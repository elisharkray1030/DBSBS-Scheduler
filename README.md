# DBSBS Night-Shift Scheduler

Night-shift scheduling for a 3-person team via a Telegram bot — the plan lives in the chat where the team already talks. No app, no server, no cost.

- **Team:** 3 people
- **Window:** 6:30pm–11pm, Sunday–Thursday; Friday 6:30–7:30pm
- **Plan:** monthly, 2 of 3 on duty every day, rotation-based
- **Cost:** $0/mo

## The problem

Keeping the monthly night-shift plan visible and up to date. The pain is the transcription ritual between chat and calendar, not missing data.

## The solution

A small Telegram bot that is the calendar:

- **`/plan`** — print the current (or any) month, 2 names per day
- **`/avail`** — state unavailable days at month start
- **`/draft`** — generate a month with nights mathematically equalized (±1)
- **`/take`** / **`/off`** — claim a slot or request cover; a day is never left open
- **Change notifications** — every take/cover posts to the group and mentions who's affected
- **Daily reminder** — cron push at 18:10: "Tonight 18:30–23:00: …"

## Stack

TypeScript + [grammY](https://grammy.dev) on [Cloudflare Workers](https://workers.cloudflare.com), data in [Cloudflare D1](https://developers.cloudflare.com/d1/). Zero server, zero ops, free forever.

## Status

Decision-ready solution plan, pending 2 open questions with the team. See [PLAN.md](PLAN.md) for the full plan, architecture, roadmap (MVP ~12–14h), and risks.

Before any code is written, run the protocol manually for one real month (pinned monthly grid + reactions) to validate the rotation rule and that all 3 teammates actually use Telegram.

## Getting started

Nothing to install yet — the plan is pre-build. Once the open questions in [PLAN.md §7](PLAN.md) are confirmed with the team, the MVP is built and deployed with `wrangler push`.
