# Hermes Setup Runbook

Step-by-step to a working assistant: Telegram first (day one), Google
Workspace second, iMessage via Photon third. Run on the target host — a small
VPS, or a Mac/Linux box that stays on.

> Commands below were cross-checked against the Hermes docs in July 2026 but
> the project moves fast — if a command errors, check
> https://hermes-agent.nousresearch.com/docs/getting-started/quickstart

---

## 0. Prerequisites

- Linux, macOS, or WSL2 host that stays running (gateway is a background
  process).
- Anthropic API key (or OpenRouter) — long-context model recommended.
- Telegram account (for the first channel).
- Access to myacaexpress@gmail.com (Google OAuth).

## 1. Install Hermes

```bash
curl -fsSLO https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh
less install.sh          # read before running
bash install.sh
source ~/.bashrc          # or ~/.zshrc
hermes --version
```

Configure the model provider when prompted (onboarding), or later:

```bash
hermes model              # pick provider + model, e.g. Anthropic Claude Sonnet
```

## 2. Telegram channel (~5 minutes)

1. In Telegram, message **@BotFather** → `/newbot` → pick a display name
   (e.g. "Shawn AI") and a username ending in `bot`. Save the token —
   anyone holding it controls the bot (`/revoke` in BotFather if it leaks).
2. Message **@userinfobot** to get your numeric Telegram user ID. Get
   Patty's the same way.
3. Run the wizard:

```bash
hermes gateway setup      # choose Telegram, paste token, allowed user IDs
hermes gateway start
hermes gateway status
```

Equivalent env config lands in `~/.hermes/.env`:
`TELEGRAM_BOT_TOKEN=...` and `TELEGRAM_ALLOWED_USERS=<shawn_id>,<patty_id>`.

4. Message the bot — it should answer within seconds.
5. Optional (group chats, e.g. a Shawn+Patty+bot planning group): in
   BotFather run `/setprivacy` → **off** so the bot sees all group messages.

**Allowlist policy:** start with Shawn + Patty only. Add specific helpers
later once guardrails are proven. Anyone not allowlisted gets ignored.

## 3. Google Workspace connection

Hermes bundles a google-workspace skill (Gmail, Calendar, Drive, Sheets,
Contacts; OAuth2; write ops require confirmation). Connect it to
myacaexpress@gmail.com and grant Gmail read, Calendar read/write,
Drive/Sheets read (Sheets write needed later for PlanningSessions).

### 3.1 Calendar topology — do this once, everything depends on it

Today myacaexpress@gmail.com sees only its own calendar. Target state — all
of life visible (at least free/busy) from the one account Hermes reads:

| Calendar | How | Purpose |
|---|---|---|
| `myacaexpress` (existing) | — | work/agency default |
| **Alina Care** | create in Google Calendar UI ("Other calendars → Create new calendar") | therapy, ABA, appointments |
| **Family/Personal** | create | date nights, family events |
| **Dev Clients** | create | freelance commitments |
| **Patty** | Patty shares her calendar → myacaexpress (Settings → Share with specific people; "See all event details", or free/busy minimum) | conflict detection |
| **Alina School** | create; Hermes cron mirrors SchoolSchedule sheet rows into it | early releases/holidays participate in availability math |

Notes:
- Any other personal Google accounts: share those calendars → myacaexpress
  too (free/busy is enough).
- The school mirror keeps the existing Apps Script (which parses Friday
  Forward emails into the sheet) as the source; Hermes just syncs sheet →
  calendar so free/busy queries see school constraints.

## 4. Install this repo's Shawn layer

```bash
git clone <this repo> ~/shawn
cp -r ~/shawn/skills/*  ~/.hermes/skills/
cp ~/shawn/soul/SOUL.md ~/.hermes/SOUL.md            # verify path: hermes docs "SOUL.md"
cp ~/shawn/memory-seeds/MEMORY.md ~/.hermes/memories/MEMORY.md
cp ~/shawn/memory-seeds/USER.md   ~/.hermes/memories/USER.md
```

Then start a fresh session (`hermes chat`) so the new SOUL/memory snapshot
loads, and sanity-check: ask *"who are you and what do you know about my
week?"* and `/availability tomorrow afternoon`.

## 5. Cron jobs

Create via plain English in a Hermes session (it writes the job), or the
cron CLI. Definitions live in `cron/jobs.md` in this repo; the initial set:

| Job | Schedule (America/Los_Angeles) | Delivers to |
|---|---|---|
| morning-briefing | 07:00 daily | Telegram (later iMessage) |
| conflict-scan | 07:05 daily + 16:00 daily | only messages if a conflict found |
| school-calendar-sync | 06:30 daily | silent; ErrorLog on failure |
| date-night-check | Mon 09:00 weekly | suggestion message when criteria met |
| memory-hygiene | Sun 21:00 weekly | silent consolidation |

## 6. iMessage via Photon (after Telegram is proven)

```bash
hermes photon setup --phone +1XXXXXXXXXX   # device-code login at app.photon.codes
```

- Assigns a dedicated iMessage line (good: distinct "assistant" identity for
  sitters/providers).
- Verify before relying on it: pricing, outbound deliverability to
  brand-new contacts (sitter outreach), and group-thread support.
- BlueBubbles (spare Mac + Apple ID) is the fallback path.

## 7. Operations

- Backup nightly: `~/.hermes/` (memory, skills, sessions DB) → Drive or
  restic. It IS the assistant's brain.
- `hermes gateway status` in a watchdog; logs via `hermes gateway` run in
  foreground when debugging.
- Model switching is `hermes model` — no code changes; keep Sonnet-tier
  default, escalate planning jobs if quality demands.
- Keep Hermes' memory-injection scanning on (default) — memory goes into the
  system prompt.
