# Cron jobs (America/Los_Angeles)

Create each in a Hermes session in plain English, e.g.:
"Every day at 7:00am Pacific, run the morning-briefing skill and send the
result to me on Telegram."

| Job | Schedule | Prompt (summary) | Delivery |
|---|---|---|---|
| morning-briefing | 07:00 daily | Run /morning-briefing | Telegram → Shawn (iMessage later) |
| conflict-scan-pm | 16:00 daily | Check tomorrow+3d across all calendars, school exceptions, pickup coverage vs Patty; message ONLY if a conflict/coverage gap found, with 2 concrete options | Telegram → Shawn |
| school-calendar-sync | 06:30 daily | Mirror SchoolSchedule sheet rows (next 30d) into "Alina School" calendar; create/update, never duplicate | silent |
| date-night-check | Mon 09:00 | If a free evening exists in next 10d for Shawn+Patty and a CoverageNetwork helper matches, propose it (helper, time, restaurant idea from memory, budget note) | Telegram → Shawn |
| memory-hygiene | Sun 21:00 | Review MEMORY.md: consolidate duplicates, expire stale loops, verify [rules] still true | silent |

Notes:
- Jobs run in isolated sessions — each prompt must name the skill(s) to use;
  don't assume chat context.
- Add `no_agent=True`-style plain watchdogs only for dumb checks (disk,
  gateway up); everything above needs reasoning.
