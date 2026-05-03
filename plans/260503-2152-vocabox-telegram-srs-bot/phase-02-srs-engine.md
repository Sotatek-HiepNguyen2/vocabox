# Phase 2 — SRS Engine

## Context

- [Design Spec](../../reports/design-spec-260503-2152-vocabox-telegram-srs-bot.md)
- [plan.md](./plan.md)

## Overview

- **Priority:** P1
- **Status:** Pending
- **Effort:** 2h
- **Blocked by:** Phase 1

Pure logic module implementing SM-2 inspired spaced repetition. No I/O — takes current state + rating, returns new state.

## Key Insights

- SM-2 ease factor has a floor of 1.3 to prevent intervals from shrinking too fast
- Module must be pure (no KV reads/writes) for testability
- Interval rounding to whole days keeps scheduling simple

## Requirements

**Functional:**
- Calculate next interval and ease based on rating (1-4)
- Compute next review date from today + interval
- Provide initial state for new words
- Cap daily reviews at 50, overflow to next day

**Non-functional:**
- Zero dependencies (pure math)
- 100% unit testable
- Exported types for SRS state

## Files to Create

| File | Purpose |
|------|---------|
| `src/srs/srs-engine.ts` | Core SRS calculation logic |
| `src/srs/srs-types.ts` | TypeScript types for SRS state and ratings |

## Implementation Steps

1. Define types (`src/srs/srs-types.ts`):
   ```ts
   export type Rating = 1 | 2 | 3 | 4; // Again, Hard, Good, Easy
   
   export interface SrsState {
     interval: number;      // days until next review
     nextReview: string;    // ISO date string YYYY-MM-DD
     ease: number;          // ease factor (min 1.3)
     reviewCount: number;
   }
   ```

2. Implement engine (`src/srs/srs-engine.ts`):
   - `createInitialState(today: string): SrsState` — interval=0, ease=2.5, nextReview=today
   - `calculateNextState(current: SrsState, rating: Rating, today: string): SrsState`
     - Again (1): interval=1, ease -= 0.2 (min 1.3)
     - Hard (2): interval = Math.ceil(current.interval * 1.2), ease -= 0.15 (min 1.3)
     - Good (3): interval = Math.ceil(current.interval * current.ease)
     - Easy (4): interval = Math.ceil(current.interval * current.ease * 1.3), ease += 0.15
     - First review (interval=0): Again→1d, Hard→2d, Good→3d, Easy→5d
   - `isDue(state: SrsState, today: string): boolean` — nextReview <= today
   - `addDays(dateStr: string, days: number): string` — date arithmetic helper

3. Export from barrel: `src/srs/index.ts`

## Todo

- [ ] Create SRS types
- [ ] Implement SRS engine with all rating calculations
- [ ] Handle first-review edge case (interval=0)
- [ ] Implement isDue helper
- [ ] Export from barrel file

## Success Criteria

- All rating paths produce correct intervals (verified by unit tests in Phase 6)
- Ease never drops below 1.3
- First-review case handled separately
- Pure functions, no side effects

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Interval grows too fast | Low | Cap max interval at 365 days |
| Date arithmetic bugs | Medium | Use simple string-based YYYY-MM-DD math, test edge cases (month/year boundaries) |
