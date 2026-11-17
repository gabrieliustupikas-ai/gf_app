# ROADCLUB

A social, gamified driving app for car enthusiasts. It launches first in Lithuania and the Baltics. Drive, explore, level up — **never for speed**.

> The name is a placeholder. To rename, edit `apps/mobile/src/config/brand.ts`. To change the neon accent colour, edit `ACCENT` in `apps/mobile/src/theme/tokens.ts`.

## Status: Phase 1 ✅

| Phase | Scope | Status |
|---|---|---|
| 1 | Scaffold, auth, full schema + RLS, map home, automatic trip tracking, trip summary, XP & levels | **done** |
| 2 | Exploration fog-of-war, badges engine, streaks, leaderboards, garage & profile | next |
| 3 | Friends, live map, Safe Areas, clubs | |
| 4 | Push-to-talk radio (LiveKit), group cruises, shared routes | |
| 5 | Challenges, events, sponsored challenges, partner dashboard, admin | |
| 6 | Polish, offline maps, share cards, full i18n, store release | |

## Repository layout

```
.
├── apps/mobile/                 Expo SDK 58 app (React Native 0.88, TypeScript strict, Expo Router)
│   ├── index.ts                 entry: registers background tasks before the router mounts
│   ├── app.config.ts            permissions copy, plugins (location, Mapbox, notifications…)
│   └── src/
│       ├── app/                 routes: onboarding/*, sign-in, (tabs)/{index,trips,radio,clubs,profile}, trip/[id]
│       ├── tracking/            background task, recorder, battery-aware controller, sync queue
│       ├── db/localDb.ts        offline-first SQLite store (raw fixes, trips, hexes, sync queue)
│       ├── demo/                demo mode: seeded history + simulated live drives
│       ├── features/            data hooks (TanStack Query) and auth side effects
│       ├── components/          glass UI kit, map, level-up overlay, hold-to-confirm button
│       ├── i18n/                lt (default), en, lv, et, pl
│       ├── theme/               design tokens (one-file accent) + light/dark/high-contrast
│       ├── stores/              Zustand: settings (persisted), session, live tracking
│       └── config/              brand + Zod-validated env
├── packages/core/               Pure TypeScript shared by app AND Edge Functions (Vitest)
│   └── src/                     tripDetection, antiCheat, modeClassifier, xp, levels, hex (H3),
│                                codec (compact track encoding), pipeline, ingest, sampling, demo routes
├── supabase/
│   ├── migrations/              0001 helpers → 0008 views (PostGIS, RLS, functions, reference data)
│   ├── functions/trip-ingest/   Edge Function: server-authoritative validation + XP
│   ├── functions/_shared/core/  GENERATED copy of packages/core (npm run sync:core)
│   ├── seed.sql                 sample Lithuanian pins & scenic roads (dev only)
│   └── tests/                   SQL test suite (RLS, ingestion, GDPR, retention)
├── scripts/                     verify-migrations.sh, sync-core.mjs, generate-ranks-sql.mjs
└── docs/                        DECISIONS.md, PRIVACY.md
```

## Quick start

Requirements: Node 20+ (22 recommended). For a real device build you also need Xcode or Android Studio, or EAS. The [Supabase CLI](https://supabase.com/docs/guides/cli) and Docker are optional.

```bash
npm install
```

```bash
cp .env.example apps/mobile/.env   # fill in Supabase + Mapbox, or leave empty for offline demo mode
```

```bash
npm run verify                     # core sync check, typecheck, unit + integration tests, lint, DB tests
```

### Run the app

The app uses native modules (Mapbox, background location, SQLite), so it needs a **development build**. Expo Go will not work.

```bash
cd apps/mobile && npx expo run:android      # or: npx expo run:ios
```

```bash
cd apps/mobile && npx eas-cli@latest build --profile development --platform android   # cloud build alternative
```

Then start Metro for later JS-only changes:

```bash
npm run mobile
```

**Try everything without driving:** finish onboarding, then tap **"Try demo mode"**. Eight Lithuanian drives are seeded (Vilnius→Trakai, Curonian Spit, Kaunas loop, Via Baltica). On the Map tab, **Simulate a drive** replays a real route through the same detector → recorder → anti-cheat → XP path that real driving uses, sped up about 12×.

### Backend (Supabase)

```bash
npx supabase start                 # local stack (Docker)
```

```bash
npx supabase db reset              # applies migrations + seed.sql
```

```bash
npx supabase functions serve trip-ingest
```

Hosted project:

```bash
npx supabase link --project-ref <ref>
```

```bash
npx supabase db push
```

```bash
npx supabase functions deploy trip-ingest
```

Put the project URL and anon key in `apps/mobile/.env`. Never put the service-role key in the app.

Without Docker, verify the SQL against plain Postgres 16 + PostGIS (this is what CI does):

```bash
npm run db:verify
```

## How a trip flows

1. **Detect.** `expo-location` delivers fixes to a headless background task. In idle mode it uses coarse sampling plus a departure geofence; in driving mode it uses 2 s high-accuracy fixes, and a saver profile on low battery. `core/tripDetection` is a serialisable reducer: a start needs ≥3 fast fixes (≥16 km/h) and ≥120 m of displacement; a trip ends after 3 min stationary or a 15 min gap. Traffic lights are ignored. A long jam is merged back into one trip.
2. **Record offline.** Fixes go to SQLite immediately. When the trip ends, the shared pipeline runs locally: anti-cheat → transport mode → H3 res-8 cells → XP. The summary is instant even with no network.
3. **Sync.** After a 6 min merge window, the trip is compressed (`core/codec`, about 12 bytes per fix) and POSTed to `trip-ingest`. The server re-runs the **same** core code, so the client is never trusted. It stores everything atomically in `ingest_trip()` under a per-user advisory lock and re-counts new hexes inside that transaction. Uploads are idempotent on `client_trip_id`, and retries use exponential backoff.
4. **Display.** The authoritative XP replaces the preview. Level-ups trigger an animated overlay with haptics.

## Safety & privacy guarantees (enforced in code)

- No speed-based scoring. `TripXpInput` has no speed field, no table stores max speed, and leaderboards (Phase 2) use only distance, exploration, streaks and XP.
- Driving mode hides the tab bar and leaves one big hold-to-confirm control. Nothing needs a tap while moving.
- New users start with location sharing **off** (`ghost`), private trips, and a 365-day raw GPS retention window. A daily `pg_cron` job deletes expired fixes and anonymises old routes.
- 16+ age gate, granular consent (only the terms are required), data export (`export_my_data()`), and full account deletion (`delete_my_account()`).
- RLS is on every table. Clients can never write XP, levels or trip stats. Blocks are honoured in every policy.

See [docs/DECISIONS.md](docs/DECISIONS.md) and [docs/PRIVACY.md](docs/PRIVACY.md).
