# Instructions for Claude Code running in this repo

This repo is the Shawn-specific layer for a Hermes Agent
(github.com/NousResearch/hermes-agent) personal assistant. If the user asks
you to "set it up", "install", or "continue setup", your job is to execute
`docs/SETUP.md` on this machine — do the steps yourself rather than telling
the user to do them, pausing only where a human is genuinely required.

## Execution notes

- Follow `docs/SETUP.md` in order. Current target: local install on Shawn's
  Mac laptop.
- Steps you run directly: the installer script (show it to the user before
  running), `hermes model`, `hermes gateway setup/install/status`, copying
  `skills/`, `soul/SOUL.md`, and `memory-seeds/` into `~/.hermes/`, cron job
  creation per `cron/jobs.md`, and verification (message round-trip test).
- Steps that need the human (ask, then continue):
  - Anthropic API key (paste when prompted by `hermes model`).
  - Telegram: user creates the bot with @BotFather on their phone and gets
    their numeric ID from @userinfobot; you take the token + IDs from there.
  - Google OAuth browser consent for the google-workspace skill.
  - Calendar topology (SETUP.md §3.1): creating calendars and Patty's
    sharing happen in the Google Calendar UI — walk the user through it,
    then verify with a calendar list call.
  - Photon device-code login (later phase).
- After any step, verify before moving on (`hermes gateway status`, a real
  Telegram round-trip, a real calendar read). Report failures honestly and
  check the upstream repo's issues — macOS launchd integration has known
  flaky spots (launchctl exit 5 fallback, /restart race).
- Never commit secrets: tokens and keys live in `~/.hermes/.env`, not here.
- Update `docs/SETUP-LOG.md` (create if absent) with what was completed,
  versions installed, and anything deviating from the runbook — so the next
  session (local or cloud) knows where setup stands.

## Repo map

`docs/HERMES-PLAN.md` architecture/plan · `docs/SETUP.md` runbook ·
`soul/SOUL.md` agent identity · `memory-seeds/` initial memory ·
`skills/` agentskills-format skills installed into `~/.hermes/skills/` ·
`cron/jobs.md` scheduled job definitions.
