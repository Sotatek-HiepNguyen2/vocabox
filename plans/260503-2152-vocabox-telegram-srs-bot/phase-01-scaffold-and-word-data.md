# Phase 1 — Project Scaffold + Word Data Pipeline

## Context

- [Design Spec](../../reports/design-spec-260503-2152-vocabox-telegram-srs-bot.md)
- [plan.md](./plan.md)

## Overview

- **Priority:** P1 (blocking all other phases)
- **Status:** Pending
- **Effort:** 4h

Scaffold Cloudflare Workers project with TypeScript, configure wrangler, and build the word data pipeline: scrape Oxford 3000 list, enrich via Free Dictionary API, output `words-data.json`.

## Key Insights

- Oxford 3000 page uses server-rendered HTML — standard fetch + parse works
- Free Dictionary API has no rate limit docs but ~3000 sequential calls need throttling (~1-2 req/sec)
- Some words may not exist in Dictionary API — graceful fallback to Oxford-only data
- `words-data.json` bundles with the Worker as a static import

## Requirements

**Functional:**
- Scrape Oxford 3000 word list (word, level, part of speech)
- Fetch dictionary data for each word (definition, example, phonetic, audio URL, synonyms, antonyms)
- Output `src/data/words-data.json` with merged data
- Handle missing dictionary entries gracefully

**Non-functional:**
- Pipeline script runs locally via `pnpm run fetch-words`
- Throttle API calls to avoid rate limiting
- Log progress and failures

## Files to Create

| File | Purpose |
|------|---------|
| `wrangler.toml` | Workers config with KV bindings and cron trigger |
| `tsconfig.json` | TypeScript config |
| `src/index.ts` | Worker entry point (stub) |
| `scripts/scrape-oxford-3000.ts` | Scrape word list → `data/oxford-3000.json` |
| `scripts/fetch-dictionary-data.ts` | Enrich words via Dictionary API → `src/data/words-data.json` |
| `src/data/words-data.json` | Final bundled word data (generated) |
| `.dev.vars` | Local dev secrets (bot token) |
| `.gitignore` | Ignore node_modules, .dev.vars, .wrangler |

## Implementation Steps

1. Init project:
   - Update `package.json` with scripts, dependencies (`wrangler`, `grammy`, `@grammyjs/runner`)
   - Add devDependencies: `typescript`, `tsx`, `cheerio` (for scraping), `@cloudflare/workers-types`
   - Create `tsconfig.json` targeting ES2022, module NodeNext
   - Create `.gitignore`

2. Configure Cloudflare Workers:
   - Create `wrangler.toml` with:
     - `name = "vocabox"`
     - `main = "src/index.ts"`
     - `compatibility_date` = current
     - KV namespace binding: `VOCABOX_KV`
     - Cron trigger: `crons = ["0 * * * *"]` (every hour)
   - Create stub `src/index.ts` exporting fetch + scheduled handlers

3. Build Oxford 3000 scraper (`scripts/scrape-oxford-3000.ts`):
   - Fetch the Oxford 3000 wordlist page
   - Parse HTML with cheerio to extract: word, CEFR level, part of speech
   - Output `data/oxford-3000.json`
   - Log count and any parsing issues

4. Build dictionary enrichment script (`scripts/fetch-dictionary-data.ts`):
   - Read `data/oxford-3000.json`
   - For each word, call `https://api.dictionaryapi.dev/api/v2/entries/en/{word}`
   - Extract: first definition, first example, phonetic, audio URL (prefer US), up to 2 synonyms, antonyms
   - Throttle: 2 requests/sec with delay
   - On API failure: keep word with Oxford data only, log warning
   - Output `src/data/words-data.json`
   - Log summary: total, enriched, fallback count

5. Add npm scripts:
   - `"scrape": "tsx scripts/scrape-oxford-3000.ts"`
   - `"enrich": "tsx scripts/fetch-dictionary-data.ts"`
   - `"fetch-words": "pnpm scrape && pnpm enrich"`
   - `"dev": "wrangler dev"`
   - `"deploy": "wrangler deploy"`

## Todo

- [ ] Update package.json with dependencies and scripts
- [ ] Create tsconfig.json
- [ ] Create wrangler.toml with KV + cron config
- [ ] Create .gitignore
- [ ] Create stub src/index.ts
- [ ] Build Oxford 3000 scraper script
- [ ] Build dictionary enrichment script
- [ ] Run pipeline, verify words-data.json output
- [ ] Verify `wrangler dev` starts without errors

## Success Criteria

- `pnpm fetch-words` produces `src/data/words-data.json` with ~3000 entries
- Each entry has at minimum: word, level, pos
- Enriched entries additionally have: definition, example, phonetic, audioUrl, synonyms
- `wrangler dev` starts and responds to HTTP requests
- KV namespace configured in wrangler.toml

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Oxford site blocks scraping | High | Use community word lists as fallback; cache scraped data |
| Dictionary API rate limits | Medium | Throttle to 2 req/sec; resume from last position on failure |
| Some words missing from API | Low | Graceful fallback — word included without enrichment |

## Security Considerations

- `.dev.vars` contains bot token — must be in `.gitignore`
- No secrets committed to repo
- Scraping scripts run locally only, not in production
