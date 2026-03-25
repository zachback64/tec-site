# TEC Site Architecture Design

**Date:** 2026-03-25
**Project:** Topology Exploration Club — topologyexploration.club
**Repo:** zachback64/tec-site

---

## Overview

Migrate the existing single-file `index.html` to a professional Astro project with working form backends, a Supabase database, unit tests, and performance optimizations. The visual design is unchanged — only the architecture changes.

**Stack:**
- **Astro** — framework and templating (`output: 'hybrid'` to support static pages + API routes)
- **Vercel** — hosting and serverless API routes (already in use)
- **Supabase** — Postgres database for signal list and applications
- **Vitest** — unit test runner
- **Vercel KV** — rate limiting

---

## Project Structure

```
tec-site/
├── src/
│   ├── components/
│   │   ├── Nav.astro
│   │   ├── Hero.astro
│   │   ├── MissionBrief.astro
│   │   ├── Protocol.astro
│   │   ├── Access.astro
│   │   ├── ExpeditionCard.astro
│   │   ├── Archive.astro
│   │   └── Footer.astro
│   ├── layouts/
│   │   └── Base.astro
│   ├── pages/
│   │   ├── index.astro
│   │   └── api/
│   │       ├── signal.ts
│   │       └── apply.ts
│   ├── lib/
│   │   ├── supabase.ts       ← thin client wrapper, no business logic
│   │   └── validation.ts
│   ├── data/
│   │   └── expedition.ts
│   └── styles/
│       └── tokens.css
├── tests/
│   ├── validation.test.ts
│   ├── signal.test.ts
│   ├── apply.test.ts
│   └── ratelimit.test.ts
├── public/
│   ├── fonts/                ← self-hosted Space Mono + EB Garamond
│   └── images/
├── astro.config.mjs
├── tsconfig.json
├── vitest.config.ts
├── .env.example
└── package.json
```

---

## Data Layer

### Expedition Config

`src/data/expedition.ts` is the single file edited each month. All components source data from here. Hero image paths are included so updating the volume means updating one file only.

```typescript
export const currentExpedition = {
  volume: 3,
  name: 'Mar Vista Survey',
  location: 'Mar Vista, Los Angeles',
  coordinates: '34.0°N 118.4°W',
  departure: 'Solar descent — April 2025',
  clearancesRemaining: 7,  // manually updated each month; see note below
  status: 'Accepting applications',
  images: {
    instrument: '/images/vol03-instrument.jpg',
    expeditionist: '/images/vol03-expeditionist.jpg',
  },
} as const

export const archive = [
  { volume: 1, date: 'Feb 2025', location: 'Private residence, Los Angeles' },
  { volume: 2, date: 'Mar 2025', location: 'Location undisclosed, Los Angeles' },
  { volume: 3, date: 'Apr 2025', location: 'Mar Vista, Los Angeles', active: true },
]
```

**`clearancesRemaining` note:** This is a static value for V1. It does not auto-decrement as applications come in. Expedition leadership updates it manually alongside reviewing the Supabase `applications` table. A future iteration could derive it from `capacity - COUNT(applications WHERE volume = current AND status = 'accepted')`.

### Supabase Schema

```sql
-- Signal list: one entry per phone per volume
CREATE TABLE signal_list (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  phone       text NOT NULL,
  name        text NOT NULL,
  email       text,                              -- NULL when not provided
  volume      int  NOT NULL,
  created_at  timestamptz DEFAULT now(),
  UNIQUE (phone, volume)                         -- prevents duplicate signups
);

-- Applications: one per phone per volume
CREATE TABLE applications (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name        text NOT NULL,
  phone       text NOT NULL,
  email       text,                              -- NULL when not provided
  referral    text,
  experience  text,
  volume      int  NOT NULL,
  status      text NOT NULL DEFAULT 'pending'
              CHECK (status IN ('pending', 'accepted', 'rejected')),
  created_at  timestamptz DEFAULT now(),
  UNIQUE (phone, volume)                         -- one application per volume
);
```

**Row-level security:** RLS is enabled on both tables. The service role key (used by API routes) bypasses RLS entirely — this is Supabase's default behavior for the service role. The anon key is not granted any access. No policies are needed for the current use case (API writes only, no client reads).

---

## API Routes

### Error Response Shape

All API errors return JSON: `{ "error": "<message>" }` with the appropriate HTTP status code. Success responses return `null` body with status `201`.

### Origin Check (defense-in-depth only)

Both routes check the `Origin` header as a lightweight defense-in-depth measure, not the primary security gate. Exact comparison: `https://topologyexploration.club` (full URL including scheme). If `Origin` is absent or `null` (e.g. same-origin requests from some browsers, curl), the request is **allowed through** — the check only blocks requests with an explicit non-matching origin. This is not a CSRF token and does not substitute for one; it merely raises the bar against trivial cross-origin abuse.

### Rate Limiting

IP extracted from the first value in the `x-forwarded-for` header (Vercel always sets this). Rate limit buckets are **per-endpoint** — `/api/signal` and `/api/apply` have independent 5-per-hour buckets. If Vercel KV is unavailable, **fail open** (availability preferred for a small club).

Time is injected via a `getNow: () => number` parameter (defaults to `Date.now()`) to allow deterministic testing without real time delays.

### `POST /api/signal`

Accepts: `{ phone, name, email? }`

1. Check `Origin` header (defense-in-depth, see above)
2. Check rate limit for this endpoint's IP bucket
3. Validate all fields via `src/lib/validation.ts`
4. Insert into `signal_list` with `volume` sourced server-side from `currentExpedition.volume` (never from the request body). On unique constraint violation `(phone, volume)`, return `409 { "error": "Already on signal list for this expedition" }`
5. Return `201` on success, `400 { "error": "<reason>" }` on validation failure, `500 { "error": "Server error" }` on DB error

### `POST /api/apply`

Accepts: `{ name, phone, email?, referral?, experience? }`

1. Check `Origin` header (defense-in-depth)
2. Check rate limit for this endpoint's IP bucket
3. Validate required fields
4. Insert into `applications` with `status: 'pending'` and `volume` sourced server-side from `currentExpedition.volume`. On unique constraint violation `(phone, volume)`, return `409 { "error": "Application already submitted for this expedition" }`
5. Return `201` on success, `400 { "error": "<reason>" }` on validation failure, `500 { "error": "Server error" }` on DB error

---

## Validation

`src/lib/validation.ts` — pure functions, no side effects, fully unit-testable.

```typescript
// Returns normalized digit string (e.g. "13105550100")
// Strips non-digits, validates length 7–15.
// Note: does not enforce country code or normalize international formats.
// Two representations of the same number (e.g. "+13105550100" and "13105550100")
// both normalize to "13105550100" — consistent for deduplication purposes.
export const validatePhone = (raw: string): string => {
  const digits = raw.replace(/\D/g, '')
  if (digits.length < 7 || digits.length > 15) throw new Error('Invalid phone number')
  return digits
}

// Returns lowercased trimmed email, or null when not provided.
// Null is stored in the DB (not empty string).
export const validateEmail = (raw: string | undefined | null): string | null => {
  if (!raw || !raw.trim()) return null
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(raw)) throw new Error('Invalid email')
  return raw.trim().toLowerCase()
}

// Returns trimmed name. Throws if blank or over 100 chars.
export const validateName = (raw: string): string => {
  const name = raw.trim()
  if (!name) throw new Error('Name is required')
  if (name.length > 100) throw new Error('Name too long')
  return name
}
```

---

## Testing

**Runner:** Vitest

### `tests/validation.test.ts`
Tests all validator functions with valid inputs, invalid inputs, and edge cases. No mocks needed.

Covers:
- `validatePhone`: valid US number, international number, too-short, too-long, non-digit stripping, both `+13105550100` and `13105550100` normalize identically
- `validateEmail`: valid email, invalid format, empty string → null, undefined → null, whitespace-only → null
- `validateName`: valid name, empty string, whitespace-only, exactly 100 chars (valid), 101 chars (throws)

### `tests/signal.test.ts`
Tests the `/api/signal` route handler with mocked Supabase client. Covers:
- Valid submission → 201
- Missing phone → 400
- Missing name → 400
- Invalid phone format → 400
- Duplicate submission (phone+volume conflict) → 409
- DB insert failure → 500
- Explicit non-matching origin → 403
- `volume` in inserted row equals `currentExpedition.volume` regardless of request body

### `tests/apply.test.ts`
Tests the `/api/apply` route handler with mocked Supabase client. Covers:
- Valid submission → 201
- Missing name → 400
- Missing phone → 400
- Invalid phone format → 400
- Duplicate application → 409
- DB insert failure → 500
- Explicit non-matching origin → 403
- `volume` in inserted row equals `currentExpedition.volume` regardless of request body

### `tests/ratelimit.test.ts`
Tests the rate-limiting middleware. Uses the injected `getNow` parameter to control time without real delays. Covers:
- First request → allowed
- Fifth request → allowed
- Sixth request → 429
- `/api/signal` and `/api/apply` use independent buckets (5 requests to one does not affect the other)
- KV unavailable → request passes through (fail open)
- After injecting `getNow` to return a time 61 minutes later, a previously-blocked IP is allowed again

### `src/lib/supabase.ts`
Thin client instantiation only — no business logic, no tests needed.

---

## Performance

### Self-hosted Fonts
Space Mono and EB Garamond are downloaded and served from `/public/fonts/`. The external Google Fonts request is removed. Eliminates a render-blocking third-party request and removes Google's ability to track site visitors via font loads.

### Image Optimization
Hero images are served via Astro's built-in `<Image>` component: automatically resized, converted to WebP, and lazy-loaded. Image paths are sourced from `expedition.ts`.

### Form Submission
Forms use `fetch`-based submission (not native HTML form POST) to avoid full-page reloads and support inline success/error states. The existing tab-switching `<script>` tag is extended to handle form submit events: prevent default, POST JSON to the appropriate API route, show inline success or error message based on the response. This keeps the JavaScript minimal and self-contained with no framework dependency.

### Zero JavaScript by Default
Astro (`output: 'hybrid'`) ships no JavaScript for static pages unless explicitly opted in. All page sections except the Access form are pure HTML+CSS.

---

## Environment Variables

```bash
# .env.example
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=   # server-only, never in client code
KV_REST_API_URL=
KV_REST_API_TOKEN=
```

---

## Migration & Cutover

1. Build the Astro project to full visual parity with the existing `index.html` (verified via side-by-side screenshot diff)
2. Wire up and smoke-test the API routes against a staging Supabase project
3. Swap the Vercel deployment — same project, same domain, no DNS changes required
4. The existing `index.html` is preserved in git history as rollback reference. If the deploy fails, run `vercel rollback` from the CLI (or use the Vercel dashboard → Deployments → select previous build → Promote to Production). Do not rely on a revert commit triggering auto-deploy — use the Vercel rollback command directly.

---

## Deployment

No change to the existing Vercel project or domain configuration. Astro's Vercel adapter handles serverless function deployment automatically. Push to `main` → auto-deploy.
