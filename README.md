# MovieMind (Reelo)

[![CI](https://github.com/VEDLESS/MovieMind/actions/workflows/ci.yml/badge.svg)](https://github.com/VEDLESS/MovieMind/actions/workflows/ci.yml)

**A social movie diary where your taste is your identity.** Log films, rate (half-stars) and review them, curate ranked or unranked lists, follow friends, discover how closely your tastes align, get hybrid recommendations, and decide what to watch together with realtime group voting (**Watch Together**).

Built with **Expo (React Native)** and **Supabase**, running entirely on free tiers.

| Documentation | Path |
| --- | --- |
| Full setup, deploy & demo guide | [docs/HANDOFF.md](docs/HANDOFF.md) |
| Architecture, ER diagrams & RLS policy matrix | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) |
| Design decisions log | [docs/DECISIONS.md](docs/DECISIONS.md) |
| Build progress | [docs/PROGRESS.md](docs/PROGRESS.md) |
| Product blueprint | [docs/Blueprint.md.txt](docs/Blueprint.md.txt) |

---

## Features

- **Watch Together (Realtime Group Voting):** Invite friends by 6-character code or link, pool picks with runtime/genre/streaming filters, and vote live via **Swipe** or **Ranked Choice (Instant-Runoff Voting)**. The winning film is automatically added to every member's watchlist.
- **Taste Engine & Recommendations:** User-to-user **Taste Match %** (mean-centred cosine similarity with shrinkage), *"People with similar taste"*, and hybrid recommendations combining item-item collaborative filtering, content similarity (genres, directors, cast, decades), and popularity cold-start fallback.
- **Film Diary, Reviews & Lists:** Half-star ratings, spoiler-masked reviews, rewatch flags, tags, likes, comments, private/public profiles with follow requests, filterable watchlist, and ranked/unranked custom lists.
- **Offline-First Sync:** Idempotent client-UUID mutation queue with FIFO replay and server-enforced last-write-wins (`upsert_diary_entry`) that syncs automatically on reconnect.
- **Stats & Yearly Wrapped:** Calendar activity heatmap, rating histogram, genre/decade/director breakdowns, and shareable yearly Wrapped cards.
- **Poster-Themed Film Pages:** Server-side poster color palette extraction in the `tmdb` Edge Function, plus cast, trailers, and regional *"Where to watch"* streaming providers.
- **Letterboxd CSV Import:** Server-side CSV parser & TMDB matcher (`import-letterboxd` Edge Function) for diary, reviews, ratings, watched, and watchlist exports.
- **Database-Level Security:** Row-Level Security (RLS) enabled on every table, verified automatically across multiple accounts in CI (`npm run test:db`).

---

## Tech Stack

- **Mobile App:** Expo SDK 58, React Native 0.88, React 19, Expo Router, TypeScript (strict), NativeWind v4 (Tailwind CSS)
- **UI & Motion:** React Native Reanimated 4, Gesture Handler, FlashList v2, `expo-image`, `react-native-svg`, Lucide icons, DM Serif Display + Inter fonts
- **State & Persistence:** TanStack Query, Zustand, MMKV (with automatic AsyncStorage fallback in Expo Go)
- **Backend:** Supabase (PostgreSQL + RLS, Auth, Realtime, Storage `avatars` bucket, `pg_cron`, Deno Edge Functions)
- **Film Metadata:** TMDB API (proxied and cached server-side via Supabase Edge Functions — the TMDB key never ships in the app)
- **Testing & CI:** Jest + React Native Testing Library, PGlite (in-process WASM Postgres for migration & RLS checks), ESLint, GitHub Actions

---

## Getting Started

### Prerequisites

- **Node.js 22+**
- **Expo Go** (SDK 58) on your phone + a free [Expo](https://expo.dev) account
- A free [Supabase](https://supabase.com) project
- A free [TMDB](https://www.themoviedb.org/settings/api) API Key (v3) or Read Access Token (v4)

### 1. Clone & Install

```bash
git clone https://github.com/VEDLESS/MovieMind.git
cd MovieMind
npm install
```

### 2. Configure Supabase

Apply the 10 SQL migrations in `supabase/migrations/` in filename order (either via CLI or the Supabase SQL Editor):

```bash
npx supabase login
npx supabase link --project-ref <your-project-ref>
npx supabase db push
```

Set the TMDB secret and deploy the 3 Edge Functions:

```bash
npx supabase secrets set TMDB_API_KEY=<your-tmdb-key> TMDB_REGION=IN
npx supabase functions deploy tmdb
npx supabase functions deploy import-letterboxd
npx supabase functions deploy send-push --no-verify-jwt
```

In the Supabase Dashboard:
- **Authentication → URL Configuration → Redirect URLs:** add `reelo://auth-callback` and `exp://**`.
- **Authentication → Providers → Email:** optionally disable **Confirm email** for faster local testing.
- **Database → Extensions:** ensure `pg_cron` is enabled for nightly similarity refreshes.

### 3. Environment Variables

Copy `.env.example` to `.env` and fill in your Supabase keys (from **Project Settings → API Keys**):

```bash
cp .env.example .env
```

```env
# Public (bundled into the app)
EXPO_PUBLIC_SUPABASE_URL=https://<your-project-ref>.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=<your-publishable-or-anon-key>

# Local seed script only (never prefix with EXPO_PUBLIC_ and never commit .env)
SUPABASE_SERVICE_ROLE_KEY=<your-secret-or-service-role-key>
TMDB_API_KEY=<optional-tmdb-key-for-seeding-posters>
```

### 4. Seed Demo Data (Optional)

Populate ~30 demo users, 82 films, ~1,100 diary entries, follows, reviews, lists, watchlists, precomputed similarity tables, and two Watch Together groups:

```bash
npm run seed
```

Demo logins (all seeded accounts use password `reelo-demo-2026`):
- `demo@example.com` / `reelo-demo-2026`
- `demo2@example.com` / `reelo-demo-2026`

### 5. Run the App

Sign into your Expo account in both the CLI and Expo Go on your phone, then start Metro:

```bash
npx expo login
npx expo start -c
```

Scan the QR code with **Expo Go** (Android) or the **Camera** app (iOS) on the same Wi-Fi network, or run `npx expo start --tunnel` if on separate networks.

---

## Scripts

| Command | Description |
| --- | --- |
| `npm start` | Start Expo development server |
| `npm run android` / `ios` / `web` | Launch Expo targeting Android, iOS, or Web |
| `npm run lint` | Run ESLint (`expo lint`) |
| `npm run typecheck` | Run TypeScript type check (`tsc --noEmit`) |
| `npm test` | Run Jest unit & component test suites |
| `npm run test:db` | Apply all migrations to PGlite and run 44 RLS, trigger, RPC & embed checks |
| `npm run check` | Run lint, typecheck, Jest, and DB tests |
| `npm run seed` | Seed demo dataset into Supabase (requires `SUPABASE_SERVICE_ROLE_KEY` in `.env`) |
| `npm run seed:dry` | Generate seed summary offline without touching Supabase |

---

## Project Structure

```text
app/                  Expo Router screens (tabs: Home, Discover, Log, Together, Profile)
components/           Reusable UI components (components/ui = design primitives)
features/             Pure domain logic (diary, lists, social, groups, stats, recs, offline)
hooks/                TanStack Query data-fetching & mutation hooks
lib/                  Supabase client, TMDB edge proxy client, offline queue, storage, validation
store/                Zustand client state stores (auth, settings)
theme/                Dark/light colour tokens and typography
supabase/migrations/  Timestamped PostgreSQL migrations (schema, RLS, triggers, RPCs, realtime)
supabase/functions/   Deno Edge Functions: tmdb, import-letterboxd, send-push (+ _shared/)
scripts/              Demo seeder (seed.mjs) and PGlite DB verification suite (test-db.mjs)
__tests__/            Jest unit and React Native Testing Library tests
docs/                 Handoff guide, architecture diagrams, design decisions, sample CSV
```

---

## Attribution & Licences

This product uses the TMDB API but is not endorsed or certified by TMDB. Film metadata and artwork © TMDB; streaming availability data by JustWatch via TMDB.

Open-source libraries are used under their respective licences: Expo, Expo Router, React Native, React (MIT); NativeWind, Tailwind CSS (MIT); React Native Reanimated, Gesture Handler, FlashList, `react-native-svg`, `react-native-view-shot`, `react-native-mmkv` (MIT); TanStack Query, Zustand, Supabase JS (MIT); Lucide icons (ISC); DM Serif Display & Inter fonts (SIL OFL 1.1); `jpeg-js` (BSD-3-Clause); PGlite (Apache-2.0 / PostgreSQL licence).
