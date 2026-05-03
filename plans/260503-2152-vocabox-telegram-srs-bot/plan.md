---
title: "VocaBox Telegram SRS Bot"
description: "Oxford 3000 vocabulary bot with spaced repetition via Telegram on Cloudflare Workers"
status: pending
priority: P1
effort: 16h
tags: [feature, backend, infra]
created: 2026-05-03
---

# VocaBox — Implementation Plan

## Overview

Build a Telegram bot delivering Oxford 3000 vocabulary via spaced repetition. Cloudflare Workers runtime, grammY framework, KV storage, pre-fetched dictionary data. Active recall flow: word + audio → reveal → rate.

## Design Spec

[design-spec](../reports/design-spec-260503-2152-vocabox-telegram-srs-bot.md)

## Phases

| # | Phase | Status | Effort | Link |
|---|-------|--------|--------|------|
| 1 | Project scaffold + word data pipeline | Pending | 4h | [phase-01](./phase-01-scaffold-and-word-data.md) |
| 2 | SRS engine | Pending | 2h | [phase-02](./phase-02-srs-engine.md) |
| 3 | Telegram bot + commands | Pending | 3h | [phase-03](./phase-03-telegram-bot-and-commands.md) |
| 4 | Session manager + card renderer | Pending | 3h | [phase-04](./phase-04-session-and-cards.md) |
| 5 | Cron trigger + delivery | Pending | 2h | [phase-05](./phase-05-cron-and-delivery.md) |
| 6 | Testing + deploy | Pending | 2h | [phase-06](./phase-06-testing-and-deploy.md) |

## Dependencies

- Telegram bot token from BotFather (user must create before Phase 3)
- Cloudflare account with Workers + KV enabled (free tier)
- Phase 1 must complete before all others (word data is foundational)
- Phase 2 before Phase 4 (session manager uses SRS engine)
- Phase 3 before Phase 4 (bot instance needed for card sending)

## Execution Order

```
Phase 1 (scaffold + data)
  ├── Phase 2 (SRS engine) ──┐
  └── Phase 3 (bot + cmds) ──┤
                              └── Phase 4 (session + cards)
                                    └── Phase 5 (cron + delivery)
                                          └── Phase 6 (testing + deploy)
```
