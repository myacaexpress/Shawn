# Shawn — Hermes assistant layer

The Shawn-specific configuration for a personal chief-of-staff agent built
on [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous
Research). Hermes provides the framework (gateway, channels, memory, skills,
cron); this repo is everything that makes it *Shawn's*.

| Path | What it is |
|---|---|
| `docs/HERMES-PLAN.md` | Architecture & phased plan (read first) |
| `docs/SETUP.md` | Step-by-step runbook: install → Telegram → Google → Photon iMessage |
| `soul/SOUL.md` | Identity, voice, privacy guardrails |
| `memory-seeds/` | Initial MEMORY.md / USER.md for `~/.hermes/memories/` |
| `skills/availability/` | "When is Shawn free?" across all calendars — any channel |
| `skills/morning-briefing/` | 7am profile-grouped digest (the anti-amnesia loop) |
| `skills/alina-care-bridge/` | Read/write the Alina Care Assistant sheet |
| `skills/delegate-coding/` | Hand implementation work to Claude Code (GitHub mention / SSH headless / scheduled sessions) |
| `cron/jobs.md` | Scheduled job definitions |

Install: follow `docs/SETUP.md`; step 4 copies these files into `~/.hermes/`.
