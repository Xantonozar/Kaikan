# Kaikansen — knowledge.md
> Everything an AI agent needs to know about internals.
> **v7 — Pure Next.js 16. MongoDB Atlas (kaikansen DB). Seed Script (5-part + by-year). Manual JWT. AnimeThemes Primary. AniList + Kitsu Fallback. Voyage AI Vector Search.**
> Read this ENTIRE file before writing any code. Do NOT skip sections.

---

## 1. MENTAL MODEL

Kaikansen is about **OP/ED themes**, not anime.

```
DATABASE: MongoDB Atlas → database name: "kaikansen"

PRIMARY DATA SOURCE (seed time only — run ONCE before first deploy):
  AnimeThemes API → ALL themes + artists + images + seasons + ALL entries

FALLBACK CHAIN (seed time only):
  1. AnimeThemes   → slug, animethemesId, song, artists, entries, videos, images
  2. AniList       → anilistId (by malId first, title search if malId absent),
                     titleEnglish, titleNative, synonyms, season, year,
                     genres, coverImage, bannerImage
  3. Kitsu         → only if AniList ALSO fails → kitsuId, titleEnglish,
                     titleJapanese, season (derived from startDate), posterImage
  4. null/unknown  → any field still missing after all three sources

LIVE APP (deploy → forever):
  ALL queries → MongoDB Atlas only
  ZERO calls to AnimeThemes, AniList, or Kitsu from live app
  MongoDB Atlas is a cloud DB — data persists between deploys
  MongoDB database name: kaikansen (set in MONGODB_URI connection string)
```

### What we store per theme (ONE doc per OP/ED theme):
- Song title + all artist names/slugs/roles
- English title + all alternative names (union from AT + AniList + Kitsu, deduplicated)
- Season + year
- ALL entries nested as `entries[]` — Standard, NC, BD, Piano variants etc
- Each entry has all quality variants in `videoSources[]` (1080p/720p/480p)
- Each entry has `audioUrl` = lowest resolution video URL (null if no videos)
- Top-level `videoUrl`/`audioUrl` = convenience fields from best standard entry
- Anime cover + grill images
- Mood tags for semantic search (`["sad","emotional","epic","calm"]`)
- Vector embedding (1024-dim, populated by embed script after seed)

---

## 2. MONGODB SCHEMAS

> All models live in `lib/models/`. MongoDB database: **kaikansen**.
> Connection string must specify db: `mongodb+srv://...@.../kaikansen`

---

### `users` — `User.model.ts`

```typescript
import mongoose, { Schema, Document } from 'mongoose'

export interface IUser extends Document {
  username: string
  displayName: string
  email: string
  passwordHash: string
  avatarUrl: string | null
  bio: string
  totalRatings: number
  totalFollowers: number
  totalFollowing: number
  isPublic: boolean
  createdAt: Date
  updatedAt: Date
}

const UserSchema = new Schema<IUser>({
  username:       { type: String, required: true, unique: true, trim: true, lowercase: true },
  displayName:    { type: String, required: true, trim: true },
  email:          { type: String, required: true, unique: true, lowercase: true },
  passwordHash:   { type: String, required: true },
  avatarUrl:      { type: String, default: null },
  bio:            { type: String, default: '', maxlength: 200 },
  totalRatings:   { type: Number, default: 0 },
  totalFollowers: { type: Number, default: 0 },
  totalFollowing: { type: Number, default: 0 },
  isPublic:       { type: Boolean, default: true },
}, { timestamps: true })

UserSchema.index({ username: 1 })
UserSchema.index({ email: 1 })

export const User = mongoose.models.User || mongoose.model<IUser>('User', UserSchema)
```

---

### `themecaches` — `ThemeCache.model.ts` (PRIMARY COLLECTION)

> ONE document per OP/ED theme (identified by animethemesId).
> ALL versions/entries nested in entries[].
> Quality variants (1080p/720p/480p) nested in entries[].videoSources[].

```typescript
import mongoose, { Schema, Document } from 'mongoose'

interface IVideoSource {
  resolution: number   // 1080, 720, 480
  url: string
  source: string | null  // "WEB", "BD", "DVD", "VHS", "RAW"
  mime: string           // always "video/webm"
}

interface IThemeEntry {
  atEntryId: number        // AnimeThemeEntry.id from AT — unique per entry
  version: string          // "Standard", "NC", "BD", "BD+NC", "Piano", "Lyrics" etc
  episodes: string | null  // "1-13", "14-26" — which episodes used this
  spoiler: boolean
  nsfw: boolean
  tags: string[]           // ["NC"], ["BD"], ["NC","BD"], [] for standard
  videoSources: IVideoSource[]  // ALL quality variants of this entry
  videoUrl: string         // highest resolution URL of this entry
  audioUrl: string | null  // lowest resolution URL — null if no videos at all
}

export interface IThemeCache extends Document {
  // Identity — animethemesId is THE unique key
  slug: string             // "${anime.slug}-${type.toLowerCase()}${sequence}-${animethemesId}"
  animethemesId: number    // AT AnimTheme.id — unique across all themes

  // Song
  songTitle: string
  artistName: string | null        // primary artist display name
  allArtists: string[]             // all artist names for search
  artistSlugs: string[]            // all artist slugs for artist page links
  artistRoles: string[]            // vocalist / band / composer etc

  // Anime info
  anilistId: number | null
  kitsuId: string | null
  animeTitle: string               // from AT (romaji)
  animeTitleEnglish: string | null // AT → AniList → Kitsu → null
  animeTitleAlternative: string[]  // union of all alt names, deduplicated
  animeSeason: 'WINTER' | 'SPRING' | 'SUMMER' | 'FALL' | null
  animeSeasonYear: number | null

  // Images
  animeCoverImage: string | null   // AT Cover facet → AniList → Kitsu poster
  animeGrillImage: string | null   // AT Grill facet → AniList banner → null

  // Theme metadata
  type: 'OP' | 'ED'
  sequence: number

  // ALL entries nested — mirrors AT structure
  entries: IThemeEntry[]

  // Convenience fields (from best standard entry — no NC/BD tags)
  videoUrl: string         // highest res of standard entry
  audioUrl: string | null  // lowest res of standard entry — null if unavailable

  // Semantic / mood
  mood: string[]           // ["sad","emotional","epic","calm","hype","dark","upbeat","nostalgic"]
  embedding: number[] | null  // 1024-dim vector, null until embed script runs

  // Community stats (theme level — not per entry)
  avgRating: number
  totalRatings: number
  totalWatches: number
  totalListens: number

  syncedAt: Date
}

const VideoSourceSchema = new Schema<IVideoSource>({
  resolution: { type: Number, required: true },
  url:        { type: String, required: true },
  source:     { type: String, default: null },
  mime:       { type: String, default: 'video/webm' },
}, { _id: false })

const ThemeEntrySchema = new Schema<IThemeEntry>({
  atEntryId:    { type: Number, required: true },
  version:      { type: String, default: 'Standard' },
  episodes:     { type: String, default: null },
  spoiler:      { type: Boolean, default: false },
  nsfw:         { type: Boolean, default: false },
  tags:         [{ type: String }],
  videoSources: [VideoSourceSchema],
  // IMPORTANT: videoUrl is NOT required — entries with no video files will have null.
  // The seed script filters these out (validEntries = parsedEntries.filter(e => e.videoUrl !== null))
  // but keeping this nullable prevents Mongoose validation errors during that filtering step.
  videoUrl:     { type: String, default: null },
  audioUrl:     { type: String, default: null },
}, { _id: false })

const ThemeCacheSchema = new Schema<IThemeCache>({
  slug:                   { type: String, required: true, unique: true },
  animethemesId:          { type: Number, required: true, unique: true },

  songTitle:              { type: String, required: true },
  artistName:             { type: String, default: null },
  allArtists:             [{ type: String }],
  artistSlugs:            [{ type: String }],
  artistRoles:            [{ type: String }],

  anilistId:              { type: Number, default: null },
  kitsuId:                { type: String, default: null },
  animeTitle:             { type: String, required: true },
  animeTitleEnglish:      { type: String, default: null },
  animeTitleAlternative:  [{ type: String }],
  animeSeason:            { type: String, enum: ['WINTER','SPRING','SUMMER','FALL'], default: null },
  animeSeasonYear:        { type: Number, default: null },

  animeCoverImage:        { type: String, default: null },
  animeGrillImage:        { type: String, default: null },

  type:                   { type: String, enum: ['OP','ED'], required: true },
  sequence:               { type: Number, required: true },

  entries:                [ThemeEntrySchema],

  videoUrl:               { type: String, required: true },
  audioUrl:               { type: String, default: null },

  mood:                   [{ type: String }],
  embedding:              { type: [Number], default: null },
  // ⚠ IMPORTANT: Mongoose coerces array fields with default:null to [] on new documents.
  // Do NOT rely on { embedding: null } alone for verification — always check BOTH:
  //   db.themecaches.countDocuments({ embedding: null })         → 0 = good
  //   db.themecaches.countDocuments({ embedding: { $size: 0 } }) → 0 = good
  // embedding:[] means the embed batch was attempted but failed.
  // Fix: db.themecaches.updateMany({ embedding:[] }, { $set:{ embedding:null } }) then rerun embed.

  avgRating:              { type: Number, default: 0 },
  totalRatings:           { type: Number, default: 0 },
  totalWatches:           { type: Number, default: 0 },
  totalListens:           { type: Number, default: 0 },

  syncedAt:               { type: Date, required: true },
}, { timestamps: true })

// TEXT INDEX — powers fast exact search
ThemeCacheSchema.index({
  songTitle:              'text',
  artistName:             'text',
  allArtists:             'text',
  animeTitle:             'text',
  animeTitleEnglish:      'text',
  animeTitleAlternative:  'text',
}, {
  weights: {
    songTitle:             10,
    artistName:            9,
    allArtists:            8,
    animeTitle:            6,
    animeTitleEnglish:     5,
    animeTitleAlternative: 3,
  },
  name: 'theme_full_search',
})

ThemeCacheSchema.index({ animeSeason: 1, animeSeasonYear: 1 })
ThemeCacheSchema.index({ avgRating: -1, totalRatings: -1 })
ThemeCacheSchema.index({ totalWatches: -1 })
ThemeCacheSchema.index({ artistSlugs: 1 })
ThemeCacheSchema.index({ type: 1 })
ThemeCacheSchema.index({ anilistId: 1 })
ThemeCacheSchema.index({ mood: 1 })
// NOTE: embedding field does NOT get a regular Mongoose index
// Use Atlas Vector Search index instead (created in Atlas UI — see §16)

export const ThemeCache = mongoose.models.ThemeCache ||
  mongoose.model<IThemeCache>('ThemeCache', ThemeCacheSchema)
```

---

### `animecaches` — `AnimeCache.model.ts`

```typescript
const AnimeCacheSchema = new Schema({
  anilistId:             { type: Number, default: null },
  malId:                 { type: Number, default: null },
  kitsuId:               { type: String, default: null },
  titleRomaji:           { type: String, required: true },
  titleEnglish:          { type: String, default: null },
  titleNative:           { type: String, default: null },
  titleAlternative:      [{ type: String }],  // union from AT + AniList + Kitsu
  synonyms:              [{ type: String }],
  season:                { type: String, enum: ['WINTER','SPRING','SUMMER','FALL'], default: null },
  seasonYear:            { type: Number, default: null },
  genres:                [{ type: String }],
  coverImageLarge:       { type: String, default: null },
  bannerImage:           { type: String, default: null },
  atCoverImage:          { type: String, default: null },
  atGrillImage:          { type: String, default: null },
  totalEpisodes:         { type: Number, default: null },
  status:                { type: String, default: null },
  averageScore:          { type: Number, default: null },
  syncedAt:              { type: Date, required: true },
}, { timestamps: true })

AnimeCacheSchema.index({ anilistId: 1 })
AnimeCacheSchema.index({ malId: 1 })
AnimeCacheSchema.index({ kitsuId: 1 })
```

---

### `artistcaches` — `ArtistCache.model.ts`

```typescript
const ArtistCacheSchema = new Schema({
  slug:          { type: String, required: true, unique: true },
  animethemesId: { type: Number, required: true },
  name:          { type: String, required: true },
  aliases:       [{ type: String }],
  imageUrl:      { type: String, default: null },
  totalThemes:   { type: Number, default: 0 },
  syncedAt:      { type: Date, required: true },
}, { timestamps: true })

ArtistCacheSchema.index({ slug: 1 })
ArtistCacheSchema.index({ name: 'text', aliases: 'text' })
```

---

### `ratings` — `Rating.model.ts`

> CRITICAL: post('save') hook DOES NOT fire on findOneAndUpdate with upsert.
> Routes must use findOne() then rating.save() to trigger hooks correctly.

```typescript
const RatingSchema = new Schema({
  userId:    { type: Schema.Types.ObjectId, ref: 'User', required: true },
  themeId:   { type: Schema.Types.ObjectId, ref: 'ThemeCache', required: true },
  themeSlug: { type: String, required: true },
  score:     { type: Number, required: true, min: 1, max: 10 },
  mode:      { type: String, enum: ['watch', 'listen'], required: true },
}, { timestamps: true })

RatingSchema.index({ userId: 1, themeId: 1 }, { unique: true })
RatingSchema.index({ userId: 1 })
RatingSchema.index({ themeId: 1 })
RatingSchema.index({ createdAt: -1 })

RatingSchema.post('save', async function () {
  const stats = await mongoose.model('Rating').aggregate([
    { $match: { themeId: this.themeId } },
    { $group: { _id: null, avg: { $avg: '$score' }, count: { $sum: 1 } } },
  ])
  await mongoose.model('ThemeCache').findByIdAndUpdate(this.themeId, {
    avgRating:    parseFloat((stats[0]?.avg ?? 0).toFixed(2)),
    totalRatings: stats[0]?.count ?? 0,
  })
  if (this.isNew) {
    await mongoose.model('User').findByIdAndUpdate(
      this.userId,
      { $inc: { totalRatings: 1 } }
    )
  }
})
```

**Rating route pattern (use this — NOT findOneAndUpdate with upsert):**
```typescript
// app/api/ratings/route.ts
let rating = await Rating.findOne({ userId: payload.userId, themeId })
if (rating) {
  rating.score = score
  rating.mode  = mode
  await rating.save()  // triggers post('save') hook ✓
} else {
  rating = new Rating({ userId: payload.userId, themeId, themeSlug, score, mode })
  await rating.save()  // triggers post('save') with this.isNew = true ✓
}
```

---

### `watchhistories` — `WatchHistory.model.ts`

```typescript
const WatchHistorySchema = new Schema({
  userId:      { type: Schema.Types.ObjectId, ref: 'User', required: true },
  themeId:     { type: Schema.Types.ObjectId, ref: 'ThemeCache', required: true },
  themeSlug:   { type: String, required: true },
  atEntryId:   { type: Number, default: null },  // which version was watched
  mode:        { type: String, enum: ['watch', 'listen'], required: true },
  viewedAt:    { type: Date, default: Date.now },
}, { timestamps: false })

WatchHistorySchema.index({ userId: 1, viewedAt: -1 })
WatchHistorySchema.index({ themeId: 1 })
WatchHistorySchema.index({ viewedAt: -1 })

WatchHistorySchema.post('save', async function () {
  const field = this.mode === 'watch' ? 'totalWatches' : 'totalListens'
  await mongoose.model('ThemeCache')
    .findByIdAndUpdate(this.themeId, { $inc: { [field]: 1 } })
})
```

---

### `follows` — `Follow.model.ts`

> CRITICAL: post('deleteOne') DOES NOT fire on findOneAndDelete().
> Use post('findOneAndDelete') hook instead.

```typescript
const FollowSchema = new Schema({
  followerId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  followeeId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
}, { timestamps: true })

FollowSchema.index({ followerId: 1, followeeId: 1 }, { unique: true })
FollowSchema.index({ followerId: 1 })
FollowSchema.index({ followeeId: 1 })

FollowSchema.post('save', async function () {
  await mongoose.model('User').findByIdAndUpdate(this.followerId, { $inc: { totalFollowing: 1 } })
  await mongoose.model('User').findByIdAndUpdate(this.followeeId, { $inc: { totalFollowers: 1 } })
})

// CORRECT hook — fires on findOneAndDelete()
FollowSchema.post('findOneAndDelete', async function (doc) {
  if (!doc) return
  await mongoose.model('User').findByIdAndUpdate(doc.followerId, { $inc: { totalFollowing: -1 } })
  await mongoose.model('User').findByIdAndUpdate(doc.followeeId, { $inc: { totalFollowers: -1 } })
})
```

---

### `friendships` — `Friendship.model.ts`

> Explicit friend request lifecycle. Read carefully.

```typescript
const FriendshipSchema = new Schema({
  requesterId:  { type: Schema.Types.ObjectId, ref: 'User', required: true },
  addresseeId:  { type: Schema.Types.ObjectId, ref: 'User', required: true },
  status:       { type: String, enum: ['pending','accepted','blocked'], default: 'pending' },
  blockerId:    { type: Schema.Types.ObjectId, ref: 'User', default: null }, // who blocked
}, { timestamps: true })

FriendshipSchema.index({ requesterId: 1, addresseeId: 1 }, { unique: true })
FriendshipSchema.index({ addresseeId: 1, status: 1 })
FriendshipSchema.index({ requesterId: 1, status: 1 })
```

**Friend system flows:**
```
ADD FRIEND:
  POST /api/friends/request { addresseeId }
  → Create Friendship { requesterId, addresseeId, status: 'pending' }
  → Create Notification for addressee: type='friend_request'

ACCEPT:
  PATCH /api/friends/[id]/accept
  → Update Friendship status → 'accepted'
  → Create Notification for requester: type='friend_accepted'

DECLINE:
  DELETE /api/friends/[id]
  → Delete Friendship doc entirely
  → No notification sent

UNFRIEND:
  DELETE /api/friends/[id]
  → Delete Friendship doc (works for accepted too)

CANCEL SENT REQUEST:
  DELETE /api/friends/[id]
  → Requester deletes their own pending request

CHECK STATUS (for profile button state):
  GET /api/friends/status/[username]
  → Returns { status: 'none'|'pending_sent'|'pending_received'|'accepted' }
```

---

### `favorites` — `Favorite.model.ts`

```typescript
const FavoriteSchema = new Schema({
  userId:    { type: Schema.Types.ObjectId, ref: 'User', required: true },
  themeId:   { type: Schema.Types.ObjectId, ref: 'ThemeCache', required: true },
  themeSlug: { type: String, required: true },
}, { timestamps: true })

FavoriteSchema.index({ userId: 1, themeId: 1 }, { unique: true })
FavoriteSchema.index({ userId: 1 })
```

---

### `notifications` — `Notification.model.ts`

```typescript
const NotificationSchema = new Schema({
  recipientId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  actorId:     { type: Schema.Types.ObjectId, ref: 'User', required: true },
  type: {
    type: String,
    enum: [
      'friend_request',
      'friend_accepted',
      'friend_rated',
      'friend_favorited',
      'follow',
    ],
    required: true,
  },
  entityId:   { type: Schema.Types.ObjectId, default: null },
  entityMeta: { type: Schema.Types.Mixed, default: null },
  read:       { type: Boolean, default: false },
}, { timestamps: true })

NotificationSchema.index({ recipientId: 1, read: 1, createdAt: -1 })
NotificationSchema.index({ createdAt: 1 }, { expireAfterSeconds: 60 * 60 * 24 * 30 })
```

---

### `searchcaches` — `SearchCache.model.ts` (NEW)

```typescript
const SearchCacheSchema = new Schema({
  query:     { type: String, required: true, unique: true, lowercase: true, trim: true },
  embedding: { type: [Number], required: true },
  hitCount:  { type: Number, default: 1 },
}, { timestamps: true })

SearchCacheSchema.index({ query: 1 })
SearchCacheSchema.index({ createdAt: 1 }, { expireAfterSeconds: 60 * 60 * 24 * 30 })
```

---

### `quizattempts` — `QuizAttempt.model.ts` (NEW)

```typescript
const QuizAttemptSchema = new Schema({
  userId:       { type: Schema.Types.ObjectId, ref: 'User', default: null }, // null = anonymous
  themeSlug:    { type: String, required: true },
  atEntryId:    { type: Number, required: true },
  quizType:     { type: String, enum: ['title','artist','anime'], required: true },
  correct:      { type: Boolean, required: true },
  timeTaken:    { type: Number, required: true },  // milliseconds
  score:        { type: Number, required: true },  // points earned
  streak:       { type: Number, default: 0 },
}, { timestamps: true })

QuizAttemptSchema.index({ userId: 1, createdAt: -1 })
QuizAttemptSchema.index({ themeSlug: 1 })
QuizAttemptSchema.index({ score: -1 })  // for leaderboard
```

---

## 3. SEED SCRIPT — FULL IMPLEMENTATION

### Overview & Strategy

```
IMPORTANT CONSTRAINTS:
- Run seed BEFORE first deploy
- NEVER re-seed on subsequent deploys
- lib/db.ts MUST NEVER trigger seeding
- Seeding is strictly manual CLI: npm run seed:all OR npm run seed:year
- MongoDB database name: kaikansen

TWO SEED MODES:
  1. npm run seed:all   → scripts/seed-all/  (5 separate files by alphabet)
  2. npm run seed:year  → scripts/seed-year.ts (filter by anime year)

SEED SPLIT (5 files for seed:all):
  seed-part1.ts → anime A-E
  seed-part2.ts → anime F-J
  seed-part3.ts → anime K-O
  seed-part4.ts → anime P-T
  seed-part5.ts → anime U-Z + numbers/symbols

WHY 5 FILES:
  - Each file is independently resumable
  - Can run in parallel on different machines if needed
  - Easier to debug — isolated failures
  - Progress tracked separately per part

DATA SOURCE PRIORITY:
  AnimeThemes → AniList (malId → title search) → Kitsu (title search) → null
```

---

### `scripts/seed-utils.ts` (shared by all seed files)

```typescript
import fs from 'fs'
import path from 'path'

// ── Logging ──────────────────────────────────────────────────────────────────

const LOG_FILE = path.join(process.cwd(), 'scripts', 'seed.log')

export function log(level: 'INFO' | 'WARN' | 'ERROR' | 'SUCCESS', context: string, message: string) {
  const timestamp = new Date().toISOString()
  const line = `[${timestamp}] [${level}] [${context}] ${message}`
  console.log(line)
  fs.appendFileSync(LOG_FILE, line + '\n')
}

export function logSeparator(label: string) {
  const line = `\n${'─'.repeat(60)}\n  ${label}\n${'─'.repeat(60)}`
  console.log(line)
  fs.appendFileSync(LOG_FILE, line + '\n')
}

// ── Progress tracking ─────────────────────────────────────────────────────────

export interface SeedProgress {
  lastCompletedPage: number
  totalProcessed: number
  totalErrors: number
  nullAudioUrlCount: number
  anilistFallbacks: number
  kitsuFallbacks: number
  unknownCount: number
  startedAt: string
  lastUpdatedAt: string
}

export function loadProgress(progressFile: string): SeedProgress {
  if (fs.existsSync(progressFile)) {
    try {
      return JSON.parse(fs.readFileSync(progressFile, 'utf-8'))
    } catch {
      log('WARN', 'PROGRESS', `Could not parse progress file ${progressFile} — starting fresh`)
    }
  }
  return {
    lastCompletedPage: 0,
    totalProcessed: 0,
    totalErrors: 0,
    nullAudioUrlCount: 0,
    anilistFallbacks: 0,
    kitsuFallbacks: 0,
    unknownCount: 0,
    startedAt: new Date().toISOString(),
    lastUpdatedAt: new Date().toISOString(),
  }
}

export function saveProgress(progressFile: string, progress: SeedProgress) {
  progress.lastUpdatedAt = new Date().toISOString()
  fs.writeFileSync(progressFile, JSON.stringify(progress, null, 2))
}

// ── Delays ────────────────────────────────────────────────────────────────────

export const delay = (ms: number) => new Promise(resolve => setTimeout(resolve, ms))

export const AT_DELAY_MS     = 700   // AnimeThemes — be polite
export const ANILIST_DELAY_MS = 1000  // AniList — avoid rate limit
export const KITSU_DELAY_MS  = 500   // Kitsu — generous free tier

// ── Mood derivation ────────────────────────────────────────────────────────────
// Simple keyword-based mood tagging from song/anime metadata
// Will be refined by vector embeddings later

export function deriveMood(songTitle: string, animeTitle: string, genres: string[]): string[] {
  const moods: string[] = []
  const combined = [songTitle, animeTitle, ...genres].join(' ').toLowerCase()

  const moodMap: Record<string, string[]> = {
    sad:        ['sad','grief','loss','cry','tear','sorrow','melancholy'],
    emotional:  ['emotional','feel','heart','soul','memory','memories','never'],
    epic:       ['epic','battle','fight','war','hero','warrior','legend','rise'],
    calm:       ['calm','peace','gentle','soft','quiet','still','breeze'],
    hype:       ['hype','fast','rush','fire','burn','explosion','racing'],
    dark:       ['dark','death','despair','demon','devil','shadow','curse'],
    upbeat:     ['happy','joy','smile','bright','fun','cheer','bounce'],
    nostalgic:  ['past','remember','childhood','youth','old','yesterday','again'],
    romantic:   ['love','kiss','heart','together','forever','your name','promise'],
    mysterious: ['mystery','unknown','secret','hidden','void','abyss'],
  }

  for (const [mood, keywords] of Object.entries(moodMap)) {
    if (keywords.some(kw => combined.includes(kw))) {
      moods.push(mood)
    }
  }

  // Genre-based moods
  if (genres.includes('Romance'))          moods.push('romantic')
  if (genres.includes('Horror'))           moods.push('dark')
  if (genres.includes('Slice of Life'))    moods.push('calm')
  if (genres.includes('Action'))           moods.push('epic')
  if (genres.includes('Comedy'))           moods.push('upbeat')
  if (genres.includes('Drama'))            moods.push('emotional')
  if (genres.includes('Mystery'))          moods.push('mysterious')

  return [...new Set(moods)]  // deduplicate
}

// ── AniList API ───────────────────────────────────────────────────────────────

export async function fetchAniListByMalId(malId: number) {
  const query = `
    query ($malId: Int) {
      Media(idMal: $malId, type: ANIME) {
        id
        idMal
        title { romaji english native }
        synonyms
        season seasonYear
        genres
        episodes
        status
        averageScore
        coverImage { large }
        bannerImage
      }
    }
  `
  try {
    const res = await fetch('https://graphql.anilist.co', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query, variables: { malId } }),
    })
    if (!res.ok) {
      log('WARN', 'ANILIST', `malId ${malId} → HTTP ${res.status}`)
      return null
    }
    const { data } = await res.json()
    return data?.Media ?? null
  } catch (err) {
    log('ERROR', 'ANILIST', `malId ${malId} → ${err}`)
    return null
  }
}

export async function fetchAniListByTitle(title: string) {
  const query = `
    query ($search: String) {
      Media(search: $search, type: ANIME) {
        id
        idMal
        title { romaji english native }
        synonyms
        season seasonYear
        genres
        episodes
        status
        averageScore
        coverImage { large }
        bannerImage
      }
    }
  `
  try {
    const res = await fetch('https://graphql.anilist.co', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query, variables: { search: title } }),
    })
    if (!res.ok) {
      log('WARN', 'ANILIST', `title search "${title}" → HTTP ${res.status}`)
      return null
    }
    const { data } = await res.json()
    return data?.Media ?? null
  } catch (err) {
    log('ERROR', 'ANILIST', `title search "${title}" → ${err}`)
    return null
  }
}

// ── Kitsu API ─────────────────────────────────────────────────────────────────

export async function fetchKitsuByTitle(title: string) {
  try {
    const encoded = encodeURIComponent(title)
    const res = await fetch(
      `https://kitsu.io/api/edge/anime?filter[text]=${encoded}&page[limit]=1`,
      { headers: { 'Accept': 'application/vnd.api+json' } }
    )
    if (!res.ok) {
      log('WARN', 'KITSU', `title search "${title}" → HTTP ${res.status}`)
      return null
    }
    const { data } = await res.json()
    if (!data || data.length === 0) return null

    const anime = data[0]
    const attrs = anime.attributes

    // Derive season from startDate
    let season: string | null = null
    let seasonYear: number | null = null
    if (attrs.startDate) {
      const d = new Date(attrs.startDate)
      seasonYear = d.getFullYear()
      const month = d.getMonth() + 1
      season = month <= 3 ? 'WINTER' : month <= 6 ? 'SPRING' : month <= 9 ? 'SUMMER' : 'FALL'
    }

    return {
      kitsuId:      anime.id,
      titleEnglish: attrs.titles?.en || attrs.titles?.en_us || null,
      titleJapanese:attrs.titles?.ja_jp || null,
      season,
      seasonYear,
      posterImage:  attrs.posterImage?.large ?? null,
      coverImage:   attrs.coverImage?.large ?? null,
    }
  } catch (err) {
    log('ERROR', 'KITSU', `title search "${title}" → ${err}`)
    return null
  }
}

// ── AnimeThemes API ───────────────────────────────────────────────────────────

const AT_BASE = 'https://api.animethemes.moe'

export async function fetchATPage(page: number, include: string) {
  const url = `${AT_BASE}/animetheme?page[size]=100&page[number]=${page}&include=${include}`
  const res = await fetch(url)
  if (!res.ok) throw new Error(`AT page ${page} → HTTP ${res.status}`)
  return res.json()
}

export async function fetchATTotalPages(): Promise<number> {
  const res = await fetch(
    `${AT_BASE}/animetheme?page[size]=1&page[number]=1&include=animethemeentries.videos`
  )
  if (!res.ok) throw new Error(`AT total pages → HTTP ${res.status}`)
  const data = await res.json()
  return Math.ceil(data.meta.total / 100)
}

// ── Theme parsing ─────────────────────────────────────────────────────────────

export function buildVersionLabel(tags: string[]): string {
  if (!tags || tags.length === 0) return 'Standard'
  return tags.map(t => t.toUpperCase()).join('+')
  // e.g. ["NC"] → "NC", ["BD","NC"] → "BD+NC"
}

export function parseATTheme(atTheme: any, genres: string[] = []): any | null {
  try {
    const anime = atTheme.anime
    if (!anime) return null

    const entries = atTheme.animethemeentries ?? []
    if (entries.length === 0) {
      log('WARN', 'PARSE', `Theme ${atTheme.id} "${atTheme.song?.title}" has no entries — skipping`)
      return null
    }

    // Build entries array — one per animethemeentry
    const parsedEntries: any[] = []
    let nullAudioCount = 0

    for (const entry of entries) {
      const videos = (entry.videos ?? []).sort((a: any, b: any) => b.resolution - a.resolution)

      if (videos.length === 0) {
        log('WARN', 'PARSE', `Entry ${entry.id} of theme ${atTheme.id} has no videos — audioUrl will be null`)
        nullAudioCount++
      }

      // Derive tags from video fields (nc, bd, source, tags string)
      const tags: string[] = []
      if (videos[0]?.nc)                           tags.push('NC')
      if (videos[0]?.bd || videos[0]?.source === 'BD') tags.push('BD')
      if (videos[0]?.tags?.includes('Lyrics'))     tags.push('Lyrics')
      if (videos[0]?.tags?.includes('Piano'))      tags.push('Piano')
      if (entry.version && typeof entry.version === 'number' && entry.version > 1) {
        tags.push(`V${entry.version}`)
      }

      const videoSources = videos.map((v: any) => ({
        resolution: v.resolution,
        url:        v.link,
        source:     v.source ?? null,
        mime:       'video/webm',
      }))

      parsedEntries.push({
        atEntryId:    entry.id,
        version:      buildVersionLabel(tags),
        episodes:     entry.episodes ?? null,
        spoiler:      entry.spoiler ?? false,
        nsfw:         entry.nsfw ?? false,
        tags,
        videoSources,
        videoUrl:     videos[0]?.link ?? null,
        audioUrl:     videos.length > 0 ? videos[videos.length - 1].link : null,
      })
    }

    // Remove entries with no videoUrl (completely unusable)
    const validEntries = parsedEntries.filter(e => e.videoUrl !== null)
    if (validEntries.length === 0) {
      log('WARN', 'PARSE', `Theme ${atTheme.id} has no usable entries after filtering — skipping`)
      return null
    }

    // Best standard entry = first entry with no NC/BD tags
    const standardEntry = validEntries.find(e => e.tags.length === 0) ?? validEntries[0]

    const artists = atTheme.song?.artists ?? []
    const allArtists  = artists.map((a: any) => a.name).filter(Boolean)
    const artistSlugs = artists.map((a: any) => a.slug).filter(Boolean)
    const artistRoles = artists.map((a: any) => a.as ?? 'performer')

    const images       = anime.images ?? []
    const atCoverImage = images.find((i: any) => i.facet === 'Cover')?.link ?? null
    const atGrillImage = images.find((i: any) => i.facet === 'Grill')?.link ?? null

    const slug = `${anime.slug}-${atTheme.type.toLowerCase()}${atTheme.sequence ?? 1}-${atTheme.id}`

    const mood = deriveMood(atTheme.song?.title ?? '', anime.name ?? '', genres)

    return {
      slug,
      animethemesId:         atTheme.id,
      songTitle:             atTheme.song?.title ?? 'Unknown',
      artistName:            allArtists[0] ?? null,
      allArtists,
      artistSlugs,
      artistRoles,
      animeTitle:            anime.name,
      animeTitleEnglish:     null,  // filled by enrichment
      animeTitleAlternative: [],    // filled by enrichment
      animeSeason:           anime.season?.toUpperCase() ?? null,
      animeSeasonYear:       anime.year ?? null,
      animeCoverImage:       atCoverImage,
      animeGrillImage:       atGrillImage,
      type:                  atTheme.type as 'OP' | 'ED',
      sequence:              atTheme.sequence ?? 1,
      entries:               validEntries,
      videoUrl:              standardEntry.videoUrl,
      audioUrl:              standardEntry.audioUrl,
      mood,
      embedding:             null,
      anilistId:             null,
      kitsuId:               null,
      syncedAt:              new Date(),
      _nullAudioCount:       nullAudioCount,  // temp field for logging, removed before save
    }
  } catch (err) {
    log('ERROR', 'PARSE', `Failed to parse theme ${atTheme?.id}: ${err}`)
    return null
  }
}

// ── Enrichment (AniList + Kitsu fallback) ─────────────────────────────────────

export async function enrichTheme(
  themeDoc: any,
  atAnime: any,
  progress: SeedProgress,
): Promise<any> {
  let anilistData = null
  let kitsuData   = null

  // Step 1: Try AniList by malId
  if (atAnime.malId) {
    await delay(ANILIST_DELAY_MS)
    anilistData = await fetchAniListByMalId(atAnime.malId)
    if (anilistData) {
      log('SUCCESS', 'ANILIST', `Theme "${themeDoc.songTitle}" → AniList id ${anilistData.id} via malId`)
      progress.anilistFallbacks++
    }
  }

  // Step 2: Try AniList by title if malId failed
  if (!anilistData) {
    await delay(ANILIST_DELAY_MS)
    anilistData = await fetchAniListByTitle(atAnime.name)
    if (anilistData) {
      log('SUCCESS', 'ANILIST', `Theme "${themeDoc.songTitle}" → AniList id ${anilistData.id} via title search`)
      progress.anilistFallbacks++
    } else {
      log('WARN', 'ANILIST', `Theme "${themeDoc.songTitle}" → AniList not found`)
    }
  }

  // Step 3: Try Kitsu if AniList completely failed
  if (!anilistData) {
    await delay(KITSU_DELAY_MS)
    kitsuData = await fetchKitsuByTitle(atAnime.name)
    if (kitsuData) {
      log('SUCCESS', 'KITSU', `Theme "${themeDoc.songTitle}" → Kitsu id ${kitsuData.kitsuId}`)
      progress.kitsuFallbacks++
    } else {
      log('WARN', 'KITSU', `Theme "${themeDoc.songTitle}" → Not found on Kitsu either → fields will be null`)
      progress.unknownCount++
    }
  }

  // Build animeTitleEnglish fallback chain
  const animeTitleEnglish =
    anilistData?.title?.english ??
    kitsuData?.titleEnglish ??
    null

  // Build animeTitleAlternative — union from all sources
  const altNames = [
    anilistData?.title?.native,
    anilistData?.title?.romaji,
    ...(anilistData?.synonyms ?? []),
    kitsuData?.titleJapanese,
  ].filter(Boolean).filter((t: string) => t !== themeDoc.animeTitle)
  const animeTitleAlternative = [...new Set(altNames)]

  // Season/year: AT is primary, AniList fills gaps
  const animeSeason = themeDoc.animeSeason ??
    anilistData?.season ??
    kitsuData?.season ??
    null

  const animeSeasonYear = themeDoc.animeSeasonYear ??
    anilistData?.seasonYear ??
    kitsuData?.seasonYear ??
    null

  // Images: AT primary, AniList fallback, Kitsu last resort
  const animeCoverImage =
    themeDoc.animeCoverImage ??
    anilistData?.coverImage?.large ??
    kitsuData?.posterImage ??
    null

  const animeGrillImage =
    themeDoc.animeGrillImage ??
    anilistData?.bannerImage ??
    kitsuData?.coverImage ??
    null

  // Mood enhancement with genres from AniList
  const genres = anilistData?.genres ?? []
  const mood = deriveMood(themeDoc.songTitle, themeDoc.animeTitle, genres)

  return {
    ...themeDoc,
    anilistId:             anilistData?.id ?? null,
    malId:                 anilistData?.idMal ?? null,
    kitsuId:               kitsuData?.kitsuId ?? null,
    animeTitleEnglish,
    animeTitleAlternative,
    titleNative:           anilistData?.title?.native ?? null,
    synonyms:              anilistData?.synonyms ?? [],
    animeSeason,
    animeSeasonYear,
    animeCoverImage,
    animeGrillImage,
    coverImageLarge:       anilistData?.coverImage?.large ?? null,
    bannerImage:           anilistData?.bannerImage ?? null,
    genres:                anilistData?.genres ?? [],
    totalEpisodes:         anilistData?.episodes ?? null,
    animeStatus:           anilistData?.status ?? null,
    averageScore:          anilistData?.averageScore ?? null,
    mood,
    _nullAudioCount:       undefined,  // remove temp field
  }
}
```

---

### `scripts/seed-shared.ts` (core loop, imported by all 5 parts + year seed)

```typescript
import 'dotenv/config'
import { connectDB } from '../lib/db'
import { ThemeCache, ArtistCache, AnimeCache } from '../lib/models'
import {
  log, logSeparator, loadProgress, saveProgress, delay,
  AT_DELAY_MS, fetchATPage, parseATTheme, enrichTheme, SeedProgress,
} from './seed-utils'
import path from 'path'

// Include string for AnimeThemes API — used by all seed parts
// song.artists.images fetches artist avatar images for ArtistCache.imageUrl
const AT_INCLUDE = 'animethemeentries.videos,song.artists.images,anime.images'

export async function runSeedForFilter(
  progressFile: string,
  label: string,
  filterFn: (atAnime: any) => boolean,  // filter which anime to process
) {
  await connectDB()
  logSeparator(`SEED: ${label}`)

  const progress = loadProgress(progressFile)
  log('INFO', 'SEED', `Resuming from page ${progress.lastCompletedPage + 1}`)
  log('INFO', 'SEED', `Stats so far: ${progress.totalProcessed} processed, ${progress.totalErrors} errors`)

  // Get total pages
  let totalPages: number
  try {
    const firstRes = await fetch(
      `https://api.animethemes.moe/animetheme?page[size]=1&page[number]=1&include=animethemeentries.videos`
    )
    const firstData = await firstRes.json()
    totalPages = Math.ceil(firstData.meta.total / 100)
    log('INFO', 'SEED', `Total AT pages: ${totalPages}`)
  } catch (err) {
    log('ERROR', 'SEED', `Failed to get total pages: ${err}`)
    process.exit(1)
  }

  for (let page = progress.lastCompletedPage + 1; page <= totalPages; page++) {
    log('INFO', 'SEED', `Page ${page}/${totalPages} starting...`)

    let pageData: any
    try {
      await delay(AT_DELAY_MS)
      pageData = await fetchATPage(page, AT_INCLUDE)
    } catch (err) {
      log('ERROR', 'SEED', `Page ${page} fetch failed: ${err} — retrying in 5s`)
      await delay(5000)
      try {
        pageData = await fetchATPage(page, AT_INCLUDE)
      } catch (retryErr) {
        log('ERROR', 'SEED', `Page ${page} retry also failed: ${retryErr} — skipping page`)
        progress.totalErrors++
        saveProgress(progressFile, progress)
        continue
      }
    }

    const themes = pageData.animethemes ?? []
    log('INFO', 'SEED', `Page ${page}: ${themes.length} themes found`)

    for (let i = 0; i < themes.length; i++) {
      const atTheme = themes[i]
      const atAnime = atTheme.anime

      // Apply filter (alphabet range or year range)
      if (!filterFn(atAnime)) continue

      const themeLabel = `"${atTheme.song?.title ?? 'Unknown'}" (${atTheme.type}${atTheme.sequence}) [${atAnime?.name}]`
      log('INFO', 'SEED', `  Theme ${i + 1}/${themes.length}: ${themeLabel}`)

      try {
        // Parse AT data
        const parsed = parseATTheme(atTheme)
        if (!parsed) {
          log('WARN', 'SEED', `  ✗ Skipped: could not parse`)
          progress.totalErrors++
          continue
        }

        if (parsed._nullAudioCount > 0) {
          log('WARN', 'SEED', `  ⚠ ${parsed._nullAudioCount} entries have null audioUrl`)
          progress.nullAudioUrlCount += parsed._nullAudioCount
        }

        // Enrich with AniList + Kitsu
        const enriched = await enrichTheme(parsed, atAnime, progress)

        // Upsert ThemeCache
        await ThemeCache.findOneAndUpdate(
          { animethemesId: enriched.animethemesId },
          { $set: enriched },
          { upsert: true, new: true }
        )
        log('SUCCESS', 'SEED', `  ✓ Upserted ThemeCache: ${enriched.slug}`)

        // Upsert ArtistCache for each artist
        const artists = atTheme.song?.artists ?? []
        for (const artist of artists) {
          if (!artist.slug) continue
          // AT returns artist images when song.artists.images is included
          const artistImageUrl = artist.images?.[0]?.link ?? null
          await ArtistCache.findOneAndUpdate(
            { slug: artist.slug },
            {
              $set: {
                slug:          artist.slug,
                animethemesId: artist.id,
                name:          artist.name,
                aliases:       artist.aliases ?? [],
                imageUrl:      artistImageUrl,
                syncedAt:      new Date(),
              },
              $inc: { totalThemes: 1 },
            },
            { upsert: true }
          )
        }

        // Upsert AnimeCache
        if (enriched.anilistId) {
          await AnimeCache.findOneAndUpdate(
            { anilistId: enriched.anilistId },
            {
              $set: {
                anilistId:        enriched.anilistId,
                malId:            enriched.malId ?? null,
                kitsuId:          enriched.kitsuId,
                titleRomaji:      enriched.animeTitle,
                titleEnglish:     enriched.animeTitleEnglish,
                titleNative:      enriched.titleNative ?? null,
                titleAlternative: enriched.animeTitleAlternative,
                synonyms:         enriched.synonyms ?? [],
                season:           enriched.animeSeason,
                seasonYear:       enriched.animeSeasonYear,
                genres:           enriched.genres ?? [],
                totalEpisodes:    enriched.totalEpisodes ?? null,
                status:           enriched.animeStatus ?? null,
                averageScore:     enriched.averageScore ?? null,
                coverImageLarge:  enriched.coverImageLarge ?? null,
                bannerImage:      enriched.bannerImage ?? null,
                atCoverImage:     enriched.animeCoverImage,
                atGrillImage:     enriched.animeGrillImage,
                syncedAt:         new Date(),
              },
            },
            { upsert: true }
          )
        }

        progress.totalProcessed++
      } catch (err) {
        log('ERROR', 'SEED', `  ✗ Failed to process theme ${atTheme.id}: ${err}`)
        progress.totalErrors++
      }
    }

    progress.lastCompletedPage = page
    saveProgress(progressFile, progress)
    log('INFO', 'SEED', `Page ${page}/${totalPages} complete. Total: ${progress.totalProcessed} processed, ${progress.totalErrors} errors`)
  }

  // Final summary
  logSeparator(`SEED COMPLETE: ${label}`)
  log('INFO', 'SEED', `Total processed:      ${progress.totalProcessed}`)
  log('INFO', 'SEED', `Total errors:         ${progress.totalErrors}`)
  log('INFO', 'SEED', `Null audioUrl count:  ${progress.nullAudioUrlCount}`)
  log('INFO', 'SEED', `AniList fallbacks:    ${progress.anilistFallbacks}`)
  log('INFO', 'SEED', `Kitsu fallbacks:      ${progress.kitsuFallbacks}`)
  log('INFO', 'SEED', `Unknown (all failed): ${progress.unknownCount}`)
  log('INFO', 'SEED', `Log written to: scripts/seed.log`)
}
```

---

### 5 Seed Part Files

```typescript
// scripts/seed-all/seed-part1.ts — anime A-E
import { runSeedForFilter } from '../seed-shared'
import path from 'path'

runSeedForFilter(
  path.join(__dirname, 'progress-part1.json'),
  'Part 1 — Anime A to E',
  (anime) => {
    const first = (anime?.name ?? '')[0]?.toUpperCase() ?? ''
    return first >= 'A' && first <= 'E'
  }
).then(() => process.exit(0)).catch(err => { console.error(err); process.exit(1) })

// scripts/seed-all/seed-part2.ts — anime F-J
// (same pattern, first >= 'F' && first <= 'J')

// scripts/seed-all/seed-part3.ts — anime K-O
// (first >= 'K' && first <= 'O')

// scripts/seed-all/seed-part4.ts — anime P-T
// (first >= 'P' && first <= 'T')

// scripts/seed-all/seed-part5.ts — anime U-Z + numbers/symbols
// (first >= 'U' || first <= '9' || !/[A-Z]/.test(first))
```

---

### Year-based seed file

```typescript
// scripts/seed-year.ts
// Usage: SEED_YEAR=2023 npm run seed:year
// Or:    SEED_YEAR_FROM=2020 SEED_YEAR_TO=2023 npm run seed:year

import 'dotenv/config'
import { runSeedForFilter } from './seed-shared'
import path from 'path'

const yearFrom = parseInt(process.env.SEED_YEAR_FROM ?? process.env.SEED_YEAR ?? '2020')
const yearTo   = parseInt(process.env.SEED_YEAR_TO   ?? process.env.SEED_YEAR ?? '2099')

if (isNaN(yearFrom) || isNaN(yearTo)) {
  console.error('Usage: SEED_YEAR=2023 npm run seed:year')
  console.error('   or: SEED_YEAR_FROM=2020 SEED_YEAR_TO=2023 npm run seed:year')
  process.exit(1)
}

console.log(`Seeding anime from year ${yearFrom} to ${yearTo}`)

runSeedForFilter(
  path.join(__dirname, `progress-year-${yearFrom}-${yearTo}.json`),
  `Year ${yearFrom}–${yearTo}`,
  (anime) => {
    const year = anime?.year ?? null
    if (!year) return false
    return year >= yearFrom && year <= yearTo
  }
).then(() => process.exit(0)).catch(err => { console.error(err); process.exit(1) })
```

---

### package.json scripts

```json
{
  "scripts": {
    "dev":         "next dev --turbopack",
    "build":       "next build",
    "start":       "next start",
    "lint":        "eslint . --max-warnings 0",
    "seed:part1":  "ts-node scripts/seed-all/seed-part1.ts",
    "seed:part2":  "ts-node scripts/seed-all/seed-part2.ts",
    "seed:part3":  "ts-node scripts/seed-all/seed-part3.ts",
    "seed:part4":  "ts-node scripts/seed-all/seed-part4.ts",
    "seed:part5":  "ts-node scripts/seed-all/seed-part5.ts",
    "seed:all":    "npm run seed:part1 && npm run seed:part2 && npm run seed:part3 && npm run seed:part4 && npm run seed:part5",
    "seed:year":   "ts-node scripts/seed-year.ts",
    "embed":       "ts-node scripts/embed.ts"
  }
}
```

---

## 4. ROUTE HANDLERS

### MongoDB Connection (singleton) — `lib/db.ts`

> CRITICAL: This file must NEVER trigger seeding. Zero seed logic here.

```typescript
import mongoose from 'mongoose'

const MONGODB_URI = process.env.MONGODB_URI!
// MONGODB_URI must point to kaikansen database:
// mongodb+srv://user:pass@cluster.mongodb.net/kaikansen

if (!MONGODB_URI) throw new Error('MONGODB_URI not set in environment')

let cached = (global as any).__mongoose ?? { conn: null, promise: null }
;(global as any).__mongoose = cached

export async function connectDB() {
  if (cached.conn) return cached.conn
  if (!cached.promise) {
    cached.promise = mongoose.connect(MONGODB_URI, {
      bufferCommands: false,
      dbName: 'kaikansen',  // explicit — never rely on URI alone
    })
  }
  try {
    cached.conn = await cached.promise
  } catch (err) {
    cached.promise = null
    throw err
  }
  return cached.conn
}
```

---

### Standard Route Handler Pattern

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { connectDB } from '@/lib/db'
import { ThemeCache } from '@/lib/models'

// IMPORTANT: params is a Promise in Next.js 16 — always await it
export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ slug: string }> }
) {
  try {
    await connectDB()
    const { slug } = await params  // ← MUST await params

    const theme = await ThemeCache.findOne({ slug }).lean()
    if (!theme) {
      return NextResponse.json(
        { success: false, error: 'Theme not found', code: 404 },
        { status: 404 }
      )
    }

    return NextResponse.json({ success: true, data: theme })
  } catch (err) {
    console.error('[API] GET /api/themes/[slug]:', err)
    return NextResponse.json(
      { success: false, error: 'Internal server error', code: 500 },
      { status: 500 }
    )
  }
}
```

### All Route Handlers needed

```
app/api/auth/login/route.ts                     POST (public)
app/api/auth/register/route.ts                  POST (public)
app/api/auth/refresh/route.ts                   POST (reads httpOnly cookie)
app/api/auth/logout/route.ts                    POST (clears httpOnly cookie)

app/api/themes/popular/route.ts                 GET ?type&page
app/api/themes/seasonal/route.ts                GET ?season&year&type&page
app/api/themes/[slug]/route.ts                  GET
app/api/themes/[slug]/versions/route.ts         GET — returns entries[] array
app/api/themes/[slug]/recommendations/route.ts  GET (public) — vector similar
app/api/themes/for-you/route.ts                 GET (auth required) — personalised
app/api/themes/random/route.ts                  GET ?type&season&year — surprise me

app/api/search/route.ts                         GET ?q&type&page ($text → vector fallback)

app/api/anime/[anilistId]/route.ts              GET
app/api/artist/[slug]/route.ts                  GET
app/api/artist/[slug]/themes/route.ts           GET

app/api/ratings/route.ts                        POST (auth required)
app/api/ratings/[themeSlug]/mine/route.ts       GET (auth required)

app/api/favorites/route.ts                      POST + DELETE (auth required)

app/api/friends/route.ts                        GET — list accepted friends
app/api/friends/request/route.ts                POST — send friend request
app/api/friends/requests/route.ts               GET — incoming pending requests
app/api/friends/requests/sent/route.ts          GET — outgoing pending requests
app/api/friends/status/[username]/route.ts      GET — none|pending_sent|pending_received|accepted
app/api/friends/[id]/accept/route.ts            PATCH — accept request
app/api/friends/[id]/route.ts                   DELETE — decline or unfriend
app/api/friends/activity/route.ts               GET (auth required)

app/api/follow/[username]/route.ts              GET + POST + DELETE (auth required)

app/api/notifications/route.ts                  GET (auth required)
app/api/notifications/unread-count/route.ts     GET (auth required)
app/api/notifications/mark-read/route.ts        PATCH (auth required)

app/api/users/me/route.ts                       GET + PATCH (auth required)
app/api/users/[username]/route.ts               GET (public)

app/api/history/route.ts                        GET + POST (auth required)
app/api/stats/live/route.ts                     GET (public)

app/api/quiz/question/route.ts                  GET ?type (public)
app/api/quiz/answer/route.ts                    POST (auth optional)
app/api/quiz/leaderboard/route.ts               GET (public)

app/api/sync/seasonal/route.ts                  GET (Vercel Cron — CRON_SECRET)
```

---

## 5. SEARCH — THREE LAYER STRATEGY

```typescript
// app/api/search/route.ts
// AUTHORITATIVE version — matches VECTOR_SEARCH_V2.md §4
// Do NOT use an older version of this route.

import { NextRequest, NextResponse } from 'next/server'
import { connectDB } from '@/lib/db'
import { ThemeCache } from '@/lib/models'
import { getQueryEmbedding } from '@/lib/embedding'

const MOOD_WORDS = [
  'sad', 'epic', 'calm', 'hype', 'emotional', 'dark',
  'upbeat', 'nostalgic', 'romantic', 'mysterious', 'peaceful',
  'intense', 'cheerful', 'melancholy', 'energetic',
]

export async function GET(req: NextRequest) {
  try {
    await connectDB()

    const { searchParams } = new URL(req.url)
    const q    = (searchParams.get('q') ?? '').trim()
    const type = searchParams.get('type')
    const page = parseInt(searchParams.get('page') ?? '1')

    if (q.length < 2) {
      return NextResponse.json(
        { success: false, error: 'Query too short — minimum 2 characters', code: 400 },
        { status: 400 }
      )
    }

    const limit = 20
    const skip  = (page - 1) * limit
    const qLower = q.toLowerCase()

    // Detect mood words in query
    const detectedMoods = MOOD_WORDS.filter(w => qLower.includes(w))
    const isMoodQuery   = detectedMoods.length > 0

    // ── LAYER 1: $text search (exact, fast, free) ──────────────────────────
    if (!isMoodQuery) {
      const textFilter: any = { $text: { $search: q } }
      if (type) textFilter.type = type

      const [textResults, textTotal] = await Promise.all([
        ThemeCache.find(textFilter, { score: { $meta: 'textScore' } })
          .sort({ score: { $meta: 'textScore' } })
          .skip(skip)
          .limit(limit)
          .lean(),
        ThemeCache.countDocuments(textFilter),
      ])

      if (textResults.length > 0) {
        return NextResponse.json({
          success: true,
          data:    textResults,
          meta: {
            page,
            total:   textTotal,
            hasMore: skip + textResults.length < textTotal,
            searchType: 'text',
          },
        })
      }
    }

    // ── LAYER 2 / 3: Vector search (semantic or mood) ─────────────────────
    const embedding = await getQueryEmbedding(q)

    if (!embedding) {
      // Voyage API unavailable — return empty gracefully
      return NextResponse.json({
        success: true,
        data:    [],
        meta: { page: 1, total: 0, hasMore: false, searchType: 'none' },
      })
    }

    // Build vector pre-filters
    const vectorFilter: any = {}
    if (type)                    vectorFilter.type = { $eq: type }
    if (detectedMoods.length > 0) vectorFilter.mood = { $in: detectedMoods }

    const semanticResults = await ThemeCache.aggregate([
      {
        $vectorSearch: {
          index:         'vector_index',
          path:          'embedding',
          queryVector:   embedding,
          numCandidates: 200,
          limit:         limit,
          ...(Object.keys(vectorFilter).length > 0 ? { filter: vectorFilter } : {}),
        },
      },
      {
        $addFields: { vectorScore: { $meta: 'vectorSearchScore' } },
      },
    ])

    return NextResponse.json({
      success: true,
      data:    semanticResults,
      meta: {
        page:       1,
        total:      semanticResults.length,
        hasMore:    false,
        searchType: isMoodQuery ? 'mood' : 'semantic',
        moods:      detectedMoods,
      },
    })

  } catch (err) {
    console.error('[API] GET /api/search:', err)
    return NextResponse.json(
      { success: false, error: 'Search failed. Please try again.', code: 500 },
      { status: 500 }
    )
  }
}
```

---

## 6. SURPRISE ME

```typescript
// app/api/themes/random/route.ts
export async function GET(req: NextRequest) {
  try {
    await connectDB()
    const { searchParams } = new URL(req.url)
    const type   = searchParams.get('type')
    const season = searchParams.get('season')
    const year   = searchParams.get('year')

    const filter: any = {}
    if (type)   filter.type = type.toUpperCase()
    if (season) filter.animeSeason = season.toUpperCase()
    if (year)   filter.animeSeasonYear = parseInt(year)

    const [theme] = await ThemeCache.aggregate([
      { $match: filter },
      { $sample: { size: 1 } },
    ])

    if (!theme) {
      return NextResponse.json(
        { success: false, error: 'No themes found', code: 404 },
        { status: 404 }
      )
    }

    return NextResponse.json({ success: true, data: theme })
  } catch (err) {
    console.error('[API] GET /api/themes/random:', err)
    return NextResponse.json(
      { success: false, error: 'Internal server error', code: 500 },
      { status: 500 }
    )
  }
}
```

---

## 7. QUIZ ROUTES

```typescript
// app/api/quiz/question/route.ts
// Returns theme with audioUrl only — song title/artist hidden from response
export async function GET(req: NextRequest) {
  try {
    await connectDB()
    const { searchParams } = new URL(req.url)
    const quizType = (searchParams.get('type') ?? 'title') as 'title' | 'artist' | 'anime'

    // Get 1 random theme + 3 decoys
    const [correct, ...decoys] = await ThemeCache.aggregate([
      { $match: { audioUrl: { $ne: null } } },
      { $sample: { size: 4 } },
    ])

    if (!correct) {
      return NextResponse.json({ success: false, error: 'Not enough themes', code: 404 }, { status: 404 })
    }

    // Build options based on quiz type — deduplicate to prevent identical answer buttons
    // (e.g. two Naruto Shippuden OPs both having the same animeTitle)
    const getValue = (t: any): string => {
      if (quizType === 'title')  return t.songTitle
      if (quizType === 'artist') return t.artistName ?? 'Unknown'
      return t.animeTitle
    }

    const correctValue = getValue(correct)
    const seen = new Set<string>([correctValue])
    const uniqueDecoys = decoys
      .map(t => getValue(t))
      .filter(v => {
        if (seen.has(v)) return false
        seen.add(v)
        return true
      })
      .slice(0, 3)

    // Pad with numbered fallbacks if fewer than 3 unique decoys available
    while (uniqueDecoys.length < 3) {
      uniqueDecoys.push(`Option ${uniqueDecoys.length + 2}`)
    }

    const options = [correctValue, ...uniqueDecoys].sort(() => Math.random() - 0.5)

    // Return audio only — hide identifying info
    return NextResponse.json({
      success: true,
      data: {
        questionId: correct._id,
        themeSlug:  correct.slug,
        atEntryId:  correct.entries[0]?.atEntryId,
        audioUrl:   correct.audioUrl,
        quizType,
        options,
        // correctAnswer is NOT sent — client must POST to /api/quiz/answer
      },
    })
  } catch (err) {
    console.error('[API] GET /api/quiz/question:', err)
    return NextResponse.json({ success: false, error: 'Server error', code: 500 }, { status: 500 })
  }
}

// app/api/quiz/answer/route.ts
export async function POST(req: NextRequest) {
  try {
    await connectDB()
    const body = await req.json()
    const { themeSlug, guess, quizType, timeTaken } = body

    const theme = await ThemeCache.findOne({ slug: themeSlug }).lean()
    if (!theme) {
      return NextResponse.json({ success: false, error: 'Theme not found', code: 404 }, { status: 404 })
    }

    let correctAnswer: string
    if (quizType === 'title')  correctAnswer = theme.songTitle
    else if (quizType === 'artist') correctAnswer = theme.artistName ?? 'Unknown'
    else correctAnswer = theme.animeTitle

    const correct = guess.trim().toLowerCase() === correctAnswer.trim().toLowerCase()

    // Score: faster = more points, max 1000
    const score = correct ? Math.max(100, 1000 - Math.floor(timeTaken / 30)) : 0

    // Optionally save attempt if user logged in
    const token = req.headers.get('authorization')?.replace('Bearer ', '')
    if (token) {
      const { verifyAccessToken } = await import('@/lib/auth')
      const payload = verifyAccessToken(token)
      if (payload) {
        const { QuizAttempt } = await import('@/lib/models')
        await QuizAttempt.create({
          userId:    payload.userId,
          themeSlug: theme.slug,
          atEntryId: theme.entries[0]?.atEntryId ?? 0,
          quizType,
          correct,
          timeTaken,
          score,
          streak:    body.streak ?? 0,
        })
      }
    }

    return NextResponse.json({
      success: true,
      data: { correct, correctAnswer, score },
    })
  } catch (err) {
    console.error('[API] POST /api/quiz/answer:', err)
    return NextResponse.json({ success: false, error: 'Server error', code: 500 }, { status: 500 })
  }
}
```

---

## 8. FRIENDS SYSTEM — COMPLETE IMPLEMENTATION

```typescript
// app/api/friends/request/route.ts — POST: send friend request
export async function POST(req: NextRequest) {
  try {
    await connectDB()
    const token = req.headers.get('authorization')?.replace('Bearer ', '')
    const payload = token ? verifyAccessToken(token) : null
    if (!payload) return NextResponse.json({ success: false, error: 'Unauthorized', code: 401 }, { status: 401 })

    const { addresseeId } = await req.json()
    if (!addresseeId) return NextResponse.json({ success: false, error: 'addresseeId required', code: 400 }, { status: 400 })
    if (addresseeId === payload.userId) return NextResponse.json({ success: false, error: 'Cannot add yourself', code: 400 }, { status: 400 })

    // Check if already exists
    const existing = await Friendship.findOne({
      $or: [
        { requesterId: payload.userId, addresseeId },
        { requesterId: addresseeId, addresseeId: payload.userId },
      ],
    })
    if (existing) {
      const msg = existing.status === 'accepted' ? 'Already friends'
        : existing.status === 'blocked' ? 'Cannot send request'
        : 'Request already sent'
      return NextResponse.json({ success: false, error: msg, code: 409 }, { status: 409 })
    }

    const friendship = await Friendship.create({ requesterId: payload.userId, addresseeId, status: 'pending' })

    // Notify addressee
    await Notification.create({
      recipientId: addresseeId,
      actorId:     payload.userId,
      type:        'friend_request',
      entityId:    friendship._id,
    })

    return NextResponse.json({ success: true, data: friendship })
  } catch (err) {
    console.error('[API] POST /api/friends/request:', err)
    return NextResponse.json({ success: false, error: 'Server error', code: 500 }, { status: 500 })
  }
}

// app/api/friends/[id]/accept/route.ts — PATCH: accept request
export async function PATCH(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    await connectDB()
    const token = req.headers.get('authorization')?.replace('Bearer ', '')
    const payload = token ? verifyAccessToken(token) : null
    if (!payload) return NextResponse.json({ success: false, error: 'Unauthorized', code: 401 }, { status: 401 })

    const { id } = await params
    const friendship = await Friendship.findById(id)
    if (!friendship) return NextResponse.json({ success: false, error: 'Not found', code: 404 }, { status: 404 })

    // Only addressee can accept
    if (friendship.addresseeId.toString() !== payload.userId) {
      return NextResponse.json({ success: false, error: 'Forbidden', code: 403 }, { status: 403 })
    }
    if (friendship.status !== 'pending') {
      return NextResponse.json({ success: false, error: 'Cannot accept — not pending', code: 400 }, { status: 400 })
    }

    friendship.status = 'accepted'
    await friendship.save()

    // Notify requester
    await Notification.create({
      recipientId: friendship.requesterId,
      actorId:     payload.userId,
      type:        'friend_accepted',
      entityId:    friendship._id,
    })

    return NextResponse.json({ success: true, data: friendship })
  } catch (err) {
    console.error('[API] PATCH /api/friends/[id]/accept:', err)
    return NextResponse.json({ success: false, error: 'Server error', code: 500 }, { status: 500 })
  }
}

// app/api/friends/status/[username]/route.ts — GET: check friendship status
export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ username: string }> }
) {
  try {
    await connectDB()
    const token = req.headers.get('authorization')?.replace('Bearer ', '')
    const payload = token ? verifyAccessToken(token) : null
    if (!payload) return NextResponse.json({ success: true, data: { status: 'none' } })

    const { username } = await params
    const targetUser = await User.findOne({ username }).lean()
    if (!targetUser) return NextResponse.json({ success: false, error: 'User not found', code: 404 }, { status: 404 })

    const friendship = await Friendship.findOne({
      $or: [
        { requesterId: payload.userId, addresseeId: targetUser._id },
        { requesterId: targetUser._id, addresseeId: payload.userId },
      ],
    }).lean()

    if (!friendship) return NextResponse.json({ success: true, data: { status: 'none' } })

    let status: string
    if (friendship.status === 'accepted') status = 'accepted'
    else if (friendship.requesterId.toString() === payload.userId) status = 'pending_sent'
    else status = 'pending_received'

    return NextResponse.json({ success: true, data: { status, friendshipId: friendship._id } })
  } catch (err) {
    console.error('[API] GET /api/friends/status/[username]:', err)
    return NextResponse.json({ success: false, error: 'Server error', code: 500 }, { status: 500 })
  }
}
```

---

## 8b. MISSING ROUTE & LIB IMPLEMENTATIONS

### `app/api/search/users/route.ts` — GET: search users for Add Friend

```typescript
// app/api/search/users/route.ts
// Used by the Add Friend dialog in friends/page.tsx
export async function GET(req: NextRequest) {
  try {
    await connectDB()
    const token = req.headers.get('authorization')?.replace('Bearer ', '')
    const payload = token ? verifyAccessToken(token) : null
    if (!payload) return NextResponse.json({ success: false, error: 'Unauthorized', code: 401 }, { status: 401 })

    const { searchParams } = new URL(req.url)
    const q = (searchParams.get('q') ?? '').trim()
    if (q.length < 2) {
      return NextResponse.json({ success: false, error: 'Query too short', code: 400 }, { status: 400 })
    }

    const users = await User.find({
      username: { $regex: q, $options: 'i' },
      _id:      { $ne: payload.userId },  // exclude self
    })
      .select('username displayName avatarUrl')
      .limit(10)
      .lean()

    return NextResponse.json({ success: true, data: users })
  } catch (err) {
    console.error('[API] GET /api/search/users:', err)
    return NextResponse.json({ success: false, error: 'Server error', code: 500 }, { status: 500 })
  }
}
```

---

### `lib/api/quiz.ts` — Client fetcher for quiz

```typescript
// lib/api/quiz.ts
import { authFetch } from './base'

export async function fetchQuizQuestion(type: 'title' | 'artist' | 'anime') {
  const res = await fetch(`/api/quiz/question?type=${type}`)
  if (!res.ok) throw new Error(`Quiz question fetch failed: HTTP ${res.status}`)
  return res.json()
}

export async function submitQuizAnswer(payload: {
  themeSlug: string
  guess: string
  quizType: 'title' | 'artist' | 'anime'
  timeTaken: number
  streak: number
}) {
  const res = await authFetch('/api/quiz/answer', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
  })
  return res.json()
}

export async function fetchQuizLeaderboard() {
  const res = await fetch('/api/quiz/leaderboard')
  return res.json()
}
```

---

### `lib/api/favorites.ts` — Client fetcher for favorites

```typescript
// lib/api/favorites.ts
import { authFetch } from './base'

export async function toggleFavorite(themeSlug: string, themeId: string, add: boolean) {
  const res = await authFetch('/api/favorites', {
    method: add ? 'POST' : 'DELETE',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ themeSlug, themeId }),
  })
  return res.json()
}

// userId param removed — endpoint uses auth token to identify user server-side
export async function fetchFavorites() {
  const res = await authFetch(`/api/favorites`)
  if (!res.ok) throw new Error('Failed to fetch favorites')
  return res.json()
}
```

---

## 9. QUERY KEYS

```typescript
// lib/queryKeys.ts
export const queryKeys = {
  themes: {
    popular:         (type?: string) => ['themes', 'popular', type],
    seasonal:        (season: string, year: number, type?: string) => ['themes', 'seasonal', season, year, type],
    bySlug:          (slug: string) => ['themes', slug],
    versions:        (slug: string) => ['themes', 'versions', slug],
    byArtist:        (slug: string) => ['themes', 'artist', slug],
    forYou:          (userId: string) => ['themes', 'for-you', userId],
    recommendations: (slug: string) => ['themes', 'recommendations', slug],
    random:          () => ['themes', 'random'],
  },
  search:    { results: (q: string, type?: string) => ['search', q, type] },
  anime:     { byId: (id: number) => ['anime', id] },
  artist:    { bySlug: (slug: string) => ['artist', slug] },
  ratings:   { mine: (themeSlug: string) => ['ratings', 'mine', themeSlug] },
  favorites: { list: () => ['favorites'] },
  friends: {
    list:         (userId: string) => ['friends', userId],
    requests:     (userId: string) => ['friends', 'requests', userId],
    requestsSent: (userId: string) => ['friends', 'requests', 'sent', userId],
    status:       (username: string) => ['friends', 'status', username],
    activity:     (userId: string) => ['friends', 'activity', userId],
  },
  follow: {
    status:    (username: string) => ['follow', 'status', username],
    followers: (username: string) => ['follow', 'followers', username],
    following: (username: string) => ['follow', 'following', username],
  },
  notifications: {
    list:        (userId: string) => ['notifications', userId],
    unreadCount: (userId: string) => ['notifications', 'count', userId],
  },
  quiz: {
    question:    (type: string) => ['quiz', 'question', type],
    leaderboard: () => ['quiz', 'leaderboard'],
  },
  profile: { byUsername: (username: string) => ['profile', username] },
  stats:   { live: () => ['stats', 'live'] },
}
```

---

## 10. ENV VARIABLES

```bash
# .env.local
# CRITICAL: The database name /kaikansen MUST appear at the end of the URI.
# Without it, Mongoose defaults to the 'test' database and all seed data goes there silently.
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/kaikansen
JWT_SECRET=<64-char-random>
JWT_REFRESH_SECRET=<64-char-random>
NEXT_PUBLIC_APP_URL=http://localhost:3000
CRON_SECRET=<random>
VOYAGE_API_KEY=<from MongoDB Atlas AI Models panel>

# Vercel dashboard (production)
MONGODB_URI=mongodb+srv://...@.../kaikansen
JWT_SECRET=...
JWT_REFRESH_SECRET=...
NEXT_PUBLIC_APP_URL=https://your-app.vercel.app
CRON_SECRET=...
VOYAGE_API_KEY=...

# Seed only (.env.local — NOT in Vercel)
# Kitsu: no key needed (public API)
# AniList: no key needed (public GraphQL)
```

---

## 11. ERROR HANDLING PATTERNS

### Backend — every route handler:
```typescript
// Standard error response shape
{ success: false, error: string, code: number }

// Always log with context:
console.error('[API] GET /api/themes/[slug]:', err)

// Never expose internal error details to client
// Log full error server-side, return generic message to client
```

### Frontend — toast notifications:
```typescript
// Use sonner for all user-facing errors
import { toast } from 'sonner'

// Success:
toast.success('Rating saved!')

// Error from API:
toast.error(data.error ?? 'Something went wrong. Please try again.')

// Network error:
toast.error('Connection error. Check your internet and try again.')

// Loading state:
const toastId = toast.loading('Saving...')
toast.dismiss(toastId)
toast.success('Saved!')
```

### authFetch error handling:
```typescript
// lib/api/base.ts
export async function authFetch(url: string, options: RequestInit = {}) {
  try {
    const res = await /* ... */
    if (!res.ok) {
      const data = await res.json().catch(() => ({ error: 'Request failed' }))
      throw new Error(data.error ?? `HTTP ${res.status}`)
    }
    return res
  } catch (err) {
    // Re-throw — caller handles with toast
    throw err
  }
}

// Usage in components/hooks:
try {
  await authFetch('/api/ratings', { method: 'POST', body: JSON.stringify(payload) })
  toast.success('Rating saved!')
} catch (err) {
  toast.error(err instanceof Error ? err.message : 'Failed to save rating')
}
```
