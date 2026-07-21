# Designing the freemium paywall and quota system

FitLifeAI runs a freemium model: core features are free, and AI-powered features (coaching, meal generation) are gated behind a subscription. The monetization layer is built on **Superwall** with **5 paywall placements**, backed by server-side usage quotas in Supabase. It currently converts at **25% trial-to-paid**.

## Design goals

1. **Let free users experience the product's value before hitting a wall.** Hard-gating everything kills conversion; the paywall should appear at the moment of demonstrated intent, not at app open.
2. **Quotas enforced server-side, not client-side.** A client-side counter is a suggestion, not a limit — anyone can bypass it by reinstalling or manipulating local state.
3. **Placements configurable without an app release.** Superwall lets paywall presentation rules and designs change remotely, so monetization experiments don't wait on App Store review.

## The placement architecture

Each of the 5 placements maps to a feature-intent moment — the points where a free user tries to do something premium (e.g., generate an AI meal plan, message the AI coach). Placement registration happens at the feature call site; whether a paywall actually appears is decided by Superwall's remote rules against the user's subscription state.

The flow for a gated action:

```
User taps premium feature
  → check subscription state
  → if subscribed: proceed
  → if not: check server-side quota (Supabase)
      → quota remaining: proceed, decrement
      → quota exhausted: register placement → Superwall presents paywall
```

## The bug that taught me about client-side state

The nastiest bug in this system was a **race condition on `subscriptionStatus` at app launch**. Superwall's subscription state loads asynchronously; feature gates that checked it too early would see a stale/unknown status — briefly treating paying subscribers as free users, or vice versa. Intermittent, unreproducible-on-demand, and reported by exactly the users you least want to annoy: paying ones.

The fix was to treat subscription status as an **async value with an explicit unknown state**, and make gate decisions wait for resolution instead of defaulting. The general lesson: any client-side cache of billing truth is eventually wrong, and the code has to be honest about the "I don't know yet" window.

## What I'd tell someone building this

- **Put the paywall after the "aha," not before it.** Our onboarding redesign moved the plan preview *before* account creation and payment — showing users their personalized plan first. Qualification converts better than obstruction.
- **Server-side quota enforcement is non-negotiable** if the gated resource costs you money per use (LLM calls do).
- **Remote-configurable placements** turn monetization into an iteration loop instead of a release cycle.
