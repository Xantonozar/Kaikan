# Kaikansen — VECTOR_SEARCH_V2.md
> MongoDB Atlas Vector Search + Voyage AI Integration.
> **v7 — Integrated into v1 (not post-launch). Mood search. For You. Recommendations. Quiz audio. Surprise Me.**
> This is a reference doc. Implementation steps are in skills.md Phase 15.
> Read knowledge.md and rules.md before this file.

---

## OVERVIEW

Kaikansen uses MongoDB Atlas Vector Search powered by Voyage AI embeddings for:

| Feature | How | API calls at runtime |
|---|---|---|
| Semantic search fallback | Query embedding → vector search | Only if $text returns 0 |
| Mood search ("sad anime op") | Query embedding + mood filter | Always (mood queries skip $text) |
| Similar themes | Stored embeddings only | Zero |
| For You (personalised) | Stored embeddings only | Zero |
| Quiz audio questions | $sample from themes with audioUrl | Zero |
| Surprise Me | $sample random | Zero |

**No ongoing API costs after initial embed script run.**
Voyage AI is an embedding model only — NOT a generative LLM.

---

## MENTAL MODEL

```
SEED TIME (run ONCE):
  scripts/seed-all/* OR scripts/seed-year.ts
  → All ~15,000 themes → MongoDB (kaikansen db)
  → audioUrl per entry = lowest res webm URL (null if no videos)

EMBED TIME (run ONCE after seed):
  scripts/embed.ts
  → Each theme → Voyage AI (voyage-4-large) → 1024-dim vector
  → Stored in ThemeCache.embedding field
  → ~4 hours on free tier (batch=20, 3 RPM limit)
  → Embed text per theme: songTitle + artistName + allArtists + artistRoles
    + animeTitle + animeTitleEnglish + animeTitleAlternative + type as text
    ("anime opening" / "anime ending")
  → mood and genres intentionally excluded:
      mood   = keyword-derived, weak signal
      genres = AniList-only, missing for themes AniList didn't find

LIVE APP (forever, zero Voyage API calls for recommendations):
  Similar themes    = $vectorSearch on stored embeddings
  For You           = average taste vector → $vectorSearch
  Mood search       = query embedding (cached) → $vectorSearch + mood filter
  Semantic fallback = query embedding (cached) → $vectorSearch
  SearchCache       = 30-day TTL prevents repeat API calls for same queries
```

---

## 1. PREREQUISITES

### 1.1 — ThemeCache schema additions

Already included in knowledge.md ThemeCache schema:
```typescript
// These fields are in the schema — do not add them separately
mood:      [{ type: String }]          // ["sad","emotional","epic"]
embedding: { type: [Number], default: null }
// DO NOT add a regular Mongoose index on embedding
// Atlas Vector Search index is created in Atlas UI
```

### 1.2 — SearchCache model

Already in knowledge.md §2. Create at:
`lib/models/SearchCache.model.ts`

30-day TTL on `createdAt` — automatic cleanup of stale query caches.

### 1.3 — Environment variable

```bash
# .env.local
VOYAGE_API_KEY=your_key_from_atlas_ai_models_panel

# How to get the key:
# 1. MongoDB Atlas → your project
# 2. Navigation → "AI Models"
# 3. "Create model API key" → name it → Create
# 4. Copy key — shown only once
```

### 1.4 — Atlas Vector Search Index

Create in Atlas UI BEFORE running embed script:

1. Atlas → cluster → **Atlas Search** tab
2. **Create Search Index** → **Atlas Vector Search** (not regular search)
3. Select collection: `themecaches` (in `kaikansen` database)
4. Index name: `vector_index`
5. JSON definition:

```json
{
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 1024,
      "similarity": "dotProduct"
    },
    {
      "type": "filter",
      "path": "type"
    },
    {
      "type": "filter",
      "path": "mood"
    },
    {
      "type": "filter",
      "path": "animeSeason"
    },
    {
      "type": "filter",
      "path": "animeSeasonYear"
    }
  ]
}
```

6. Click **Create** → wait for status to show **Active** (takes a few minutes)

> **Important:** Index must be Active before embed script runs, otherwise vector inserts succeed but search won't work until index rebuilds.

---

## 2. EMBEDDING HELPER — `lib/embedding.ts`

```typescript
// lib/embedding.ts — SERVER SIDE ONLY
// Never import this in client components

import { connectDB } from '@/lib/db'
import { SearchCache } from '@/lib/models'

const VOYAGE_API_KEY = process.env.VOYAGE_API_KEY!

// Primary: Atlas proxy endpoint (key from Atlas AI Models panel)
// Fallback: Direct Voyage AI endpoint
const VOYAGE_ENDPOINT_PRIMARY  = 'https://ai.mongodb.com/v1/embeddings'
const VOYAGE_ENDPOINT_FALLBACK = 'https://api.voyageai.com/v1/embeddings'

async function callVoyageAPI(
  texts: string[],
  inputType: 'query' | 'document',
  endpoint: string,
): Promise<number[][] | null> {
  try {
    const res = await fetch(endpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${VOYAGE_API_KEY}`,
      },
      body: JSON.stringify({
        input:      texts,
        model:      'voyage-4-large',
        input_type: inputType,
      }),
    })

    if (!res.ok) {
      console.error(`[Voyage] ${endpoint} → HTTP ${res.status}`)
      return null
    }

    const { data } = await res.json()
    return data.map((d: any) => d.embedding)
  } catch (err) {
    console.error(`[Voyage] ${endpoint} error:`, err)
    return null
  }
}

// Get embeddings for multiple texts (used by embed script)
export async function getEmbeddings(
  texts: string[],
  inputType: 'query' | 'document' = 'document',
): Promise<number[][] | null> {
  // Try primary (Atlas proxy) first
  let result = await callVoyageAPI(texts, inputType, VOYAGE_ENDPOINT_PRIMARY)
  if (result) return result

  // Fallback to direct Voyage endpoint
  console.warn('[Voyage] Primary endpoint failed, trying fallback...')
  result = await callVoyageAPI(texts, inputType, VOYAGE_ENDPOINT_FALLBACK)
  return result
}

// Get embedding for a single search query (cached in SearchCache)
export async function getQueryEmbedding(query: string): Promise<number[] | null> {
  await connectDB()
  const normalised = query.toLowerCase().trim()

  // Check SearchCache first — same query never hits API twice
  try {
    const cached = await SearchCache.findOneAndUpdate(
      { query: normalised },
      { $inc: { hitCount: 1 } },
      { new: true },
    ).lean()

    if (cached) {
      return cached.embedding as number[]
    }
  } catch (err) {
    console.error('[Embedding] SearchCache lookup failed:', err)
    // Continue to API call even if cache fails
  }

  // Call Voyage API
  const embeddings = await getEmbeddings([normalised], 'query')
  if (!embeddings || embeddings.length === 0) return null

  const embedding = embeddings[0]

  // Cache for future use (fire and forget — don't block response)
  SearchCache.create({ query: normalised, embedding }).catch(err => {
    console.error('[Embedding] SearchCache save failed:', err)
  })

  return embedding
}

// Average multiple vectors into one — used for taste profiles (For You)
export function averageVectors(vectors: number[][]): number[] {
  if (vectors.length === 0) return []
  const dim = vectors[0].length
  const avg = new Array(dim).fill(0)
  for (const vec of vectors) {
    for (let i = 0; i < dim; i++) avg[i] += vec[i]
  }
  return avg.map(v => v / vectors.length)
}
```

---

## 3. EMBED SCRIPT — `scripts/embed.ts`

```typescript
import 'dotenv/config'
import fs from 'fs'
import path from 'path'
import { connectDB } from '../lib/db'
import { ThemeCache } from '../lib/models'
import { getEmbeddings } from '../lib/embedding'

const PROGRESS_FILE = path.join(__dirname, 'embed-progress.json')
const LOG_FILE      = path.join(__dirname, 'embed.log')
const BATCH_SIZE    = 20      // 20 themes per Voyage API call
const DELAY_MS      = 20_000  // 20s between batches → stay under 3 RPM

interface EmbedProgress {
  processed: number
  errors: number
  startedAt: string
  lastUpdatedAt: string
}

function log(level: string, message: string) {
  const line = `[${new Date().toISOString()}] [${level}] ${message}`
  console.log(line)
  fs.appendFileSync(LOG_FILE, line + '\n')
}

function loadProgress(): EmbedProgress {
  if (fs.existsSync(PROGRESS_FILE)) {
    try { return JSON.parse(fs.readFileSync(PROGRESS_FILE, 'utf-8')) }
    catch { log('WARN', 'Could not parse embed-progress.json — starting fresh') }
  }
  return { processed: 0, errors: 0, startedAt: new Date().toISOString(), lastUpdatedAt: new Date().toISOString() }
}

function saveProgress(p: EmbedProgress) {
  p.lastUpdatedAt = new Date().toISOString()
  fs.writeFileSync(PROGRESS_FILE, JSON.stringify(p, null, 2))
}

const delay = (ms: number) => new Promise(r => setTimeout(r, ms))

async function main() {
  await connectDB()
  log('INFO', '─'.repeat(50))
  log('INFO', 'EMBED SCRIPT STARTING — database: kaikansen')
  log('INFO', '─'.repeat(50))

  const progress = loadProgress()
  log('INFO', `Resuming. Previously processed: ${progress.processed}`)

  // Query for themes without valid embeddings — check both null AND empty array.
  // Mongoose coerces null-default [Number] fields to [] on new documents, so
  // countDocuments({ embedding: null }) alone may undercount un-embedded themes.
  const total = await ThemeCache.countDocuments({
    $or: [{ embedding: null }, { embedding: { $size: 0 } }]
  })
  log('INFO', `Themes without embeddings: ${total}`)

  if (total === 0) {
    log('SUCCESS', 'All themes already have embeddings!')
    process.exit(0)
  }

  let batchNum = 0

  while (true) {
    // Always query for themes missing valid embeddings — includes both null and []
    // ([] marks a previously failed batch — safe to retry)
    const batch = await ThemeCache.find({
      $or: [{ embedding: null }, { embedding: { $size: 0 } }]
    })
      .limit(BATCH_SIZE)
      .lean()

    if (batch.length === 0) {
      log('SUCCESS', 'All themes embedded!')
      break
    }

    batchNum++
    log('INFO', `Batch ${batchNum}: ${batch.length} themes`)

    // Build embed text for each theme
    //
    // Included fields and why:
    //   songTitle             — core identity of the theme
    //   artistName/allArtists — all credited artists
    //   artistRoles           — vocalist/band/composer adds stylistic signal
    //   animeTitle            — romaji title (AT source)
    //   animeTitleEnglish     — English title (AniList source)
    //   animeTitleAlternative — native title + synonyms, so searches like
    //                           "Attack on Titan" hit "Shingeki no Kyojin" themes
    //   type as text          — "anime opening" / "anime ending" so mood searches
    //                           like "sad anime ending" understand the concept
    //
    // Excluded:
    //   mood   — keyword-derived, low quality signal compared to real text
    //   genres — only available from AniList, missing for themes AniList didn't find
    const texts = batch.map(t => {
      const parts = [
        t.songTitle,
        t.artistName,
        ...(t.allArtists ?? []),
        ...(t.artistRoles ?? []),
        t.animeTitle,
        t.animeTitleEnglish,
        ...(t.animeTitleAlternative ?? []),
        t.type === 'OP' ? 'anime opening' : 'anime ending',
      ].filter(Boolean) as string[]
      return parts.join(' ')
    })

    try {
      const embeddings = await getEmbeddings(texts, 'document')

      if (!embeddings || embeddings.length !== batch.length) {
        log('ERROR', `Batch ${batchNum}: Voyage API returned unexpected response — skipping batch`)
        progress.errors += batch.length
        saveProgress(progress)

        // Mark these as failed with empty array to avoid infinite loop
        // They'll have [] not null — can be fixed by clearing and rerunning
        await ThemeCache.bulkWrite(
          batch.map(t => ({
            updateOne: {
              filter: { _id: t._id },
              update: { $set: { embedding: [] } },  // empty array marks as "attempted"
            },
          }))
        )
        continue
      }

      // Bulk update embeddings
      await ThemeCache.bulkWrite(
        batch.map((theme, i) => ({
          updateOne: {
            filter: { _id: theme._id },
            update: { $set: { embedding: embeddings[i] } },
          },
        }))
      )

      progress.processed += batch.length
      saveProgress(progress)

      log('SUCCESS', `Batch ${batchNum} done. Total: ${progress.processed} embedded, ${progress.errors} errors`)

      // Rate limit delay (3 RPM free tier)
      const remaining = await ThemeCache.countDocuments({
        $or: [{ embedding: null }, { embedding: { $size: 0 } }]
      })
      if (remaining > 0) {
        log('INFO', `${remaining} remaining. Waiting ${DELAY_MS / 1000}s...`)
        await delay(DELAY_MS)
      }

    } catch (err) {
      log('ERROR', `Batch ${batchNum} exception: ${err}`)
      progress.errors += batch.length
      saveProgress(progress)
      log('INFO', 'Waiting 30s before retry...')
      await delay(30_000)
    }
  }

  // Final report
  log('INFO', '─'.repeat(50))
  log('INFO', `EMBED COMPLETE`)
  log('INFO', `Total embedded: ${progress.processed}`)
  log('INFO', `Total errors:   ${progress.errors}`)
  log('INFO', 'Verify: db.themecaches.countDocuments({ embedding: null }) should be 0')
  log('INFO', 'Note: themes with embedding:[] had API failures — clear and rerun if needed')
  log('INFO', '─'.repeat(50))
  process.exit(0)
}

main().catch(err => {
  console.error('[FATAL]', err)
  process.exit(1)
})
```

---

## 4. SEARCH — ADVANCED IMPLEMENTATION WITH AI INTEGRATION

```typescript
// app/api/search/route.ts — complete implementation with AI search support

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
    const q        = (searchParams.get('q') ?? '').trim()
    const type     = searchParams.get('type')
    const page     = parseInt(searchParams.get('page') ?? '1')
    const aiSearch = searchParams.get('ai') === 'true'  // AI search toggle

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
    const isMoodQuery    = detectedMoods.length > 0

    // ── AI SEARCH MODE: Skip text search, go directly to vector ───────────────
    if (aiSearch) {
      const embedding = await getQueryEmbedding(q)
      if (!embedding) {
        return NextResponse.json({
          success: true,
          data:    [],
          meta:   { page: 1, total: 0, hasMore: false, searchType: 'ai-failed' },
        })
      }

      const vectorFilter: any = {}
      if (type) vectorFilter.type = { $eq: type }
      if (detectedMoods.length > 0) vectorFilter.mood = { $in: detectedMoods }

      const aiResults = await ThemeCache.aggregate([
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
        data:    aiResults,
        meta:   {
          page:       1,
          total:      aiResults.length,
          hasMore:    false,
          searchType: isMoodQuery ? 'ai-mood' : 'ai',
          moods:      detectedMoods,
        },
      })
    }

    // ── STANDARD MODE: Three-layer search ─────────────────────────────────────
    
    // LAYER 1: $text search (exact, fast, free)
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

      // Apply ranking: Exact match > Title match > Partial match > Related
      const rankedResults = rankSearchResults(textResults, q)
      
      if (rankedResults.length > 0) {
        return NextResponse.json({
          success: true,
          data:    rankedResults,
          meta: {
            page,
            total:   textTotal,
            hasMore: skip + rankedResults.length < textTotal,
            searchType: 'text',
          },
        })
      }
    }

    // LAYER 2 / 3: Vector search (semantic or mood)
    const embedding = await getQueryEmbedding(q)

    if (!embedding) {
      // Voyage API unavailable — return empty gracefully
      return NextResponse.json({
        success: true,
        data:    [],
        meta:   { page: 1, total: 0, hasMore: false, searchType: 'none' },
      })
    }

    // Build vector pre-filters
    const vectorFilter: any = {}
    if (type)                    vectorFilter.type = { $eq: type }
    if (detectedMoods.length > 0) vectorFilter.mood = { $in: detectedMoods }

    // numCandidates must be >> limit to compensate for any post-filtering
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
      meta:   {
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

// ── RANKING LOGIC ─────────────────────────────────────────────────────────────
// Exact match > Title match > Partial match > Related match

function rankSearchResults(results: any[], query: string): any[] {
  const qLower = query.toLowerCase().trim()
  
  return results
    .map(theme => {
      let rankScore = 0
      
      // Exact match (song title = query or anime title = query)
      if (theme.songTitle?.toLowerCase() === qLower || 
          theme.animeTitle?.toLowerCase() === qLower ||
          theme.animeTitleEnglish?.toLowerCase() === qLower) {
        rankScore = 1000
      }
      // Title match (query is contained in title)
      else if (theme.songTitle?.toLowerCase().includes(qLower) ||
               theme.animeTitle?.toLowerCase().includes(qLower) ||
               theme.animeTitleEnglish?.toLowerCase().includes(qLower)) {
        rankScore = 500
      }
      // Partial match (title contains any word from query)
      else {
        const queryWords = qLower.split(' ')
        const hasPartialMatch = queryWords.some(word => 
          word.length >= 2 && (
            theme.songTitle?.toLowerCase().includes(word) ||
            theme.animeTitle?.toLowerCase().includes(word) ||
            theme.animeTitleEnglish?.toLowerCase().includes(word) ||
            theme.allArtists?.some((a: string) => a.toLowerCase().includes(word))
          )
        )
        rankScore = hasPartialMatch ? 200 : 50
      }
      
      // Bonus for exact match on artist
      if (theme.artistName?.toLowerCase() === qLower) {
        rankScore += 100
      }
      
      // Include original text score for final sort
      return { ...theme, rankScore }
    })
    .sort((a, b) => {
      // Primary: rankScore descending
      if (b.rankScore !== a.rankScore) return b.rankScore - a.rankScore
      // Secondary: textScore descending (from MongoDB $text)
      return (b.score?.$meta?.textScore ?? 0) - (a.score?.$meta?.textScore ?? 0)
    })
}
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

    // numCandidates must be >> limit to compensate for any post-filtering
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

### UI Feedback for Search Types

```tsx
// app/components/search/SearchClient.tsx — search type banners
{meta?.searchType === 'semantic' && (
  <div className="flex items-center gap-2 px-3 py-2 bg-accent-container rounded-[12px] mb-3">
    <Sparkles className="w-4 h-4 text-accent flex-shrink-0" />
    <p className="text-xs font-body text-accent">
      No exact matches — showing semantically related results
    </p>
  </div>
)}

{meta?.searchType === 'mood' && (
  <div className="flex items-center gap-2 px-3 py-2 bg-accent-container rounded-[12px] mb-3">
    <Music className="w-4 h-4 text-accent flex-shrink-0" />
    <p className="text-xs font-body text-accent">
      Showing themes matching the mood: {meta.moods?.join(', ')}
    </p>
  </div>
)}

{meta?.searchType === 'none' && query.length >= 2 && (
  <EmptyState type="NoResults" message="No results found. Try different keywords." />
)}
```

---

## 5. THEME RECOMMENDATIONS

```typescript
// app/api/themes/[slug]/recommendations/route.ts

import { NextRequest, NextResponse } from 'next/server'
import { connectDB } from '@/lib/db'
import { ThemeCache } from '@/lib/models'

export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ slug: string }> }
) {
  try {
    await connectDB()
    const { slug } = await params  // MUST await params

    const theme = await ThemeCache.findOne({ slug }).lean()

    if (!theme) {
      return NextResponse.json(
        { success: false, error: 'Theme not found', code: 404 },
        { status: 404 }
      )
    }

    // No embedding yet — return empty (embed script hasn't run)
    if (!theme.embedding || theme.embedding.length === 0) {
      return NextResponse.json({ success: true, data: [] })
    }

    // Similar themes — zero API calls, pure stored vector math
    // Prefer same type (OP → OP, ED → ED) but don't require it
    // numCandidates=100 >> limit=15 ensures post-$match exclusion never starves results
    // (the current theme is always rank #1 for its own embedding, so we need limit > 6)
    const similar = await ThemeCache.aggregate([
      {
        $vectorSearch: {
          index:         'vector_index',
          path:          'embedding',
          queryVector:   theme.embedding,
          numCandidates: 100,
          limit:         15,         // buffer: lose 1 to self-exclusion, still have 14
          filter: {
            type: { $eq: theme.type },  // same type (OP or ED)
          },
        },
      },
      {
        $match: {
          slug: { $ne: slug },  // exclude current theme
        },
      },
      {
        $addFields: { vectorScore: { $meta: 'vectorSearchScore' } },
      },
      { $limit: 6 },
    ])

    // If not enough same-type results, fill with any type
    if (similar.length < 3) {
      const anyType = await ThemeCache.aggregate([
        {
          $vectorSearch: {
            index:         'vector_index',
            path:          'embedding',
            queryVector:   theme.embedding,
            numCandidates: 100,
            limit:         15,
          },
        },
        {
          $match: { slug: { $ne: slug } },
        },
        { $limit: 6 },
      ])
      return NextResponse.json({ success: true, data: anyType })
    }

    return NextResponse.json({ success: true, data: similar })

  } catch (err) {
    console.error('[API] GET /api/themes/[slug]/recommendations:', err)
    return NextResponse.json(
      { success: false, error: 'Could not load recommendations', code: 500 },
      { status: 500 }
    )
  }
}
```

---

## 6. FOR YOU — PERSONALISED HOME PAGE

```typescript
// app/api/themes/for-you/route.ts

import { NextRequest, NextResponse } from 'next/server'
import { connectDB } from '@/lib/db'
import { ThemeCache, Rating, WatchHistory } from '@/lib/models'
import { verifyAccessToken } from '@/lib/auth'
import { averageVectors } from '@/lib/embedding'

export async function GET(req: NextRequest) {
  try {
    await connectDB()

    const token   = req.headers.get('authorization')?.replace('Bearer ', '')
    const payload = token ? verifyAccessToken(token) : null
    if (!payload) {
      return NextResponse.json(
        { success: false, error: 'Unauthorized', code: 401 },
        { status: 401 }
      )
    }

    // Get user's highest rated themes (score 7+)
    const topRatings = await Rating.find({
      userId: payload.userId,
      score:  { $gte: 7 },
    }).lean()

    if (topRatings.length < 3) {
      return NextResponse.json({
        success: true,
        data:    [],
        meta:    { reason: 'not_enough_ratings', needed: 3, have: topRatings.length },
      })
    }

    // Get embeddings of those rated themes
    const themeIds    = topRatings.map(r => r.themeId)
    const ratedThemes = await ThemeCache.find(
      { _id: { $in: themeIds }, embedding: { $exists: true, $ne: null, $not: { $size: 0 } } }
    ).select('embedding').lean()

    if (ratedThemes.length === 0) {
      // Embeddings not generated yet
      return NextResponse.json({
        success: true,
        data:    [],
        meta:    { reason: 'embeddings_pending' },
      })
    }

    // Build taste vector — average of all highly rated embeddings
    const tasteVector = averageVectors(
      ratedThemes.map(t => t.embedding as number[])
    )

    // Collect slugs user has already interacted with
    const [watchedSlugs, ratedSlugs] = await Promise.all([
      WatchHistory.distinct('themeSlug', { userId: payload.userId }),
      Rating.distinct('themeSlug', { userId: payload.userId }),
    ])
    const excludeSlugs = [...new Set([...watchedSlugs, ...ratedSlugs])]

    // Vector search with large numCandidates to compensate for $match exclusions
    // Rule: numCandidates >> limit to ensure we get enough after filtering
    const candidates = await ThemeCache.aggregate([
      {
        $vectorSearch: {
          index:         'vector_index',
          path:          'embedding',
          queryVector:   tasteVector,
          numCandidates: 300,   // fetch many more than needed
          limit:         150,   // take 150 from vector search
        },
      },
      {
        $match: {
          slug: { $nin: excludeSlugs },  // exclude already seen
        },
      },
      {
        $addFields: { vectorScore: { $meta: 'vectorSearchScore' } },
      },
      { $limit: 10 },  // trim to final 10
    ])

    return NextResponse.json({
      success: true,
      data:    candidates,
      meta:    { reason: 'personalised', basedOnRatings: topRatings.length },
    })

  } catch (err) {
    console.error('[API] GET /api/themes/for-you:', err)
    return NextResponse.json(
      { success: false, error: 'Could not load recommendations', code: 500 },
      { status: 500 }
    )
  }
}
```

### UI — For You Section

```tsx
// In HomeClient.tsx — For You section
const { data: forYou, isLoading: forYouLoading } = useQuery({
  queryKey: queryKeys.themes.forYou(user?.id ?? ''),
  queryFn:  () => authFetch('/api/themes/for-you').then(r => r.json()),
  enabled:  !!user,
  staleTime: 5 * 60 * 1000,
})

{user && !forYouLoading && (
  <section>
    {forYou?.data?.length > 0 ? (
      <>
        <div className="flex items-center justify-between mb-3">
          <div>
            <p className="text-xs font-body font-semibold text-accent uppercase tracking-wide">
              Based on your ratings
            </p>
            <h2 className="text-xl font-display font-bold text-ktext-primary">For You</h2>
          </div>
        </div>
        <div className="space-y-2">
          {forYou.data.map(theme => <ThemeListRow key={theme.slug} {...theme} />)}
        </div>
      </>
    ) : forYou?.meta?.reason === 'not_enough_ratings' ? (
      <div className="bg-bg-surface rounded-[16px] border border-border-subtle p-4">
        <p className="text-sm font-body text-ktext-secondary text-center">
          Rate {3 - (forYou.meta.have ?? 0)} more themes to unlock personalised recommendations
        </p>
      </div>
    ) : null}
  </section>
)}
```

---

## 7. SURPRISE ME

```typescript
// app/api/themes/random/route.ts

import { NextRequest, NextResponse } from 'next/server'
import { connectDB } from '@/lib/db'
import { ThemeCache } from '@/lib/models'

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

    // $sample is the correct MongoDB way — never Math.random() on JS arrays
    const [theme] = await ThemeCache.aggregate([
      { $match: filter },
      { $sample: { size: 1 } },
    ])

    if (!theme) {
      return NextResponse.json(
        { success: false, error: 'No themes found matching filters', code: 404 },
        { status: 404 }
      )
    }

    return NextResponse.json({ success: true, data: theme })

  } catch (err) {
    console.error('[API] GET /api/themes/random:', err)
    return NextResponse.json(
      { success: false, error: 'Could not get random theme', code: 500 },
      { status: 500 }
    )
  }
}
```

### SurpriseButton Component

```tsx
// app/components/shared/SurpriseButton.tsx ("use client")
'use client'
import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { Shuffle, Loader2 } from 'lucide-react'
import { toast } from 'sonner'

export function SurpriseButton() {
  const [loading, setLoading] = useState(false)
  const router = useRouter()

  const handleSurprise = async () => {
    setLoading(true)
    const toastId = toast.loading('Finding something for you...')
    try {
      const res  = await fetch('/api/themes/random')
      const data = await res.json()
      if (!data.success) throw new Error(data.error)
      toast.dismiss(toastId)
      router.push(`/theme/${data.data.slug}?autoplay=true`)
    } catch (err) {
      toast.dismiss(toastId)
      toast.error('Could not find a theme. Try again!')
    } finally {
      setLoading(false)
    }
  }

  return (
    <button
      onClick={handleSurprise}
      disabled={loading}
      className="flex items-center gap-2 h-12 px-6 rounded-full
                 bg-accent-container border border-border-accent
                 text-accent font-body font-semibold text-sm
                 transition-all duration-150 interactive
                 disabled:opacity-50 disabled:cursor-not-allowed"
    >
      {/* Use Loader2 during loading — Shuffle has asymmetric arms that look broken spinning */}
      {loading
        ? <Loader2 className="w-4 h-4 animate-spin" />
        : <Shuffle className="w-4 h-4" />
      }
      Surprise Me
    </button>
  )
}
```

---

## 8. QUIZ AUDIO ROUTES

See knowledge.md §7 for full quiz route implementations.

Key constraints for quiz:
- `GET /api/quiz/question` — filter: `{ audioUrl: { $ne: null } }` — never return null audio
- Response must NOT include `songTitle`, `artistName`, `animeTitle` — these are the answers
- Response includes: `audioUrl`, `options[]`, `quizType`, `themeSlug` (for answer submission)
- `POST /api/quiz/answer` — validates guess, returns `{ correct, correctAnswer, score }`
- Score: `correct ? Math.max(100, 1000 - Math.floor(timeTaken / 30)) : 0`

---

## 9. RATE LIMIT SUMMARY

| Operation | Rate Limited | Free Limit | Notes |
|---|---|---|---|
| Voyage embed API (embed script) | ✅ Yes | 3 RPM | Batch 20 = ~4hrs total |
| Voyage embed API (search fallback) | ✅ Yes | 3 RPM | Cached in SearchCache |
| MongoDB Vector Search | ❌ No | Unlimited | Just a DB query |
| Theme recommendations | ❌ No | Unlimited | Stored embeddings |
| For You | ❌ No | Unlimited | Stored embeddings |
| Surprise Me | ❌ No | Unlimited | $sample |
| Quiz audio | ❌ No | Unlimited | $sample |
| AniList (seed only) | ⚠️ Soft | ~90/min | 1000ms delay in seed |
| Kitsu (seed only) | ❌ No | Generous | 500ms delay in seed |

---

## 10. IMPLEMENTATION CHECKLIST

```
PREREQUISITES:
[ ] MongoDB Atlas cluster running, kaikansen database created
[ ] VOYAGE_API_KEY obtained from Atlas → AI Models panel
[ ] Added VOYAGE_API_KEY to .env.local
[ ] Atlas Vector Search index 'vector_index' created and Active
[ ] embedding field in ThemeCache schema (no Mongoose index on it)
[ ] SearchCache model created and exported from lib/models/index.ts
[ ] QuizAttempt model created and exported

SEED:
[ ] scripts/seed-utils.ts created
[ ] scripts/seed-shared.ts created
[ ] All 5 part files created (seed-all/)
[ ] scripts/seed-year.ts created
[ ] All progress files gitignored
[ ] npm run seed:all completed (or seed:year for testing)
[ ] Verified: db.themecaches.countDocuments() ~15,000+
[ ] Checked: db.themecaches.countDocuments({ audioUrl: null }) (note the number)

EMBED:
[ ] scripts/embed.ts created
[ ] npm run embed completed
[ ] Verified: db.themecaches.countDocuments({ embedding: null }) === 0
[ ] Verified: db.themecaches.countDocuments({ embedding: { $size: 0 } }) === 0
    (Mongoose coerces null-default array fields → []. Check BOTH. embedding:[] = batch failed.)
    (Fix: db.themecaches.updateMany({ embedding:[] }, { $set:{ embedding:null } }) then rerun embed)

ROUTES:
[ ] lib/embedding.ts created with primary + fallback endpoint
[ ] app/api/search/route.ts updated with 3-layer strategy
[ ] app/api/themes/[slug]/recommendations/route.ts created
[ ] app/api/themes/for-you/route.ts created
[ ] app/api/themes/random/route.ts created
[ ] app/api/quiz/question/route.ts created
[ ] app/api/quiz/answer/route.ts created
[ ] app/api/quiz/leaderboard/route.ts created
[ ] All new routes added to proxy.ts protected list where needed
[ ] queryKeys.ts updated with new keys

UI:
[ ] SearchClient shows searchType banners (text/semantic/mood/none)
[ ] HomeClient has For You section with not_enough_ratings placeholder
[ ] SurpriseButton component created
[ ] ThemePageClient has VersionSwitcher
[ ] ThemePageClient has recommendations section
[ ] QuizClient created with full quiz flow
[ ] Listen mode disabled if audioUrl === null

TESTING:
[ ] Search "gurenge" → text results
[ ] Search "sad anime opening" → mood results with mood banner
[ ] Search "something obscure" → semantic results with semantic banner
[ ] Rate 3+ themes 7+ → For You section appears
[ ] Surprise Me → random theme plays
[ ] Quiz → audio plays, no title/artist visible, answer submits
[ ] Version switcher shows on themes with multiple entries
```

---

## 11. NOTES

- **voyage-4-large** — 1024 dimensions, best accuracy, free within 200M tokens
- **200M token budget**: 15,000 themes × ~30 tokens avg = ~450,000 tokens for full embed. Well within limit.
- **Empty array `[]` vs null**: embed.ts marks failed embeddings as `[]` to distinguish "tried but failed" from "not yet attempted". Clean up with: `db.themecaches.updateMany({ embedding: [] }, { $set: { embedding: null } })` then rerun embed.
- **SearchCache TTL**: 30 days. Same query never hits Voyage API twice within that window.
- **averageVectors minimum**: For You uses minimum 3 ratings. With 1-2, the taste vector is too narrow to be useful.
- **dotProduct vs cosine**: dotProduct is used because Voyage AI normalises vectors, making dotProduct equivalent to cosine similarity but faster.
- **$vectorSearch must be pipeline stage 1**: Never put $match or other stages before $vectorSearch. It is always the first stage.
- **numCandidates rule**: Always set numCandidates >> limit. For post-$match filtering, set numCandidates at 10-30x the final desired count.
