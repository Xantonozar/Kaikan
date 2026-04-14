# Kaikansen — design.md
> Complete design system. Mobile-first. Light + Dark mode.
> **v7 — Tidal UI. Pure Next.js 16. Vercel. Seed Script. No Express. Manual JWT.**
> Reference this before building any UI component or page.

---

## 1. VISUAL IDENTITY

**Concept:** "Ethereal Tide" — flowing teal and navy inspired by bioluminescent ocean depths.
The app should feel like a premium music platform at the intersection of anime culture and modern design.

**Mood:** Clean, immersive, slightly editorial. Premium but not corporate. Warm but not playful.

**Design Language:** Material You (M3) 2026 — adaptive layouts, tonal elevation, container-based composition, expressive state layers.

**Logo Mark:** `≋` wave character + "Kaikansen" wordmark in display font.

**Primary Content:** Anime OP/EDs — the hero of every screen is a ThemeListRow or ThemeCard, not an anime poster.

---

## 2. COLOR SYSTEM

### CSS Custom Properties (globals.css)

```css
/* ── LIGHT MODE (default) ── */
:root {
  /* Backgrounds */
  --bg-base:     #E8F4F7;
  --bg-surface:  #FFFFFF;
  --bg-elevated: #F0F8FA;
  --bg-overlay:  #E2EFF3;
  --bg-toast:    #D4E8ED;
  --bg-header:   #FFFFFF;

  /* Primary Accent — Dark Teal */
  --accent:              #0A8A96;
  --accent-hover:        #0C9DA8;
  --accent-pressed:      #086E78;
  --accent-subtle:       rgba(10, 138, 150, 0.10);
  --accent-subtle-hover: rgba(10, 138, 150, 0.18);
  --accent-container:    rgba(10, 138, 150, 0.12);
  --on-accent-container: #065962;
  --accent-glow:         rgba(10, 138, 150, 0.20);

  /* Secondary Accent — Mint Teal */
  --accent-mint:           #4ECDC4;
  --accent-mint-hover:     #60D5CD;
  --accent-mint-container: rgba(78, 205, 196, 0.15);
  --on-accent-mint:        #063E3A;

  /* ED Accent — Warm Peach */
  --accent-ed:           #F0A05A;
  --accent-ed-subtle:    rgba(240, 160, 90, 0.12);
  --accent-ed-container: rgba(240, 160, 90, 0.15);
  --on-accent-ed:        #5C3200;

  /* Text */
  --text-primary:   #0D1B2A;
  --text-secondary: #4A7A85;
  --text-tertiary:  #8FAAB0;
  --text-disabled:  #B5CDD2;
  --text-on-accent: #FFFFFF;

  /* Borders */
  --border-subtle:  rgba(10, 138, 150, 0.08);
  --border-default: rgba(10, 138, 150, 0.15);
  --border-strong:  rgba(10, 138, 150, 0.30);
  --border-accent:  rgba(10, 138, 150, 0.50);

  /* Semantic */
  --success:  #16A34A;
  --warning:  #D97706;
  --error:    #DC2626;
  --info:     #0284C7;

  /* Special */
  --logout-bg: #0A8A96;  /* Teal in light mode */
}

/* ── DARK MODE ── */
/* next-themes uses attribute="data-theme" — it never applies a .dark class to <html>  */
/* Use ONLY [data-theme="dark"] here. The .dark selector is dead code with next-themes. */
[data-theme="dark"] {
  /* Backgrounds */
  --bg-base:     #070F18;
  --bg-surface:  #0C1C28;
  --bg-elevated: #112233;
  --bg-overlay:  #162840;
  --bg-toast:    #1A3050;
  --bg-header:   #0A1828;

  /* Primary Accent — Mint Teal (brighter in dark) */
  --accent:              #4ECDC4;
  --accent-hover:        #60D5CD;
  --accent-pressed:      #3EBDB5;
  --accent-subtle:       rgba(78, 205, 196, 0.10);
  --accent-subtle-hover: rgba(78, 205, 196, 0.18);
  --accent-container:    rgba(78, 205, 196, 0.15);
  --on-accent-container: #4ECDC4;
  --accent-glow:         rgba(78, 205, 196, 0.25);

  /* Secondary Accent — Mint same as primary in dark */
  --accent-mint:           #4ECDC4;
  --accent-mint-hover:     #60D5CD;
  --accent-mint-container: rgba(78, 205, 196, 0.15);
  --on-accent-mint:        #063E3A;

  /* ED Accent */
  --accent-ed:           #F0A05A;
  --accent-ed-subtle:    rgba(240, 160, 90, 0.12);
  --accent-ed-container: rgba(240, 160, 90, 0.15);
  --on-accent-ed:        #F0A05A;

  /* Text */
  --text-primary:   #FFFFFF;
  --text-secondary: #7AAAB8;
  --text-tertiary:  #4A6878;
  --text-disabled:  #2A4858;
  --text-on-accent: #063E3A;

  /* Borders */
  --border-subtle:  rgba(78, 205, 196, 0.06);
  --border-default: rgba(78, 205, 196, 0.12);
  --border-strong:  rgba(78, 205, 196, 0.25);
  --border-accent:  rgba(78, 205, 196, 0.45);

  /* Semantic */
  --success:  #22C55E;
  --warning:  #F59E0B;
  --error:    #F87171;
  --info:     #38BDF8;

  /* Special */
  --logout-bg: #8B1A1A;  /* Crimson in dark mode */
}
```

### Tailwind Config Extension

```typescript
// tailwind.config.ts
const config: Config = {
  darkMode: ['class', '[data-theme="dark"]'],
  // IMPORTANT: no src/ directory — project uses --no-src-dir
  content: [
    './app/**/*.{ts,tsx}',
    './lib/**/*.{ts,tsx}',
    './hooks/**/*.{ts,tsx}',
    './providers/**/*.{ts,tsx}',
    './types/**/*.{ts,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        bg: {
          base:     'var(--bg-base)',
          surface:  'var(--bg-surface)',
          elevated: 'var(--bg-elevated)',
          overlay:  'var(--bg-overlay)',
          toast:    'var(--bg-toast)',
          header:   'var(--bg-header)',
        },
        accent: {
          DEFAULT:        'var(--accent)',
          hover:          'var(--accent-hover)',
          pressed:        'var(--accent-pressed)',
          subtle:         'var(--accent-subtle)',
          'subtle-hover': 'var(--accent-subtle-hover)',
          container:      'var(--accent-container)',
          on:             'var(--on-accent-container)',
          glow:           'var(--accent-glow)',
          mint:           'var(--accent-mint)',
          'mint-hover':   'var(--accent-mint-hover)',
          'mint-container': 'var(--accent-mint-container)',
          'on-mint':      'var(--on-accent-mint)',
          ed:             'var(--accent-ed)',
          'ed-subtle':    'var(--accent-ed-subtle)',
          'ed-container': 'var(--accent-ed-container)',
          'on-ed':        'var(--on-accent-ed)',
        },
        ktext: {
          primary:   'var(--text-primary)',
          secondary: 'var(--text-secondary)',
          tertiary:  'var(--text-tertiary)',
          disabled:  'var(--text-disabled)',
          'on-accent': 'var(--text-on-accent)',
        },
        border: {
          subtle:  'var(--border-subtle)',
          default: 'var(--border-default)',
          strong:  'var(--border-strong)',
          accent:  'var(--border-accent)',
        },
        semantic: {
          success: 'var(--success)',
          warning: 'var(--warning)',
          error:   'var(--error)',
          info:    'var(--info)',
        }
      },
      fontFamily: {
        display: ['Outfit', 'sans-serif'],
        body:    ['Inter', 'sans-serif'],
        mono:    ['JetBrains Mono', 'monospace'],
      },
      borderRadius: {
        'sm':      '8px',
        DEFAULT:   '12px',
        'md':      '16px',
        'card':    '20px',
        'card-lg': '24px',
        'pill':    '9999px',
        'input':   '12px',
        'btn':     '12px',
        'btn-lg':  '9999px',
        // NOTE: dark mode header bottom rounding (rounded-b-[24px]) is applied
        // via direct className in AppHeader.tsx — NOT as a Tailwind config token
      },
      boxShadow: {
        'none':        'none',
        'card':        '0 2px 12px rgba(10, 138, 150, 0.08)',
        'card-hover':  '0 8px 32px rgba(10, 138, 150, 0.15)',
        'modal':       '0 24px 80px rgba(0,0,0,0.25)',
        'accent-glow': '0 0 32px var(--accent-glow)',
        'avatar-glow': '0 0 0 3px var(--accent-mint)',
      }
    }
  }
}
```

---

## 3. TYPOGRAPHY SYSTEM

### Google Fonts Setup (Next.js 16 — use next/font/google, NOT raw link tags)

```typescript
// app/layout.tsx
import { Outfit, Inter, JetBrains_Mono } from 'next/font/google'

const outfit = Outfit({
  subsets: ['latin'],
  weight: ['600', '700', '800'],
  variable: '--font-outfit',
  display: 'swap',
})

const inter = Inter({
  subsets: ['latin'],
  weight: ['400', '500', '600'],
  variable: '--font-inter',
  display: 'swap',
})

const jetbrainsMono = JetBrains_Mono({
  subsets: ['latin'],
  weight: ['700'],
  variable: '--font-jetbrains',
  display: 'swap',
})

// Apply to <html> in layout.tsx:
// <html className={`${outfit.variable} ${inter.variable} ${jetbrainsMono.variable}`}>
```

> **Do NOT use raw `<link>` tags for fonts.** `next/font/google` self-hosts fonts, avoids
> layout shift, and is the correct approach for Next.js 16.

### Type Scale

| Role | Class | Size | Weight | Use |
|---|---|---|---|---|
| Display Large | `text-4xl font-display font-extrabold tracking-tight` | 36px | 800 | Hero moments only |
| Display Medium | `text-3xl font-display font-bold tracking-tight` | 30px | 700 | Page titles (desktop) |
| Display Small | `text-2xl font-display font-bold` | 24px | 700 | Page titles (mobile), song titles |
| Headline | `text-xl font-display font-semibold` | 20px | 600 | Section titles |
| Title | `text-base font-body font-semibold` | 16px | 600 | Card titles, tab labels |
| Body | `text-sm font-body font-normal leading-relaxed` | 14px | 400 | Descriptions, meta |
| Label | `text-xs font-body font-medium tracking-wide` | 12px | 500 | Badges, chips, timestamps |
| Score | `text-2xl font-mono font-bold tabular-nums` | 24px | 700 | Large score display |
| Score SM | `text-sm font-mono font-bold tabular-nums` | 14px | 700 | Inline scores |

**Section labels** (TODAY, YOUR CIRCLE, etc.):
```tsx
<span className="text-xs font-body font-semibold tracking-[0.1em] uppercase text-ktext-tertiary">
  TODAY
</span>
```

---

## 4. ELEVATION & SURFACE SYSTEM

### Light Mode Surfaces

| Level | Usage | bg Token |
|---|---|---|
| 0 | Page canvas | `bg-bg-base` |
| 1 | Cards, panels, nav | `bg-bg-surface` (white) |
| 2 | Hovered cards, inputs | `bg-bg-elevated` |
| 3 | Modals, drawers | `bg-bg-overlay` |
| 4 | Toasts, tooltips | `bg-bg-toast` |

Light mode uses `shadow-card` on Level 1 cards instead of tonal tint.

### Dark Mode Surfaces

| Level | Usage | bg Token |
|---|---|---|
| 0 | Page canvas | `bg-bg-base` (`#070F18`) |
| 1 | Cards, panels | `bg-bg-surface` (`#0C1C28`) |
| 2 | Hover, inputs | `bg-bg-elevated` (`#112233`) |
| 3 | Modals | `bg-bg-overlay` (`#162840`) |

Dark mode uses border instead of shadow: `border border-border-subtle`.

### Dark Mode Header (Unique)
```tsx
// Apply rounded-b-[24px] directly in AppHeader.tsx via className — not via Tailwind config
<header className="bg-bg-header rounded-b-[24px] px-4 py-3 flex items-center justify-between">
  {/* Rounded bottom corners only in dark mode */}
</header>
```

---

## 5. STATE LAYERS (Interactive surfaces)

```css
/* globals.css */
.interactive {
  position: relative;
  overflow: hidden;
  cursor: pointer;
}

.interactive::before {
  content: '';
  position: absolute;
  inset: 0;
  background: var(--accent);
  opacity: 0;
  border-radius: inherit;
  transition: opacity 150ms cubic-bezier(0.2, 0, 0, 1);
  pointer-events: none;
  z-index: 0;
}

.interactive:hover::before        { opacity: 0.06; }
.interactive:active::before       { opacity: 0.12; }
.interactive:focus-visible::before { opacity: 0.08; }

.interactive:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
}
```

Apply `.interactive` to: ThemeCards, list rows, nav items, buttons, friend cards, all clickable surfaces.

---

## 6. CANONICAL LAYOUTS

### Breakpoints
- **Compact** (< 600px — Mobile): Single pane, Bottom Nav
- **Medium** (600–1240px — Tablet): Navigation Rail (80px)
- **Expanded** (> 1240px — Desktop): Navigation Rail (240px) + two-pane

### Page Wrapper
```tsx
<div className="min-h-screen bg-bg-base flex">
  <NavigationRail className="hidden md:flex" />
  <main className="
    flex-1 min-w-0
    pb-20 md:pb-0
    md:pl-20 lg:pl-60
    px-4 md:px-6 lg:px-8
  ">
    <div className="max-w-2xl mx-auto md:max-w-7xl">
      {children}
    </div>
  </main>
  <BottomNav className="flex md:hidden" />
</div>
```

---

## 7. BORDER RADIUS SYSTEM

| Element | Value | Class |
|---|---|---|
| Page cards, ThemeCard | 20px | `rounded-[20px]` |
| Modals, large drawers | 24px | `rounded-[24px]` |
| Dark mode header (bottom only) | 24px | `rounded-b-[24px]` (direct className) |
| List row cards | 16px | `rounded-[16px]` |
| Inputs | 12px | `rounded-[12px]` |
| Standard buttons | 12px | `rounded-[12px]` |
| Primary CTA, nav pills, badges | Full | `rounded-full` |
| Avatar | Full circle | `rounded-full` |
| Video player | 20px | `rounded-[20px]` |
| Featured card | 20px | `rounded-[20px]` |
| Stats box | 16px | `rounded-[16px]` |
| Setting section card | 20px | `rounded-[20px]` |

---

## 8. COMPONENT SPECIFICATIONS

### 8.1 App Header

**Light Mode:**
```tsx
<header className="sticky top-0 z-40 bg-bg-surface border-b border-border-subtle px-4 h-14 flex items-center justify-between">
  <button className="interactive rounded-full p-2">
    <Menu className="w-5 h-5 text-ktext-secondary" />
  </button>
  <div className="flex items-center gap-2">
    <span className="text-accent text-xl">≋</span>
    <span className="font-display font-bold text-lg text-ktext-primary">Kaikansen</span>
  </div>
  <div className="flex items-center gap-2">
    <button className="interactive rounded-full p-2">
      <Search className="w-5 h-5 text-ktext-secondary" />
    </button>
    <div className="w-9 h-9 rounded-full overflow-hidden border-2 border-accent-mint bg-bg-elevated">
      <img src={avatarUrl} className="w-full h-full object-cover" />
    </div>
  </div>
</header>
```

**Dark Mode Header (rounded bottom — applied via className, not Tailwind config):**
```tsx
// AppHeader.tsx — use useTheme().isDark to conditionally apply rounding
"use client"
export function AppHeader() {
  const { isDark } = useTheme()
  return (
    <header className={`
      sticky top-0 z-40 h-14 px-4 flex items-center justify-between
      bg-bg-header
      ${isDark ? 'rounded-b-[24px] shadow-md' : 'border-b border-border-subtle'}
    `}>
      {/* content */}
    </header>
  )
}
```

---

### 8.2 Bottom Navigation

```tsx
<nav className="
  flex md:hidden fixed bottom-0 left-0 right-0 z-40
  bg-bg-surface border-t border-border-subtle
  h-16 pb-[env(safe-area-inset-bottom)]
">
  {navItems.map(item => (
    <NavLink key={item.path} to={item.path} className={({ isActive }) => `
      flex-1 flex flex-col items-center justify-center gap-1 relative interactive
      ${isActive ? 'text-accent' : 'text-ktext-tertiary'}
    `}>
      {isActive && (
        <span className="absolute inset-x-auto w-14 h-9 rounded-full bg-accent-container" />
      )}
      <item.Icon className="w-6 h-6 relative z-10" />
      {item.badge > 0 && (
        <span className="absolute top-2 right-1/4 min-w-[16px] h-4
                         bg-error text-white text-[10px] font-mono font-bold
                         rounded-full flex items-center justify-center px-1">
          {item.badge > 9 ? '9+' : item.badge}
        </span>
      )}
    </NavLink>
  ))}
</nav>
```

Bottom nav items (5): Home (drop/wave icon) · Search · Quiz · Notifications (bell) · Profile

---

### 8.3 Navigation Rail (Tablet/Desktop)

```tsx
<nav className="hidden md:flex flex-col fixed left-0 top-0 bottom-0 w-20 lg:w-60
                bg-bg-surface border-r border-border-subtle z-40 py-4">
  <div className="flex items-center gap-3 px-4 mb-8">
    <span className="text-accent text-2xl">≋</span>
    <span className="hidden lg:block font-display font-bold text-lg text-ktext-primary">Kaikansen</span>
  </div>
  {navItems.map(item => (
    <NavLink to={item.path} className={({ isActive }) => `
      flex items-center gap-3 mx-2 px-3 py-3 rounded-full
      transition-colors duration-150 interactive
      ${isActive ? 'bg-accent-container text-accent' : 'text-ktext-secondary hover:text-ktext-primary'}
    `}>
      <item.Icon className="w-6 h-6 flex-shrink-0" />
      <span className="hidden lg:block text-sm font-body font-medium">{item.label}</span>
    </NavLink>
  ))}
  <div className="mt-auto px-4">
    <Avatar className="w-10 h-10" />
  </div>
</nav>
```

---

### 8.4 ThemeFeaturedCard (Home — Large Landscape)

```tsx
<div className="
  flex-shrink-0 relative overflow-hidden rounded-[20px]
  w-[75vw] md:w-72 aspect-video
  interactive cursor-pointer
  bg-bg-surface shadow-card
">
  <img src={animeCoverImage ?? atCoverImage} className="w-full h-full object-cover" />
  <div className="absolute inset-0 bg-gradient-to-t from-black/70 via-black/20 to-transparent" />
  <div className="absolute top-3 left-3 flex items-center gap-1
                  bg-black/50 backdrop-blur-sm rounded-full px-2 py-1">
    <Star className="w-3 h-3 text-yellow-400 fill-yellow-400" />
    <span className="text-xs font-mono font-bold text-white">{avgRating.toFixed(1)}</span>
  </div>
  <div className="absolute bottom-0 left-0 right-0 p-3">
    <p className="text-sm font-display font-bold text-white truncate">{animeTitle}</p>
    <p className="text-xs font-body text-white/70 truncate">{studio}</p>
  </div>
</div>
```

---

### 8.5 ThemeListRow (Popular, Search, Friends Activity)

```tsx
<div className="
  flex items-center gap-3 p-3
  bg-bg-surface rounded-[16px]
  border border-border-subtle
  shadow-card interactive cursor-pointer
  transition-all duration-200 hover:shadow-card-hover hover:border-border-default
">
  <div className="w-16 h-16 flex-shrink-0 rounded-[12px] overflow-hidden bg-bg-elevated">
    <img src={animeCoverImage} className="w-full h-full object-cover" />
  </div>
  <div className="flex-1 min-w-0 space-y-0.5">
    <div className="flex items-center gap-1.5 mb-1">
      <span className={`
        text-[10px] font-mono font-bold px-1.5 py-0.5 rounded-full
        ${type === 'OP'
          ? 'bg-accent-container text-accent'
          : 'bg-accent-ed-container text-accent-ed'
        }
      `}>
        {type}{sequence}
      </span>
      {qualityBadges?.map(badge => (
        <span key={badge} className="text-[10px] font-mono px-1.5 py-0.5 rounded-full
                                     bg-bg-elevated text-ktext-tertiary border border-border-subtle">
          {badge}
        </span>
      ))}
    </div>
    <p className="text-sm font-body font-semibold text-ktext-primary truncate">{songTitle}</p>
    <p className="text-xs font-body text-ktext-secondary truncate">
      {artistName} · {animeTitle}
    </p>
    {friendUsername && (
      <p className="text-xs font-body text-accent truncate">
        @{friendUsername} rated {friendScore}/10
      </p>
    )}
    {!friendUsername && (
      <div className="flex items-center gap-2 pt-0.5">
        <Star className="w-3 h-3 text-yellow-500 fill-yellow-500" />
        <span className="text-xs font-mono font-bold text-ktext-secondary">
          {avgRating.toFixed(1)}
        </span>
        <span className="text-xs text-ktext-tertiary">({totalRatings})</span>
      </div>
    )}
  </div>
  <button className="w-9 h-9 rounded-full bg-accent-container
                     flex items-center justify-center flex-shrink-0 interactive">
    <Play className="w-4 h-4 text-accent" />
  </button>
</div>
```

---

### 8.6 ThemeCard (Grid — Season/Search grid variant)

```tsx
<div className="
  group relative overflow-hidden rounded-[20px]
  bg-bg-surface border border-border-default
  shadow-card interactive cursor-pointer
  transition-all duration-200 hover:shadow-card-hover hover:-translate-y-0.5
">
  <div className="relative aspect-video overflow-hidden rounded-t-[20px]">
    <img src={animeCoverImage} className="w-full h-full object-cover
               group-hover:scale-105 transition-transform duration-500" />
    <div className="absolute inset-0 bg-gradient-to-t from-black/60 via-transparent to-transparent" />
    <span className={`absolute top-2 left-2 text-xs font-mono font-bold px-2 py-0.5 rounded-full
      ${type === 'OP' ? 'bg-accent text-white' : 'bg-accent-ed text-white'}`}>
      {type}{sequence}
    </span>
    {avgRating > 0 && (
      <div className="absolute top-2 right-2 w-8 h-8 rounded-full
                      flex items-center justify-center text-xs font-mono font-bold text-white"
           style={{ backgroundColor: getScoreColor(Math.round(avgRating)) }}>
        {avgRating.toFixed(1)}
      </div>
    )}
  </div>
  <div className="p-3 space-y-0.5">
    <p className="text-sm font-body font-semibold text-ktext-primary truncate">{songTitle}</p>
    <p className="text-xs font-body text-ktext-secondary truncate">{artistName}</p>
    <p className="text-xs font-body text-ktext-tertiary truncate">{animeTitle}</p>
    <div className="flex items-center gap-1 pt-1">
      <Star className="w-3 h-3 text-yellow-500 fill-yellow-500" />
      <span className="text-xs font-mono font-bold text-ktext-secondary">
        {avgRating > 0 ? avgRating.toFixed(1) : '—'}
      </span>
      <span className="text-xs text-ktext-tertiary">· {totalRatings} ratings</span>
    </div>
  </div>
</div>
```

---

### 8.7 Video Player (Vidstack)

Vidstack is a modern, accessible media player that supports HLS, DASH, and webm.
It handles both video playback and audio-only (background) playback gracefully.

```tsx
// /app/components/theme/VideoPlayer.tsx
'use client'
import { useRef, useEffect } from 'react'
import { Player, Media, Video } from '@vidstack/player-react'
import '@vidstack/player-react/styles/default.css'

interface VideoPlayerProps {
  videoUrl: string | null
  poster?: string | null
  mode: 'watch' | 'listen'
  onPlay?: () => void
}

export function VideoPlayer({ videoUrl, poster, mode, onPlay }: VideoPlayerProps) {
  const playerRef = useRef<any>(null)
  
  useEffect(() => {
    if (playerRef.current && onPlay) {
      const handlePlay = () => onPlay()
      const media = playerRef.current?.querySelector('media-player')
      if (media) {
        media.addEventListener('play', handlePlay)
        return () => media.removeEventListener('play', handlePlay)
      }
    }
  }, [onPlay])

  if (!videoUrl) {
    return (
      <div className={`relative w-full aspect-video rounded-[20px] overflow-hidden bg-bg-elevated flex items-center justify-center`}>
        <div className={`absolute inset-0 flex items-center justify-center bg-bg-surface`}>
          <img src={poster ?? ''} className={`absolute inset-0 w-full h-full object-cover ${mode === 'listen' ? 'opacity-20' : ''}`} />
          <div className={`flex items-end gap-1 h-12 relative z-10`}>
            {[...Array(8)].map((_, i) => (
              <div key={i} className={`w-1.5 bg-accent-mint rounded-full eq-bar-${i + 1}`} />
            ))}
          </div>
        </div>
      </div>
    )
  }

  if (mode === 'listen') {
    // Audio-only mode: hide video element, show equalizer animation
    return (
      <div className={`relative w-full aspect-video rounded-[20px] overflow-hidden bg-bg-elevated`}>
        <div className={`absolute inset-0 flex items-center justify-center bg-bg-surface`}>
          <img src={poster ?? ''} className={`absolute inset-0 w-full h-full object-cover opacity-20`} />
          <div className={`flex items-end gap-1 h-12 relative z-10`}>
            {[...Array(8)].map((_, i) => (
              <div key={i} className={`w-1.5 bg-accent-mint rounded-full eq-bar-${i + 1}`} />
            ))}
          </div>
        </div>
        {/* Hidden audio player for background playback */}
        <div className={`hidden`}>
          <Player
            ref={playerRef}
            src={videoUrl}
            autoplay
            muted={false}
            playsInline
          >
            <Media>
              <Video />
            </Media>
          </Player>
        </div>
      </div>
    )
  }

  // Watch mode: show full video player
  return (
    <div className={`relative w-full aspect-video rounded-[20px] overflow-hidden bg-bg-elevated`}>
      <Player
        ref={playerRef}
        src={videoUrl}
        poster={poster ?? undefined}
        autoplay
        muted={false}
        playsInline
        controls
      >
        <Media>
          <Video />
        </Media>
      </Player>
    </div>
  )
}
```

#### Vidstack Installation
```bash
npm install @vidstack/player-react @vidstack/player
```

#### Custom CSS for Vidstack (globals.css)
Vidstack uses CSS custom properties for theming. Add these to match the design system:
```css
/* Vidstack custom theme */
media-player {
  --media-control-background: rgba(10, 138, 150, 0.9);
  --media-accent-color: var(--accent-mint);
  --media-accent: var(--accent);
  --media-text-color: var(--text-primary);
  --media-range-bar-color: var(--accent-mint);
  --media-range-track-background: var(--bg-overlay);
  --media-range-thumb-background: var(--accent-mint);
}

media-player[data-theme='dark'] {
  --media-control-background: rgba(78, 205, 196, 0.9);
}
```

### Watch/Listen Toggle

```tsx
<div className="flex gap-2 mt-4">
  <button onClick={() => setMode('watch')}
          className={`flex items-center gap-2 px-5 py-2.5 rounded-full font-body text-sm font-semibold
            transition-colors duration-150
            ${mode === 'watch'
              ? 'bg-accent text-white'
              : 'bg-bg-elevated text-ktext-secondary border border-border-default'
            }`}>
    <Eye className="w-4 h-4" />
    Watch
  </button>
  <button onClick={() => setMode('listen')}
          className={`flex items-center gap-2 px-5 py-2.5 rounded-full font-body text-sm font-semibold
            transition-colors duration-150
            ${mode === 'listen'
              ? 'bg-accent text-white'
              : 'bg-bg-elevated text-ktext-secondary border border-border-default'
            }`}>
    <Headphones className="w-4 h-4" />
    Listen
  </button>
</div>
```

---

### 8.8 RatingWidget

**Circles variant (Your Rating):**
```tsx
<div className="space-y-3">
  <div className="flex items-center justify-between">
    <p className="text-xs font-body font-semibold text-ktext-secondary uppercase tracking-wide">Your Rating</p>
    {userRating && <p className="text-xs font-body text-ktext-tertiary">Tap to score</p>}
  </div>
  <div className="flex gap-2">
    {[1,2,3,4,5].map(score => (
      <button key={score} onClick={() => onRate(score)}
              className={`flex-1 h-11 rounded-full font-mono font-bold text-sm
                transition-all duration-150 interactive
                ${userRating === score
                  ? 'text-white scale-110 shadow-accent-glow'
                  : 'bg-bg-elevated text-ktext-tertiary border border-border-default'
                }`}
              style={userRating === score ? { backgroundColor: getScoreColor(score) } : {}}
              aria-label={`Rate ${score} out of 10`}>
        {score}
      </button>
    ))}
  </div>
  <div className="flex gap-2">
    {[6,7,8,9,10].map(score => (
      <button key={score} onClick={() => onRate(score)}
              className={`flex-1 h-11 rounded-full font-mono font-bold text-sm
                transition-all duration-150 interactive
                ${userRating === score
                  ? 'text-white scale-110 shadow-accent-glow'
                  : 'bg-bg-elevated text-ktext-tertiary border border-border-default'
                }`}
              style={userRating === score ? { backgroundColor: getScoreColor(score) } : {}}
              aria-label={`Rate ${score} out of 10`}>
        {score}
      </button>
    ))}
  </div>
  {userRating && (
    <button className="w-full h-12 bg-accent text-white rounded-full font-body font-semibold
                       interactive transition-colors duration-150 hover:bg-accent-hover">
      Confirm Rating
    </button>
  )}
</div>
```

**Bars variant (Community Score display — dark mode ThemePage):**
```tsx
<div className="bg-bg-elevated rounded-[16px] p-4 space-y-2">
  <p className="text-xs font-body font-semibold text-ktext-tertiary uppercase tracking-wide">Community Score</p>
  <div className="flex items-end gap-1 h-12">
    {scoreDistribution.map((count, i) => (
      <div key={i}
           onClick={() => onRate(i + 1)}
           className="flex-1 rounded-t-sm cursor-pointer transition-all duration-150 hover:opacity-80"
           style={{
             height: `${(count / maxCount) * 48}px`,
             backgroundColor: i === (userRating - 1) ? 'var(--accent-mint)' : 'var(--bg-overlay)'
           }} />
    ))}
  </div>
  <p className="font-mono font-bold text-3xl text-ktext-primary">
    {avgRating.toFixed(1)} <span className="text-sm font-body text-ktext-tertiary">/ 10</span>
  </p>
  <p className="text-xs text-ktext-tertiary">Tap a bar to rate</p>
</div>
```

---

### 8.9 ProfileHeader

```tsx
<div className="flex flex-col items-center text-center pt-6 pb-4 space-y-4">
  <div className="relative">
    <div className="w-24 h-24 rounded-full overflow-hidden ring-2 ring-accent-mint ring-offset-2 ring-offset-bg-base">
      <img src={avatarUrl} className="w-full h-full object-cover" />
    </div>
    {isOwn && (
      <button className="absolute bottom-0 right-0 w-7 h-7 rounded-full
                         bg-accent-mint flex items-center justify-center border-2 border-bg-base">
        <Pencil className="w-3 h-3 text-on-accent-mint" />
      </button>
    )}
  </div>
  <div>
    <h1 className="text-2xl font-display font-bold text-ktext-primary">{displayName}</h1>
    <p className="text-sm font-body text-ktext-secondary mt-1 max-w-[240px]">{bio}</p>
  </div>
  {isOwn ? (
    <button className="px-8 h-11 bg-accent-container border border-border-accent
                       text-accent font-body font-semibold rounded-full interactive">
      Edit Profile
    </button>
  ) : (
    <div className="flex gap-3">
      {/* FollowButton with label="Add Friend" for user profiles */}
      <FollowButton username={username} label="Add Friend" />
      <button className="px-6 h-11 bg-bg-elevated border border-border-default
                         text-ktext-primary font-body font-semibold rounded-full interactive">
        Message
      </button>
    </div>
  )}
  <div className="flex gap-3 w-full max-w-xs">
    {[
      { label: 'RATINGS', value: totalRatings },
      { label: 'FRIENDS', value: friends },
      { label: 'FOLLOWING', value: following },
    ].map(stat => (
      <div key={stat.label} className="flex-1 bg-bg-elevated rounded-[16px] p-3 text-center">
        <p className="text-xl font-display font-bold text-accent">{formatCount(stat.value)}</p>
        <p className="text-[10px] font-body text-ktext-tertiary tracking-wide">{stat.label}</p>
      </div>
    ))}
  </div>
</div>
```

---

### 8.10 NotificationCard

```tsx
<div className="bg-bg-surface rounded-[16px] border border-border-subtle p-4 space-y-3 shadow-card">
  <div className="flex items-start gap-3">
    <div className="relative flex-shrink-0">
      <div className="w-12 h-12 rounded-full overflow-hidden bg-bg-elevated">
        <img src={actorAvatar} className="w-full h-full object-cover" />
      </div>
      <div className={`absolute -bottom-1 -right-1 w-5 h-5 rounded-full
                       flex items-center justify-center border-2 border-bg-surface
                       ${type === 'friend_request' ? 'bg-accent' : 'bg-accent-mint'}`}>
        {type === 'friend_request' && <UserPlus className="w-2.5 h-2.5 text-white" />}
        {type === 'friend_rated' && <Star className="w-2.5 h-2.5 text-white" />}
        {type === 'friend_favorited' && <Heart className="w-2.5 h-2.5 text-white" />}
      </div>
    </div>
    <div className="flex-1 min-w-0">
      <p className="text-sm font-body text-ktext-primary leading-relaxed">
        <span className="font-semibold">{actorName}</span>{' '}
        {notificationText}
        {type === 'friend_rated' && (
          <span className="text-accent font-semibold"> {score}/10</span>
        )}
      </p>
      <p className="text-xs text-ktext-tertiary mt-1">{timeAgo}</p>
    </div>
    {(type === 'friend_rated' || type === 'friend_favorited') && themeImage && (
      <div className="w-10 h-10 rounded-full overflow-hidden flex-shrink-0 bg-bg-elevated">
        <img src={themeImage} className="w-full h-full object-cover" />
      </div>
    )}
    {type === 'friend_favorited' && (
      <Heart className="w-6 h-6 text-red-500 fill-red-500 flex-shrink-0" />
    )}
  </div>
  {type === 'friend_request' && (
    <div className="flex gap-2 ml-15">
      <button className="flex-1 h-10 bg-accent text-white rounded-full font-body font-semibold text-sm interactive">
        Accept
      </button>
      <button className="flex-1 h-10 bg-bg-elevated border border-border-default
                         text-ktext-secondary rounded-full font-body font-semibold text-sm interactive">
        Decline
      </button>
    </div>
  )}
</div>
```

---

### 8.11 FriendCard

```tsx
<div className="flex items-center gap-3 bg-bg-surface rounded-[16px] border border-border-subtle p-4 shadow-card">
  <div className="relative flex-shrink-0">
    <div className="w-12 h-12 rounded-full overflow-hidden bg-bg-elevated">
      <img src={avatarUrl} className="w-full h-full object-cover" />
    </div>
    {isOnline && (
      <div className="absolute bottom-0 right-0 w-3.5 h-3.5 rounded-full
                      bg-green-500 border-2 border-bg-surface" />
    )}
  </div>
  <div className="flex-1 min-w-0">
    <p className="text-sm font-body font-semibold text-ktext-primary">{displayName}</p>
    <p className="text-xs font-body text-ktext-secondary truncate">{bio}</p>
  </div>
  <div className="flex items-center gap-2 flex-shrink-0">
    <button className="w-9 h-9 rounded-full bg-accent-container
                       flex items-center justify-center interactive">
      <MessageCircle className="w-4 h-4 text-accent" />
    </button>
    <button className="w-9 h-9 rounded-full bg-bg-elevated border border-border-default
                       flex items-center justify-center interactive">
      <MoreVertical className="w-4 h-4 text-ktext-tertiary" />
    </button>
  </div>
</div>
```

---

### 8.12 HistoryCard

```tsx
<div className="flex items-center gap-3 bg-bg-surface rounded-[16px] border border-border-subtle p-4 shadow-card interactive cursor-pointer">
  <div className="w-16 h-16 flex-shrink-0 rounded-[12px] overflow-hidden bg-bg-elevated">
    <img src={animeCoverImage} className="w-full h-full object-cover" />
  </div>
  <div className="flex-1 min-w-0">
    <p className="text-[10px] font-body font-semibold tracking-wide uppercase
                  text-accent flex items-center gap-1 mb-1">
      {mode === 'watch' ? <Eye className="w-3 h-3" /> : <Headphones className="w-3 h-3" />}
      {type === 'OP' ? 'Opening Theme' : 'Ending Theme'}
    </p>
    <p className="text-sm font-body font-bold text-ktext-primary truncate">{songTitle}</p>
    <p className="text-xs font-body text-ktext-secondary italic truncate">{artistName}</p>
    <div className="flex items-center gap-3 mt-1 text-xs text-ktext-tertiary">
      <span className="flex items-center gap-1">
        <Clock className="w-3 h-3" />
        {timeAgo}
      </span>
      {userRating && (
        <span className="flex items-center gap-1">
          <Star className="w-3 h-3 text-yellow-500 fill-yellow-500" />
          {userRating}/10
        </span>
      )}
    </div>
  </div>
  <button className="w-9 h-9 rounded-full bg-accent-container flex items-center justify-center flex-shrink-0 interactive">
    <Play className="w-4 h-4 text-accent" />
  </button>
</div>
```

Section dividers:
```tsx
<div className="flex items-center gap-3 my-4">
  <div className="flex-1 h-px bg-border-subtle" />
  <span className="text-xs font-body font-semibold tracking-[0.1em] uppercase text-ktext-tertiary">TODAY</span>
  <div className="flex-1 h-px bg-border-subtle" />
</div>
```

---

### 8.13 AnimePage Layout

```tsx
<div className="relative h-56 md:h-72 overflow-hidden -mx-4 md:-mx-6">
  <img src={atGrillImage ?? bannerImage} className="w-full h-full object-cover object-top" />
  <div className="absolute inset-0 bg-gradient-to-b from-transparent via-transparent to-bg-base" />
  <div className="absolute bottom-4 left-4 flex gap-2">
    {genres.slice(0, 2).map(genre => (
      <span key={genre} className="text-xs font-body font-semibold px-3 py-1 rounded-full
                                    bg-black/40 backdrop-blur-sm text-white border border-white/20">
        {genre.toUpperCase()}
      </span>
    ))}
  </div>
</div>

<div className="px-4 -mt-8 relative z-10">
  <h1 className="text-2xl font-display font-bold text-ktext-primary">{titleRomaji}</h1>
  <div className="flex items-center gap-4 mt-1 text-sm text-ktext-secondary">
    <span className="flex items-center gap-1">
      <Star className="w-3.5 h-3.5 text-yellow-500 fill-yellow-500" />
      {(averageScore / 10).toFixed(1)} AniList
    </span>
    <span>·</span>
    <span>{totalEpisodes} Episodes</span>
  </div>
</div>

<div className="mt-6 bg-bg-surface rounded-[20px] border border-border-subtle p-4 shadow-card">
  <p className="text-xs font-body font-semibold text-ktext-tertiary uppercase tracking-wide mb-2">Openings</p>
  {openingThemes.map(theme => <ThemeListRow key={theme.slug} {...theme} />)}
  <p className="text-xs font-body font-semibold text-ktext-tertiary uppercase tracking-wide mb-2 mt-4">Endings</p>
  {endingThemes.map(theme => <ThemeListRow key={theme.slug} {...theme} />)}
</div>
```

---

### 8.14 ArtistPage Layout

```tsx
<div className="flex flex-col items-center text-center pt-6 space-y-4">
  <div className="relative">
    <div className="w-28 h-28 rounded-full overflow-hidden ring-2 ring-accent-mint ring-offset-2 ring-offset-bg-base">
      <img src={artistImage} className="w-full h-full object-cover" />
    </div>
  </div>
  <div>
    <h1 className="text-3xl font-display font-extrabold text-ktext-primary tracking-tight uppercase">
      {artistName}
    </h1>
  </div>
  <div className="flex gap-3 w-full max-w-xs">
    <div className="flex-1 bg-bg-elevated rounded-[16px] p-3 text-center">
      <p className="text-xl font-display font-bold text-accent">{formatCount(totalThemes)}</p>
      <p className="text-[10px] font-body text-ktext-tertiary tracking-wide uppercase">Total Themes</p>
    </div>
  </div>
  <div className="flex gap-3">
    {/* FollowButton with label="Follow Artist" for artist pages */}
    <FollowButton username={artistSlug} label="Follow Artist" />
    <button className="px-6 h-11 bg-bg-elevated border border-border-default
                       text-ktext-primary rounded-full font-body font-semibold interactive">
      Play Latest
    </button>
  </div>
</div>
```

---

### 8.15 Settings Page Layout

```tsx
<div className="bg-bg-surface rounded-[20px] border border-border-subtle p-4 shadow-card mb-4">
  <div className="flex items-center gap-3">
    <div className="w-14 h-14 rounded-full overflow-hidden ring-2 ring-accent-mint">
      <img src={avatarUrl} className="w-full h-full object-cover" />
    </div>
    <div>
      <p className="font-body font-semibold text-ktext-primary">{displayName}</p>
      <p className="text-xs font-body text-ktext-secondary">{userTag}</p>
    </div>
  </div>
</div>

<p className="text-xs font-body font-semibold text-accent tracking-[0.1em] uppercase mb-2 px-1">Account</p>
<div className="bg-bg-surface rounded-[20px] border border-border-subtle shadow-card mb-4 overflow-hidden">
  {accountItems.map((item, i) => (
    <button key={item.label} className={`
      w-full flex items-center gap-3 p-4 interactive
      ${i < accountItems.length - 1 ? 'border-b border-border-subtle' : ''}
    `}>
      <div className="w-9 h-9 rounded-[10px] bg-accent-container flex items-center justify-center">
        <item.Icon className="w-4 h-4 text-accent" />
      </div>
      <div className="flex-1 text-left">
        <p className="text-sm font-body font-medium text-ktext-primary">{item.label}</p>
        <p className="text-xs font-body text-ktext-tertiary">{item.subtitle}</p>
      </div>
      <ChevronRight className="w-4 h-4 text-ktext-tertiary" />
    </button>
  ))}
</div>

{/* Logout — uses CSS var for color: teal in light, crimson in dark */}
<button className="w-full h-14 rounded-full font-body font-bold text-white
                   flex items-center justify-center gap-2 interactive mt-6"
        style={{ backgroundColor: 'var(--logout-bg)' }}>
  <LogOut className="w-4 h-4" />
  Logout from Kaikansen
</button>
```

---

### 8.16 Auth Page Layout

```tsx
<div className="min-h-screen flex flex-col items-center justify-center px-4
                bg-bg-base relative overflow-hidden">
  <div className="flex flex-col items-center gap-3 mb-8">
    <div className="w-14 h-14 rounded-[16px] bg-accent-container flex items-center justify-center">
      <span className="text-accent text-2xl">≋</span>
    </div>
    <div className="text-center">
      <h1 className="text-3xl font-display font-extrabold text-ktext-primary">Kaikansen</h1>
      <p className="text-xs font-body tracking-[0.2em] uppercase text-ktext-tertiary mt-1">
        The Ethereal Tide
      </p>
    </div>
  </div>

  <div className="w-full max-w-sm bg-bg-surface rounded-[24px] border border-border-subtle p-6 shadow-modal">
    <h2 className="text-2xl font-display font-bold text-ktext-primary mb-1">Welcome back</h2>
    <p className="text-sm font-body text-ktext-secondary mb-6">
      Continue your journey through the tide.
    </p>
    <div className="space-y-4 mb-6">
      <div className="flex items-center gap-3 h-12 bg-bg-elevated rounded-[12px] px-4
                      border border-border-default focus-within:border-border-accent">
        <Mail className="w-4 h-4 text-ktext-tertiary flex-shrink-0" />
        <input type="email" placeholder="name@kaikansen.io"
               className="flex-1 bg-transparent outline-none text-sm font-body
                          text-ktext-primary placeholder:text-ktext-disabled" />
      </div>
      <div className="flex items-center gap-3 h-12 bg-bg-elevated rounded-[12px] px-4
                      border border-border-default focus-within:border-border-accent">
        <Lock className="w-4 h-4 text-ktext-tertiary flex-shrink-0" />
        <input type={showPassword ? 'text' : 'password'} placeholder="••••••••"
               className="flex-1 bg-transparent outline-none text-sm font-body text-ktext-primary" />
        <button onClick={() => setShowPassword(!showPassword)} className="interactive rounded-full p-1">
          {showPassword ? <EyeOff className="w-4 h-4 text-ktext-tertiary" /> : <Eye className="w-4 h-4 text-ktext-tertiary" />}
        </button>
      </div>
    </div>
    <button className="w-full h-12 bg-accent text-white rounded-full font-body font-semibold
                       flex items-center justify-center gap-2 interactive hover:bg-accent-hover">
      Sign In <ArrowRight className="w-4 h-4" />
    </button>
    <p className="text-center text-sm font-body text-ktext-secondary mt-4">
      Don't have an account?{' '}
      <Link to="/register" className="text-accent font-semibold interactive">Create one</Link>
    </p>
  </div>
</div>
```

---

### 8.17 ThemePage Layout (Full)

```tsx
<div className="pb-8">
  <div className="-mx-4 md:-mx-6">
    <VideoPlayer videoSources={videoSources} poster={animeCoverImage} mode={mode} />
  </div>
  <div className="flex gap-2 mt-4 px-4">
    <WatchListenToggle mode={mode} onModeChange={setMode} />
  </div>
  <div className="px-4 mt-4">
    <h1 className="text-2xl font-display font-bold text-ktext-primary leading-tight">{songTitle}</h1>
    <p className="text-sm font-body text-accent font-semibold mt-1">{artistName}</p>
    <Link to={`/anime/${anilistId}`} className="text-xs font-body text-ktext-tertiary mt-0.5 hover:text-accent">
      ∞ {animeTitle}
    </Link>
  </div>
  <div className="flex gap-3 mx-4 mt-4">
    {[
      { value: avgRating.toFixed(1), label: 'AVG RATING', color: getScoreColor(Math.round(avgRating)) },
      { value: formatCount(totalRatings), label: 'RATINGS' },
      { value: formatCount(totalWatches), label: 'WATCHES' },
    ].map(stat => (
      <div key={stat.label} className="flex-1 bg-bg-elevated rounded-[16px] p-3 text-center">
        <p className="text-xl font-mono font-bold text-ktext-primary"
           style={stat.color ? { color: stat.color } : {}}>
          {stat.value}
        </p>
        <p className="text-[10px] font-body text-ktext-tertiary tracking-wide">{stat.label}</p>
      </div>
    ))}
  </div>
  <div className="mx-4 mt-4 bg-bg-surface rounded-[20px] border border-border-subtle p-4 shadow-card">
    <RatingWidget userRating={userRating} onRate={handleRate} />
  </div>
</div>
```

---

### 8.18 Home Page Layout

```tsx
<div className="space-y-6 pt-4">
  <section>
    <div className="flex items-center justify-between mb-3">
      <div>
        <p className="text-xs font-body font-semibold text-accent uppercase tracking-wide">Current Season</p>
        <h2 className="text-2xl font-display font-bold text-ktext-primary">Winter 2026</h2>
      </div>
      <Link to="/season/winter/2026" className="text-sm font-body text-accent font-semibold interactive">
        View All
      </Link>
    </div>
    <div className="flex gap-3 overflow-x-auto scrollbar-hide -mx-4 px-4 pb-2">
      {featuredThemes.map(theme => <ThemeFeaturedCard key={theme.slug} {...theme} />)}
    </div>
  </section>

  {isLoggedIn && friendActivity.length > 0 && (
    <section>
      <div className="flex items-center justify-between mb-3">
        <h2 className="text-lg font-display font-bold text-ktext-primary">👥 Friends Activity</h2>
        <Link to="/friends" className="text-sm font-body text-accent font-semibold interactive">See all</Link>
      </div>
      <div className="space-y-2">
        {friendActivity.slice(0, 5).map(activity => (
          <ThemeListRow key={`${activity.userId}-${activity.themeSlug}`}
                        {...activity.theme}
                        friendUsername={activity.username}
                        friendScore={activity.score} />
        ))}
      </div>
    </section>
  )}

  <section>
    <div className="flex items-center justify-between mb-3">
      <h2 className="text-lg font-display font-bold text-ktext-primary">🔥 Popular Themes</h2>
      <div className="flex gap-1 p-1 bg-bg-elevated rounded-full">
        {(['OP', 'ED'] as const).map(t => (
          <button key={t} onClick={() => setTypeFilter(t === typeFilter ? null : t)}
                  className={`h-7 px-3 rounded-full text-xs font-body font-bold interactive
                    ${typeFilter === t ? 'bg-accent text-white' : 'text-ktext-secondary'}`}>
            {t}
          </button>
        ))}
      </div>
    </div>
    <div className="space-y-2">
      {popularThemes.map(theme => <ThemeListRow key={theme.slug} {...theme} />)}
    </div>
    <div ref={loadMoreRef} className="h-8" />
  </section>

  <div className="flex gap-3">
    <div className="flex-1 bg-bg-surface rounded-[16px] border border-border-subtle p-4 shadow-card">
      <TrendingUp className="w-5 h-5 text-accent mb-1" />
      <p className="text-xs font-body text-ktext-tertiary uppercase tracking-wide">Active Users</p>
      <p className="text-xl font-display font-bold text-ktext-primary">{formatCount(activeUsers)}</p>
    </div>
    <div className="flex-1 bg-bg-surface rounded-[16px] border border-border-subtle p-4 shadow-card">
      <div className="flex items-center gap-1 mb-1">
        {listeningAvatars.slice(0, 3).map((a, i) => (
          <img key={i} src={a} className="w-5 h-5 rounded-full -ml-1 first:ml-0 border border-bg-surface" />
        ))}
      </div>
      <p className="text-xs font-body text-ktext-tertiary uppercase tracking-wide">Listening Now</p>
      <p className="text-xl font-display font-bold text-ktext-primary">{listeningNow}</p>
    </div>
  </div>
</div>
```

---

### 8.19 Search Page Layout

```tsx
<div className={`pt-4 space-y-4`}>
  {/* Search Bar with AI Button */}
  <div className={`flex items-center gap-2 h-12 bg-bg-elevated rounded-full px-4
                  border border-border-default focus-within:border-border-accent`}>
    <Search className={`w-4 h-4 text-ktext-tertiary flex-shrink-0`} />
    <input 
      value={query} 
      onChange={e => setQuery(e.target.value)}
      onKeyDown={e => {
        if (e.key === 'Enter' && !aiSearchEnabled) {
          // Normal search on Enter
        } else if (e.key === 'Enter' && aiSearchEnabled) {
          // AI search trigger
        }
      }}
      placeholder={`Search songs, artists, anime…`}
      className={`flex-1 bg-transparent outline-none text-sm font-body text-ktext-primary`} 
    />
    {/* AI Search Toggle Button */}
    <button 
      onClick={() => setAiSearchEnabled(!aiSearchEnabled)}
      className={`flex-shrink-0 p-2 rounded-full transition-all ${
        aiSearchEnabled 
          ? 'bg-accent text-white' 
          : 'bg-bg-surface border border-border-default text-ktext-tertiary'
      }`}
      title={`AI Search ${aiSearchEnabled ? 'enabled' : 'disabled'}`}
    >
      <Sparkles className={`w-4 h-4`} />
    </button>
    {query && (
      <button onClick={clearQuery} className={`interactive rounded-full p-1`}>
        <X className={`w-4 h-4 text-ktext-tertiary`} />
      </button>
    )}
  </div>
  
  {/* AI Search Active Indicator */}
  {aiSearchEnabled && (
    <div className={`flex items-center gap-2 px-3 py-2 bg-accent-container rounded-[12px]`}>
      <Sparkles className={`w-4 h-4 text-accent flex-shrink-0`} />
      <p className={`text-xs font-body text-accent`}>
        AI Search — Enhanced results powered by semantic understanding
      </p>
    </div>
  )}
  
  {/* Search Type Banner - Show when using semantic/mood search */}
  {searchType === 'semantic' && (
    <div className={`flex items-center gap-2 px-3 py-2 bg-accent-container rounded-[12px]`}>
      <Sparkles className={`w-4 h-4 text-accent flex-shrink-0`} />
      <p className={`text-xs font-body text-accent`}>
        No exact matches — showing semantically related results
      </p>
    </div>
  )}
  
  {searchType === 'mood' && (
    <div className={`flex items-center gap-2 px-3 py-2 bg-accent-container rounded-[12px]`}>
      <Music className={`w-4 h-4 text-accent flex-shrink-0`} />
      <p className={`text-xs font-body text-accent`}>
        Showing themes matching the mood: {moods?.join(', ')}
      </p>
    </div>
  )}
  
  {searchType === 'none' && query.length >= 2 && (
    <div className={`flex items-center gap-2 px-3 py-2 bg-semantic-warning/10 rounded-[12px]`}>
      <AlertCircle className={`w-4 h-4 text-semantic-warning flex-shrink-0`} />
      <p className={`text-xs font-body text-semantic-warning`}>
        No results found. Try different keywords or disable AI search.
      </p>
    </div>
  )}
  
  <div className={`flex gap-2 overflow-x-auto scrollbar-hide pb-1`}>
    {filters.map(filter => (
      <button key={filter.value} onClick={() => setFilter(filter.value)}
              className={`flex-shrink-0 h-9 px-4 rounded-full text-sm font-body font-medium interactive
                ${activeFilter === filter.value
                  ? 'bg-accent text-white'
                  : 'bg-bg-surface border border-border-default text-ktext-secondary'
                }`}>
        {filter.label}
      </button>
    ))}
  </div>
  <div className={`space-y-2`}>
    {results.map(theme => <ThemeListRow key={theme.slug} {...theme} />)}
  </div>
</div>
```

#### AI Search Implementation Notes
- AI button in search bar toggles semantic search mode
- When enabled: all queries go through vector search + AI enhancement
- When disabled: uses standard $text search with vector fallback
- AI search uses Voyage AI for query embedding (cached in SearchCache)
- Fallback for AI failures: automatically fall back to text search

---

## 9. SPACING SYSTEM

| Context | Value |
|---|---|
| Page horizontal padding | `px-4` → `md:px-6` → `lg:px-8` |
| Page top padding | `pt-4` |
| Section gap | `space-y-6` |
| Card inner padding | `p-4` |
| List row gap | `space-y-2` |
| Grid gap | `gap-3` → `md:gap-4` |
| Chip/badge gap | `gap-2` |
| Icon (nav) | `w-6 h-6` |
| Icon (inline) | `w-4 h-4` |
| Min touch target | `min-h-11 min-w-11` (44px) |
| Bottom nav height | `h-16` |

---

## 10. IMAGE TREATMENT

```tsx
/* Anime cover — square for list rows */
<div className="w-16 h-16 flex-shrink-0 rounded-[12px] overflow-hidden bg-bg-elevated">
  <img src={atCoverImage ?? animeCoverImage} className="w-full h-full object-cover"
       loading="lazy" alt={animeTitle} />
</div>

/* Featured card — 16:9 landscape */
<div className="aspect-video rounded-[20px] overflow-hidden bg-bg-elevated">
  <img src={atCoverImage ?? animeCoverImage} className="w-full h-full object-cover" loading="lazy" />
</div>

/* Grill/Banner — wide hero */
<div className="relative h-56 overflow-hidden">
  <img src={atGrillImage ?? bannerImage ?? animeCoverImage}
       className="w-full h-full object-cover object-top" />
  <div className="absolute inset-0 bg-gradient-to-b from-transparent to-bg-base" />
</div>

/* Image fallback chain — always use this priority:
   1. atCoverImage (AnimeThemes Cover)
   2. animeCoverImage (from AnimeCache — AniList)
   3. placeholder bg-bg-elevated
*/
```

---

## 11. ANIMATION SYSTEM

### Easing: `cubic-bezier(0.2, 0, 0, 1)` — M3 standard

```css
/* globals.css */
@keyframes fadeUp {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}
.fade-up { animation: fadeUp 220ms cubic-bezier(0.2, 0, 0, 1) forwards; }

@keyframes scaleIn {
  from { opacity: 0; transform: scale(0.96); }
  to   { opacity: 1; transform: scale(1); }
}
.scale-in { animation: scaleIn 250ms cubic-bezier(0.2, 0, 0, 1) forwards; }

/* Stagger list items */
.stagger > * { opacity: 0; }
.stagger > *:nth-child(1) { animation: fadeUp 220ms 0ms   cubic-bezier(0.2,0,0,1) forwards; }
.stagger > *:nth-child(2) { animation: fadeUp 220ms 40ms  cubic-bezier(0.2,0,0,1) forwards; }
.stagger > *:nth-child(3) { animation: fadeUp 220ms 80ms  cubic-bezier(0.2,0,0,1) forwards; }
.stagger > *:nth-child(4) { animation: fadeUp 220ms 120ms cubic-bezier(0.2,0,0,1) forwards; }
.stagger > *:nth-child(5) { animation: fadeUp 220ms 160ms cubic-bezier(0.2,0,0,1) forwards; }

/* Skeleton shimmer */
@keyframes shimmer {
  0%   { background-position: -800px 0; }
  100% { background-position:  800px 0; }
}
.shimmer {
  background: linear-gradient(
    90deg,
    var(--bg-elevated) 0%,
    var(--bg-overlay)  50%,
    var(--bg-elevated) 100%
  );
  background-size: 800px 100%;
  animation: shimmer 1.4s infinite linear;
}

/* Listen mode equalizer bars */
@keyframes eq1 { 0%,100%{height:6px}  50%{height:20px} }
@keyframes eq2 { 0%,100%{height:14px} 50%{height:6px}  }
@keyframes eq3 { 0%,100%{height:20px} 50%{height:14px} }
@keyframes eq4 { 0%,100%{height:6px}  50%{height:18px} }
.eq-bar-1 { animation: eq1 0.8s ease-in-out infinite; }
.eq-bar-2 { animation: eq2 0.9s ease-in-out infinite; }
.eq-bar-3 { animation: eq3 1.0s ease-in-out infinite; }
.eq-bar-4 { animation: eq4 0.7s ease-in-out infinite; }
.eq-bar-5 { animation: eq1 1.1s ease-in-out infinite; }
.eq-bar-6 { animation: eq2 0.85s ease-in-out infinite; }
.eq-bar-7 { animation: eq3 0.95s ease-in-out infinite; }
.eq-bar-8 { animation: eq4 0.75s ease-in-out infinite; }

/* Reduced motion */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## 12. SCORE UTILITIES

```typescript
// lib/utils.ts

export function getScoreColor(score: number): string {
  if (score >= 9)  return '#10b981'
  if (score >= 7)  return '#22c55e'
  if (score >= 6)  return '#84cc16'
  if (score >= 5)  return '#eab308'
  if (score >= 4)  return '#f97316'
  return '#ef4444'
}

export function getScoreLabel(score: number): string {
  const labels: Record<number, string> = {
    10: 'Masterpiece', 9: 'Excellent', 8: 'Great',
    7:  'Good',        6: 'Fine',      5: 'Average',
    4:  'Below Avg',   3: 'Bad',       2: 'Terrible', 1: 'Unwatchable'
  }
  return labels[score] ?? '—'
}

export function formatCount(n: number): string {
  if (n >= 1_000_000_000) return `${(n / 1_000_000_000).toFixed(1)}B`
  if (n >= 1_000_000)     return `${(n / 1_000_000).toFixed(1)}M`
  if (n >= 1_000)         return `${(n / 1_000).toFixed(1)}k`
  return n.toString()
}

export function timeAgo(date: Date | string): string {
  // "2 hours ago", "Yesterday", "Mar 24"
}

export function cn(...inputs: (string | undefined | null | false)[]): string {
  // clsx + tailwind-merge
}
```

---

## 13. MOBILE-SPECIFIC RULES

- All interactive elements: `min-h-11 min-w-11` (44px) — non-negotiable
- Bottom nav: `h-16` + `pb-[env(safe-area-inset-bottom)]`
- Page content: `pb-20 md:pb-0`
- Horizontal scrolls: `overflow-x-auto scrollbar-hide -mx-4 px-4 pb-2`
- Video: always `playsInline` — required for iOS (plyr-react passes this via options)
- Listen mode: keep audio playing, show equalizer overlay
- Font sizes: minimum `text-sm` (14px) for body copy, never smaller

---

## 14. ACCESSIBILITY

- All icon-only buttons have `aria-label`
- Rating buttons: `aria-label="Rate {score} out of 10"`
- Score displays: `aria-label="Community score: {score} from {count} ratings"`
- Bottom nav: `role="link"` + `aria-current="page"` on active
- Focus rings: `.interactive:focus-visible` with 2px accent outline
- Color contrast: text-primary on bg-surface ≥ 7:1 (light) / ≥ 7:1 (dark)
- Scores always shown with label, never color-only

---

## 15. AGENT PROMPT CHEAT SHEET

```
Apply the Kaikansen Tidal UI design system:
- Use CSS variables (var(--token)) for all colors — never hardcode hex values
- .interactive class on ALL clickable surfaces
- rounded-[20px] for cards, rounded-[16px] for list rows, rounded-full for pills
- Image fallback chain: atCoverImage → animeCoverImage (AniList)
- Grill image fallback: atGrillImage → bannerImage → animeCoverImage
- Bottom nav: flex md:hidden | Navigation Rail: hidden md:flex
- Min touch target: min-h-11 on every interactive element
- Dark mode: use ONLY [data-theme="dark"] — next-themes never adds a .dark class
- Dark mode header: rounded-b-[24px] applied via className in AppHeader.tsx
- Section labels: text-xs uppercase tracking-wide text-ktext-tertiary (light) / text-accent (dark)
- OP badge: bg-accent-container text-accent
- ED badge: bg-accent-ed-container text-accent-ed
- Score colors: use getScoreColor() from lib/utils.ts
- Logout button: var(--logout-bg) — teal in light, crimson in dark
- FollowButton label prop: "Follow Artist" on artist pages, "Add Friend" on user profiles
- VideoPlayer: uses plyr-react (not plain plyr) — import from 'plyr-react'
- useTheme hook: import from 'hooks/useTheme' (root hooks/ folder)
```

**Token quick-reference:**
```
bg-bg-base        → page background
bg-bg-surface     → cards, panels
bg-bg-elevated    → inputs, hover states
accent            → primary interactive (dark teal light / mint teal dark)
accent-mint       → badges, active indicators
accent-container  → chip/indicator bg
accent-ed         → ED themes, peach color
ktext-primary     → headings, titles
ktext-secondary   → subtitles, artists
ktext-tertiary    → timestamps, metadata
border-default    → card borders
border-accent     → focused/selected
```

---

## 16. EXACT MOBILE LAYOUTS (Screen-by-Screen)

> Pixel-accurate specs from design mockups.
> All measurements assume 390px wide (iPhone 14 Pro viewport).

---

### 16.1 HOME PAGE — Mobile Layout

#### Light Mode Flow (top → bottom)

```
┌─────────────────────────────────────────┐  h=56px
│  ☰  Kaikansen              [avatar 36px]│  bg: #FFFFFF
│  (hamburger)  (text-accent font-display)│  border-b: border-border-subtle
└─────────────────────────────────────────┘

CURRENT SEASON label: text-xs color #0A8A96 (accent)
Winter 2026 / View All →: text-2xl font-display font-bold / text-sm text-accent

FEATURED STRIP (horizontal scroll, -mx-4 px-4)
Cards: w-[75vw] aspect-video rounded-[20px] gap-12px

Popular Themes heading + [OP][ED] toggle
LIST ROWS: bg-bg-surface rounded-[16px] border border-border-subtle shadow-card

STATS FOOTER (2 cards, mt-24px)
Active Users + Listening Now: bg-bg-surface rounded-[16px] p-16px

BOTTOM NAV h=64px + safe-area
Active: large teal pill w-20 h-10 rounded-full bg-accent-mint behind icon
```

#### Dark Mode Flow
```
HEADER: bg-bg-header (#0A1828) rounded-b-[24px] — NO border
FEATURED: cards bg-bg-surface (#0C1C28)
STATS labels: text-[#4ECDC4] (accent-mint)
BOTTOM NAV active: bg-accent-container oval, icon text-accent-mint
```

---

### 16.2 SEARCH PAGE — Mobile Layout

```
SEARCH BAR h=48px: bg-bg-elevated rounded-full border border-border-default
FILTER CHIPS: overflow-x-auto gap-8px
  Active: bg-accent text-white | Inactive: bg-bg-surface border text-ktext-secondary
RESULTS: ThemeListRow format, heart icon instead of play for favorites
```

---

### 16.3 THEME PAGE — Mobile Layout

```
VIDEO PLAYER: -mx-4 full bleed, aspect-video, rounded-[20px]
WATCH/LISTEN: Active bg-accent text-white rounded-full h-10 px-5
SONG INFO: text-2xl font-display font-bold + text-sm text-accent artist
STATS ROW: 3 cards bg-bg-elevated rounded-[16px], AVG RATING colored by getScoreColor()
RATING WIDGET: circles variant with confirm button
```

---

### 16.4 NOTIFICATIONS PAGE — Mobile Layout

```
HEADER: "Notifications" + "Mark all as read" text-accent right
NOTIFICATION CARDS: bg-bg-surface rounded-[20px] p-16px shadow-card
  Friend request: Accept (bg-accent) + Decline (bg-bg-elevated) h-10 rounded-full
  Rated: score pill bg-accent-container text-accent + theme thumbnail w-10 rounded-full
OLDER NOTIFICATIONS divider: text-[10px] uppercase + h-px bg-border-subtle
Dark: unread cards left border 3px accent-mint
```

---

### 16.5 FRIENDS / CONNECTIONS PAGE — Mobile Layout

```
"Connections" text-3xl text-accent font-bold + [👤+ Add Friend] pill right
TABS: Friends | Requests [badge] — active underline h-[2px] bg-accent
FRIEND ROWS: bg-bg-surface rounded-[24px] h=72px
  Avatar w-12 + online dot + name + tagline + [💬][⋮] buttons
PENDING REQUEST CARDS: Accept (teal pill) + Decline (gray pill)
Dark: friend rows use lighter steel-blue tint, fully rounded pill-shaped
```

---

### 16.6 WATCH HISTORY PAGE — Mobile Layout

```
"Watch History" + subtitle text-sm text-ktext-secondary
FILTER TABS: [All][Watched][Listened] pill tabs
DATE DIVIDERS: ──── TODAY ──── text-[10px] uppercase text-ktext-tertiary
HISTORY CARDS: bg-bg-surface rounded-[20px] p-16px
  Image w-20 h-20 rounded-[12px] + mode label teal + title + time
```

---

### 16.7 PROFILE PAGE — Mobile Layout

```
AVATAR: w-24 h-24 ring-2 ring-accent-mint (own: edit pencil badge)
Name text-2xl font-bold + bio text-sm text-ktext-secondary centered
Own profile: "Edit Profile" border-2 border-accent text-accent rounded-full h-11
Other: FollowButton label="Add Friend" + Message button
STATS: RATINGS | FRIENDS | FOLLOWING — bg-bg-elevated rounded-[16px]
RECENT ACTIVITY: circular image w-14 + score circle right side
Dark: ring glow box-shadow rgba(78,205,196,0.4)
```

---

### 16.8 SETTINGS PAGE — Mobile Layout

```
USER CARD: bg-bg-surface rounded-[20px] avatar w-14 ring-2 ring-accent-mint
SECTION LABEL: text-[11px] uppercase tracking-[0.12em] text-accent
SECTION CARD: bg-bg-surface rounded-[20px] overflow-hidden
  Rows h=60px: icon container w-9 h-9 bg-accent-container rounded-[10px]
LOGOUT: full-width h-14 rounded-full style={{ backgroundColor: 'var(--logout-bg)' }}
Dark logout: #8B1A1A (crimson)
```

---

### 16.11 ARTIST PAGE — Mobile Layout

```
AVATAR: w-28 h-28 rounded-full ring-2 ring-accent-mint (dark: teal glow)
Name: text-3xl font-extrabold uppercase tracking-tight centered
STATS: 2 cards bg-bg-elevated rounded-[16px] — Total Themes + Global Rank
ACTIONS: FollowButton label="Follow Artist" + "Play Latest" outline button
DISCOGRAPHY ROWS: circular img w-12 + title + OP badge + year + ⋮ icon
Dark: rows no card bg, image stays circular, "The Story" card with genre tags
```

---

### 16.12 BOTTOM NAVIGATION — Exact Specs

```
Container: h=64px + safe-area, fixed bottom-0, z-50
Light: bg-bg-surface border-t border-border-subtle
Dark: bg-bg-surface (no border)

4 Items — 25% each:

LIGHT active: w-20 h-10 rounded-full bg-accent-mint behind icon+label
  Icon: text-accent | Label: text-[10px] text-accent (active only)

DARK active: w-20 h-10 rounded-full bg-accent-container
  Icon: text-accent-mint | Label: text-[10px] text-accent-mint

Bell badge: w-4 h-4 rounded-full bg-error text-white text-[8px] font-mono
  absolute top-2, content: count if ≤9, "9+" if more
```

---

### 16.13 SPACING & VISUAL RHYTHM

```
Header:              h=56px   fixed top
Page content start:  pt=16px  (pt=20px dark — rounded header offset)
Section gap:         mt=24px  between major sections
Card gap:            gap-12px between list rows
Card padding:        p=16px   inside all cards
Inner element gap:   gap=8px  between small elements
Section label → card: mb=8px
Bottom nav buffer:   pb=80px  last item clears nav

Horizontal padding:
  Mobile: px=16px (px-4)
  Bleed items: -mx-4 to remove padding
```

---

### 16.14 TYPOGRAPHY SIZES — Mobile Exact

| Element | Size | Weight | Color token |
|---|---|---|---|
| App name header | 18px | 700 | ktext-primary |
| Page title (Display) | 24-30px | 700-800 | ktext-primary |
| Artist name (hero) | 28-36px | 800 | ktext-primary |
| Song title | 20-24px | 700 | ktext-primary |
| Section heading | 18-20px | 700 | ktext-primary |
| List row title | 14px | 600 | ktext-primary |
| List row subtitle | 12px | 400 | ktext-secondary |
| List row meta | 12px | 400 | ktext-tertiary |
| Section label | 10-11px | 600 | ktext-tertiary / accent |
| Badge/chip | 10px | 700 | varies |
| Timestamp | 11-12px | 400 | ktext-tertiary |
| Score (large) | 32-40px | 700 | getScoreColor() |
| Score (inline) | 13-14px | 700 | ktext-secondary |
| Button label | 14px | 600 | varies |
| Input text | 14px | 400 | ktext-primary |

---

## 17. NEXT.JS DARK MODE INTEGRATION

### next-themes Setup
```tsx
// /app/layout.tsx
import { ThemeProvider } from 'next-themes'

export default function RootLayout({ children }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider attribute="data-theme" defaultTheme="system" enableSystem disableTransitionOnChange>
          <AuthProvider>
            <QueryProvider>
              {children}
              <Toaster />
            </QueryProvider>
          </AuthProvider>
        </ThemeProvider>
      </body>
    </html>
  )
}
```

### ThemeToggle Component (Settings Page)
```tsx
// /app/components/shared/ThemeToggle.tsx
"use client"
import { useTheme } from 'next-themes'
import { useEffect, useState } from 'react'
import { Sun, Moon, Monitor } from 'lucide-react'

export function ThemeToggle() {
  const { theme, setTheme } = useTheme()
  const [mounted, setMounted] = useState(false)
  useEffect(() => setMounted(true), [])
  if (!mounted) return null

  const options = [
    { value: 'light', icon: Sun,     label: 'Light'  },
    { value: 'dark',  icon: Moon,    label: 'Dark'   },
    { value: 'system',icon: Monitor, label: 'System' },
  ]

  return (
    <div className="flex gap-1 p-1 bg-bg-elevated rounded-full">
      {options.map(opt => (
        <button
          key={opt.value}
          onClick={() => setTheme(opt.value)}
          className={`flex items-center gap-1.5 h-8 px-3 rounded-full text-xs font-body font-medium
            transition-colors duration-150 interactive
            ${theme === opt.value
              ? 'bg-accent text-white'
              : 'text-ktext-secondary hover:text-ktext-primary'
            }`}
        >
          <opt.icon className="w-3.5 h-3.5" />
          {opt.label}
        </button>
      ))}
    </div>
  )
}
```

### useTheme Hook (App-level)
```tsx
// hooks/useTheme.ts  ← root hooks/ folder (NOT /app/hooks/)
"use client"
import { useTheme as useNextTheme } from 'next-themes'
import { useEffect, useState } from 'react'

export function useTheme() {
  const { theme, setTheme, resolvedTheme } = useNextTheme()
  const [mounted, setMounted] = useState(false)
  useEffect(() => setMounted(true), [])

  return {
    theme:   mounted ? resolvedTheme : undefined,
    rawTheme: mounted ? theme : undefined,
    setTheme,
    isDark:  mounted ? resolvedTheme === 'dark' : false,
    isLight: mounted ? resolvedTheme === 'light' : true,
    mounted,
    toggle:  () => setTheme(resolvedTheme === 'dark' ? 'light' : 'dark'),
  }
}
```

### Dark Mode Header Detection
```tsx
// AppHeader.tsx — uses useTheme().isDark to apply rounded-b-[24px]
// This is a direct className — NOT a Tailwind config token
"use client"
import { useTheme } from '@/hooks/useTheme'

export function AppHeader() {
  const { isDark } = useTheme()
  return (
    <header className={`
      sticky top-0 z-40 h-14 px-4 flex items-center justify-between
      bg-bg-header
      ${isDark ? 'rounded-b-[24px] shadow-md' : 'border-b border-border-subtle'}
    `}>
      {/* content */}
    </header>
  )
}
```

### CSS Variable Switching (globals.css)
The CSS variables defined in §2 automatically switch when `[data-theme="dark"]` is applied to `<html>` by next-themes. No JavaScript needed for color switching — it is entirely CSS-driven.

---

## 18. ARTIST DISCOGRAPHY ROW

```tsx
<div className="flex items-center gap-3 py-3 border-b border-border-subtle interactive cursor-pointer">
  {/* Circular image */}
  <div className="w-12 h-12 rounded-full overflow-hidden bg-bg-elevated flex-shrink-0">
    <img src={animeCoverImage} alt={songTitle} className="w-full h-full object-cover rounded-full" />
  </div>

  {/* Info */}
  <div className="flex-1 min-w-0">
    <p className="text-sm font-body font-semibold text-ktext-primary truncate">{songTitle}</p>
    <div className="flex items-center gap-2 mt-0.5">
      <span className={`text-[10px] font-mono font-bold px-1.5 py-0.5 rounded-full
        ${type === 'OP' ? 'bg-accent-container text-accent' : 'bg-accent-ed-container text-accent-ed'}`}>
        {type}
      </span>
      <p className="text-xs text-ktext-tertiary truncate">{animeTitle} · {year}</p>
    </div>
  </div>

  {/* More menu */}
  <button className="w-8 h-8 rounded-full flex items-center justify-center interactive">
    <MoreVertical className="w-4 h-4 text-ktext-tertiary" />
  </button>
</div>
```

Note: No artist bio field — `ArtistCache` does not store a bio. Do not render a bio card on artist pages.

---

## 19. FOLLOW BUTTON DESIGN

### FollowButton Component — accepts label prop

```tsx
// /app/components/shared/FollowButton.tsx ("use client")
interface FollowButtonProps {
  username: string
  label?: string  // default: 'Follow' — pass 'Follow Artist' or 'Add Friend' as needed
}

export function FollowButton({ username, label = 'Follow' }: FollowButtonProps) {
  const { isFollowing, follow, unfollow, isPending } = useFollow(username)

  return (
    <button
      onClick={() => isFollowing ? unfollow() : follow()}
      disabled={isPending}
      className={`
        flex items-center gap-2 h-11 px-6 rounded-full font-body font-semibold text-sm
        transition-all duration-150 interactive
        ${isFollowing
          ? 'bg-accent-container text-accent border border-border-accent'
          : 'bg-accent text-white hover:bg-accent-hover'
        }
        ${isPending ? 'opacity-50 cursor-not-allowed' : ''}
      `}
    >
      {isPending ? (
        <Loader2 className="w-4 h-4 animate-spin" />
      ) : isFollowing ? (
        <>
          <Check className="w-4 h-4" />
          Following
        </>
      ) : (
        <>
          <Plus className="w-4 h-4" />
          {label}
        </>
      )}
    </button>
  )
}

// Usage:
// Artist page:        <FollowButton username={artistSlug} label="Follow Artist" />
// User profile:       <FollowButton username={username} label="Add Friend" />
// Notifications (follow-back): <FollowButton username={actorUsername} label="Follow Back" />
// ↑ "Follow Back" on notifications because it calls the Follow API — NOT the Friends API.
//   Reserve "Add Friend" for the explicit friend request flow only.
```

---

## 20. LIVE STATS CARDS DESIGN

```tsx
<div className="flex gap-3 mt-6">
  <div className="flex-1 bg-bg-surface rounded-[16px] border border-border-subtle p-4 shadow-card">
    <TrendingUp className="w-5 h-5 text-accent mb-2" />
    <p className="text-[10px] font-body font-semibold uppercase tracking-wide
                  text-ktext-tertiary dark:text-accent-mint">
      Active Users
    </p>
    <p className="text-2xl font-display font-bold text-ktext-primary mt-0.5">
      {formatCount(activeUsers)}
    </p>
  </div>

  <div className="flex-1 bg-bg-surface rounded-[16px] border border-border-subtle p-4 shadow-card">
    <div className="flex items-center mb-2">
      {avatars.slice(0, 3).map((src, i) => (
        <img key={i} src={src} alt="" width={20} height={20}
               className={`w-5 h-5 rounded-full border border-bg-surface ${i > 0 ? '-ml-1' : ''}`} />
      ))}
      {listeningCount > 3 && (
        <span className="text-[10px] text-ktext-tertiary ml-1.5">+{listeningCount - 3}</span>
      )}
    </div>
    <p className="text-[10px] font-body font-semibold uppercase tracking-wide
                  text-ktext-tertiary dark:text-accent-mint">
      Listening Now
    </p>
    <p className="text-2xl font-display font-bold text-ktext-primary mt-0.5">
      {listeningNow}
    </p>
  </div>
</div>
```

Dark mode stats (automatic via `dark:text-accent-mint`):
- `ACTIVE USERS` and `LISTENING NOW` labels switch to mint teal in dark mode
- Numbers inherit `text-ktext-primary` which resolves to `#FFFFFF` in dark mode
- Card bg: `bg-bg-surface` with darker elevation via CSS var switching
- Requires `darkMode: ['class', '[data-theme="dark"]']` in tailwind.config.ts (already set)
