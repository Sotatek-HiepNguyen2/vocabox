# Phase 3 — Telegram Bot + Commands

## Context

- [Design Spec](../../reports/design-spec-260503-2152-vocabox-telegram-srs-bot.md)
- [plan.md](./plan.md)

## Overview

- **Priority:** P1
- **Status:** Pending
- **Effort:** 3h
- **Blocked by:** Phase 1

Set up grammY bot with Cloudflare Workers adapter, webhook handler, and core commands (`/start`, `/stats`, `/settings`). Wire up KV for user storage.

## Key Insights

- grammY has a dedicated `@grammyjs/adapter-cloudflare-workers` package
- Webhook mode: Telegram POSTs updates to Worker URL, grammY processes them
- KV operations are async — all handlers must be async
- Bot token stored in Worker secrets (not env vars)

## Requirements

**Functional:**
- `/start` — register user, prompt CEFR level selection via inline keyboard
- `/stats` — show words learned, review count, current level
- `/settings` — change daily word count, review hour, timezone
- Handle callback queries from inline keyboards
- Store/retrieve user data in KV

**Non-functional:**
- Webhook setup via wrangler secret + bot API call
- Graceful error responses to user on failures

## Files to Create

| File | Purpose |
|------|---------|
| `src/bot/bot-setup.ts` | grammY bot instance creation and middleware |
| `src/bot/commands/start-command.ts` | /start registration flow |
| `src/bot/commands/stats-command.ts` | /stats display |
| `src/bot/commands/settings-command.ts` | /settings management |
| `src/bot/callbacks/level-selection-callback.ts` | CEFR level picker handler |
| `src/kv/kv-user-store.ts` | KV read/write helpers for user data |
| `src/kv/kv-types.ts` | TypeScript types for KV stored data |

## Files to Modify

| File | Change |
|------|--------|
| `src/index.ts` | Wire bot webhook handler into fetch export |

## Implementation Steps

1. Define KV types (`src/kv/kv-types.ts`):
   ```ts
   export interface UserSettings {
     telegramId: number;
     username?: string;
     level: string;           // A1, A2, B1, B2, C1
     dailyNewWords: number;   // default 10
     reviewHour: number;      // 0-23, default 8
     timezone: string;        // e.g. "Asia/Ho_Chi_Minh"
     active: boolean;
     createdAt: string;
   }
   ```

2. Build KV user store (`src/kv/kv-user-store.ts`):
   - `getUser(kv, telegramId): Promise<UserSettings | null>`
   - `saveUser(kv, user: UserSettings): Promise<void>`
   - `getAllUserIds(kv): Promise<number[]>` — read from `user-list` key
   - `addUserToList(kv, telegramId): Promise<void>` — append to `user-list`

3. Create bot instance (`src/bot/bot-setup.ts`):
   - Factory function `createBot(token: string)` returning grammY Bot instance
   - Register command handlers
   - Register callback query handlers
   - Error handler middleware

4. Implement `/start` (`src/bot/commands/start-command.ts`):
   - Check if user already registered → welcome back message
   - New user → send inline keyboard with CEFR levels (A1, A2, B1, B2, C1)
   - Message: "Welcome to VocaBox! Pick your starting level:"

5. Implement level selection callback (`src/bot/callbacks/level-selection-callback.ts`):
   - Parse callback data `level:{A1|A2|B1|B2|C1}`
   - Create UserSettings with defaults (dailyNewWords=10, reviewHour=8, timezone=Asia/Ho_Chi_Minh)
   - Save to KV, add to user list
   - Confirm: "You're set! Starting from {level}. Your first words arrive tomorrow at 8:00 AM."

6. Implement `/stats` (`src/bot/commands/stats-command.ts`):
   - Read user settings from KV
   - Count SRS records by scanning `srs:{userId}:*` (KV list with prefix)
   - Display: level, total words seen, reviews completed, words mastered (interval >= 30)

7. Implement `/settings` (`src/bot/commands/settings-command.ts`):
   - Show current settings with inline keyboard options
   - Options: change daily count (5/10/15/20), change hour (6-22), change timezone
   - Update KV on selection

8. Wire into Worker (`src/index.ts`):
   - Import bot setup
   - In fetch handler: pass request to grammY webhookCallback
   - Type the Env bindings (VOCABOX_KV, BOT_TOKEN)

## Todo

- [ ] Create KV types and user store helpers
- [ ] Create bot setup with grammY + Workers adapter
- [ ] Implement /start command with level picker
- [ ] Implement level selection callback handler
- [ ] Implement /stats command
- [ ] Implement /settings command
- [ ] Wire webhook handler into src/index.ts
- [ ] Test locally with `wrangler dev` + ngrok/cloudflared tunnel

## Success Criteria

- `/start` registers user and saves to KV
- Level selection callback stores user with correct level
- `/stats` displays user progress
- `/settings` allows changing preferences
- Bot responds within Telegram's 10-second webhook timeout

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| grammY Workers adapter compatibility | Medium | Well-maintained package; fallback to raw fetch if needed |
| KV list operation for user scanning | Low | Use `user-list` key instead of KV list API |
| Webhook timeout (10s) | Medium | Keep handlers fast; no heavy computation in webhook path |

## Security Considerations

- Bot token stored as Cloudflare secret, never in code
- Validate callback query data format before processing
- User IDs from Telegram are trusted (Telegram signs webhook payloads)
