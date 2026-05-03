# Phase 5 — Cron Trigger + Delivery

## Context

- [Design Spec](../../reports/design-spec-260503-2152-vocabox-telegram-srs-bot.md)
- [plan.md](./plan.md)

## Overview

- **Priority:** P1
- **Status:** Pending
- **Effort:** 2h
- **Blocked by:** Phase 4

Implement the hourly cron trigger that iterates users, checks timezone match, builds sessions, and sends the first card. This is the "push" mechanism that makes the bot proactive.

## Key Insights

- Single cron fires every hour at :00 UTC
- For each user, compare current UTC hour against user's configured local hour
- Cloudflare Workers cron has 30-second CPU time limit — must be efficient
- Skip users who already have an active session (avoid duplicate sends)

## Requirements

**Functional:**
- Hourly cron iterates all registered users
- Deliver session only when UTC hour matches user's local review hour
- Build word queue and create session for matched users
- Send first recall card to start the session
- Skip users with existing active sessions

**Non-functional:**
- Complete within Workers cron CPU limit (30s)
- Stagger Telegram API calls to avoid rate limits (30 msgs/sec)
- Log delivery results for debugging

## Files to Create

| File | Purpose |
|------|---------|
| `src/cron/cron-handler.ts` | Scheduled event handler |
| `src/cron/timezone-utils.ts` | UTC-to-local hour matching |

## Files to Modify

| File | Change |
|------|--------|
| `src/index.ts` | Wire scheduled handler export |

## Implementation Steps

1. Build timezone utils (`src/cron/timezone-utils.ts`):
   - `getLocalHour(utcHour: number, timezone: string): number` — convert UTC hour to local hour
   - Use `Intl.DateTimeFormat` with timeZone option (available in Workers runtime)
   - Handle DST transitions automatically via Intl API

2. Implement cron handler (`src/cron/cron-handler.ts`):
   ```ts
   export async function handleScheduled(env: Env): Promise<void> {
     const now = new Date();
     const utcHour = now.getUTCHours();
     const today = now.toISOString().slice(0, 10);
     
     const userIds = await getAllUserIds(env.VOCABOX_KV);
     
     for (const userId of userIds) {
       // 1. Get user settings
       // 2. Check if local hour matches review hour
       // 3. Skip if active session exists
       // 4. Build word queue
       // 5. Create session
       // 6. Send first recall card via Telegram API
     }
   }
   ```

3. Delivery logic per user:
   - Get user settings from KV
   - Skip if `active === false`
   - Compute local hour from UTC + user timezone
   - Skip if local hour !== user's reviewHour
   - Check for existing session — skip if found
   - Build word queue via `buildQueue()`
   - If queue empty (no due reviews, no new words at level): skip, optionally notify level complete
   - Create session in KV
   - Send first recall card

4. Stagger sends:
   - Small delay between users (50ms) to stay under Telegram rate limits
   - For small group (<50 users) this adds <2.5s total — well within cron limits

5. Wire into Worker (`src/index.ts`):
   - Export `scheduled` handler that calls `handleScheduled(env)`

## Todo

- [ ] Build timezone utility with Intl API
- [ ] Implement cron handler with user iteration
- [ ] Add active session check (skip if exists)
- [ ] Add staggered sending between users
- [ ] Wire scheduled export in src/index.ts
- [ ] Test with `wrangler dev --test-scheduled`

## Success Criteria

- Cron fires hourly and correctly matches users by timezone
- Users receive first recall card at their configured hour
- No duplicate sessions created
- Inactive users and users with existing sessions are skipped
- Completes within 30s CPU time for <50 users

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Intl API timezone edge cases | Low | Well-supported in Workers; test DST transitions |
| Cron CPU timeout with many users | Low | <50 users, well within limits; batch if needed later |
| Telegram rate limit (30 msg/sec) | Low | 50ms delay between users keeps well under limit |

## Security Considerations

- Cron handler has no external input — no injection risk
- Bot token accessed from env secrets only
