# Phase 4 — Session Manager + Card Renderer

## Context

- [Design Spec](../../reports/design-spec-260503-2152-vocabox-telegram-srs-bot.md)
- [plan.md](./plan.md)

## Overview

- **Priority:** P1
- **Status:** Pending
- **Effort:** 3h
- **Blocked by:** Phase 2, Phase 3

Manages active review sessions per user. Builds word queues from due reviews + new words, tracks progress through the session, renders recall/reveal cards, and handles rating callbacks.

## Key Insights

- Session state lives in KV — stateless Worker reads/writes it per request
- Two-phase card flow: recall (word + audio) → reveal (full card + rating buttons)
- Audio sent as Telegram voice message via URL — no need to download/re-upload
- Callback data must be compact (Telegram 64-byte limit)

## Requirements

**Functional:**
- Build session: collect due reviews + new words up to daily limit, cap at 50
- Send recall card: word, phonetic, audio
- Send reveal card: definition, example, synonyms, antonyms, rating buttons
- Process rating: update SRS state, advance to next word or end session
- Session summary on completion

**Non-functional:**
- Callback data fits in 64 bytes
- Session survives Worker restarts (persisted in KV)
- Handle out-of-order button taps gracefully

## Files to Create

| File | Purpose |
|------|---------|
| `src/session/session-manager.ts` | Session lifecycle: create, advance, complete |
| `src/session/session-types.ts` | Session state types |
| `src/session/word-queue-builder.ts` | Build word queue from SRS state + new words |
| `src/bot/cards/recall-card.ts` | Format recall phase message |
| `src/bot/cards/reveal-card.ts` | Format reveal phase message + rating buttons |
| `src/bot/callbacks/reveal-callback.ts` | Handle "Reveal" button tap |
| `src/bot/callbacks/rating-callback.ts` | Handle rating button tap |
| `src/kv/kv-srs-store.ts` | KV read/write helpers for SRS records |

## Implementation Steps

1. Define session types (`src/session/session-types.ts`):
   ```ts
   export interface Session {
     words: string[];       // ordered word list for this session
     currentIndex: number;
     phase: "recall" | "reveal";
     startedAt: string;     // ISO timestamp
     reviewedCount: number;
   }
   ```

2. Build KV SRS store (`src/kv/kv-srs-store.ts`):
   - `getSrsState(kv, userId, word): Promise<SrsState | null>`
   - `saveSrsState(kv, userId, word, state: SrsState): Promise<void>`
   - `getDueWords(kv, userId, today: string): Promise<string[]>` — list keys with prefix `srs:{userId}:`, filter by isDue
   - `getReviewedWords(kv, userId): Promise<string[]>` — all words user has seen

3. Build word queue (`src/session/word-queue-builder.ts`):
   - `buildQueue(kv, userId, userLevel, dailyNewWords, today): Promise<string[]>`
   - Get due reviews (oldest nextReview first)
   - Cap reviews at 50
   - Fill remaining slots with new words from user's level (not yet in SRS)
   - New words selected sequentially from words-data.json by level
   - Return combined list: reviews first, then new words

4. Implement session manager (`src/session/session-manager.ts`):
   - `createSession(kv, userId, words): Promise<Session>` — save to KV
   - `getSession(kv, userId): Promise<Session | null>`
   - `advanceSession(kv, userId): Promise<Session | null>` — increment index, set phase to recall
   - `completeSession(kv, userId): Promise<void>` — delete session key
   - `setPhase(kv, userId, phase): Promise<void>`

5. Build recall card (`src/bot/cards/recall-card.ts`):
   - Lookup word in words-data.json
   - Format message: word in caps, phonetic
   - Send audio via `sendAudio` if audioUrl exists
   - Inline keyboard: single "Reveal Answer" button
   - Callback data: `rv:{wordIndex}` (compact)

6. Build reveal card (`src/bot/cards/reveal-card.ts`):
   - Full card: word, phonetic, POS, definition, example, synonyms, antonyms
   - Inline keyboard: 4 rating buttons
   - Callback data: `rt:{wordIndex}:{rating}` (e.g. `rt:3:2` = word index 3, Hard)

7. Implement reveal callback (`src/bot/callbacks/reveal-callback.ts`):
   - Parse `rv:{index}` from callback data
   - Update session phase to "reveal"
   - Send reveal card for current word
   - Answer callback query

8. Implement rating callback (`src/bot/callbacks/rating-callback.ts`):
   - Parse `rt:{index}:{rating}` from callback data
   - Get current SRS state (or create initial for new words)
   - Calculate next state via SRS engine
   - Save updated SRS state to KV
   - Advance session to next word
   - If more words: send next recall card
   - If done: send summary message, delete session

9. Register callbacks in bot setup:
   - Add `bot.callbackQuery(/^rv:/)` for reveal
   - Add `bot.callbackQuery(/^rt:/)` for rating

## Todo

- [ ] Create session types
- [ ] Build KV SRS store helpers
- [ ] Build word queue builder
- [ ] Implement session manager
- [ ] Build recall card renderer
- [ ] Build reveal card renderer
- [ ] Implement reveal callback handler
- [ ] Implement rating callback handler
- [ ] Register callbacks in bot setup

## Success Criteria

- Session creates with correct word queue (reviews + new words)
- Recall card shows word + audio + reveal button
- Reveal card shows full info + 4 rating buttons
- Rating updates SRS state correctly
- Session advances through all words and shows summary
- Callback data fits within 64-byte Telegram limit

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| KV list for due words is slow with many records | Medium | Acceptable for small group; D1 migration path for scale |
| Audio URL expired or broken | Low | Skip audio, send text-only card |
| Concurrent button taps cause race condition | Low | KV eventual consistency acceptable; last-write-wins is fine |

## Security Considerations

- Validate callback data format before parsing
- Verify session belongs to the user making the callback
- Don't expose internal word indices in user-facing messages
