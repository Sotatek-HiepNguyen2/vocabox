# VocaBox — Design Spec

## Problem Statement

Learning vocabulary effectively requires consistent, spaced exposure — but most learners forget to practice. VocaBox is a Telegram bot that delivers Oxford 3000 words via spaced repetition, using active recall and audio pronunciation, directly to the user's chat. Zero friction, zero cost.

**User Stories:**
- As a learner, I want daily vocabulary cards pushed to my Telegram so I don't need to remember to open an app
- As a learner, I want to hear pronunciation and see definitions so I build both recognition and recall
- As a learner, I want the system to schedule reviews automatically so I retain words long-term
- As a learner, I want to start from my current level so I don't waste time on words I already know

## Scope

**In scope (MVP):**
- Oxford 3000 word list with pre-fetched dictionary data (definitions, examples, audio, synonyms, antonyms)
- Telegram bot with webhook on Cloudflare Workers
- User registration with CEFR level selection
- SRS engine (SM-2 inspired) with daily delivery
- Active recall flow: word + audio → reveal → rate
- Per-user timezone-aware delivery via single cron
- `/start`, `/stats`, `/settings` commands
- Safety cap on daily reviews for backlog scenarios

**Out of scope (Phase 2+):**
- Oxford 5000 expansion
- Placement quiz onboarding
- Web dashboard
- `/skip`, `/pause` commands
- Leaderboards

## Evaluated Approaches

### Bot Framework
| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| grammY | Lightweight, Workers-compatible, good TS support | Smaller community than Telegraf | **Selected** |
| Telegraf | Large community, mature | Heavier, Node.js-oriented, Workers compat issues | Rejected |
| Raw Bot API | No dependencies | Verbose, reinventing the wheel | Rejected |

grammY is purpose-built for serverless/edge runtimes. Native Workers adapter available.

### Storage
| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| Cloudflare KV | Free tier (100k reads/day, 1k writes/day), simple key-value | Eventually consistent, no queries | **Selected** |
| Cloudflare D1 | SQL queries, relational | Overkill for small group, more setup | Phase 2 if needed |
| Supabase | Full Postgres, generous free tier | External dependency, latency | Rejected |

KV is sufficient for small group. Key design: `user:{id}` for settings, `srs:{userId}:{word}` for review state.

### Word Data
| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| Runtime API calls | Always fresh | Unreliable, adds latency, API has no SLA | Rejected |
| Pre-fetch at build time | Zero runtime dependency, fast | Static, one-time effort | **Selected** |

Pre-fetch all 3000 words from Free Dictionary API → store as static JSON bundled with Worker.

## Architecture

```
┌─────────────────────────────────────┐
│  Build-time: Pre-fetch Pipeline     │
│  Oxford 3000 list + Dictionary API  │
│  → words-data.json (bundled)        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  Cloudflare Worker                  │
│                                     │
│  ┌─────────────┐  ┌──────────────┐ │
│  │ Webhook      │  │ Cron Trigger │ │
│  │ Handler      │  │ (hourly)     │ │
│  └──────┬──────┘  └──────┬───────┘ │
│         │                │          │
│  ┌──────▼────────────────▼───────┐ │
│  │      Bot Logic                │  │
│  │  ┌──────────┐ ┌────────────┐ │  │
│  │  │ Commands │ │ SRS Engine │ │  │
│  │  └──────────┘ └────────────┘ │  │
│  │  ┌──────────┐ ┌────────────┐ │  │
│  │  │ Card     │ │ Session    │ │  │
│  │  │ Renderer │ │ Manager    │ │  │
│  │  └──────────┘ └────────────┘ │  │
│  └──────────────┬────────────────┘ │
│                 │                   │
│  ┌──────────────▼────────────────┐ │
│  │  Cloudflare KV                │ │
│  │  user:{id} → settings         │ │
│  │  srs:{userId}:{word} → state  │ │
│  │  session:{userId} → active    │ │
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
```

## Components

### 1. Pre-fetch Pipeline (build-time script)

Runs once before deploy. Scrapes Oxford 3000 word list, then fetches each word from Free Dictionary API.

**Output:** `words-data.json` — array of:
```json
{
  "word": "albeit",
  "level": "B2",
  "pos": "conjunction",
  "definition": "although; even though",
  "example": "The pay is good, albeit less than my last job.",
  "phonetic": "/ɔːlˈbiːt/",
  "audioUrl": "https://...mp3",
  "synonyms": ["although", "though"],
  "antonyms": []
}
```

Fallback: if Dictionary API returns no data for a word, log it and include the word with Oxford-sourced data only (word, level, pos). No audio/synonyms for that entry.

### 2. SRS Engine

SM-2 inspired. Per-word per-user state:

```json
{
  "interval": 1,
  "nextReview": "2026-05-04",
  "ease": 2.5,
  "reviewCount": 0
}
```

**Rating logic:**
| Rating | Action |
|--------|--------|
| Again (1) | interval = 1 day, ease -= 0.2 (min 1.3) |
| Hard (2) | interval × 1.2, ease -= 0.15 |
| Good (3) | interval × ease |
| Easy (4) | interval × ease × 1.3, ease += 0.15 |

`nextReview = today + interval` after each rating.

**Backlog cap:** Max 50 reviews per day. Overflow carries to next day, oldest first.

### 3. Session Manager

Manages the active review session for a user. Tracks which words are queued, current word index, and session state.

**KV key:** `session:{userId}`
```json
{
  "words": ["albeit", "abandon", "..."],
  "currentIndex": 0,
  "phase": "recall",
  "startedAt": "2026-05-03T08:00:00Z"
}
```

**Phase flow per word:**
1. `recall` — bot sends word + audio, "Tap to reveal" button
2. `reveal` — bot sends full card (definition, example, synonyms, antonyms), rating buttons
3. User taps rating → SRS updates → advance to next word or end session

### 4. Card Renderer

Formats Telegram messages for each phase.

**Recall message:**
```
━━━━━━━━━━━━━━━
📖 ALBEIT  /ɔːlˈbiːt/
━━━━━━━━━━━━━━━

🔊 [audio attached]

[👀 Reveal Answer]
```

**Reveal message:**
```
━━━━━━━━━━━━━━━
📖 ALBEIT  /ɔːlˈbiːt/
━━━━━━━━━━━━━━━
(conjunction) — although; even though

📝 "The pay is good, albeit less than my last job."

🔗 Synonyms: although, though

How well did you know this?
[😰 Again]  [😅 Hard]  [😊 Good]  [🎯 Easy]
```

### 5. Bot Commands

| Command | Behavior |
|---------|----------|
| `/start` | Register user → pick CEFR level → save to KV → confirm |
| `/stats` | Show: words learned, current streak, reviews today, mastery breakdown by level |
| `/settings` | Change: daily new word count (default 10), review time (default 08:00), timezone |

### 6. Cron Trigger

Single cron runs every hour. On each tick:
1. List all users from KV (via `user:*` prefix listing)
2. For each user, check if current UTC hour matches their configured local hour
3. If match: build session (due reviews + new words), save to `session:{userId}`, send first card

## Data Flow

```
Cron tick (hourly)
  → iterate users
  → check timezone match
  → query SRS records where nextReview <= today
  → add new words (up to daily limit, from user's current level)
  → cap at 50 total
  → create session in KV
  → send first recall card

User taps "Reveal"
  → webhook fires
  → read session from KV
  → send reveal card with rating buttons

User taps rating
  → webhook fires
  → update SRS record in KV
  → advance session index
  → if more words: send next recall card
  → if done: send summary ("✅ Session complete! 12 words reviewed")
```

## KV Key Design

| Key Pattern | Value | Purpose |
|-------------|-------|---------|
| `user:{telegramId}` | User settings JSON | Registration, preferences |
| `srs:{telegramId}:{word}` | SRS state JSON | Per-word review tracking |
| `session:{telegramId}` | Session state JSON | Active review session |
| `user-list` | Array of telegram IDs | Cron iteration (avoids KV list API limits) |

## Error Handling

- **Dictionary API failure during pre-fetch:** Log warning, include word without audio/synonyms
- **KV write failure:** Retry once, then skip word and log. User sees next word.
- **Telegram API failure:** Retry with exponential backoff (max 3 attempts)
- **No words due for review + no new words left at level:** Notify user they've completed current level, prompt to advance
- **User sends message mid-session:** Ignore non-button inputs during active session, reply with "Tap a button to continue"

## Testing Strategy

- **Unit tests:** SRS engine (interval calculations, edge cases like ease floor)
- **Unit tests:** Session manager (word queue building, phase transitions)
- **Integration tests:** Webhook handler with mocked Telegram API
- **Build verification:** Pre-fetch script runs against a small word subset
- **Manual testing:** Full flow via real Telegram bot in dev environment

## Tech Stack Summary

| Component | Choice |
|-----------|--------|
| Runtime | Cloudflare Workers (TypeScript) |
| Bot framework | grammY (with Workers adapter) |
| Storage | Cloudflare KV |
| Word data | Static JSON, bundled at build time |
| Dictionary source | Free Dictionary API (build-time only) |
| Word list source | Oxford 3000 (scraped once) |
| Build tool | Wrangler CLI |
| Package manager | pnpm |

## Implementation Risks

| Risk | Mitigation |
|------|------------|
| Oxford site blocks scraping | Use cached/community word lists as fallback |
| Free Dictionary API missing words | Graceful degradation — word included without audio/synonyms |
| KV free tier write limits (1k/day) | Batch writes where possible; small group stays well under limit |
| Telegram rate limits | Stagger message sends with small delays between users |
| grammY Workers adapter issues | Well-documented, active maintenance; fallback to raw fetch calls |

## Success Metrics

- Bot delivers daily sessions without manual intervention
- SRS intervals produce measurable retention (user self-report)
- Session completion rate > 70% (users finish most sessions they start)
- Zero runtime dependency on external APIs

## Next Steps

1. Scrape Oxford 3000 → `words.json`
2. Build pre-fetch script → `words-data.json`
3. Scaffold Cloudflare Workers project with grammY
4. Implement SRS engine
5. Implement session manager + card renderer
6. Wire up webhook handler + cron trigger
7. Deploy and test with real Telegram bot
