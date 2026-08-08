# TrocaCopa

Mobile-first sticker-album tracker for the 2026 World Cup. Collectors can manage their album, count duplicates, follow their progress, and publish a profile for trading.

I designed and built the product as an installable PWA using Next.js, React, PostgreSQL, and server-side session enforcement.

[Live application](https://troca-copa-26.vercel.app/) · [Source code](https://github.com/rmb34/trocacopa) · [LinkedIn](https://linkedin.com/in/lucas-da-silva-santos-a46879285)

> TrocaCopa is an independent project and is not affiliated with FIFA, Panini, or Coca-Cola.

---

## Product Overview

Physical sticker albums are easy to start and surprisingly difficult to track. Collectors need to know which stickers they already own, how many duplicates they have, what is missing, and what they can offer in a trade.

TrocaCopa brings that workflow to the phone. The current catalog contains 993 stickers across the tournament teams and special sets. Collection changes are reflected immediately in the interface and persisted to the collector's account.

The product is free and can be installed directly from the browser without an app store.

---

## Main Features

### Album Management

Collectors can:

- Browse the complete album by group and team.
- Search teams without case or accent sensitivity.
- Filter stickers by collection status.
- Mark individual stickers as owned or missing.
- Register the total number of copies owned.
- Update an entire team at once.
- Navigate a mobile-first album interface.

A stored count represents the total number of copies: zero means missing, one means collected, and values above one contribute to the duplicate inventory.

### Progress Dashboard

The dashboard derives collection statistics from the stored entries:

- Total stickers collected.
- Missing stickers.
- Duplicate copies.
- Overall completion percentage.
- Progress by team.

These values are calculated from the catalog instead of being stored as independent counters, avoiding synchronization problems between collection data and displayed statistics.

### Duplicate Inventory

Duplicates are grouped by team in a dedicated view. Collectors can use the native mobile share sheet or copy a formatted list when sharing through messaging applications.

The sharing layer falls back to the clipboard when the Web Share API is unavailable and reports whether the content was shared, copied, dismissed, or could not be sent.

### Public Collector Profile

Each account receives a unique profile slug. Profiles are currently public by default, and the collector can disable public access from the profile settings.

A published profile can show:

- Display name and city.
- Collection progress.
- Missing stickers.
- Duplicate inventory.
- An optional WhatsApp contact button.

The phone number is included in the public response only when the owner explicitly enables that option. Email addresses are never returned by the public profile query.

### Installable PWA

TrocaCopa includes:

- Web application manifest.
- Installable mobile experience.
- Generated application icons.
- PWA caching for production navigation.
- Responsive layouts for phone and desktop use.

---

## Engineering Highlights

### Multi-Tenant Isolation by Construction

Collection mutations are implemented as Server Actions. They never accept an account identifier from the browser; the authenticated user is derived from the server-side session.

Read operations that need an internal user identifier live in a server-only query module instead of a callable Server Action. This prevents a browser from invoking a read with another collector's identifier.

The separation is deliberate:

```text
Authenticated mutation
        |
        v
Server Action derives userId from session
        |
        v
Validated update scoped to that user

Server-rendered read
        |
        v
Server-only query module
        |
        v
Explicit public or private projection
```

This design addresses insecure direct object reference risks at the application boundary instead of relying on every caller to remember an ownership check.

### Validated Collection Updates

The server does not accept arbitrary sticker identifiers or counts.

For every update, it:

1. Resolves the authenticated account.
2. Applies the per-user request limiter.
3. Checks the sticker code against the catalog.
4. Normalizes the count to an accepted range.
5. Upserts the `(userId, stickerCode)` record.
6. Revalidates the affected collection pages.

The database has a unique constraint on the account and sticker-code pair, matching the optimistic upsert used by the interface.

### Race-Safe Profile Creation

Profiles are created lazily when an authenticated user first enters the application.

Slug generation handles two possible concurrent conflicts: another profile using the same slug and two requests trying to create a profile for the same account. The insert uses conflict protection and reads the winning record when another request completes first.

### Derived Collection Statistics

Collection totals and progress are pure domain calculations over the static catalog and stored counts. The same functions support the dashboard, team progress, missing lists, duplicate lists, and future trade matching.

Keeping those calculations outside the UI makes them independently testable and prevents different pages from implementing slightly different counting rules.

### Token-Based Visual System

The interface uses semantic `oklch` design tokens with Tailwind CSS and shadcn/ui primitives. Branding and theme changes are applied through tokens rather than repeated component-level color values.

---

## Architecture

```text
Browser
   |
   v
Next.js App Router
   |
   +---- Server Components for authenticated and public reads
   |
   +---- Server Actions for collection and profile mutations
   |
   +---- better-auth session resolution
   |
   v
Server-only queries and domain calculations
   |
   v
Drizzle ORM
   |
   v
PostgreSQL / Neon
```

The data model is intentionally small:

```text
user / session / account / verification
                 |
                 +---- profile (one per account)
                 |
                 +---- sticker entries (account + sticker code + count)
```

The public route uses a restricted field projection and applies the profile's publication and contact preferences before returning data.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16, App Router |
| UI | React 19, TypeScript |
| Styling | Tailwind CSS 4, shadcn/ui, semantic `oklch` tokens |
| Authentication | better-auth |
| Database | PostgreSQL, Neon |
| ORM | Drizzle ORM, drizzle-kit |
| PWA | `@ducanh2912/next-pwa` |
| Testing | Vitest |
| Analytics | Vercel Analytics |
| Hosting | Vercel |

---

## Security and Privacy

- Collection writes derive the account from the authenticated session.
- Private reads are kept out of callable RPC-style actions.
- Collectors can disable public access to their profile.
- Public queries never return account email addresses.
- WhatsApp contact information requires a separate explicit opt-in.
- Sticker codes are checked against the application catalog.
- Counts are normalized on the server.
- Profile fields have server-enforced length and format limits.
- Mutations use per-account fixed-window throttling.
- CSP, frame, content-type, referrer, and browser-permission headers are applied globally.

The current rate limiter stores counters in application memory. It is useful for controlling bursts within an instance, but is not described as a globally distributed rate-limit guarantee.

---

## Testing

The Vitest suite focuses on the product rules that are easiest to break during interface changes:

- Catalog size, groups, codes, and sticker ranges.
- Sticker-code parsing and labeling.
- Case- and accent-insensitive team search.
- Overall and per-team progress.
- Missing and duplicate calculations.
- Trade-match calculations.
- Rate-limit windows, expiry, blocking, and account isolation.

```bash
npm test
```

---

## Running Locally

Requirements:

- Node.js 20 or newer.
- PostgreSQL database.
- npm.

```bash
npm install
cp .env.example .env.local
npx drizzle-kit push
npm run dev
```

Required environment variables:

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `BETTER_AUTH_SECRET` | Secret used to sign authentication state |
| `BETTER_AUTH_URL` | Public base URL of the application |

---

## Current Scope

The product currently provides collection tracking, duplicate organization, profile publication, and sharing. It does not yet automatically discover compatible collectors, even though the domain layer already contains the comparison logic required to calculate mutually useful trades.

Other possible evolutions include making profile publication opt-in, richer Open Graph images for shared profiles, and a dedicated custom domain.

---

## Author

**Lucas da Silva Santos** — Full Stack Developer

[LinkedIn](https://linkedin.com/in/lucas-da-silva-santos-a46879285)

> TrocaCopa is not affiliated with or endorsed by FIFA, Panini, or Coca-Cola.
