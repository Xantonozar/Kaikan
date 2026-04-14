# Kaikansen — skills.md
> Phase-by-phase implementation guide.
> **v7 — Next.js 16. Vercel. 5-part + Year Seed. Manual JWT. Voyage AI Vector Search. Quiz. Surprise Me. Friends System.**
> Each phase = one focused AI coding session. Follow phases in order.
> Read rules.md and knowledge.md BEFORE starting any phase.

---

## BEFORE YOU START — AI AGENT CHECKLIST

```
[ ] Read rules.md completely
[ ] Read knowledge.md completely  
[ ] MongoDB database name is: kaikansen
[ ] Never rename proxy.ts to middleware.ts
[ ] Always await params in Next.js 16
[ ] tsconfig.json must have baseUrl: "."
[ ] audioUrl may be null — always null-check
[ ] Never auto-seed from lib/db.ts
[ ] All errors → toast.error() on frontend
[ ] All API errors → console.error('[API] METHOD /path:', err)
```

---

# PHASE 0 — Seed Scripts (Run Before First Deploy)

## Step 0.1 — Project Bootstrap

```
The AI will create a fresh Next.js 16 project first, THEN seed data.
Seed must run BEFORE first Vercel deploy.

Database: MongoDB Atlas → database name: kaikansen
Create the database manually in Atlas UI first if needed.

Install seed dependencies:
  npm install -D ts-node dotenv

Files to create (from knowledge.md §3):
  scripts/seed-utils.ts       ← shared helpers, logging, API fetchers
  scripts/seed-shared.ts      ← core loop (runSeedForFilter function)
  scripts/seed-all/seed-part1.ts  ← anime A-E
  scripts/seed-all/seed-part2.ts  ← anime F-J
  scripts/seed-all/seed-part3.ts  ← anime K-O
  scripts/seed-all/seed-part4.ts  ← anime P-T
  scripts/seed-all/seed-part5.ts  ← anime U-Z + numbers/symbols
  scripts/seed-year.ts        ← filter by year range

Add to .gitignore:
  scripts/seed.log
  scripts/embed.log
  scripts/seed-all/progress-part*.json
  scripts/embed-progress.json
  scripts/progress-year-*.json

Add tsconfig path for scripts:
  "ts-node": { "files": true, "transpileOnly": true }
```

## Step 0.2 — Seed Part Files Pattern

Each part file follows this exact pattern:
```typescript
// scripts/seed-all/seed-part1.ts
import 'dotenv/config'
import { runSeedForFilter } from '../seed-shared'
import path from 'path'

runSeedForFilter(
  path.join(__dirname, 'progress-part1.json'),
  'Part 1 — Anime A to E',
  (anime: any) => {
    const first = (anime?.name ?? '')[0]?.toUpperCase() ?? ''
    return first >= 'A' && first <= 'E'
  }
).then(() => process.exit(0)).catch(err => {
  console.error('[FATAL]', err)
  process.exit(1)
})

// seed-part2.ts: first >= 'F' && first <= 'J'
// seed-part3.ts: first >= 'K' && first <= 'O'
// seed-part4.ts: first >= 'P' && first <= 'T'
// seed-part5.ts: first >= 'U' || !/^[A-Z]/.test(first)
```

## Step 0.3 — Run Seed

```bash
# Option A: Run all 5 parts sequentially (~2-4 hours total)
npm run seed:all

# Option B: Run parts individually (can run in parallel on different machines)
npm run seed:part1
npm run seed:part2
# etc.

# Option C: Seed by year (faster for testing)
SEED_YEAR=2023 npm run seed:year
SEED_YEAR_FROM=2020 SEED_YEAR_TO=2023 npm run seed:year
```

## Step 0.4 — Verify Seed

```
Check scripts/seed.log for error summary.
Verify in MongoDB Atlas (kaikansen database):
  db.themecaches.countDocuments()       → ~15,000+
  db.artistcaches.countDocuments()      → ~3,000+
  db.themecaches.getIndexes()           → should show "theme_full_search"
  db.themecaches.findOne({ animeTitleEnglish: { $ne: null } })
  db.themecaches.countDocuments({ audioUrl: null })  → note this number
  db.themecaches.findOne({ 'entries.1': { $exists: true } })  → verify nested entries

Expected log summary format:
  Total processed:      ~15,000
  Null audioUrl count:  [some number — acceptable]
  AniList fallbacks:    [most themes]
  Kitsu fallbacks:      [small number]
  Unknown:              [small number]

Only proceed to Phase 1 after seed verified.
```

## Step 0.4b — Verify Embed (run after Phase 15)

```
IMPORTANT — Mongoose array coercion: embedding fields typed as [Number] are coerced
to [] (empty array) by Mongoose when default is null — never to literal null in the DB.
Always check BOTH conditions:

  db.themecaches.countDocuments({ embedding: null })         → should be 0
  db.themecaches.countDocuments({ embedding: { $size: 0 } }) → should be 0 (failed batches)

If embedding:[] exists, those batches failed. Clear and rerun embed:
  db.themecaches.updateMany({ embedding: [] }, { $set: { embedding: null } })
  npm run embed

The vector_index in Atlas MUST be Active before running embed — confirm in Atlas UI first.
```

---

# PHASE 1 — Project Setup

## Step 1.1 — Next.js 16 Init

```bash
npx create-next-app@16 . \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --no-src-dir \
  --import-alias "@/*"

# Turbopack is default in Next.js 16 — no flag needed
# React Compiler enabled via next.config.ts

# Install all runtime dependencies:
npm install \
  jsonwebtoken \
  bcryptjs \
  next-themes \
  @tanstack/react-query \
  @tanstack/react-query-devtools \
  mongoose \
  plyr-react \
  sonner \
  lucide-react \
  clsx \
  tailwind-merge \
  zod

# plyr-react ships its own TypeScript types — NO @types/plyr

# Install type definitions:
npm install -D \
  @types/jsonwebtoken \
  @types/bcryptjs \
  ts-node \
  dotenv

# Create vercel.json:
{
  "crons": [{ "path": "/api/sync/seasonal", "schedule": "0 3 * * 1" }]
}
```

## Step 1.2 — next.config.ts

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  reactCompiler: true,   // React Compiler — stable in Next.js 16
  // Turbopack is default — no config needed
}

export default nextConfig
```

## Step 1.3 — tsconfig.json (CRITICAL)

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"]
    },
    "plugins": [{ "name": "next" }]
  },
  "ts-node": {
    "files": true,
    "transpileOnly": true
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

> **CRITICAL FIX:** `"ts-node"` MUST be a **top-level key** in tsconfig.json — NOT inside `"compilerOptions"`. Placing it inside compilerOptions causes ts-node to silently ignore the config, breaking all seed and embed scripts.

> `baseUrl: "."` is REQUIRED for proxy.ts at root to resolve `@/lib/auth` imports.

## Step 1.4 — ESLint Flat Config

```
Next.js 16 uses flat config — delete any .eslintrc.json if present.
eslint.config.mjs is created by create-next-app.
Verify content matches rules.md §13.

Add to package.json:
  "lint": "eslint . --max-warnings 0"

Note: next lint is removed in Next.js 16. Run: npm run lint
Note: next build no longer runs linting automatically.
```

## Step 1.5 — Tailwind + Global CSS

```
Update tailwind.config.ts:
  - darkMode: ['class', '[data-theme="dark"]']
  - content: [
      './app/**/*.{ts,tsx}',
      './lib/**/*.{ts,tsx}',
      './hooks/**/*.{ts,tsx}',
      './providers/**/*.{ts,tsx}',
      './types/**/*.{ts,tsx}',
    ]
    ↑ NO src/ directory — project uses --no-src-dir
  - Add all design tokens from design.md §2

Create app/globals.css:
  - @tailwind base/components/utilities
  - :root light mode CSS vars (design.md §2)
  - [data-theme="dark"] dark mode CSS vars
  - .interactive state layer
  - Animation keyframes: fadeUp, scaleIn, shimmer, eq1-eq8
  - Plyr custom CSS overrides (mint teal controls)
```

## Step 1.6 — Database + Models

```
CRITICAL: lib/db.ts must NEVER trigger seeding.
Database name: kaikansen — set explicitly in connectDB.

Create lib/db.ts — MongoDB singleton from knowledge.md §4
  - dbName: 'kaikansen' explicit in mongoose.connect options

Create lib/models/index.ts — export all models
Create all models from knowledge.md §2:
  User.model.ts
  ThemeCache.model.ts       ← entries[] nested, animethemesId unique key
  AnimeCache.model.ts       ← includes kitsuId, titleAlternative[]
  ArtistCache.model.ts
  Rating.model.ts           ← post('save') hook — NOT findOneAndUpdate
  WatchHistory.model.ts     ← atEntryId field added
  Follow.model.ts           ← post('findOneAndDelete') NOT post('deleteOne')
  Friendship.model.ts       ← status: pending|accepted|blocked
  Favorite.model.ts
  Notification.model.ts
  SearchCache.model.ts      ← NEW: 30-day TTL
  QuizAttempt.model.ts      ← NEW: quiz scoring
```

## Step 1.7 — Auth Setup

```
Create lib/auth.ts (server-side) from knowledge.md §4:
  signAccessToken, signRefreshToken, verifyAccessToken, verifyRefreshToken
  hashPassword, comparePassword

Create lib/auth-client.ts (client-side):
  _accessToken in-memory (never localStorage)
  getAccessToken, setAccessToken, refreshAccessToken
  authFetch — auto-retry on 401 with toast.error on final failure
  login, logout helpers

Create proxy.ts at PROJECT ROOT (NOT inside /app/):
  CRITICAL: File name = proxy.ts, export function = proxy
  NEVER rename to middleware.ts
  This is a CUSTOM auth helper module — NOT Next.js automatic middleware.
  Next.js auto-detects middleware only from middleware.ts with export { middleware }.
  proxy.ts is instead explicitly imported by route handlers and server components
  that need auth protection. Page-level redirects use server-side auth checks
  inside page.tsx (read cookie → verify → redirect). API routes import proxy helpers
  directly to verify the Bearer token.
  Reads Authorization: Bearer <token>
  Page routes: redirect to /login if unauthorized (called from page.tsx server code)
  API routes: return 401 JSON if unauthorized (called from route handler)
```

## Step 1.8 — Auth API Routes

```
Create app/api/auth/login/route.ts (POST)
Create app/api/auth/register/route.ts (POST)
Create app/api/auth/refresh/route.ts (POST)
Create app/api/auth/logout/route.ts (POST)

All from knowledge.md §4.
Remember: await params pattern — though auth routes have no params.
All errors: console.error + NextResponse.json({ success: false, error, code })
```

## Step 1.9 — Providers + Root Layout

```
Create providers/ThemeProvider.tsx ("use client")
Create providers/QueryProvider.tsx ("use client")
  staleTime: 5min, gcTime: 10min

Create providers/AuthProvider.tsx ("use client")
  On mount: refreshAccessToken() → fetch /api/users/me
  Exposes: { user, isLoading, setUser, login, logout }

Create app/layout.tsx:
  - Outfit + Inter from next/font/google
  - suppressHydrationWarning on <html>
  - ThemeProvider → QueryProvider → AuthProvider → children + Toaster (sonner)

Create lib/embedding.ts (server-side only):
  getQueryEmbedding(query): Promise<number[] | null>
    - Check SearchCache first (30-day TTL)
    - Call Voyage AI if not cached
    - Cache result in SearchCache
  averageVectors(vectors): number[]

Create lib/queryKeys.ts — full object from knowledge.md §9
Create lib/utils.ts:
  getScoreColor(score), getScoreLabel(score)
  formatCount(n), timeAgo(date)
  cn(...inputs) — clsx + tailwind-merge

Create hooks/useAuth.ts — re-export useAuth from AuthProvider
  ALWAYS import useAuth from @/hooks/useAuth — never from provider directly
Create hooks/useUser.ts — return useAuth().user

Create types/app.types.ts — IThemeCache, IThemeEntry, IArtistCache, IAnimeCache, AuthUser
Create types/api.types.ts — ApiResponse<T>, PaginatedResponse<T>

Create .env.local with all variables from knowledge.md §10
```

---

# PHASE 2 — Core API Routes

## Step 2.1 — Theme Routes

```
app/api/themes/popular/route.ts (GET):
  Auth: none
  Params: type (OP|ED), page
  Filter: totalRatings >= 3
  Sort: avgRating DESC, totalRatings DESC, totalWatches DESC
  Returns: ThemeCache docs (entries[] included)
  Error logging: console.error('[API] GET /api/themes/popular:', err)

app/api/themes/seasonal/route.ts (GET):
  Auth: none
  Params: season, year, type, page
  Error logging: console.error('[API] GET /api/themes/seasonal:', err)

app/api/themes/[slug]/route.ts (GET):
  Auth: none
  MUST: const { slug } = await params
  Returns: full ThemeCache doc including entries[]

app/api/themes/[slug]/versions/route.ts (GET):
  Auth: none
  MUST: const { slug } = await params
  Returns: { entries: theme.entries, slug, songTitle, type, sequence }
  Purpose: version switcher on theme page

app/api/themes/random/route.ts (GET):
  Auth: none
  Params: type?, season?, year?
  Uses $sample — knowledge.md §6
  Purpose: Surprise Me feature

app/api/themes/[slug]/recommendations/route.ts (GET):
  Auth: none
  MUST: const { slug } = await params
  Uses stored embedding — zero API calls
  Returns 6 similar themes (same type preferred)
  Falls back to empty array if no embedding yet
```

## Step 2.2 — Search Route

```
app/api/search/route.ts (GET):
  Auth: none
  Params: q (min 2 chars), type, page
  THREE LAYERS from knowledge.md §5:
    Layer 1: $text search (exact, fast, free)
    Layer 2: Vector semantic search (if text returns 0)
    Layer 3: Mood search (if query contains sad/epic/calm etc — skip text)
  Returns searchType: 'text' | 'semantic' | 'mood' | 'none' in meta
  Error: toast hint shown in UI when searchType='none'
```

## Step 2.3 — Artist + Anime Routes

```
app/api/artist/[slug]/route.ts (GET):
  MUST: const { slug } = await params
  Returns: { artist: ArtistCache, themes: ThemeCache[] with entries[] }
  Theme count badge: themes.length
  Entries badge: theme.entries.length > 1 → show "X versions"

app/api/artist/[slug]/themes/route.ts (GET):
  MUST: const { slug } = await params
  ThemeCache.find({ artistSlugs: slug }).lean()

app/api/anime/[anilistId]/route.ts (GET):
  MUST: const { anilistId } = await params
  Returns: { anime: AnimeCache, themes: ThemeCache[], openings, endings }
  openings: themes.filter(t => t.type === 'OP').sort by sequence
  endings:  themes.filter(t => t.type === 'ED').sort by sequence
  Each theme includes entries[] for version switcher

app/api/stats/live/route.ts (GET):
  Auth: none — public
  From knowledge.md §11
```

## Step 2.4 — User Routes

```
app/api/users/[username]/route.ts (GET):
  MUST: const { username } = await params
  Public profile — exclude passwordHash

app/api/users/me/route.ts (GET + PATCH):
  Auth: required
  GET: return current user
  PATCH: Zod validate { displayName?, bio?, avatarUrl? }
```

## Step 2.5 — Protected Action Routes

```
app/api/ratings/route.ts (POST):
  Auth: required
  Zod: { themeSlug, score (1-10), mode (watch|listen) }
  MUST use findOne + .save() pattern — NOT findOneAndUpdate with upsert
  (post('save') hook triggers avgRating recalculation)
  Success: return rating + updated theme avgRating
  Frontend: toast.success('Rating saved!')

app/api/ratings/[themeSlug]/mine/route.ts (GET):
  MUST: const { themeSlug } = await params
  Auth: required

app/api/favorites/route.ts (POST + DELETE):
  Auth: required
  POST: Favorite.create — toast.success('Added to favorites')
  DELETE: Favorite.findOneAndDelete — toast.success('Removed from favorites')

app/api/history/route.ts (GET + POST):
  Auth: required
  POST body: { themeSlug, atEntryId, mode }
  atEntryId stored so history shows which version was watched
```

## Step 2.6 — Friends Routes (Complete)

```
ALL routes from knowledge.md §8 — implement exactly:

app/api/friends/request/route.ts (POST):
  Create Friendship { requesterId, addresseeId, status: 'pending' }
  Create Notification for addressee: type='friend_request'
  Error if already exists: 409 with appropriate message
  Error if self-add: 400

app/api/friends/route.ts (GET):
  Auth: required
  Return accepted friendships, populate username + avatarUrl

app/api/friends/requests/route.ts (GET):
  Auth: required
  Pending where addresseeId = current user

app/api/friends/requests/sent/route.ts (GET):
  Auth: required
  Pending where requesterId = current user

app/api/friends/status/[username]/route.ts (GET):
  MUST: const { username } = await params
  Returns: { status: 'none'|'pending_sent'|'pending_received'|'accepted', friendshipId? }

app/api/friends/[id]/accept/route.ts (PATCH):
  MUST: const { id } = await params
  Auth: required (addressee only)
  Update status → 'accepted'
  Notify requester: type='friend_accepted'

app/api/friends/[id]/route.ts (DELETE):
  MUST: const { id } = await params
  Auth: required
  Works for: decline pending, cancel sent, unfriend accepted
  No notification sent

app/api/friends/activity/route.ts (GET):
  Auth: required
  Last 10 ratings by accepted friends
```

## Step 2.7 — Follow, Notifications, Quiz Routes

```
app/api/follow/[username]/route.ts (GET + POST + DELETE):
  MUST: const { username } = await params
  Follow.post('findOneAndDelete') hook — NOT post('deleteOne')

app/api/search/users/route.ts (GET):
  Auth: required
  Params: q (min 2 chars)
  Returns up to 10 users matching username (case-insensitive), excludes self
  Used by: Add Friend dialog in friends/page.tsx

app/api/notifications/route.ts (GET)
app/api/notifications/unread-count/route.ts (GET)
app/api/notifications/mark-read/route.ts (PATCH)

app/api/quiz/question/route.ts (GET):
  Filter: { audioUrl: { $ne: null } }
  $sample: { size: 4 } — 1 correct + 3 decoys
  Returns: audioUrl, options, quizType — NO title/artist in response

app/api/quiz/answer/route.ts (POST):
  Validate guess, return { correct, correctAnswer, score }
  Save QuizAttempt if user logged in

app/api/quiz/leaderboard/route.ts (GET):
  Top 10 by score from QuizAttempt

app/api/themes/for-you/route.ts (GET):
  Auth: required
  Min 3 ratings scoring 7+ to activate
  averageVectors of top rated theme embeddings → taste vector
  $vectorSearch numCandidates: 300, limit: 150, then $limit: 10
  Exclude already watched/rated slugs

app/api/sync/seasonal/route.ts (GET):
  Vercel Cron — GET handler
  Verify: Authorization: Bearer {CRON_SECRET}
```

---

# PHASE 3 — Auth UI

## Step 3.1 — Login + Register Pages

```
app/login/page.tsx (Server Component shell)
app/components/auth/LoginForm.tsx ("use client"):
  Uses useAuth().login(email, password)
  Success: router.push('/')
  Error: toast.error(error message from API)
  Loading: spinner on button

app/register/page.tsx + RegisterForm.tsx ("use client"):
  Uses authFetch POST /api/auth/register
  On success: setUser(data.user), router.push('/')
  Error: toast.error(message)
  Zod client-side validation before submit
```

---

# PHASE 4 — Home Page

## Step 4.1 — Home Page

```
app/page.tsx (Server Component):
  Fetch on server: popular, featured seasonal, live stats
  Pass as initialData to HomeClient

app/components/home/HomeClient.tsx ("use client"):
  SECTIONS (in order):
    1. FEATURED STRIP (current season horizontal scroll)
    2. FOR YOU (if logged in + 3+ ratings)
       - If not enough ratings: "Rate a few more themes..." placeholder
    3. FRIENDS ACTIVITY (if logged in + has friends)
    4. TRENDING NOW (ratingVelocity — fast-gaining themes)
    5. POPULAR THEMES (infinite scroll, OP/ED filter)
    6. SURPRISE ME button (SurpriseButton component)
    7. LIVE STATS FOOTER

SurpriseButton component:
  Dice/shuffle icon + "Surprise Me"
  GET /api/themes/random
  On success: router.push(`/theme/${theme.slug}?autoplay=true`)
  Loading: toast.loading('Finding something for you...')
  Error: toast.error('Could not find a theme. Try again!')
```

---

# PHASE 5 — Theme Page

## Step 5.1 — Theme Page + Version Switcher

```
app/theme/[slug]/page.tsx (Server Component):
  MUST: const { slug } = await params
  Fetch: /api/themes/${slug}
  Pass as initialData to ThemePageClient

app/theme/[slug]/ThemePageClient.tsx ("use client"):
  selectedEntry: useState(theme.entries.find(e => e.version === 'Standard') ?? theme.entries[0])

  LAYOUT (in order):
    - VersionSwitcher (if entries.length > 1)
    - VideoPlayer (selectedEntry.videoUrl, mode)
    - WatchListenToggle
      ↑ Listen mode disabled if selectedEntry.audioUrl === null
        Show tooltip: "Audio not available for this version"
    - Song info card
    - Stats row (avgRating, totalRatings, totalWatches)
    - RatingWidget
    - "Similar Themes" section (recommendations)
    - Link to Anime page

app/components/theme/VersionSwitcher.tsx ("use client"):
  Props: { entries: IThemeEntry[], selected: IThemeEntry, onSelect: (entry) => void }
  Only render if entries.length > 1
  Pill tabs: Standard | NC | BD | BD+NC | Piano | etc
  Active: bg-accent text-white
  Inactive: bg-bg-elevated border text-ktext-secondary

VideoPlayer.tsx:
  import Plyr from 'plyr-react'  — NO @types/plyr
  import 'plyr-react/plyr.css'
  If audioUrl is null and mode is 'listen': show disabled state
  On first play: POST /api/history { themeSlug, atEntryId, mode }
  Error on history save: console.error only (don't toast — non-critical)
```

---

# PHASE 6 — Search Page

## Step 6.1 — Search

```
app/search/page.tsx (Server Component shell)
app/components/search/SearchClient.tsx ("use client"):
  300ms debounce
  Three layer feedback in UI:
    searchType='text'     → normal results
    searchType='semantic' → "No exact matches — showing related results" banner
    searchType='mood'     → "Showing themes matching the mood: [detected mood]" banner
    searchType='none'     → "No results found. Try different keywords." + toast hint
  Filter chips: All | OP | ED
  Infinite scroll with useInfiniteQuery
  Error: toast.error('Search failed. Please try again.')
```

---

# PHASE 7 — Anime + Artist Pages

## Step 7.1 — Anime Page

```
app/anime/[anilistId]/page.tsx (Server Component):
  MUST: const { anilistId } = await params

Layout:
  - Hero banner (atGrillImage → bannerImage)
  - Genre tags overlay
  - Title (animeTitleEnglish ?? animeTitle) + AniList score
  - OPENINGS section: each OP as ThemeListRow
    - Show version count badge if entries.length > 1
    - Link to /theme/[slug] for each
  - ENDINGS section: same pattern
  - Artist names link to /artist/[slug]
```

## Step 7.2 — Artist Page

```
app/artist/[slug]/page.tsx (Server Component):
  MUST: const { slug } = await params

Layout:
  - ArtistHeader: avatar, name, stats, FollowButton label="Follow Artist"
  - Discography: ArtistDiscographyRow per theme
    - Show version count badge: entries.length > 1 → "NC + BD"
  - Error: toast.error if artist not found (use notFound() from next/navigation)
```

---

# PHASE 8 — User Profile + History

## Step 8.1 — Profile Page

```
app/user/[username]/page.tsx (Server Component):
  MUST: const { username } = await params

Friend button state from GET /api/friends/status/[username]:
  none             → "Add Friend"
  pending_sent     → "Request Sent" (disabled)
  pending_received → "Accept Request" (green)
  accepted         → "Friends ✓" + ··· menu with Unfriend

Stats: RATINGS | FOLLOWERS | FOLLOWING
(NOT "Friends" — User schema has totalFollowers/totalFollowing, not a friends count)
```

---

# PHASE 9 — Friends System UI

## Step 9.1 — Friends Page

```
app/friends/page.tsx ("use client"):
  Redirect to /login if no user

TABS: Friends | Requests [incoming count badge] | Sent [outgoing count badge]

Friends tab:
  FriendCard per accepted friendship
  "No friends yet" empty state with "Find People" button → /search

Requests tab (INCOMING):
  Per request: avatar + name + "Accept" (green) + "Decline" (grey)
  Accept: PATCH /api/friends/[id]/accept → toast.success('Friend request accepted!')
  Decline: DELETE /api/friends/[id] → toast.success('Request declined')
  Error: toast.error(message)

Sent tab (OUTGOING):
  Per sent request: avatar + name + "Pending" badge + "Cancel" button
  Cancel: DELETE /api/friends/[id] → toast.success('Request cancelled')

Add Friend:
  Search input → GET /api/search/users?q=...
  Results: UserCard with "Add Friend" button
  POST /api/friends/request → toast.success('Friend request sent!')
  Error: toast.error(message)
```

---

# PHASE 10 — Notifications

## Step 10.1 — Notifications Page

```
app/notifications/page.tsx ("use client"):
  useNotifications() hook: refetchInterval: 60_000
  "Mark all as read" button

NotificationCard per type:
  friend_request:  Accept (bg-accent) + Decline buttons inline
                   Accept → PATCH /api/friends/[entityId]/accept
  friend_accepted: simple text
  friend_rated:    theme thumbnail right side + score
  friend_favorited:heart icon right side + theme thumbnail
  follow:          FollowButton label="Follow Back"
                   ↑ This calls /api/follow/[username] (Follow system).
                     Must be "Follow Back" — NOT "Add Friend".
                     "Add Friend" belongs only to the Friends system (/api/friends/request).

All actions: toast.success on success, toast.error on failure
Bell badge in BottomNav: red dot if unreadCount > 0
```

---

# PHASE 11 — Quiz Mode

## Step 11.1 — Quiz Page

```
app/quiz/page.tsx (Server Component shell)
app/components/quiz/QuizClient.tsx ("use client"):

STATE:
  phase: 'idle' | 'playing' | 'answered' | 'summary'
  quizType: 'title' | 'artist' | 'anime'
  currentQuestion, selectedAnswer, score, streak

IDLE SCREEN:
  "Quiz Mode" heading + ≋ icon
  3 quiz type buttons: Guess the Title | Guess the Artist | Guess the Anime
  How it works: "Audio plays — no visuals. Pick the right answer."

PLAYING SCREEN:
  - Audio player (plyr-react in audio mode, NO video element)
  - animeCoverImage blurred (filter: blur(20px)) as background
  - Timer bar animation (30 seconds countdown)
  - 4 option buttons (A/B/C/D)
  - Submit button (disabled until option selected)

ANSWERED SCREEN:
  - Correct: green flash + score + streak
  - Wrong: red flash + correct answer revealed
  - Theme info revealed: cover image unblurred + title + artist + anime
  - "Next Question" button

SUMMARY (after 10 questions):
  Total score, correct count, streak record
  Share score button (copies to clipboard)
  "Play Again" button

ERROR HANDLING:
  If audioUrl null for question: skip, fetch new question silently
  Network error: toast.error('Connection issue. Try again.')
  No questions available: toast.error('Not enough themes available for quiz')
```

---

# PHASE 12 — Settings + Dark Mode

## Step 12.1 — Settings Page

```
app/settings/page.tsx ("use client"):
  Redirect to /login if no user

ThemeToggle: Light | Dark | System (3-option pill)
Logout: style={{ backgroundColor: 'var(--logout-bg)' }}
  On logout: useAuth().logout() → toast.success('Logged out') → router.push('/login')

Profile edit:
  PATCH /api/users/me → toast.success('Profile updated') / toast.error(message)
```

---

# PHASE 13 — Navigation + Layout

## Step 13.1 — Navigation

```
AppHeader.tsx ("use client"):
  useTheme().isDark → rounded-b-[24px] (dark) or border-b (light)
  Surprise Me button: dice icon in header
  Search icon → /search

BottomNav.tsx ("use client"):
  5 items: Home | Search | Quiz | Notifications | Profile
  Bell: red dot if unreadCount > 0
  flex md:hidden

NavigationRail.tsx ("use client"):
  hidden md:flex
  Same items as BottomNav
```

---

# PHASE 14 — Loading States + Polish

## Step 14.1 — Skeletons + Empty States

```
Create loading.tsx in each route folder
Create LoadingSkeleton.tsx:
  ThemeListRowSkeleton, ThemeFeaturedCardSkeleton
  ProfileHeaderSkeleton, NotificationSkeleton
  All use .shimmer from globals.css

Create EmptyState.tsx:
  NoResults, NoHistory, NoFriends, NoNotifications
  NoPopular, SeasonEmpty, NoArtistThemes, NoForYou

Create ErrorBoundary.tsx ("use client"):
  Catch render errors, show retry button
  Log to console.error
```

---

# PHASE 15 — Embed Script (After Seed)

## Step 15.1 — Vector Embedding

```
Create scripts/embed.ts:
  Connect to kaikansen DB
  Batch size: 20 themes per API call
  Delay: 20s between batches (free tier 3 RPM)
  Embed text per theme:
    [
      songTitle,
      artistName,
      ...allArtists,
      ...artistRoles,
      animeTitle,
      animeTitleEnglish,
      ...animeTitleAlternative,
      type === 'OP' ? 'anime opening' : 'anime ending',
    ].filter(Boolean).join(' ')

  NOTE — intentionally excluded:
    mood   → keyword-derived tags, weak semantic signal
    genres → AniList-only, missing for themes AniList didn't find

  Progress: scripts/embed-progress.json
  Logging: scripts/embed.log (same format as seed.log)

Create Atlas Vector Search index in Atlas UI:
  Collection: themecaches
  Index name: vector_index
  JSON definition from rules.md §14

Run: npm run embed
Expected: ~4 hours on free tier (750 API calls at batch=20)
Verify:
  db.themecaches.countDocuments({ embedding: null }) === 0
  db.themecaches.countDocuments({ embedding: { $size: 0 } }) === 0
  (embedding:[] means the batch failed — clear with updateMany and rerun embed if non-zero)
```

---

# PHASE 16 — Deploy

## Step 16.1 — Pre-Deploy Checklist

```
LOCAL:
[ ] Seed complete — db.themecaches.countDocuments() ~15,000+
[ ] Embed complete — db.themecaches.countDocuments({ embedding: null }) === 0
[ ] No failed embeds — db.themecaches.countDocuments({ embedding: { $size: 0 } }) === 0
    (Mongoose coerces null-default array fields to [] — check BOTH conditions)
[ ] Atlas Vector Search index 'vector_index' active (must be Active BEFORE embed ran)
[ ] npm run dev → test all features
[ ] Test dark mode toggle
[ ] Test auth: register → login → refresh → logout
[ ] Test protected routes redirect to /login
[ ] Test rating → avgRating updates on theme
[ ] Test quiz → audio plays without video visible
[ ] Test surprise me → random theme plays
[ ] Test search: exact → text results, vague → semantic, "sad anime" → mood
[ ] Test friends: send → accept → unfriend flow
[ ] npm run lint → 0 errors
[ ] npm run build → 0 errors
```

## Step 16.2 — Vercel Setup

```
1. Push to GitHub
2. Connect repo to Vercel
3. Add ALL environment variables in Vercel dashboard:
   MONGODB_URI=mongodb+srv://...@.../kaikansen
   JWT_SECRET=<64-char>
   JWT_REFRESH_SECRET=<64-char>
   NEXT_PUBLIC_APP_URL=https://your-app.vercel.app
   CRON_SECRET=<random>
   VOYAGE_API_KEY=<from Atlas AI Models>
4. Deploy

POST-DEPLOY VERIFY:
[ ] /api/themes/popular → returns data with entries[]
[ ] /api/search?q=gurenge → returns results
[ ] /api/search?q=sad+anime → returns mood-filtered results
[ ] /api/themes/random → returns random theme
[ ] /api/stats/live → returns { activeUsers, listeningNow }
[ ] /api/quiz/question → returns audioUrl (no title)
[ ] Auth flow works end-to-end
[ ] Cron: Monday 3am auto-syncs new season data
```

## Step 16.3 — DEPLOY.md

```
Create /DEPLOY.md with full checklist above.

REMINDERS:
- Seed runs ONCE — never re-run on future deploys
- MongoDB kaikansen DB persists between deploys
- Weekly cron handles new seasons automatically
- Live app NEVER calls AniList/AnimeThemes/Kitsu
- proxy.ts (root level) is Next.js 16 middleware — NEVER rename
- audioUrl may be null — always null-check in components
- embedding field uses Atlas Vector Search — NOT Mongoose index
```
