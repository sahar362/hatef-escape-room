# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install       # first-time setup
npm run dev       # dev server at http://localhost:3000
npm run build     # production build
npm run lint      # ESLint
```

No test suite exists. Verify changes by running the dev server and playing through the game.

## What This Is

**اتاق فرار هاتف** — a Persian-language (RTL) educational escape room that teaches how LLMs work. Players solve 5 puzzles in 10 minutes, each representing a transformer operation: tokenization → embedding → attention → generation → prompt engineering.

## Architecture

### State & Flow

The single Zustand store in [lib/store.ts](lib/store.ts) drives everything. `phase` moves through `"intro" → "playing" → "win" | "lose"`. `currentLayer` (1–5) determines which puzzle renders. The store persists to localStorage via `zustand/middleware`.

`remainingSeconds()` is a computed getter (not stored): `(GAME_DURATION - elapsed) - penaltySeconds`. Penalties come from hints (+90s each).

Routing: `/` → Intro screen → sets phase to `"playing"` → `/game` renders LayerShell wrapping the current puzzle component. `/game` redirects to `/` if phase is not `"playing"`.

### Puzzle Data

All puzzle content lives in [lib/puzzles.ts](lib/puzzles.ts): token arrays, word clusters, sentence structures, generation trees, and correct answers. Each layer's puzzle component in [components/puzzles/](components/puzzles/) imports from there. **Change puzzle content in `puzzles.ts`, not in the component.**

### Layer 5 AI Integration

[app/api/oracle/route.ts](app/api/oracle/route.ts) handles the prompt-engineering puzzle's LLM calls with a three-tier fallback:

1. **OpenRouter** (priority 1) — tries `OPENROUTER_MODEL` env var first, then falls back through `OPENROUTER_MODELS` array on 429 rate-limit errors
2. **Anthropic Claude** (`claude-opus-4-8`) — used if `ANTHROPIC_API_KEY` is set and OpenRouter is unavailable
3. **Offline mode** — validates the prompt contains required keywords, returns a pre-composed fallback poem from [lib/acrostic.ts](lib/acrostic.ts)

The goal: produce a Persian poem where the first letter of each line spells **"رها"** (ر، ه، ا). `checkAcrostic()` in [lib/acrostic.ts](lib/acrostic.ts) normalizes Arabic diacritics and Aleph variants before checking.

### Audio

[lib/audio.ts](lib/audio.ts) uses Tone.js for fully generative audio (no required audio files). It initializes on first user interaction (browser autoplay policy). `updateTension(fractionRemaining)` is called every second to ramp BPM and reduce reverb as time runs out. Dropping `public/audio/ambient.mp3` switches to file-based mode.

### Styling

Tailwind with a custom dark theme: `ink-*` for backgrounds, `cyanGlow`/`amberGlow`/`dangerGlow` for accents. Global scanline and grain effects are in [app/globals.css](app/globals.css). All layouts are RTL-first with Vazir font. Framer Motion handles layer transition animations.

## Environment Variables

Copy `.env.example` to `.env.local` and fill in at least one key:

```
OPENROUTER_API_KEY=   # preferred — free models available
OPENROUTER_MODEL=google/gemma-4-31b-it:free   # override model if desired
ANTHROPIC_API_KEY=    # fallback
```

Layer 5 works offline without any keys, but the AI response will be the pre-composed fallback poem.

## Key Design Constraints

- **Persian/RTL throughout** — all UI text is Persian, `dir="rtl"` on root, Vazir font loaded via Next.js layout
- **No backend persistence** — scores and state are client-only (localStorage). Supabase env vars are stubbed for a future leaderboard
- **Offline-first puzzles** — layers 1–4 have no network dependency; only layer 5 optionally calls the API
- **10-minute hard limit** — `GAME_DURATION` in store.ts; timer drives win/lose transition
