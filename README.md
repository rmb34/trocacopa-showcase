# TrocaCopa

> A freemium sticker-album tracker for the FIFA World Cup 2026 — track what you have, what you're missing, and unlock trading by supporting the project. Built mobile-first as an installable PWA.

**By Lucas da Silva Santos — Full Stack Developer**

🌎 **[Live demo](ai-chatbot-beta-plum.vercel.app)**

> 🔒 Source code is private. This README documents the architecture and technical decisions behind the product.

---

## Overview

Every World Cup, millions of people fill a physical sticker album and end up with a drawer full of duplicates and no good way to track what they still need or who to trade with. TrocaCopa moves that whole ritual to the phone: mark each of the 993 stickers as owned or missing, watch your completion percentage climb, count your duplicates, and share a public link so other collectors can find you.

It's built as a **freemium product with a real payment wall**. The full album tracker is free forever (the acquisition and virality layer). The trading layer — your organized duplicates list, the public trade inventory, and a supporter badge — unlocks with a single **R$ 10,99** payment, framed as supporting an independent project rather than a cold paywall.

---

## What It Does

**Album Tracking (free)**
All 993 stickers — 48 national teams × 20, plus the FIFA World Cup History and Coca-Cola special sets — with real country flags served from a CDN. Tap to mark owned/missing, use +/− to register duplicates, filter by group (A–L + Especial) and by status, and "complete a whole team" in one tap. Optimistic UI updates instantly and reconciles with the server.

**Progress & Stats (free)**
Live completion percentage, per-team progress bars, and counts of owned / missing / duplicate stickers on a dashboard.

**Public Collector Profile (free)**
A shareable `/u/[slug]` page showing your completion and the stickers you're looking for — every shared link is organic marketing. No authentication required to view.

**Trading Layer (Apoiador — paid)**
The organized duplicates list, the "trade inventory" section on your public profile, and a ⭐ Supporter badge. The wall lands exactly where the hobby gets real: you have duplicates in hand and want to trade them.

**Installable PWA**
Add to home screen on iOS or Android — no app store. Themed manifest and icons.

---

## Architecture

```
Browser ──▶ Next.js App Router (Server Components, SSR)
                 │
                 ├── Server Actions  (mutations only — userId from session)
                 ├── Server-only queries (reads — never exposed as endpoints)
                 ├── lib/entitlements (pure, tested access decisions)
                 │
                 ├──▶ Drizzle ORM ──▶ Neon (serverless PostgreSQL)
                 └──▶ Stripe REST API (checkout) + webhook (signature-verified)
```

The freemium gate is enforced **server-side, in one place**. Access decisions live in a pure module (`lib/entitlements.ts`) that the server components call; the UI only renders the result. The key invariant — *a free user's duplicates are never sent to the client* — is expressed as a pure function (`publicProfileSections`) and pinned by unit tests, so it can't silently regress.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router), React 19, TypeScript |
| Styling | Tailwind CSS v4, shadcn/ui, oklch design tokens |
| Auth | better-auth (session-based) |
| Database | PostgreSQL (Neon — serverless) |
| ORM | Drizzle ORM |
| Payments | Stripe (Checkout + signature-verified webhooks) |
| PWA | Web manifest + generated icons |
| Testing | Vitest |
| Deploy | Vercel |

---

## Engineering Highlights

**Payments that actually work on serverless.**
The Stripe Node SDK threw `StripeConnectionError` on Vercel's runtime — its internal fetch didn't play well with the platform. I replaced the SDK call with a direct `fetch` to the Stripe REST API for checkout, which made the integration reliable. The webhook still uses the SDK's `constructEvent` (pure crypto, no network) to verify signatures.

**Freemium gating as a tested invariant.**
Rather than scattering `if (isPaid)` checks through components, all access decisions live in `lib/entitlements.ts` as pure functions applied server-side. `publicProfileSections()` returns an empty duplicates list for non-supporters, guaranteeing the gated data never reaches the client — and a unit test asserts exactly that.

**Closing an IDOR by construction.**
Read functions were initially Next.js Server Actions (`'use server'`), which exposes them as callable RPC endpoints — meaning a client could request any user's collection by passing an arbitrary id. I moved all reads into a server-only query module so they're no longer reachable as endpoints, and kept mutations as actions that derive the user id from the session. Multi-tenant isolation is enforced by construction, not by convention.

**Rebrand with zero component churn.**
The visual identity (Brazilian flag palette) is defined entirely as semantic `oklch` design tokens. Re-theming the whole app meant editing tokens, not components — every button, badge and gradient inherited the new colors automatically.

---

## Security

- **Multi-tenant writes:** every mutation derives `userId` from the authenticated session — a user can only ever modify their own data.
- **No read IDOR:** data reads are server-only functions, not exposed actions.
- **Webhook integrity:** Stripe events are signature-verified before granting access — a forged event can't unlock the paid tier.
- **No PII leakage:** email is never sent to public pages; phone numbers are stripped from public payloads.
- **Input hardening:** sticker codes are validated against the catalog and counts are clamped server-side.

---

## Testing

Vitest, with coverage focused on the logic that matters:

- **Catalog integrity** — 993 stickers across 50 sets, groups A–L + Especial.
- **Statistics** — completion, per-team progress, missing/duplicate computation.
- **Entitlements** — the freemium access rules, including the invariant that free users' duplicates are never exposed publicly.

---

## What I'd Do Differently / Roadmap

- **Rate limiting** on mutating actions before scaling traffic.
- **Custom domain** and a per-profile Open Graph image for richer link previews.
- **Trade matching** — surface collectors whose duplicates fill your gaps (and vice-versa).
- Revisit "public by default" with an explicit share-to-publish flow.

---

## Author

**Lucas da Silva Santos** — Full Stack Developer
Building production software end-to-end: product, architecture, payments, and deploy.

> Not affiliated with FIFA, Panini, or Coca-Cola.
