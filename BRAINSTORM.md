# Oxford Vocabulary Learning Bot — Project Brainstorm

## 🎯 Project Goal

Build a **free vocabulary learning system** using the Oxford 3000 word list, delivered via a **Telegram Bot** with spaced repetition scheduling (SRS). No AI needed in the app itself — AI (Claude Code) is only used during development.

---

## 📚 Word Source

- **Oxford 3000**: https://www.oxfordlearnersdictionaries.com/wordlist/american_english/oxford3000/
- **Oxford 5000** (phase 2): same site, 5000 list
- Scrape once → save as static `words.json`
- Each word entry should include: `word`, `level` (A1/A2/B1/B2/C1/C2), `part_of_speech`

---

## 🔊 Pronunciation & Definitions — Free Dictionary API

No paid API needed. Use the **Free Dictionary API**:

```
GET https://api.dictionaryapi.dev/api/v2/entries/en/{word}
```

Returns:
- IPA phonetic string (e.g. `/ɔːlˈbiːt/`)
- MP3 audio URL (US & UK pronunciation)
- Definition
- Example sentence
- Part of speech
- Synonyms

No API key required. 100% free.

---

## 📬 Delivery — Telegram Bot

**Why Telegram:**
- Free Bot API, no approval needed
- Supports inline buttons (perfect for SRS ratings)
- Can send audio messages (pronunciation MP3)
- Bot messages the USER — no need for user to remember to open an app
- Works globally, works on mobile natively

**How to create a bot:**
1. Open Telegram → search `@BotFather`
2. Send `/newbot` → follow steps
3. Receive a token like `7412938:AAFxyz...`
4. Use that token in your backend to control the bot

---

## 🧠 Core Algorithm — Spaced Repetition (SRS)

Use a simple **SM-2-inspired** interval system:

| Rating | Next Review |
|--------|-------------|
| 😰 Again (forgot) | 1 day |
| 😅 Hard | 3 days |
| 😊 Good | 7 days |
| 🎯 Easy | 14 days → doubles each time |

Each word per user stores:
- `interval` (days until next review)
- `next_review_date`
- `ease_factor` (optional for full SM-2)
- `review_count`

---

## 🤖 Telegram Bot Daily Flow

Every morning at a user-configured time (default 8:00 AM), the bot sends:

```
🤖 YourVocabBot

Good morning! 5 words to review today ☀️

━━━━━━━━━━━━━━━
📖 ALBEIT  /ɔːlˈbiːt/
━━━━━━━━━━━━━━━
(conjunction) — although; even though

Example: "The pay is good, albeit less than my last job."

🔊 [audio message attached]

Did you know this word?
[😰 Again]  [😅 Hard]  [😊 Good]  [🎯 Easy]
```

User taps a button → SRS updates → next word loads automatically.

**Bot Commands:**
- `/start` — register, begin onboarding
- `/stats` — show progress (words learned, streak, mastered count)
- `/settings` — change daily word count, review time
- `/skip` — skip today's session
- `/pause` — pause for N days

---

## 🏗️ Architecture — 100% Free Stack

```
┌─────────────────────────────────┐
│  Static JSON (Oxford 3000)      │
│  words.json — hosted on GitHub  │
└────────────────┬────────────────┘
                 │
┌────────────────▼────────────────┐
│  Backend (Cloudflare Workers)   │
│  - Daily cron job (8 AM)        │
│  - SRS scheduling logic         │
│  - Telegram Bot API calls       │
│  - Fetches audio from Dict API  │
└────────────────┬────────────────┘
                 │ Telegram Bot API
┌────────────────▼────────────────┐
│  Telegram Bot                   │
│  - Sends word cards with audio  │
│  - Handles button callbacks     │
│  - Updates SRS schedule         │
└─────────────────────────────────┘
                 │ (optional later)
┌────────────────▼────────────────┐
│  Web Dashboard (GitHub Pages)   │
│  - Visual progress chart        │
│  - Word list browser            │
│  - Heatmap / streak             │
└─────────────────────────────────┘
```

---

## 💾 Free Hosting Plan

| Component | Free Service |
|-----------|-------------|
| Word list JSON | GitHub (raw file) |
| Backend + cron | Cloudflare Workers (free tier: 100k req/day) |
| Database | Cloudflare KV (free tier) or Supabase free tier |
| Web dashboard | GitHub Pages or Vercel |
| Telegram bot | BotFather — completely free |
| Pronunciation audio | Free Dictionary API — no key needed |

---

## 📐 Data Schema

### `words.json`
```json
[
  {
    "word": "albeit",
    "level": "B2",
    "pos": "conjunction"
  }
]
```

### User SRS record (per word per user, stored in KV/DB)
```json
{
  "user_id": "123456789",
  "word": "albeit",
  "interval": 7,
  "next_review": "2025-05-10",
  "review_count": 3,
  "ease": 2.5
}
```

### User settings
```json
{
  "user_id": "123456789",
  "daily_new_words": 10,
  "review_time": "08:00",
  "timezone": "Asia/Ho_Chi_Minh",
  "active": true
}
```

---

## 🚀 Build Phases

### Phase 1 — MVP (Bot only)
- [ ] Scrape Oxford 3000 → `words.json`
- [ ] Create Telegram bot via BotFather
- [ ] Set up Cloudflare Worker project
- [ ] `/start` command — register user
- [ ] Daily cron — sends today's due words
- [ ] Fetch audio from Free Dictionary API, send as Telegram voice message
- [ ] Inline buttons: Again / Hard / Good / Easy
- [ ] SRS logic updates schedule on button tap
- [ ] `/stats` command

### Phase 2 — Polish
- [ ] `/settings` — daily count, time, timezone
- [ ] Onboarding quiz — "how many words do you already know?" fast-track
- [ ] `/skip` and `/pause` commands
- [ ] Oxford 5000 unlock after Oxford 3000 completion

### Phase 3 — Web Dashboard (optional)
- [ ] Progress heatmap
- [ ] Words browser (filter by level, known/unknown)
- [ ] Streak leaderboard

---

## 🛠️ Suggested Tech Stack

**Option A — JavaScript (recommended for Cloudflare Workers)**
- Runtime: Cloudflare Workers (JS/TS native)
- Bot framework: `grammy` or `telegraf` (Node-compatible)
- Storage: Cloudflare KV

**Option B — Python**
- Runtime: Vercel serverless functions or Railway free tier
- Bot framework: `python-telegram-bot`
- Storage: Supabase (PostgreSQL, free tier)

---

## 📝 First Task for Claude Code

1. Scrape https://www.oxfordlearnersdictionaries.com/wordlist/american_english/oxford3000/ and output a clean `words.json` with fields: `word`, `level`, `pos`
2. Scaffold a Cloudflare Workers project with:
   - `wrangler.toml` config
   - KV namespace for user data
   - Cron trigger at `0 1 * * *` (UTC, = 8 AM Vietnam time UTC+7)
   - Telegram webhook handler
3. Implement the SRS scheduling module
4. Implement the word card sender (fetch audio from dictionaryapi.dev, send via Telegram)