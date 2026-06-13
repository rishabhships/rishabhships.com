---
title: 'The 12×12 cross-grade problem: subscriptions when you have more than one product'
description: "Single-product subscriptions are a state machine. Multi-product subscriptions are a state machine sitting on top of a matrix. Here's the design, and the operational discipline that goes with it."
pubDate: 'Jun 14 2026'
---

In [my last post](/blog/subscription-state-machine) I argued that subscription state in a Play Billing app is a state machine, and that you should make it explicit. That post assumed one subscription product.

This post is about what happens when you have twelve.

The first time a subscription codebase moves from one product to two, the code is usually fine. Someone adds a second SKU, a checkbox somewhere, an `if (productId == "yearly")` branch in a paywall. It works.

The codebase that has twelve products — six product lines crossed with monthly and yearly cadences, plus the occasional regional variant — does not work the same way. By the time you have that many SKUs, the actually-hard thing isn't subscription state. Subscription state is solved. The actually-hard thing is *transitions between subscriptions*, and they are not solved by your state machine.

### The shape of the problem

Acrobat has roughly six premium product lines, each available as monthly or yearly. So twelve SKUs, give or take, depending on which markets you're counting.

Users do not stay on the SKU they signed up for. They:

- **Upgrade** — basic monthly to premium yearly.
- **Downgrade** — premium yearly to basic monthly.
- **Cross-grade** — premium yearly to premium-with-AI yearly. Same tier, different feature bundle. Same cadence, different product line.
- **Cadence switch** — premium monthly to premium yearly, at the same tier.

If you have N SKUs, there are N*(N-1) directed transitions. Twelve SKUs gives you 132. Each one has its own set of decisions to make about proration, entitlement granting, and server reconciliation.

A single state machine cannot encode 132 distinct rules. You need a second structure on top of it. That structure is a matrix.

### The naive approaches, and why each of them collapses

When people first encounter this problem, they reach for one of three patterns. I'll go through them in the order I've seen teams try them.

**Pattern one: if-else per pair.** Write a function that takes (from_sku, to_sku) and returns the right behaviour. Naturally collapses into a giant switch statement. Works at two SKUs. Becomes unreadable at five. By twelve it's a 132-branch function that nobody can review and nobody knows is correct.

**Pattern two: hierarchical ordering.** Encode SKUs as tiers — basic, standard, premium. Implement transitions as "compare tier ordinals." Falls apart the first time you ship a product that doesn't fit the hierarchy. Premium-with-AI and Premium-without-AI both sit at the same tier ordinal, but they have different entitlements. The hierarchy claims they're equivalent. Your support team will tell you they're not.

**Pattern three: defer to Play.** Just call `BillingFlowParams.SubscriptionUpdateParams` with the right proration mode and let Google figure it out. This actually works fine for the billing event itself. What it loses is everything around the event: the preview UI ("you'll be charged ₹399 today and your old plan ends immediately"), the entitlement reconciliation on your server, the audit trail your support team needs three months later when the user disputes the charge, and the business rules ("this transition isn't allowed in this region"). All of those still have to live somewhere in your code, and now they're not centralized.

Each of these patterns is encoding the same data in a worse representation. The right data structure is a matrix.

### The matrix

The matrix is exactly what it sounds like. A 2D table, indexed by `from_sku` and `to_sku`, where each cell encodes everything you need to know about that specific transition.

A simplified version of what a cell holds:

```kotlin
data class TransitionRule(
    val isAllowed: Boolean,
    val disallowReason: String? = null,
    val prorationMode: ProrationMode,
    val entitlementToGrant: Entitlement,
    val serverReconciliation: ReconciliationStrategy,
    val confirmationCopyKey: String,
)
```

And the matrix itself:

```kotlin
class CrossGradeMatrix(
    private val table: Map<Pair<SkuId, SkuId>, TransitionRule>
) {
    fun lookup(from: SkuId, to: SkuId): TransitionRule =
        table[from to to] ?: TransitionRule.disallowed("Unknown transition")
}
```

This looks unexciting on the page. The point isn't the shape, it's the property: there is exactly one place in the code that says what any given transition does. Twelve SKUs, 132 cells. Add a thirteenth SKU and you add one row and one column. Drop a SKU and you delete one row and one column. The logic that consumes the matrix does not change.

In practice the matrix is rarely a literal hardcoded `Map`. It's more often loaded from server-side config so product can tune the rules without a release. But the shape on the read path is the same: `lookup(from, to)` returns a `TransitionRule` and the rest of your code does what the rule says.

The first time I built one of these I made the mistake of allowing the matrix to also encode the state-machine logic. That was a regret. The matrix should know about SKUs, prices, entitlements, and product-portfolio rules. It should not know about Google Play notification timing, grace periods, or whether a Cancelled subscription can be cross-graded. That's the state machine's job, and the two should not be entangled.

### The matrix sits on top of the state machine

Here's how the two compose in practice. The user is in some subscription state — say, Active on `pro_monthly`. They tap a paywall offering `pro_yearly`. Your code does roughly this:

```kotlin
val rule = matrix.lookup(from = "pro_monthly", to = "pro_yearly")

if (!rule.isAllowed) {
    return UiResult.NotAllowed(rule.disallowReason)
}

val event = SubscriptionEvent.CrossGrade(
    fromSku = "pro_monthly",
    toSku = "pro_yearly",
    prorationMode = rule.prorationMode,
)

val transition = stateMachine.reduce(currentState, event)

when (transition) {
    is TransitionResult.Success ->
        showConfirmation(rule.confirmationCopyKey, transition.newState)
    is TransitionResult.Invalid ->
        showError(transition.reason)
}
```

Two layers. The matrix says: *is this transition meaningful at all, and what are its commercial terms?* The state machine says: *is the user in a state where this kind of event is even legal right now?*

The cleanest example of why you need both: a user in `Cancelled` state (paid period still running, but they've cancelled) attempts a cross-grade. The matrix says the transition is well-defined — `pro_monthly` to `pro_yearly` has a known rule. The state machine says no, cross-grades aren't valid from a Cancelled state. The right behaviour is "reject, with the state-machine reason." If you only had the matrix, you'd let the transition through and then fight a race condition between the user's existing cancel and the new purchase. If you only had the state machine, you'd accept the event but have no idea what proration mode to apply.

Composing them gives you a system where each decision happens once, in the right layer.

### Operational things that don't fit on a whiteboard

A few things about running a system like this in production that are worth saying out loud.

**Every matrix lookup gets logged.** Not the result, the *query*. Six months from now a user is going to dispute a transition, and the only way you'll reconstruct what happened is having (`user_id, from_sku, to_sku, ts, rule_returned`) in your warehouse. Don't skip this.

**The matrix lives in config, not code, beyond a certain size.** Once product wants to A/B test rules ("offer a free month for the monthly-to-yearly transition in this region"), the matrix has to be tunable without a release. We landed on a small typed JSON, fetched on app start and cached locally for offline robustness.

**Server is the source of truth for the matrix too.** The client renders UI based on it, but the actual transition is reconciled server-side, against the server's copy of the matrix. If they disagree, the server wins. This matters during gradual rollouts: clients in the wild will have a stale matrix for weeks.

**Confirmations are matrix-driven, not state-machine-driven.** "You'll be charged ₹X today" is a function of the matrix's `prorationMode` and the SKU prices, not a function of subscription state. Don't let the confirmation UI sneak into the state machine's vocabulary.

### Where this lives in renew-kt

[renew-kt](https://github.com/rishabhships/renew-kt) — the small library I extracted in the last post — deliberately doesn't ship a matrix.

The reasoning: the matrix is product logic. It's specific to what subscriptions your business sells and what rules govern them. A library can't have an opinion on whether your `pro_monthly` to `pro_yearly` is allowed in Germany, because the library doesn't know what `pro_monthly` is.

What renew-kt does provide is the substrate the matrix sits on. The library has `SubscriptionEvent.Upgrade`, `SubscriptionEvent.Downgrade`, and `SubscriptionEvent.CrossGrade` as first-class events. Your matrix layer turns a user intent into one of these events. The state machine then validates whether the event is legal from the user's current state. Two layers, clean seam between them.

The sample app demos the substrate side. Tap any of the Upgrade / Downgrade / CrossGrade buttons and you can watch the state machine accept or reject the event based on the current state. The product-portfolio side — the matrix — is for you to build, because only you know what your products are.

### The broader takeaway

The broader takeaway is the same one I keep coming back to in subscription systems. Subscription engineering looks like one problem from the outside. It is actually two:

1. **Lifecycle logic.** Which state is a single subscription in, what events can move it to other states, how do we handle invalid events. This is the state machine. Same shape for every subscription product on earth.

2. **Portfolio logic.** What SKUs do we sell, how do they relate to each other, what are the rules for moving between them. This is the matrix. Unique to your product, your pricing, your geography.

Single-product apps can skip the second layer, because their portfolio has exactly one cell. Multi-product apps cannot, and trying to encode the second layer in the data structures of the first is where the codebase goes off the rails.

The discipline is to keep these two pieces structurally apart. Lifecycle logic in a state machine. Portfolio logic in a matrix. They compose. They do not merge.

If you're building a subscription stack from scratch today and you can already see more than one SKU in your future, draw the matrix first. The state machine fits underneath whenever you need it.

---

If this resonates and you want to use renew-kt as the lifecycle substrate, the repo is at [github.com/rishabhships/renew-kt](https://github.com/rishabhships/renew-kt). I'm at [rishabhships.com](https://rishabhships.com), or [rishabh@rishabhships.com](mailto:rishabh@rishabhships.com).
