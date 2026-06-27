---
title: 'Anatomy of a mobile payments stack at billion-install scale'
description: "Most Android developers have wired up Play Billing. Fewer have had to make it production-reliable at scale. Here's what the full stack actually looks like, and where the complexity lives."
pubDate: 'Jun 27 2026'
---

Most Android developers have wired up Google Play Billing at some point. You add the dependency, call `launchBillingFlow`, handle `PurchasesUpdatedListener`, acknowledge the purchase. It works in your test account. You ship it.

A year later, you have support tickets. Users who paid but see a free experience. Users who cancelled and still have premium. Users who upgraded and got charged twice. A Monday morning Slack message with a graph you didn't expect.

I've spent the better part of four years owning the in-app payments stack for a billion-install consumer Android app — 95M+ monthly active users, Google Play Billing, subscriptions across dozens of markets. This post is about what the full stack actually looks like once you've learned the hard lessons, and where the complexity that isn't in the docs lives.

## The layers

A production-grade mobile payments stack has five layers. Most codebases only think about two.

```
┌──────────────────────────────────────┐
│  1. Surface — paywall, checkout UI   │
├──────────────────────────────────────┤
│  2. State machine — subscription     │
│     lifecycle, event validation      │
├──────────────────────────────────────┤
│  3. Billing client — Play Billing    │
│     SDK, flow launch, ack            │
├──────────────────────────────────────┤
│  4. Reconciliation — server verify,  │
│     entitlement grant, RTDN          │
├──────────────────────────────────────┤
│  5. Observability — events, alerts,  │
│     on-call runbooks                 │
└──────────────────────────────────────┘
```

Most tutorials cover layer 3 and the happy path of layer 1. The complexity is in layers 2, 4, and 5. That's also where every production incident I've seen has lived.

## Layer 1: Surface

The paywall and checkout UI are the most visible layer and the least technically interesting — until they're not.

The hard parts aren't layout or animation. They're:

**Price loading.** `BillingClient.queryProductDetails()` is async and can fail. Your paywall needs a loading state, a retry, and a graceful error that doesn't strand the user on a blank screen. On slow networks this takes 2–4 seconds. Users don't wait.

**Eligibility.** Not every user is eligible for every offer. Free trials can only be granted once per Google account per product. Intro pricing has its own eligibility. You need to query `SubscriptionOfferDetails` and check `pricingPhases` before deciding what to show — and that decision has to be consistent with what the server thinks the user is eligible for.

**The confirmation screen.** Before calling `launchBillingFlow`, your UI should show the user exactly what they're about to pay and when. For a cross-grade — switching between two paid products — the proration calculation matters. Get it wrong and users file chargebacks.

**Regional variance.** In some markets, specific payment forms are regulated. In others, price rounding rules differ. The Play Billing SDK handles currency display, but the business logic around which offers are available in which regions lives in your code.

None of this is impossible. All of it is omitted from the quickstart guides.

## Layer 2: State machine

I've written about this in detail in a [previous post](/blog/subscription-state-machine), so I'll be brief here.

The Google Play Billing subscription lifecycle has seven states and eleven events. The bugs come from treating state as just-a-field rather than something you derive, deterministically, from a stream of events.

The fix is to make the state machine explicit: sealed classes for states, sealed classes for events, a pure `reduce(state, event)` function that either produces a new state or rejects the event with a reason.

The rejection case is as important as the success case. Events arrive out of order. Notifications duplicate. Races happen between Real-Time Developer Notifications and direct `Subscriptions.get` calls. Your state machine needs to handle invalid transitions gracefully — log them, alert on them, and not silently corrupt state.

I extracted this into [renew-kt](https://github.com/rishabhships/renew-kt), a small Apache 2.0 library with the 7-state model, all transitions, and an RTDN adapter. It's about 500 lines of Kotlin and has no Android dependency, so it's testable in pure JVM.

When you have more than one subscription product, the state machine gets a layer on top of it: a transition matrix. That's what the [12×12 cross-grade post](/blog/cross-grade-12x12) covers.

## Layer 3: Billing client

The Play Billing SDK is where most developers spend most of their time, and it's actually the most reliable layer once you understand its contract.

A few things the docs understate:

**`BillingClient` connection state is not sticky.** The connection drops. Your app goes to background, the service disconnects, and the next call you make will fail with `SERVICE_DISCONNECTED`. You need a reconnect strategy — typically a wrapper that queues operations and retries on reconnect, with exponential backoff and a max retry count.

**`onPurchasesUpdated` is not the only path.** It fires for purchases initiated in the current session. But a purchase can complete while your app is in the background — the user was sent to the Play store, approved, and returned to a cold-started app. `queryPurchasesAsync` on `BillingClient.startConnection` is how you catch these. Miss this path and you'll have users who completed payment but whose app never acknowledged it.

**Acknowledgement is your responsibility, and unacknowledged purchases get refunded.** Google Play automatically refunds any purchase not acknowledged within three days. Your acknowledgement flow needs to be robust to network failure, app kills, and process death. If you're doing server-side acknowledgement (recommended), make the server endpoint idempotent.

**Pending purchases are real.** `PurchaseState.PENDING` means the user has initiated payment via a method that hasn't cleared yet — cash payment at a kiosk, bank transfer, etc. These are significant in some markets. Your UI needs a pending state, and your backend needs to handle the `SUBSCRIPTION_PURCHASED` RTDN that fires when the payment eventually clears.

## Layer 4: Reconciliation

This is the layer that makes everything reliable, and it's the layer most mobile engineers haven't had to build because it lives on the server.

The architecture is:

1. **Purchase token verification.** When a purchase completes on-device, the client sends the purchase token to your backend. The backend verifies it against the Google Play Developer API (`purchases.subscriptions.get`). Never trust a purchase token that hasn't been server-verified.

2. **Entitlement grant.** The server decides what the user is entitled to based on the verified subscription state. The mobile client reads entitlement from the server — it does not infer it from local purchase state.

3. **Real-Time Developer Notifications (RTDN).** Google sends subscription lifecycle events — renewals, cancellations, holds, expirations — to a Pub/Sub topic you configure in the Play Console. Your backend listens to this topic and updates entitlement in real time. This is how you handle renewals without requiring the user to open the app.

4. **Reconciliation fallback.** RTDNs occasionally arrive late or not at all. Your backend should run a periodic reconciliation job that queries `purchases.subscriptions.get` for active subscribers and corrects any drift between what Google says and what your database says.

The failure mode when you skip server verification: a motivated user can replay a valid purchase token across accounts, use a token from a refunded purchase, or use a token from a different app entirely. These don't happen at 1% of users. They happen at enough users at scale that it matters.

The failure mode when you skip RTDN: your users' access state drifts from reality silently, until they contact support or until you notice the discrepancy in your analytics.

## Layer 5: Observability

Payments infrastructure without observability is infrastructure you can't operate.

The minimum viable set of events to emit, from the mobile client:

```
paywall_impression       { product_id, offer_id, surface }
checkout_initiated       { product_id, offer_id }
purchase_completed       { product_id, purchase_token, state }
purchase_acknowledged    { product_id, purchase_token }
purchase_failed          { product_id, error_code, response_code }
entitlement_granted      { product_id, entitlement }
entitlement_revoked      { product_id, reason }
```

From the server side, on top of RTDN:

```
rtdn_received            { notification_type, subscription_id }
verification_success     { purchase_token }
verification_failed      { purchase_token, reason }
reconciliation_drift     { subscription_id, expected_state, actual_state }
```

With these events, you can answer:

- What's my paywall-to-purchase conversion rate by surface and offer?
- What's my purchase-to-acknowledgement success rate?
- How many verification failures am I seeing, and on what error codes?
- How often does entitlement state drift from Google's source of truth?

Without them, you find out something is wrong when a support ticket volume spikes or a metric moves. With them, you find out before the user does.

**Alerting.** At minimum: alert on purchase failure rate above a threshold, alert on verification failure rate, alert on RTDN processing lag. These are your three leading indicators for a broken payments flow.

**On-call runbooks.** Document the steps to take for each alert. "Purchase failure rate elevated" → check error code distribution → if `BILLING_UNAVAILABLE` widespread, likely Play outage, check status.google.com. If `ITEM_UNAVAILABLE`, check product configuration in Play Console. Every alert should have a runbook. The runbook is the institutional memory that outlasts any individual engineer on the team.

## What this looks like at scale

At 95M+ MAU, a few things change qualitatively:

**Even rare events happen a lot.** An event that affects 0.01% of users is 9,500 users. If that event is "purchase not acknowledged," that's 9,500 potential chargebacks. Reliability work that feels academic at 10K users is operational at 100M.

**Rollouts are careful.** Any change to the payments flow goes out behind a feature flag, to 1% of traffic, with automated rollback on error rate thresholds, before it reaches everyone. The blast radius of a bad payments deploy is immediate and measurable.

**Support is a signal source.** At this scale, your support queue is a leading indicator for production issues. "I paid but the app doesn't know" tickets spiking is an alert before your metrics surface the problem. Instrument the support resolution paths. When a support agent grants manual entitlement, emit an event. That event is a metric.

**Cross-org alignment is engineering work.** A cross-grade rollout — allowing users to switch between subscription tiers — isn't just an Android engineering problem. It touches billing, entitlement, pricing, legal review (refund implications differ by region), and the server teams. The technical design is maybe 40% of the work. Getting alignment across the orgs on what the rules are, and what "correct" looks like in every state, is the other 60%. I wrote about the technical side of the cross-grade design in [the 12×12 post](/blog/cross-grade-12x12).

## The honest summary

Mobile payments is one of the few areas of Android engineering where the gap between "it works in my test account" and "it works reliably in production" is genuinely large. The Play Billing SDK is well-designed. The documentation is thorough. The complexity isn't in the API.

The complexity is in the state management across async event sources, the server-side reconciliation that most mobile engineers don't own, and the observability that lets you know something is wrong before your users do.

If you're building a new subscription stack, spend the design time on layers 2, 4, and 5. The paywall and billing client will take care of themselves. The state machine, reconciliation, and observability are where the incidents live — and where the interesting engineering is.

---

*I write about mobile payments engineering and Android infrastructure. The [renew-kt](https://github.com/rishabhships/renew-kt) library covers the state machine layer. I'm at [rishabhships.com](https://rishabhships.com) or [rishabh@rishabhships.com](mailto:rishabh@rishabhships.com).*
