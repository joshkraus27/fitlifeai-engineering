# Transactional email infrastructure: Edge Functions, idempotency, and the Apple private-relay problem

FitLifeAI sends two kinds of transactional email: an on-demand "email me this week's plan" export, and a founder welcome email on onboarding. Both run as **Supabase Edge Functions** (TypeScript/Deno) calling the **Resend** API from a verified custom domain, with delivery logged to Postgres under row-level security.

## Architecture

```
iOS app ──invoke──▶ Supabase Edge Function ──▶ Resend API ──▶ user's inbox
                          │
                          ├── reads: user profile, planned meals / workout plan
                          └── writes: email_log (RLS-secured)
```

Two properties were non-negotiable:

**1. Idempotency.** The welcome email must send exactly once per user, even if the trigger fires twice (retries, re-onboarding, duplicate invocations). The function checks `email_log` for an existing `type='welcome'` row before sending — the log is both the audit trail and the guard.

**2. Server-side changes ship instantly.** Because the email logic lives in Edge Functions rather than the app binary, fixes and template changes go live with a deploy — no App Store review cycle. This mattered within days of launch (see below).

## Row-level security

The `email_log` and `email_preferences` tables are RLS-secured: users can read their own rows, and a scoped insert policy lets the logging path write. Getting RLS policies right is fiddly — the failure mode is silent (queries return empty, inserts fail quietly) rather than loud, which makes it worth testing policies as deliberately as application code.

## The Apple private-relay bounce problem

The launch surprise: **a majority of our Sign in with Apple users (53 of 97) had Apple private-relay addresses** (`@privaterelay.appleid.com`), and mail from our newly-verified domain **bounced** when sent to the relay.

This forced a product-level fix, not just an infrastructure one:

- **Email capture became part of onboarding.** After Apple sign-in, users are asked "Where should we send your plan?" — framed as delivery, not data collection. Real addresses get prefilled; relay addresses get an empty field with domain chips.
- **DMARC on the sending domain** to improve deliverability and reduce relay rejections.

The deeper lesson: **third-party auth providers shape your data.** "We have every user's email" was technically true and practically false — half the addresses couldn't receive mail. The bug also only surfaced because the delivery log made bounces visible; without it, the welcome email would have silently failed for half of new users.

## One production bug worth confessing

An early version of the welcome-email function was deployed **stale** — an older file that read the user's profile name but not their email, falling back to the (bouncing) Apple relay address. The failure chain: correct code on my laptop → wrong artifact deployed → bounces in the log. The fix was trivial; the lesson wasn't: **verify what's actually deployed, not what's on disk.** Deploy pipelines exist because "it works on my machine" extends all the way to "I'm sure I deployed the right file."
