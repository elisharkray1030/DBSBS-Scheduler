# Night-Shift Scheduling — Solution Plan

**Team:** 3 people · **Window:** 6:30pm–11pm, Sunday–Friday (fixed) · **Plan:** monthly, 2 of 3 on every day, rotation-based
**Date:** 2026-08-14 · **Constraint:** $0/mo · **Status:** Decision-ready, pending 5 open questions

---

## Executive Summary

1. **Verdict:** build a small Telegram bot — do NOT build an app. The pain is keeping the plan updated, not missing data; a chat-native plan removes the update ritual entirely.
2. **Why:** the plan should live where the team already talks. Every change happens in chat, the bot records it and notifies everyone — no one "updates the calendar" anymore.
3. **Zero-build alternative** (shared Google Calendar + sheet) fixes visibility but keeps manual transcription — which is literally the friction you named. It's the fallback, not the pick.
4. **Stack:** TypeScript + grammY (Telegram bot) on Cloudflare Workers, data in Cloudflare D1 (SQLite). No server, no database to run, no auth to build.
5. **Cost:** $0/mo forever. Verified free caps: 100k req/day, 5 cron triggers, 500MB D1 — our usage will be under 1% of every limit.
6. **Effort:** MVP ~12–14h · v1 ~+5–8h · total ~17–22h. The MVP is two evenings plus a polish session.
7. **Maintenance:** near zero — no OS to patch, no hosting bill, deploys are `wrangler push`. This is the "boring, stable" choice.
8. **De-risk first:** run the protocol manually for one real month (pinned monthly grid + reactions) before writing any code. Confirms the rotation rule and that all 3 of you actually use Telegram.
9. **Confidence: 0.9.** Planning model now locked: availability in, mathematically equalized draft out (Q6 resolved). Remaining variance: Q1 (Telegram usage) and Q7 (swap semantics).
10. **Build only after** confirming the remaining open questions in §7 (Q1, Q4, Q5, Q7) with the team. All other decisions below are contingent on those answers.

---

## 1. Strategy

### 1.1 Build vs configure — honest assessment

The problem decomposes into three needs:

| Need | Shared GCal + Sheet | Telegram bot (this plan) |
|---|---|---|
| Visibility: who's on which day | ✅ strong | ✅ strong (`/plan` prints the week) |
| Change notifications | ⚠️ partial — GCal app/email alerts, easy to ignore | ✅ bot pings the group, hard to miss |
| Swap agreement flow | ❌ none — happens in chat, then someone transcribes | ✅ built-in: /take, /drop, /swap, confirm |
| Reminder before shift | ⚠️ per-event reminders, only if someone adds them | ✅ automatic daily 18:10 push |
| Effort to keep it truthful | ❌ *this is the pain* — someone must open GCal and edit | ✅ the chat IS the update; zero extra steps |
| Cost / maintenance | ✅ $0, zero | ✅ $0, near zero |
| Phone-native | ✅ | ✅ (it lives inside your messenger) |

**Verdict:** GCal+Sheet solves *visibility*, not *coordination*. Your stated pain — "keep them in mind, consistently update the calendar" — is the transcription ritual between chat and calendar. A bot collapses that gap: the plan is whatever the group last agreed in chat, and the bot is the source of truth. This is a case where a *tiny* build genuinely beats configuration, because the problem isn't missing software, it's a missing feedback loop.

**Anti-patterns rejected:**
- No web app/PWA (MVP): 3 users don't need a UI, sessions, or a second app to open. The group chat is the UI.
- No full scheduling optimizer beyond one rule: the month draft equalizes nights worked (greedy, deterministic, explainable). It **proposes** — the team always overrides via /off + /take. Swaps remain social decisions.

### 1.2 MVP scope

The MVP is a **shared, versioned plan in a Telegram group chat**:

1. `Month plan visible at all times` — `/plan` prints the current month; `/plan may` prints any month. **Two names per day, all 6 days (Sun–Fri) always filled** — must-cover-2 is the invariant.
2. `Availability in` — `/avail` (no args = all clear) or `/avail 3rd 5-7th` states when you can't work this month. Default: available.
3. `Equalized draft out` — `/draft <month>` generates the month: 2 on per day, availability respected, **nights mathematically equalized** (target 2×days/3, ±1). Humans keep final say — /take and /off override the draft freely.
4. `Claim and request cover` — `/take wed` fills an open slot; `/off wed` marks "I can't work Wed" and pings the group for cover. A day is never left open.
5. `Everyone knows when the plan changes` — every take/cover posts a change message ("📅 Wed 19 Aug: Kai covers Elijah"), and mentions the affected person.
6. `Nobody forgets a shift` — cron push daily at 18:10: "Tonight 18:30–23:00: Ana & Kai."
7. `Setup in under a minute` — bot added to a private group; each member sends `/whoami` once. No accounts, no passwords.

### 1.3 User stories (MVP)

- As a team member, I want to state my unavailable days once at month start so the draft respects them — `/avail 3rd 5-7th`.
- As a team member, I want the month drafted fairly — equal nights for everyone (±1), availability respected — `/draft`.
- As a team member, I want to see who's on each day of the month so I can plan around my nights — `/plan`.
- As a team member, I want to claim a day that still needs a second person — `/take wed`.
- As a team member, I want to request cover when something comes up, without leaving the day open — `/off wed` → group ping → `/take wed`.
- As a team member, I want to know the rotation is fair — balance line in `/plan` (nights worked per person).
- As a team member, I want to be told immediately when the plan changes so I never work from stale info — bot change-message.
- As a team member, I want a reminder before my shift so I don't lose track during busy weeks — 18:10 cron.

### 1.4 Explicit assumptions (each maps to an open question)

- **A1:** All 3 use Telegram daily (Q1).
- **A2:** **Resolved — exactly 2 people on every day, all 6 days.** One person is off each day.
- **A3:** **Resolved — plan horizon is one month**, re-made at month end.
- **A4:** The 18:30–23:00 window is fixed; only the *day* assignment varies (Q4 — still to confirm).
- **A5:** Last-write-wins is acceptable for a 3-person team (Q5). No locking — the group chat resolves disputes socially.
- **A6 (resolved):** Planning = availability in, mathematically equalized draft out. When everyone is free (the usual case) the draft degenerates to a clean rotation. The bot proposes, humans override via /take + /off — the team keeps final say.
- **A7 (new):** Swaps happen as cover-requests (`/off` → someone `/take`s), not day-for-day exchanges (Q7 — to confirm).

---

## 2. Solution Design

### 2.1 Delivery vehicle — trade-off table

| Option | Cost | Maintenance | Friction removed? | Verdict |
|---|---|---|---|---|
| **Telegram bot (picked)** | $0 | ~zero | Yes — chat-native plan + push notifications | ✅ **Recommend** |
| WhatsApp group + pinned message | $0 | zero | Partial — no change alerts, manual edits by one admin | Fallback pilot |
| Google Calendar + shared sheet | $0 | zero | Visibility only — transcription ritual remains | Fallback if Q1 fails |
| Tiny web app / PWA | $0 (host) | low | New app to open + accounts for 3 people | Overkill |
| Self-hosted bot on your PC (Hermes or raw) | $0 | **high** | Yes, but your machine becomes a dependency; dies when PC sleeps/reboots | Rejected |
| Existing scheduling apps (WhenIWork etc.) | usually paid | — | — | Violates $0 |

### 2.2 Why Telegram, not WhatsApp

- **Bot API is free and open** — WhatsApp's business API is not (no free tier for this; unofficial libraries break the ToS and get banned).
- **Push notifications for group messages actually work** — bots can send messages to the group at any time; members get real notifications.
- **Inline keyboards / reactions** exist for lightweight confirm flows later.
- Bots join private groups directly; the group's membership IS the auth boundary.

### 2.3 Core loop (the whole product in one sentence)

> Once a month: everyone states availability → the bot drafts an equalized plan → the team adjusts. Daily: the bot tells the group who's on tonight. Every change: the bot records it and tells the group.

---

## 3. Software Stack (all free, limits verified 2026-08-14)

| Layer | Choice | Why | Verified free-tier limit |
|---|---|---|---|
| Bot framework | **grammY** (TypeScript) | Purpose-built for Telegram; first-class Webhook + D1 story on Workers | — |
| Runtime / hosting | **Cloudflare Workers** | No server, no OS, no uptime bill; instant cold start (irrelevant for webhook) | **100,000 requests/day** · 10ms CPU/req · 128MB · 5 cron triggers/account · free plan, no card required |
| Database | **Cloudflare D1** (SQLite) | One schema, zero ops, time-travel backups | **500MB/db · 5GB/account · 10 databases (free)** · 50 queries/invocation — usage will be a few KB and 2–4 queries |
| Scheduler | **Workers Cron Triggers** | Native scheduled invocations, no extra service | 2 of our 5 free triggers used |
| Notifications | **Telegram Bot API** (free) | Push to group + mention users | Bots: unlimited group sends; rate ~30 msg/s (irrelevant at our volume) |
| Calendar API | **None in MVP** | The bot IS the calendar. Optional `.ics` export in v2 (no API key needed) | — |
| Auth | **None** | The private Telegram group + member allowlist is the auth boundary | — |
| Domain/HTTPS | `*.workers.dev` | Built-in HTTPS, no cert management | free |

**Rejected hosts and why (verified):**
- **Supabase** (free: 500MB DB, 5GB egress, 2 projects) — **free projects pause after 7 days of inactivity**; a calendar that naps for a week is disqualifying for a schedule app.
- **Vercel Hobby** (free: 100GB transfer, 1M edge reqs, 6,000 build min) — fine, but it's a general web host; Workers+D1 is fewer moving parts for a bot.
- **Render/Railway free tiers** — spin down after inactivity; a bot that cold-starts for 30s on every message is worse than broken.
- **Oracle Cloud Always Free VPS** — genuinely free forever VM, but it's a server: you patch it, you babysit it. Violates the low-maintenance constraint.

---

## 4. Architecture

### 4.1 Diagram

```
Telegram (3 phones + bot in a private group)
        │  HTTPS webhook (Telegram → workers.dev)
        ▼
Cloudflare Workers  (grammY webhook handler)
        │
        ├── D1 (SQLite): members · slots · changes
        │
        └── Cron Triggers
              ├── 18:10 daily  → "Tonight: <names>" reminder
              └── Sat 17:00    → "/plan next" nag
```

### 4.2 Data model (SQLite, D1)

```sql
CREATE TABLE members (
  id          INTEGER PRIMARY KEY,
  tg_user_id  INTEGER UNIQUE NOT NULL,   -- Telegram user id (registered via /whoami)
  name        TEXT NOT NULL              -- display name, editable via /name <text>
);

CREATE TABLE slots (                       -- one row per coverage need per day
  id          INTEGER PRIMARY KEY,
  date        TEXT NOT NULL,               -- 'YYYY-MM-DD' LOCAL date, no TZ math
  label       TEXT NOT NULL DEFAULT 'a',   -- 'a' | 'b' — the 2 duty slots
  member_id   INTEGER NOT NULL REFERENCES members(id),  -- must-cover-2: never NULL
  UNIQUE(date, label)
);
CREATE INDEX idx_slots_date ON slots(date);

CREATE TABLE changes (                     -- audit log (v1): who changed what, when
  id          INTEGER PRIMARY KEY,
  ts          TEXT NOT NULL,               -- ISO local
  tg_user_id  INTEGER NOT NULL,
  action      TEXT NOT NULL,               -- 'take' | 'drop' | 'swap' | 'plan'
  detail      TEXT
);
```

Design notes:
- **Local dates only** (`YYYY-MM-DD`), week starts Sunday. No timezone library, no DST — boring on purpose. If the team is all in one TZ this never bites.
- Exactly two rows per date, both always filled — the **must-cover-2 invariant**. Pending cover-requests live in `changes` as a pending flag, never as an empty slot.
- Balance line = `SELECT member_id, COUNT(*) FROM slots GROUP BY member_id` over the month — the fairness number at the bottom of `/plan`.

### 4.3 Core flows

**Set availability** (`/avail`)
1. `/avail` with no args = "I'm free all month". `/avail 3rd 5-7th` = unavailable on those dates.
2. Availability only feeds the next `/draft` — it never edits a live plan.

**Draft the month** (`/draft <month>` — the monthly ritual, MVP)
1. Collects everyone's availability; for each day picks the 2 available people with the lowest nights-worked count (deterministic tie-break, so re-running is stable).
2. Posts the proposed month + resulting balance. Anyone adjusts with /off + /take — the draft only proposes.
3. If a day can't get 2 available people, it's flagged: `⚠️ Sun 24 Aug: only 1 available — needs a volunteer`.
4. In the usual all-free case this is exactly the fair rotation (each person off ~2 days/week, counts within ±1).

**Claim a slot** (`/take wed`)
1. Bot resolves requester via Telegram user id → member (auto-register if unknown).
2. Fills an open slot; if the day already has 2: "Wed is full (Ana & Kai) — use /off to request cover".
3. Post to group: `📅 Wed 19 Aug: Kai takes Wed ✅ (1 open → full)`.

**Request cover** (`/off wed` — the day is never left open)
1. Only the on-duty owner can request cover (else: "You're not on Wed").
2. The slot stays assigned to the requester; bot posts: `⚠️ Wed 19 Aug needs cover — /take wed (Elijah can't)`.
3. Whoever `/take wed`'s replaces them; group sees `📅 Wed 19 Aug: Kai covers Elijah ✅`.

**View the plan** (`/plan` = current month, `/plan may` = any month)
- Print a grid (Sun→Fri, two names per day) + balance line: nights worked per person this month. Under 1,000 chars, Telegram-renderable. Pending covers shown inline: `⚠️`.

**Swap day-for-day** (`/swap` — v1, only if the team prefers exchanges over cover-requests)
- Proposes: "Elijah wants to swap Wed ↔ Thu. Kai, confirm?" with inline ✅/❌ buttons. Both confirm → both slots flip; balance unchanged.

**Reminders (cron)**
- 18:10 daily: `🌙 Tonight 18:30–23:00: Ana & Kai.`
- 18:00 bump if a cover-request is still pending: `⚠️ Wed still needs cover — /take wed`.
- Last working day of the month, 23:05: `🗓 Plan for <next month>? /draft <month> or fill it with /take`.

**Security model**
- Bot only serves the allowlisted private group chat_id; unknown chats get nothing.
- Commands from unknown user ids → "Register with /whoami".
- **Bot privacy mode must be disabled** in BotFather (`/setprivacy` → Disable) so the bot sees group messages. (Classic gotcha — see §6.)

---

## 5. Roadmap

### Phase 0 — Validate before building (2–3h, this week)
- Ask the remaining open questions (§7: Q1, Q4, Q5, Q7) with your colleagues.
- **Manual pilot:** create a private Telegram group, pin the month grid ("Sun: Ana+Kai · Mon: Kai+Ming · …"), run it for **one real month**. Cover-requests via message + reactions; balance tracked by hand at month end.
- Exit criteria: the pinned-message ritual feels *better* than the current process, all 3 people actually opened Telegram on their shift days, and the rotation stayed fair without an argument. If the group goes quiet for a week, the bot won't save you either — reconsider.

### Phase 1 — MVP (12–14h, two evenings + a polish session)
Scope: `/whoami`, `/avail`, `/draft` (greedy equalizer), `/plan` (monthly, 2-per-day, balance line), `/take`, `/off` + cover ping, change notifications, 18:10 daily reminder, D1 schema (2 slots/day NOT NULL, avail rows), member allowlist, deploy to Workers.
Acceptance:
- All 3 can register, set availability, take, and request cover from their phones.
- `/draft` equalizes: with everyone available, each person's count is within ±1 of the others; with availability holes the constraint is respected and infeasible days are flagged, never silently broken.
- Every change posts a group notification within 2 seconds.
- No day is ever left with fewer than 2 people — the invariant holds in data and UI.
- Balance line matches a hand count of nights worked.
- Daily reminder fires at 18:10 every day; month rollover renders correctly — test it explicitly.
- Bot survives 48h of real use without intervention.

### Phase 2 — v1 (+5–8h)
Scope: `/swap` with inline confirm buttons, `/name` (nicknames), audit log (`/log`), month-end plan-nag, `/help`.
Acceptance:
- Two-party swap completes with both confirmations and notifies the group.
- `/log` shows who changed what, when — disputes resolve in 30 seconds.
- No feature requires more than 2 messages from the user.

### Phase 3 — v2 (optional, +3–4h)
Scope: `.ics` export so the plan lands in Google Calendar automatically (static file, no API key), `/backup` (dumps the month to a chat message), `/newterm` (clears the month and rolls the rotation across DBSBS term boundaries).
Acceptance: teammates with a calendar habit see the week in GCal without any manual entry.

**Total effort: 17–22h** to a complete, durable solution. Cost: $0. Ongoing: nothing.

---

## 6. Risks & Gotchas

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Telegram privacy mode** left on → bot ignores all messages | certain-once | High | BotFather `/setprivacy` → Disable. Documented in setup steps, not discoverable by debugging. |
| Bot can't join a **secret** group | certain | Low | Use a *private* group (not secret). Members-only is still enforced. |
| Cloudflare free-tier drift (terms change) | low | Medium | Bot logic is ~200 lines; port to any VPS in a day. Worst case = re-type 6 lines into a pinned message. |
| D1 data loss (account lockout, etc.) | low | Low | Stakes are tiny: losing the plan = 5 minutes to re-type. `/backup` (v2) removes it entirely. |
| Free 10ms CPU/request on Workers | low | Low | We do 2–4 D1 queries/req (~2ms). Avoid heavy JSON parsing in handlers. Verified cap, not a surprise. |
| Team grows (4th person) | medium | None | Add to group + `/whoami`. Scale was never the constraint. |
| **Someone leaves mid-week** | medium | Low | `/off` their remaining days; group covers via `/take` — same as any cover request. |
| **Greedy draft ≠ perfect optimum** | low | Low | With availability holes, equalization can drift beyond ±1. The balance line makes it visible; humans override with /off + /take. If it ever matters, the solver is a ~30-line swap. |
| **Everyone silently stops using it** | medium | High | The #1 killer. Mitigated by Phase 0 pilot — the habit is validated *before* code exists. |
| WhatsApp "free" temptation | — | — | Not free at the API level, unofficial bridges break ToS. Telegram is the free, stable path. |

---

## 7. Open Questions (confirm with the team before Phase 1)

1. **Does everyone use Telegram daily?** (Decides everything. If no → fallback: GCal+Sheet.)
2. ✅ **RESOLVED — exactly 2 people on every day, Sun–Fri (6 days).** One person is off each day.
3. ✅ **RESOLVED — monthly plan**, re-made at month end.
4. **Is the 18:30–23:00 window truly fixed, or does it vary by day?** (If it varies, slots gain a `start/end` column — still trivial.)
5. **Is last-write-wins acceptable?** (No conflict resolution — the group chat settles disputes socially. If you want formal approval on every change, that's +2h on v1.)
6. ✅ **RESOLVED — availability in, mathematical equalization out.** Usually everyone's free; the draft equalizes nights (±1) and respects stated availability. Preferences = availability, nothing more.
7. **NEW — how do swaps work?** Cover-request style (`/off` → someone `/take`s) is the recommended default. Day-for-day exchange (`/swap`) is optional v1. Confirm which matches how you actually trade nights.

---

## Appendix — Verified sources (2026-08-14)

- Cloudflare Workers limits (requests, CPU, cron, memory): https://developers.cloudflare.com/workers/platform/limits/
- Cloudflare D1 limits (500MB free, 50 queries/invocation): https://developers.cloudflare.com/d1/platform/limits/
- Supabase free tier (500MB DB, 5GB egress, **paused after 1 week inactivity**, 2 projects): https://supabase.com/pricing
- Telegram bot group privacy mode (commands-only unless disabled): https://core.telegram.org/bots/features · https://www.teleme.io/articles/group_privacy_mode_of_telegram_bots
- Vercel Hobby (100GB transfer, 1M edge requests, 6,000 build min, personal non-commercial): https://vercel.com/docs/plans/hobby

*Confidence: overall 0.9. Remaining variance: Q1 (Telegram usage) and Q7 (swap semantics); each moves it ±0.05.*

---

## Update log
- 2026-08-14: Requirements refined — monthly plan, exactly 2 of 3 on duty every day (Sun–Fri), rotation-based. Plan updated throughout: flows (`/off` cover-requests replace `/drop`), data model (2 filled slots/day, balance line), roadmap (month pilot, `/draft` generator in v1), effort 15–20h, confidence 0.85.
- 2026-08-14 (2nd): Rotation rule locked — preferences = availability, the draft equalizes nights mathematically (±1), usually a clean rotation. `/avail` added; `/draft` promoted from v1 to MVP; effort 17–22h; confidence 0.9.
