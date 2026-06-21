# TrocaCopa

> A free sticker-album tracker for the FIFA World Cup 2026 — track what you have, what you're missing, and trade duplicates with other collectors. Built mobile-first as an installable PWA.

**By Lucas da Silva Santos — Full Stack Developer**

🌎 **[Live demo](https://troca-copa-26.vercel.app/)**

> 🔒 Source code is private. This README documents the architecture and technical decisions behind the product.

---

## Overview

Every World Cup, millions of people fill a physical sticker album and end up with a drawer full of duplicates and no good way to track what they still need or who to trade with. TrocaCopa moves that whole ritual to the phone: mark each of the 993 stickers as owned or missing, watch your completion percentage climb, count your duplicates, and share a public link so other collectors can find you.

Every feature — album tracking, the duplicates list, and the public trade profile — is completely free, with no subscription or in-app purchase of any kind.

---

## What It Does

**Album Tracking**
All 993 stickers — 48 national teams × 20, plus the FIFA World Cup History and Coca-Cola special sets — with real country flags served from a CDN. Tap to mark owned/missing, use +/− to register duplicates, filter by group (A–L + Especial) and by status, and "complete a whole team" in one tap. Optimistic UI updates instantly and reconciles with the server.

**Progress & Stats**
Live completion percentage, per-team progress bars, and counts of owned / missing / duplicate stickers on a dashboard.

**Trading Layer**
An organized duplicates list and a public trade inventory — see exactly what you have spare and what you're missing, ready to share with other collectors.

**Public Collector Profile**
A shareable `/u/[slug]` page showing your completion, the stickers you're looking for, and your trade inventory — every shared link is organic marketing. No authentication required to view.

**Installable PWA**
Add to home screen on iOS or Android — no app store. Themed manifest and icons.

---

## Architecture

```
Browser ──▶ Next.js App Router (Server Components, SSR)
                 │
                 ├── Server Actions  (mutations only — userId from session)
                 ├── Server-only queries (reads — never exposed as endpoints)
                 │
                 └──▶ Drizzle ORM ──▶ Neon (serverless PostgreSQL)
```

Mutations are Server Actions that derive `userId` from the session — never from client input. Reads live in a server-only query module, not exposed as callable endpoints, closing off a class of cross-user data-access bugs by construction rather than by convention.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router), React 19, TypeScript |
| Styling | Tailwind CSS v4, shadcn/ui, oklch design tokens |
| Auth | better-auth (session-based) |
| Database | PostgreSQL (Neon — serverless) |
| ORM | Drizzle ORM |
| PWA | Web manifest + generated icons |
| Testing | Vitest |
| Deploy | Vercel |

---

## Engineering Highlights

**Closing an IDOR by construction.**
Read functions were initially Next.js Server Actions (`'use server'`), which exposes them as callable RPC endpoints — meaning a client could request any user's collection by passing an arbitrary id. I moved all reads into a server-only query module so they're no longer reachable as endpoints, and kept mutations as actions that derive the user id from the session. Multi-tenant isolation is enforced by construction, not by convention.

**Rebrand with zero component churn.**
The visual identity (Brazilian flag palette) is defined entirely as semantic `oklch` design tokens. Re-theming the whole app meant editing tokens, not components — every button, badge and gradient inherited the new colors automatically.

---

## Security

- **Multi-tenant writes:** every mutation derives `userId` from the authenticated session — a user can only ever modify their own data.
- **No read IDOR:** data reads are server-only functions, not exposed actions.
- **No PII leakage:** email is never sent to public pages; phone numbers are stripped from public payloads.
- **Input hardening:** sticker codes are validated against the catalog and counts are clamped server-side.

---

## Testing

Vitest, with coverage focused on the logic that matters:

- **Catalog integrity** — 993 stickers across 50 sets, groups A–L + Especial.
- **Statistics** — completion, per-team progress, missing/duplicate computation.

---

## What I'd Do Differently / Roadmap

- **Rate limiting** on mutating actions before scaling traffic.
- **Custom domain** and a per-profile Open Graph image for richer link previews.
- **Trade matching** — surface collectors whose duplicates fill your gaps (and vice-versa).
- Revisit "public by default" with an explicit share-to-publish flow.

---

## Author

**Lucas da Silva Santos** — Full Stack Developer
Building production software end-to-end: product, architecture, and deploy.

> Not affiliated with FIFA, Panini, or Coca-Cola.
