# Kaikansen — rules.md
> Hard rules for every AI agent, developer, or code generation session.
> **v8 — Pure Next.js 16 App Router. Vercel hosting. MongoDB Atlas (kaikansen DB). Seed scripts (5-part + year). Manual JWT. Voyage AI Vector Search. Vidstack Player. AI Search Integration. Advanced Search. No shadcn.**
> Do NOT deviate from these unless explicitly told to in the session prompt.

---

## 1. PROJECT IDENTITY

- App name: **Kaikansen** — anime OP/ED rating, discovery, and social platform
- Primary entity: **OP/ED themes** — anime is metadata attached to themes, not the focus
- Homepage: Popular OP/EDs + friends activity + For You + current season strip
- Target: Mobile-first anime fans
- Language: TypeScript everywhere. No plain JS.
- Database: **MongoDB Atlas — database name: kaikansen** (always specified in URI and connectDB)

---

## 2. ARCHITECTURE — PURE NEXT.JS ON VERCEL

```
┌─────────────────────────────────────────────────────┐
│                   VERCEL                             │
│                                                      │
│   Next.js 16 App Router                              │
│   ├── /app/api/...  (Route Handlers = API)           │
│   ├── Server Components (data fetching)              │
│   ├── Client Components (interactions)               │
│   └── proxy.ts (JWT auth protection — root level)    │
│                                                      │
│   MongoDB Atlas — database: kaikansen                │
│   └── All data pre-seeded before first deploy only   │
└─────────────────────────────────────────────────────┘

ZERO external API calls from live app:
  ✅ Search      → MongoDB $text + Atlas Vector Search
  ✅ Ratings     → MongoDB only
  ✅ Browse      → MongoDB only
  ✅ Recommend   → MongoDB Vector Search (stored embeddings)
  ✅ For You     → MongoDB Vector Search (stored embeddings)

AnimeThemes + AniList + Kitsu → ONLY called by seed scripts
Seed runs ONCE. MongoDB Atlas persists forever.
```

### NO Express. NO separate server. NO backend folder.

---

## 3. TECH STACK

| Layer | Choice | Notes |
|---|---|---|
| Framework | Next.js 16 App Router | Pages = `page.tsx`, API = `route.ts` |
| Hosting | Vercel | Free tier + Pro for cron |
| Database | MongoDB Atlas | Database name: **kaikansen** |
| Auth | Manual JWT | Access token in memory, refresh in httpOnly cookie |
| Dark mode | next-themes | `data-theme` attribute |
| Video player | Vidstack | `"use client"` — modern, accessible, robust player |
| Client state | TanStack Query v5 | Mutations + client queries |
| Styling | Tailwind CSS + CSS vars | Light/dark via CSS custom properties |
| UI components | Custom Tailwind | **No shadcn** — hand-built only |
| Validation | Zod | All API route inputs |
| Toast notifications | sonner | ALL user-facing feedback — success + error |
| Icons | lucide-react | Consistent icon set throughout |
| Class utilities | clsx + tailwind-merge | Used in `cn()` helper in lib/utils.ts |
| Seed scripts | scripts/ folder | 5-part alphabetical OR by-year — run ONCE |
| Scheduled sync | Vercel Cron Jobs | Weekly re-sync for new seasons |
| Bundler | Turbopack | Default in Next.js 16 — `--turbopack` flag in dev script is redundant but harmless |
| React Compiler | Enabled via next.config.ts | `reactCompiler: true` |
| Node.js minimum | 20.9 | Required by Next.js 16 |
| Vector search | MongoDB Atlas Vector Search + Voyage AI | Voyage AI via Atlas AI Models panel |
| Embed model | voyage-4-large | 1024-dim, free 200M tokens |
| AI Search | Voyager AI | Integrated into search bar with dedicated button |

---

## 4. FOLDER STRUCTURE

```
/
├── app/
│   ├── layout.tsx
│   ├── globals.css
│   ├── not-found.tsx
│   │
│   ├── page.tsx                              ← HomePage (Server Component)
│   ├── loading.tsx
│   ├── search/page.tsx
│   ├── theme/[slug]/page.tsx
│   ├── anime/[anilistId]/page.tsx
│   ├── artist/[slug]/page.tsx
│   ├── season/[season]/[year]/page.tsx
│   ├── user/[username]/page.tsx
│   ├── friends/page.tsx
│   ├── notifications/page.tsx
│   ├── history/page.tsx
│   ├── settings/page.tsx
│   ├── login/page.tsx
│   ├── register/page.tsx
│   ├── quiz/page.tsx
│   │
│   ├── api/
│   │   ├── auth/login/route.ts
│   │   ├── auth/register/route.ts
│   │   ├── auth/refresh/route.ts
│   │   ├── auth/logout/route.ts
│   │   ├── themes/popular/route.ts
│   │   ├── themes/seasonal/route.ts
│   │   ├── themes/random/route.ts            ← Surprise Me
│   │   ├── themes/for-you/route.ts           ← Personalised (auth)
│   │   ├── themes/[slug]/route.ts
│   │   ├── themes/[slug]/versions/route.ts   ← All entries of a theme
│   │   ├── themes/[slug]/recommendations/route.ts
│   │   ├── search/route.ts                   ← $text → vector fallback
│   │   ├── anime/[anilistId]/route.ts
│   │   ├── artist/[slug]/route.ts
│   │   ├── artist/[slug]/themes/route.ts
│   │   ├── ratings/route.ts
│   │   ├── ratings/[themeSlug]/mine/route.ts
│   │   ├── favorites/route.ts
│   │   ├── friends/route.ts                  ← GET accepted friends
│   │   ├── friends/request/route.ts          ← POST send request
│   │   ├── friends/requests/route.ts         ← GET incoming pending
│   │   ├── friends/requests/sent/route.ts    ← GET outgoing pending
│   │   ├── friends/status/[username]/route.ts← GET friendship status
│   │   ├── friends/[id]/accept/route.ts      ← PATCH accept
│   │   ├── friends/[id]/route.ts             ← DELETE decline/unfriend
│   │   ├── friends/activity/route.ts
│   │   ├── follow/[username]/route.ts
│   │   ├── notifications/route.ts
│   │   ├── notifications/unread-count/route.ts
│   │   ├── notifications/mark-read/route.ts
│   │   ├── users/me/route.ts
│   │   ├── users/[username]/route.ts
│   │   ├── search/users/route.ts             ← GET ?q= for Add Friend search
│   │   ├── history/route.ts
│   │   ├── stats/live/route.ts
│   │   ├── quiz/question/route.ts
│   │   ├── quiz/answer/route.ts
│   │   ├── quiz/leaderboard/route.ts
│   │   └── sync/seasonal/route.ts
│   │
│   └── components/
│       ├── auth/LoginForm.tsx
│       ├── auth/RegisterForm.tsx
│       ├── theme/ThemeListRow.tsx
│       ├── theme/ThemeFeaturedCard.tsx
│       ├── theme/ThemeCard.tsx
│       ├── theme/VideoPlayer.tsx
│       ├── theme/WatchListenToggle.tsx
│       ├── theme/RatingWidget.tsx
│       ├── theme/VersionSwitcher.tsx         ← NEW: Standard/NC/BD tabs
│       ├── artist/ArtistHeader.tsx
│       ├── artist/ArtistDiscographyRow.tsx
│       ├── layout/AppHeader.tsx
│       ├── layout/BottomNav.tsx
│       ├── layout/NavigationRail.tsx
│       ├── layout/PageWrapper.tsx
│       ├── quiz/QuizClient.tsx               ← NEW
│       ├── home/HomeClient.tsx
│       ├── search/SearchClient.tsx
│       └── shared/
│           ├── LoadingSkeleton.tsx
│           ├── EmptyState.tsx
│           ├── ThemeToggle.tsx
│           ├── FollowButton.tsx
│           ├── SurpriseButton.tsx            ← NEW
│           └── ErrorBoundary.tsx
│
├── lib/
│   ├── db.ts                                 ← MongoDB singleton (kaikansen db)
│   ├── auth.ts                               ← JWT sign/verify (server)
│   ├── auth-client.ts                        ← Client token store + auto-refresh
│   ├── embedding.ts                          ← Voyage AI helpers (server only)
│   ├── models/
│   │   ├── index.ts
│   │   ├── User.model.ts
│   │   ├── ThemeCache.model.ts
│   │   ├── AnimeCache.model.ts
│   │   ├── ArtistCache.model.ts
│   │   ├── Rating.model.ts
│   │   ├── WatchHistory.model.ts
│   │   ├── Favorite.model.ts
│   │   ├── Friendship.model.ts
│   │   ├── Follow.model.ts
│   │   ├── Notification.model.ts
│   │   ├── SearchCache.model.ts              ← NEW
│   │   └── QuizAttempt.model.ts              ← NEW
│   ├── api/
│   │   ├── themes.ts
│   │   ├── artist.ts
│   │   ├── search.ts
│   │   ├── ratings.ts
│   │   ├── favorites.ts
│   │   ├── friends.ts
│   │   ├── follow.ts
│   │   ├── notifications.ts
│   │   ├── users.ts
│   │   ├── history.ts
│   │   ├── quiz.ts
│   │   └── stats.ts
│   ├── queryKeys.ts
│   └── utils.ts
│
├── hooks/
│   ├── useUser.ts
│   ├── useAuth.ts                            ← re-export from AuthProvider
│   ├── useRating.ts
│   ├── useFavorite.ts
│   ├── useFriends.ts
│   ├── useNotifications.ts
│   ├── useSearch.ts
│   ├── useTheme.ts
│   ├── useFollow.ts
│   ├── useStats.ts
│   └── useQuiz.ts                            ← NEW
│
├── providers/
│   ├── QueryProvider.tsx
│   ├── ThemeProvider.tsx
│   └── AuthProvider.tsx
│
├── types/
│   ├── app.types.ts
│   └── api.types.ts
│
├── scripts/
│   ├── seed-utils.ts                         ← shared helpers, logging, API fetchers
│   ├── seed-shared.ts                        ← core seed loop, imported by all parts
│   ├── seed-year.ts                          ← seed by year range
│   ├── seed-all/
│   │   ├── seed-part1.ts                     ← anime A-E
│   │   ├── seed-part2.ts                     ← anime F-J
│   │   ├── seed-part3.ts                     ← anime K-O
│   │   ├── seed-part4.ts                     ← anime P-T
│   │   └── seed-part5.ts                     ← anime U-Z + symbols
│   ├── embed.ts
│   ├── seed.log                              ← gitignored
│   ├── embed.log                             ← gitignored
│   ├── seed-all/progress-part*.json          ← gitignored
│   ├── embed-progress.json                   ← gitignored
│   └── progress-year-*.json                  ← gitignored
│
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json                             ← MUST have baseUrl: "." for proxy.ts
├── eslint.config.mjs                         ← Flat config (Next.js 16)
├── vercel.json
├── proxy.ts                                  ← JWT middleware ROOT LEVEL
├── AGENTS.md                                 ← AI agent instructions (rules summary for Cursor/Copilot/etc.)
└── package.json
```

---

## 5. DATABASE RULES

- MongoDB database name: **kaikansen** — always explicit in URI and `connectDB({ dbName: 'kaikansen' })`
- **MONGODB_URI must include the database name**: `mongodb+srv://user:pass@cluster.mongodb.net/kaikansen`
  Without it, Mongoose defaults to `test` — all data goes to the wrong DB silently.
- All Mongoose models in `/lib/models/`
- MongoDB connection: singleton in `/lib/db.ts`
- `connectDB()` called at top of EVERY Route Handler
- **`lib/db.ts` must NEVER trigger seeding** — seeding is strictly manual CLI
- `animethemesId` is THE unique key for ThemeCache — not slug alone
- Slug formula: `${anime.slug}-${type.toLowerCase()}${sequence}-${animethemesId}` — guaranteed unique
- `audioUrl` may be **null** — if null, play audio using the video URL but keep the video hidden (background playback). Never disable playback because `audioUrl` is missing.
- `animeTitleEnglish` may be null — always fallback to `animeTitle` (Romaji) for display.
- `entries[]` — ALL versions nested in ThemeCache. Never separate docs per version.
- `videoSources[]` inside each entry — ALL quality variants. Never discard lower resolutions.
- `mood[]` — keep as an empty array `[]` for now.
- Title normalization: Seed and normalize Romaji, Native, English, and Alternative titles.
- Fallback priority: AnimeThemes (primary) → AniList → Kitsu.
- Kitsu matching: Use both `slug` and `title` to choose the most accurate match dynamically.
- Image seeding: Seed Small cover and Banner images. Priority: AnimeThemes (largest available) → AniList → Kitsu.
- Studio/Series: Use data from AnimeThemes if available, fallback to AniList, then empty.
- Full Data Seeding: Store ALL available data from AnimeThemes API without skipping important fields.
- `embedding` field — NO regular Mongoose index. Use Atlas Vector Search index only.
- `.lean()` for all read-only queries
- Max 50/page default, 30 for popular/seasonal

---

## 6. SEED RULES — CRITICAL

```
TWO SEED MODES:
  1. npm run seed:all  → 5 separate files (A-E, F-J, K-O, P-T, U-Z+symbols)
  2. npm run seed:year → filter by anime year (SEED_YEAR=2023 or SEED_YEAR_FROM/TO)

EACH SEED FILE:
  - Independently resumable via its own progress-*.json file
  - Detailed logging to console AND scripts/seed.log
  - Progress saved after every page

NEVER:
  - Re-seed on subsequent deploys
  - Trigger seeding from lib/db.ts
  - Call AniList/AnimeThemes/Kitsu from live app routes

DATA SOURCE PRIORITY:
  AnimeThemes → AniList (malId first, title fallback) → Kitsu → null/unknown

DELAYS (respect rate limits):
  AnimeThemes: 700ms between pages
  AniList:     1000ms between requests
  Kitsu:       500ms between requests

LOGGING FORMAT:
  [ISO_TIMESTAMP] [LEVEL] [CONTEXT] message
  Levels: INFO | WARN | ERROR | SUCCESS
  Written to both console and scripts/seed.log

FINAL SUMMARY MUST INCLUDE:
  - Total themes processed
  - Total errors
  - Null audioUrl count
  - AniList fallback count
  - Kitsu fallback count
  - Unknown/null count
```

---

## 7. AUTH RULES (Manual JWT)

- **Access token**: signed JWT, 15min expiry, stored in memory — never localStorage
- **Refresh token**: signed JWT, 7 day expiry, stored in httpOnly `refresh_token` cookie
- JWT signing: `jsonwebtoken` with `JWT_SECRET` and `JWT_REFRESH_SECRET`
- Password hashing: `bcryptjs` (12 rounds)
- `proxy.ts` at root: custom auth helper module — **NOT Next.js automatic middleware**
  - Next.js only auto-detects middleware from `middleware.ts` with `export { middleware }`
  - `proxy.ts` is instead explicitly imported by route handlers and server pages that need protection
  - API routes: import and call `verifyAccessToken()` from proxy.ts (or lib/auth.ts) on each request
  - Protected pages: call `verifyAccessToken()` server-side in `page.tsx`, use `redirect('/login')` if invalid
- Export function named `proxy` — **NEVER rename to `middleware`** (avoids confusion with Next.js middleware)
- `proxy.ts` uses `@/lib/auth` — requires `tsconfig.json` `baseUrl: "."` to resolve
- On 401: auto-refresh via `/api/auth/refresh`, retry once, logout if fails

---

## 8. API ROUTE RULES

Every Route Handler must:
1. Call `await connectDB()` at top
2. For protected routes: `verifyAccessToken()` from Bearer header
3. Validate input with Zod
4. Return consistent shape: `{ success: true, data: T }` or `{ success: false, error: string, code: number }`
5. Wrap in try/catch — never let unhandled errors crash
6. Log errors: `console.error('[API] METHOD /path:', err)`
7. **`await params` before destructuring** — params is a Promise in Next.js 16

```typescript
// CORRECT params pattern:
export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ slug: string }> }
) {
  const { slug } = await params  // ← ALWAYS await
}
```

---

## 9. DARK MODE RULES

- Library: `next-themes`, `attribute="data-theme"`, `defaultTheme="system"`
- `suppressHydrationWarning` on `<html>`
- All colors via CSS custom properties — auto-switch on `[data-theme="dark"]`
- `useTheme()` hook in `hooks/useTheme.ts`

---

## 10. UI RULES

- **No shadcn/ui** — all components hand-built with Tailwind
- No `components/ui/` directory
- All design tokens from `design.md` — colors via CSS vars
- `FollowButton` accepts `label` prop (default `'Follow'`)
- `audioUrl` handling: If `audioUrl` is null, play the video URL in the background (video element hidden). Never disable playback.
- **Vidstack Player**: Used for both video and audio playback. Modern, robust, and accessible.
- All user-facing errors shown via `sonner` toast — `toast.error(message)`
- All user-facing successes shown via `toast.success(message)`
- Loading states: use `toast.loading()` for async actions

---

## 11. VERCEL RULES

- Sync functions max 300s (Pro)
- Cron Jobs call routes via **GET**
- `MONGODB_URI`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `VOYAGE_API_KEY` in Vercel dashboard
- `CRON_SECRET` protects `/api/sync/seasonal`

```json
// vercel.json
{
  "crons": [{ "path": "/api/sync/seasonal", "schedule": "0 3 * * 1" }]
}
```

---

## 12. MOBILE-FIRST RULES

- Tailwind mobile-first — `md:` and `lg:` for larger screens
- Bottom Nav: `flex md:hidden`
- Navigation Rail: `hidden md:flex`
- Touch targets: `min-h-11 min-w-11` (44px)
- Video: `playsInline` always
- Font minimum: `text-sm` (14px)

---

## 13. ESLINT RULES (Next.js 16)

- Flat config format: `eslint.config.mjs` — no `.eslintrc.json`
- `next lint` removed in Next.js 16 — run via `npm run lint`
- `next build` no longer runs linting automatically

```js
// eslint.config.mjs
// create-next-app@16 generates this file automatically — use whatever it produces.
// If manually creating, use the default export from eslint-config-next:

import { dirname } from 'path'
import { fileURLToPath } from 'url'
import { FlatCompat } from '@eslint/eslintrc'

const __filename = fileURLToPath(import.meta.url)
const __dirname  = dirname(__filename)

const compat = new FlatCompat({ baseDirectory: __dirname })

export default [
  ...compat.extends('next/core-web-vitals', 'next/typescript'),
  {
    ignores: ['.next/**', 'out/**', 'next-env.d.ts'],
  },
]
```

> **Note:** Do NOT use `import nextVitals from 'eslint-config-next/core-web-vitals'` — those
> subpath exports do not exist. Use `FlatCompat` with `compat.extends(...)` instead, or use
> whatever `create-next-app@16` generates (preferred — it will always produce a working config).

---

## 14. VECTOR SEARCH RULES

- `embedding` field on ThemeCache — NO regular Mongoose index
- Atlas Vector Search index named `vector_index` — created in Atlas UI
- **Index must be `Active` status BEFORE running `npm run embed`** — if index is created after embed, search will return no results until a manual index rebuild. Confirm Active status in Atlas → cluster → Atlas Search tab.
- Index on `themecaches` collection, 1024 dimensions, dotProduct similarity
- Filter fields in index: `type`, `mood`, `animeSeason`, `animeSeasonYear`
- Voyage AI endpoint: `https://api.voyageai.com/v1/embeddings` (direct) OR `https://ai.mongodb.com/v1/embeddings` (Atlas proxy)
- Use Atlas proxy as primary (key from Atlas AI Models panel)
- `VOYAGE_API_KEY` in env — never hardcode
- Search ranking logic: **Exact match > Title match > Partial match > Related match**.
- Search implements intelligent partial-match search:
  - Loose match (broad search): Query 'nar' finds 'Naruto', 'Naruto Shippuden', 'Boruto: Naruto Next Generations'
  - Exact refinement: Query 'naruto shippuden' returns only 'Naruto Shippuden'
  - Extended query support: Query 'naruto op 9' finds 'Naruto OP 9', 'Naruto Shippuden OP 9', 'Boruto Naruto OP 9'
- AI Search: Add an AI button in the search bar to trigger Voyage AI for semantic/enhanced query handling.
- Search fallback order: `$text` first → zero results → vector search
- Mood queries (sad, epic, etc.) → skip `$text`, go direct to vector + mood filter
- SearchCache: 30-day TTL, cache all query embeddings to minimize API calls
- For You: `numCandidates: 300`, `limit: 150`, then `$limit: 10` after `$match`
- `$vectorSearch` must always be the **first** pipeline stage — never put `$match` before it

---

## 15. FRIEND SYSTEM RULES

```
STATUS VALUES: 'none' | 'pending_sent' | 'pending_received' | 'accepted'

FLOWS:
  Send request:    POST /api/friends/request { addresseeId }
  Accept:          PATCH /api/friends/[id]/accept  (addressee only)
  Decline/Unfriend:DELETE /api/friends/[id]
  Cancel sent:     DELETE /api/friends/[id]  (requester cancels own request)
  Check status:    GET /api/friends/status/[username]

PROFILE BUTTON STATES:
  none             → "Add Friend" (filled accent)
  pending_sent     → "Request Sent" (disabled grey)
  pending_received → "Accept Request" (green filled)
  accepted         → "Friends ✓" + Unfriend in ··· menu

NOTIFICATIONS:
  Send request  → notify addressee: type='friend_request'
  Accept        → notify requester: type='friend_accepted'
  Decline       → no notification
```

---

## 16. SURPRISE ME + QUIZ RULES

```
SURPRISE ME:
  GET /api/themes/random?type=OP&season=WINTER&year=2026
  Uses MongoDB $sample — do NOT use Math.random() on JS arrays
  Frontend: dice icon button → fetch random → navigate to /theme/[slug]?autoplay=true

QUIZ:
  GET  /api/quiz/question?type=title  → returns audioUrl ONLY, hides title/artist
  POST /api/quiz/answer               → { themeSlug, guess, quizType, timeTaken }
  GET  /api/quiz/leaderboard          → top QuizAttempt scores

  audioUrl must not be null for quiz questions — filter: { audioUrl: { $ne: null } }
  Score formula: correct ? Math.max(100, 1000 - Math.floor(timeTaken / 30)) : 0
  Options: 1 correct + 3 random decoys from $sample
```

---

## 17. HARD CONSTRAINTS FOR AI AGENTS

> These are absolute. Never violate them regardless of context.

1. **Never rename `proxy.ts` to `middleware.ts`** — `proxy.ts` is a custom auth module explicitly imported by routes/pages. Naming it `middleware.ts` would imply it is Next.js automatic middleware (which requires `export function middleware`) and cause confusion. Keep the naming distinct.
2. **Never auto-seed from `lib/db.ts`** — seeding is manual CLI only
3. **Always `await params`** in Next.js 16 route handlers and pages
4. **`tsconfig.json` must have `baseUrl: "."`** — required for `@/` alias in root-level `proxy.ts`
5. **`audioUrl` may be null** — if null, play audio using video URL in background; never disable playback.
6. **Use Vidstack Player** — robust and accessible player for video/audio.
7. **Never use `findOneAndUpdate` with upsert for Rating saves** — use `findOne` + `.save()` to trigger hooks
8. **`animethemesId` is the unique ThemeCache key** — slug alone is not unique enough
9. **Never call AniList/AnimeThemes/Kitsu from live app routes** — seed only
10. **MongoDB database name is `kaikansen`** — always explicit, never default
11. **All errors must show toast in frontend** — never silent failures
12. **All API errors must be logged** — `console.error('[API] METHOD /path:', err)`
13. **Remove fake `npx skills add` commands** — they reference non-existent packages

---

## 18. MVP SCOPE

### ✅ Included
- Next.js 16 App Router + Turbopack + React Compiler
- Vercel hosting
- 5-part seed + year-based seed (AnimeThemes → AniList → Kitsu)
- All OP/EDs from AnimeThemes (~15,000+ themes)
- All versions (NC, BD, etc.) nested in `entries[]`
- English titles + alternative names from all sources
- Mood tagging + vector embeddings
- Popular, seasonal, search (text + vector + mood)
- Surprise Me (random OP/ED)
- Quiz mode (audio only, 4 options)
- For You (personalised recommendations)
- Similar themes (vector recommendations)
- Artist pages + Anime pages with all themes
- Version switcher on theme page
- Rating 1–10
- Full-text + semantic search
- User profiles + follow/unfollow
- Friends system (request → accept → unfriend)
- Notifications (polling 60s)
- Watch/Listen history (with atEntryId tracking)
- Light + Dark mode
- Live stats
- Manual JWT auth
- Detailed seed logging

### ❌ Frozen for v2
- Comments
- Leaderboard (quiz leaderboard is MVP)
- OAuth
- Push/email notifications
- Real-time WebSockets
- Lyrics API
- Playlist/Library
- Admin panel
- Block user
- Dynamic color extraction (node-vibrant)

---

## 19. AGENTS.md — REQUIRED PROJECT ROOT FILE

`AGENTS.md` must exist at the project root. It is the first file AI coding assistants (Cursor, GitHub Copilot, Claude Code, etc.) read when opening the project. It should be a short, scannable summary pointing to the full doc set.

```markdown
# Kaikansen — AI Agent Quick Reference
> v7. Read the full docs before writing any code.

## Read these files IN ORDER before any session:
1. rules.md       — hard rules, stack, constraints
2. knowledge.md   — schemas, seed, all route implementations
3. skills.md      — phase-by-phase build guide
4. design.md      — design system, components, tokens
5. VECTOR_SEARCH_V2.md — authoritative search/embed implementation

## Top 5 things that will break everything if you get them wrong:
1. Never rename proxy.ts to middleware.ts — proxy.ts is a custom auth module imported explicitly by routes/pages; naming it middleware.ts implies Next.js auto-interception (which requires middleware.ts + export { middleware }) and will cause auth confusion
2. Always await params — params is a Promise in Next.js 16 route handlers and pages
3. Never seed from lib/db.ts — seeding is manual CLI only (npm run seed:all)
4. audioUrl may be null — always null-check before use, never assume it exists
5. MongoDB database name is kaikansen — always explicit, never rely on URI default

## Stack summary:
- Next.js 16 App Router + Turbopack + React Compiler
- MongoDB Atlas (database: kaikansen) — Mongoose ODM
- Manual JWT (access token in memory, refresh in httpOnly cookie)
- Voyage AI vector search via MongoDB Atlas AI Models panel
- Tailwind CSS + CSS vars (no shadcn — all components hand-built)
- TanStack Query v5, sonner toasts, lucide-react icons, Vidstack player

## Seed flow (run ONCE before first deploy):
  npm run seed:all   → scripts/seed-all/ (A-E, F-J, K-O, P-T, U-Z)
  npm run embed      → scripts/embed.ts (Voyage AI embeddings)

## Key Rules:
1. audioUrl may be null — play video URL in background; never disable playback.
2. Search ranking: Exact > Title > Partial > Related.
3. Seeding fallback: AnimeThemes > AniList > Kitsu.

## Auth pattern:
  proxy.ts at root — custom auth helper, NOT Next.js auto-middleware
  Imported explicitly by route handlers: verifyAccessToken() from Bearer header
  Protected pages: call verifyAccessToken() server-side in page.tsx → redirect('/login')
  Export named 'proxy' — NEVER rename to 'middleware' (avoids confusion with Next.js middleware.ts)
```
