---
title: '5 failure modes in mobile in-app purchase flows (and how we catch them)'
description: "After years owning subscription payments on a high-scale Android app, these are the five failure modes that caused the most production incidents — and the patterns that catch them before users do."
pubDate: 'Jun 27 2026'
---

In-app purchase flows fail in boring ways. Not dramatic crashes — those are easy to catch. The expensive failures are the quiet ones: the purchase that completed but never got acknowledged, the renewal that fired but left the user locked out, the upgrade that billed correctly but granted the wrong entitlement.

I've spent years owning subscription payments on a high-scale Android app. These are the five failure modes I've seen cause the most production incidents, and the patterns we use to catch them before users do.

---

## 1. The unacknowledged purchase

**What happens.** The user taps buy. The Play Billing flow completes. `onPurchasesUpdated` fires with `BillingResponseCode.OK`. Your app sends the purchase token to the backend for verification. The network call fails — timeout, server blip, process death, the user force-closed the app. The purchase token sits unacknowledged.

Google Play's contract: any purchase not acknowledged within **three days** is automatically refunded. The user paid. Three days later, they're silently refunded without knowing it. They still think they have a subscription. You find out when they contact support two weeks later.

**How to catch it.** Two things, both required:

First, make your backend acknowledgement endpoint idempotent and retry it aggressively. Don't acknowledge from the client — acknowledge from the server after verification, so the retry logic has a reliable execution environment.

Second, on every `BillingClient` connection (including cold starts and reconnects), call `queryPurchasesAsync` and check every returned purchase for `isAcknowledged == false`. If you find one, re-trigger the verification + acknowledgement flow immediately. This is your safety net for every session where the first attempt failed.

```kotlin
billingClient.queryPurchasesAsync(
    QueryPurchasesParams.newBuilder()
        .setProductType(BillingClient.ProductType.SUBS)
        .build()
) { result, purchases ->
    if (result.responseCode == BillingResponseCode.OK) {
        purchases
            .filter { !it.isAcknowledged }
            .forEach { triggerVerificationFlow(it) }
    }
}
```

---

## 2. The out-of-order RTDN

**What happens.** Real-Time Developer Notifications are not guaranteed to arrive in order. A `SUBSCRIPTION_RENEWED` notification can arrive before the `SUBSCRIPTION_ON_HOLD` notification from the previous billing cycle. A `SUBSCRIPTION_PURCHASED` can arrive after a `SUBSCRIPTION_CANCELLED` for the same token if the user cancelled seconds after purchasing.

The naive backend processes each RTDN as it arrives and writes the resulting state. Out-of-order delivery means state gets written backwards. You end up with users in `Active` state when they should be `OnHold`, or vice versa.

**How to catch it.** Don't process RTDNs as absolute state writes. Process them as events into a state machine that rejects illegal transitions.

When an RTDN arrives, fetch the current subscription state from your database. Feed the RTDN as an event into your state machine. If the transition is valid, write the new state. If it's invalid — because this event can't follow the current state — log it, emit a metric, and drop it. Then schedule a `Subscriptions.get` call against the Play Developer API to fetch ground truth and reconcile.

The `SUBSCRIPTION_ON_HOLD` arriving *after* `SUBSCRIPTION_RENEWED` gets rejected by the state machine (you can't go from Active→OnHold via an old hold event), the reconciliation job runs, and you end up with the correct state. Without the state machine, you'd silently write the wrong state.

---

## 3. The background purchase that never surfaces

**What happens.** The user taps buy from your paywall. You call `launchBillingFlow`. The OS sends them to the Play Store. Your app goes to background. The user approves the purchase in Play, then navigates away — checks their email, answers a call, whatever. When they eventually return to your app, it cold-starts.

`onPurchasesUpdated` never fires for this purchase. It only fires for purchases that complete while your `BillingClient` is connected and your app is in the foreground. A purchase that completed while your app was in the background is invisible unless you explicitly query for it.

**How to catch it.** Always call `queryPurchasesAsync` on `BillingClient` connection — not just on the first launch, but on every reconnect. Every time `onBillingServiceConnected` fires:

```kotlin
override fun onBillingSetupFinished(result: BillingResult) {
    if (result.responseCode == BillingResponseCode.OK) {
        queryAndProcessPendingPurchases()
    }
}
```

This is the path most quickstart tutorials omit. It's also the path that catches every background-completed purchase. Skipping it means a non-trivial fraction of purchases — especially on slower devices where users context-switch more — never get processed.

---

## 4. The grace period entitlement flip

**What happens.** A user's subscription renewal fails — insufficient funds, expired card. Google Play puts them in grace period: billing retries for a few days, and the user retains access during that window.

At some point in this window, your app opens. Your backend checks subscription state. Depending on timing and how you've implemented state polling, you might see `purchaseState == PURCHASED` (still technically purchased, just not renewed yet) and interpret it as active. Or you might see the RTDN `SUBSCRIPTION_IN_GRACE_PERIOD` and incorrectly revoke access. Either direction is a bug.

The first bug — incorrectly revoking access during grace — generates support tickets immediately. Users who haven't done anything wrong get locked out. The second bug — granting access past grace period expiry — leaks entitlement silently.

**How to catch it.** Grace period is a distinct state, not a degraded version of Active. Model it explicitly:

```kotlin
sealed class SubscriptionState {
    data class Active(val sku: String, val expiryMs: Long) : SubscriptionState()
    data class InGracePeriod(val sku: String, val gracePeriodEndMs: Long) : SubscriptionState()
    // ...
}
```

Entitlement logic reads the state, not the raw `purchaseState` field. `InGracePeriod` grants access. `OnHold` (grace expired, billing still retrying) revokes access. The distinction is in your state, not in the Play Billing `Purchase` object.

The RTDN `SUBSCRIPTION_IN_GRACE_PERIOD` is the trigger to move from Active to InGracePeriod. `SUBSCRIPTION_RESTARTED` or `SUBSCRIPTION_RENEWED` moves back to Active. `SUBSCRIPTION_ON_HOLD` moves to OnHold. These are the transitions your state machine needs to handle explicitly — not your entitlement-check code.

---

## 5. The upgrade that granted the wrong entitlement

**What happens.** The user upgrades from a basic plan to a premium plan. The billing event succeeds — Play charges the right amount, the new purchase token is valid, your backend verifies it. But entitlement is granted based on the *product ID in the purchase token*, and somewhere in the upgrade flow, the product ID lookup goes wrong. The user ends up with basic entitlement on a premium subscription, or premium entitlement on a basic one.

This is rarer than the other four, but the support cost is high. Overgranting is a revenue leak. Undergranting (user paid for premium, got basic) generates angry tickets and chargebacks.

**How to catch it.** Two things:

First, make the entitlement mapping explicit and centralized. Don't derive entitlement inline in the purchase handler. Have a single function, `entitlementFor(productId: String): Entitlement`, that's the only place in the codebase that knows what each product ID grants. Test it exhaustively.

Second, emit an event every time entitlement is granted or changed, with both `productId` and `entitlement` in the payload:

```
entitlement_granted { user_id, product_id, entitlement, source }
```

Run a daily reconciliation query: for every user where `entitlement != expectedEntitlementFor(product_id)`, alert. This catches drift before users notice it, and gives you the audit trail to understand how it happened.

---

## The common thread

All five of these share the same root cause: treating in-app purchases as a simple request/response rather than a distributed system with async events, eventual consistency, and multiple failure points.

The Play Billing SDK is reliable. The failure modes aren't in the SDK — they're in the assumptions apps make about timing, ordering, and state derivation around it.

The fixes are also consistent: explicit state machines, idempotent operations, queryPurchasesAsync on every connection, and enough instrumentation that you find out about failures before your users file support tickets.

If you're auditing your own purchase flow, these five are the checklist. Fix whichever ones you haven't handled yet. The cost of fixing them proactively is a few days of engineering. The cost of finding out from users is much higher.

---

*The [subscription-state-machine post](/blog/subscription-state-machine) covers the state machine model in detail. [renew-kt](https://github.com/rishabhships/renew-kt) is the open-source library. I'm at [rishabhships.com](https://rishabhships.com) or [rishabh@rishabhships.com](mailto:rishabh@rishabhships.com).*
