# Phase 6 — Testing + Deploy

## Context

- [Design Spec](../../reports/design-spec-260503-2152-vocabox-telegram-srs-bot.md)
- [plan.md](./plan.md)

## Overview

- **Priority:** P1
- **Status:** Pending
- **Effort:** 2h
- **Blocked by:** Phase 5

Unit tests for SRS engine and session logic, integration test for webhook handler, deploy to Cloudflare Workers, set webhook URL with Telegram.

## Key Insights

- Vitest works well with Cloudflare Workers via `@cloudflare/vitest-pool-workers`
- SRS engine is pure logic — easiest to test thoroughly
- Webhook integration test can use grammY's test utilities
- Deploy requires: wrangler login, KV namespace creation, bot token secret

## Requirements

**Functional:**
- Unit tests for SRS engine covering all 4 ratings + edge cases
- Unit tests for word queue builder
- Unit tests for timezone utils
- Integration test for webhook handler
- Successful deploy to Cloudflare Workers
- Telegram webhook URL configured

**Non-functional:**
- Tests run via `pnpm test`
- CI-friendly (no external dependencies needed for unit tests)

## Files to Create

| File | Purpose |
|------|---------|
| `vitest.config.ts` | Vitest configuration |
| `tests/srs-engine.test.ts` | SRS engine unit tests |
| `tests/word-queue-builder.test.ts` | Queue builder tests |
| `tests/timezone-utils.test.ts` | Timezone conversion tests |

## Files to Modify

| File | Change |
|------|--------|
| `package.json` | Add vitest dependency and test script |

## Implementation Steps

1. Setup Vitest:
   - Install `vitest` as devDependency
   - Create `vitest.config.ts` with TypeScript support
   - Add `"test": "vitest run"` and `"test:watch": "vitest"` to package.json

2. SRS engine tests (`tests/srs-engine.test.ts`):
   - Test `createInitialState` returns correct defaults
   - Test each rating (1-4) produces expected interval and ease
   - Test ease floor at 1.3 (repeated "Again" ratings)
   - Test first-review special case (interval=0)
   - Test `isDue` with past, today, and future dates
   - Test interval cap at 365 days
   - Test date arithmetic across month/year boundaries

3. Word queue builder tests (`tests/word-queue-builder.test.ts`):
   - Test due reviews sorted oldest first
   - Test backlog cap at 50
   - Test new words fill remaining slots
   - Test empty queue (no reviews, no new words)
   - Test mixed queue (reviews + new words)
   - Mock KV with in-memory Map

4. Timezone utils tests (`tests/timezone-utils.test.ts`):
   - Test UTC+7 conversion (Vietnam)
   - Test UTC+0 (London)
   - Test negative offset (US timezones)
   - Test hour boundary (23:00 UTC → next day in positive offset)

5. Deploy:
   - `wrangler login`
   - Create KV namespace: `wrangler kv namespace create VOCABOX_KV`
   - Update `wrangler.toml` with namespace ID
   - Set bot token: `wrangler secret put BOT_TOKEN`
   - Deploy: `wrangler deploy`
   - Set webhook: `curl https://api.telegram.org/bot{TOKEN}/setWebhook?url=https://vocabox.{account}.workers.dev`

6. Smoke test:
   - Send `/start` to bot in Telegram
   - Select a level
   - Verify user saved in KV via `wrangler kv key get`
   - Trigger cron manually: `wrangler dev --test-scheduled`
   - Verify session card received

## Todo

- [ ] Setup Vitest config and dependencies
- [ ] Write SRS engine unit tests
- [ ] Write word queue builder tests
- [ ] Write timezone utils tests
- [ ] Run tests, verify all pass
- [ ] Create KV namespace on Cloudflare
- [ ] Deploy Worker
- [ ] Set Telegram webhook URL
- [ ] Smoke test full flow in Telegram

## Success Criteria

- All unit tests pass
- SRS engine has full coverage of rating paths and edge cases
- Worker deploys without errors
- Bot responds to `/start` in Telegram
- Cron trigger delivers session cards

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Vitest Workers pool compatibility | Low | Use standard Vitest for pure logic; Workers pool only if needed |
| Deploy fails due to config | Medium | Verify wrangler.toml before deploy; check free tier limits |
| Webhook URL not reachable | Low | Workers URLs are public by default; verify with curl |
