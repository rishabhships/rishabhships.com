---
title: 'Modeling Google Play Billing subscriptions as a state machine'
description: "Subscription state in a Play Billing app isn't a single source of truth. It's a derived view. Here's how I model it explicitly, and the small Kotlin library I extracted from doing this at scale."
pubDate: 'Jun 11 2026'
---

There's a bug pattern that's specific to subscription apps. It looks like this:

> A user reports they paid yesterday, the receipt cleared, and the app still shows them as a free user. Or the inverse. They cancelled three weeks ago, payment never went through, and we're still serving them premium features.

If you've shipped Google Play Billing subscriptions in any consumer Android app, you've seen some flavor of this. And when you sit down to fix it, the fix is always in the same kind of place: somewhere between an event arriving from the platform and the state your app *thinks* the user is in.

The reason is structural. Subscription state isn't a thing you store. It's a thing you derive, repeatedly, from a stream of events that arrive at different times through different channels. The gap between "what just happened" and "what state should the user be in" is where the bugs live.

I work on the in-app payments stack of a popular Android app. The single most useful mental shift in that codebase has been treating subscriptions as a deterministic state machine: explicit states, explicit events, explicit rejections for invalid transitions. This post is about why that helps, and the small open-source library I extracted from doing it.

### The Play Billing lifecycle, briefly

The Google Play Billing subscription lifecycle has seven canonical states. The docs describe them well, but they're scattered across multiple pages. Compactly:

- **NotPurchased.** The user has never bought a subscription.
- **Active.** They have one. It's billing normally.
- **InGracePeriod.** A billing attempt failed, Play is retrying, and the user still has access.
- **OnHold.** Grace ran out and the subscription is in account hold. The user loses access, but recovery is still possible.
- **Paused.** The user proactively paused (Play allows this for some subscriptions).
- **Cancelled.** The user cancelled, but the paid period is still running. Access continues until expiry.
- **Expired.** The subscription has fully ended.

There are events that move between them: `Purchase`, `Renew`, `Cancel`, `Upgrade`, `Downgrade`, `CrossGrade`, `EnterGracePeriod`, `EnterHold`, `Pause`, `Resume`, `Expire`. Eleven of them.

The reason this is hard to ship correctly isn't that there are too many states or too many events. The reason is that the transitions between them are *conditional*, and most apps treat them as if they aren't.

### What "treat them as if they aren't" looks like

Common pattern in subscription code:

```kotlin
fun onPurchaseEvent(purchase: Purchase) {
    if (purchase.purchaseState == PURCHASED) {
        user.hasPremium = true
        user.expiryTime = purchase.purchaseTime + ONE_MONTH
    }
}
```

This is wrong in subtle ways that take a while to surface. Three obvious ones.

**First, it doesn't know whether the user had a subscription before this event.** A `Purchase` that arrives on top of an `Active` state means something different from one that arrives on top of `Cancelled`. The first might be a duplicate event. The second is a winback.

**Second, it assumes "purchased" maps to "active right now."** It doesn't. A purchase can be pending, deferred, or replaced by an upgrade. Without checking purchase state alongside the prior known state, you can't tell.

**Third, it silently writes the new state.** If the transition was actually invalid (say, a stale duplicate event for a user who's already moved past Active), the code blissfully overwrites the more-correct state with less-correct fields.

The fix is to make the transition explicit and rejectable.

### Explicit transitions, explicit rejections

The principle I keep coming back to: every event must either produce a new state with a *reason*, or be rejected with a *reason*. No silent fall-through.

In the library I'll get to in a moment, the reducer signature is:

```kotlin
fun reduce(state: SubscriptionState, event: SubscriptionEvent): TransitionResult
```

Where `TransitionResult` is either `Success(newState)` or `Invalid(from, event, reason)`. So:

```kotlin
val current = SubscriptionState.InGracePeriod("pro_monthly", gracePeriodEndEpochMs = ...)
val attempt = machine.reduce(current, SubscriptionEvent.Upgrade("pro_yearly", ...))
// attempt is TransitionResult.Invalid(
//   reason = "Only Renew, Cancel, EnterHold, Pause, or Expire are valid during grace period."
// )
```

The win isn't that invalid transitions stop happening. They happen all the time. Stale notifications. Out-of-order events. Races between RTDN delivery and follow-up `Subscriptions.get` calls. The win is that when an invalid transition shows up, the code knows, and you get a single observable surface to log it, alert on it, and route it for investigation. Nothing silently drifts.

### Real-time developer notifications as the source of truth

The other half of the picture is picking the right input.

Mobile clients are bad sources of truth for subscription state. They're online intermittently, the OS kills them, they're vulnerable to clock skew, and a determined user can sideload an old purchase token to lie about state. The Play Billing SDK gives you `Purchase` objects locally, but the canonical signal is on the server: Real-Time Developer Notifications (RTDN), delivered to your Pub/Sub topic the moment Google sees a transition.

RTDN carries a `notificationType` integer. `SUBSCRIPTION_RENEWED = 2`. `SUBSCRIPTION_ON_HOLD = 5`. And so on. There are 14 of them.

Once you map those notification types to your event vocabulary, your server processes them and writes the derived state. Your mobile client reads that state. The client never infers state from purchase tokens alone.

If you do this correctly, the mobile experience is fully reactive to server state. You stop asking "what does the user think they have." You ask "what does the source of truth say." Your state machine is the bridge that turns each RTDN into a single intent.

### What I open-sourced

I extracted the state machine and RTDN adapter into a small library: [**renew-kt**](https://github.com/rishabhships/renew-kt).

It's deliberately tiny. About 500 lines of Kotlin. No Android dependency. Apache 2.0. The whole library is:

- The 7-state, 11-event model above, as a sealed class hierarchy
- A pure `reduce(state, event)` function with explicit rejection
- An RTDN adapter mapping all 14 Google Play notification types
- 40 unit tests covering every transition I could think of, including the invalid ones

The state diagram is in the README. The point of the library isn't novelty. Most of this model is faithful to the Play Billing docs. The point is to have one place to encode that model so every subsequent person who touches subscription code starts from the same vocabulary.

### What it deliberately doesn't do

Honesty is important here. The library doesn't:

- Talk to the Play Billing SDK. It produces events. You connect them.
- Persist state. Use Room, DataStore, or whatever your app already has.
- Resolve race conditions between RTDN and `Subscriptions.get`. The state machine is downstream of whichever order events arrive.
- Replace your server. Subscription verification still needs to happen on a trusted backend.

It's a tiny piece, deliberately. A subscription stack has many parts. This is the part that's annoying to reinvent and easy to get wrong, so I put a verified version somewhere stable.

### The bigger takeaway

I'd phrase the broader lesson like this. **Subscription state isn't observable.** It's never sitting in one place ready to be read. It's always derived, and the derivation can fail. You need to make the derivation deterministic, and you need to make the failures visible.

If you're shipping a Play Billing subscription product and you've been treating subscription state as just-another-field, I'd quietly suggest looking at how often your support team gets the "I paid but the app doesn't show it" tickets. That's the diagnostic.

---

If renew-kt is useful to you, please use it. If it's incorrect for an edge case I missed, please open an issue. The most valuable thing for a library like this is real-world stress-testing.

I'm at [rishabhships.com](https://rishabhships.com), or [rishabh@rishabhships.com](mailto:rishabh@rishabhships.com).
