# FitLifeAI — Engineering Notes

I'm the co-founder of [FitLifeAI](https://apps.apple.com/us/app/fitlifeai), an AI-powered iOS fitness app that generates personalized meal and workout plans with macro tracking. It's live on the App Store with 600+ downloads, 150 monthly active users, 40 paying subscribers (25% trial-to-paid conversion), and a 4.9-star App Store rating.

The app's source code is private — it's the company's core IP. This repo documents the engineering behind it: the architecture, the problems that came up in production, and how I solved them.

## Stack

| Layer | Technology |
|---|---|
| iOS app | Swift, SwiftUI (UIKit for select flows) |
| Backend | Supabase — PostgreSQL, Auth, Edge Functions (TypeScript/Deno) |
| Monetization | Superwall (5 freemium gate placements), usage-quota enforcement |
| Analytics | PostHog — event taxonomy, DAU/retention dashboards |
| Email | Resend API via Supabase Edge Functions |
| Data tooling | Python validation pipelines |

## Write-ups

1. **[Optimizing search over a 745K-row food database](docs/01-query-optimization.md)** — how a full-text search query went from ~9s to ~130ms with a GIN index, and what I learned about PostgreSQL indexing along the way.

2. **[Designing the freemium paywall and quota system](docs/02-paywall-and-quota-system.md)** — 5 paywall placements, server-side usage quotas, and the race condition that taught me not to trust client-side subscription state.

3. **[Transactional email infrastructure](docs/03-email-infrastructure.md)** — Supabase Edge Functions + Resend, idempotent delivery logging, row-level security, and the Apple private-relay bounce problem that forced a redesign of email capture.

## Why this repo exists

I'm a Mechanical Engineering major, so I don't have a CS transcript to point to. What I have instead is a production app with paying users and the engineering decisions that got it there. These write-ups are the proof of work.

— Josh Kraus · joshkraus27@gmail.com · [LinkedIn](https://www.linkedin.com/in/josh-kraus-99b51a39b/)
