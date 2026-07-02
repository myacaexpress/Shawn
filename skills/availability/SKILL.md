---
name: availability
description: Answer "when is Shawn free?" questions from any channel (Telegram, iMessage, CLI). Computes real free windows across all connected calendars plus school/care constraints, and proposes concrete slots without leaking private event details.
---

# Availability

Use this skill whenever someone asks about Shawn's availability, proposes a
meeting time, or asks "does <time> work?" — whether Shawn asks directly or an
allowlisted person messages the bot.

## Procedure

1. **Parse the ask**: date range, duration needed (default 30 min if a call,
   60 min if unspecified meeting), and who is asking (Shawn himself, Patty,
   or a third party).
2. **Gather constraints** — always all of these, not just the primary
   calendar:
   - `cal_list_calendars`, then `cal_list_events` for the window across:
     myacaexpress (work), Alina Care, Family/Personal, Dev Clients, Patty
     (shared), Alina School.
   - School-day rules from the Alina School calendar / SchoolSchedule sheet:
     early-release days change pickup time.
   - Standing rules (see MEMORY.md `[rules]` entries), e.g.:
     - school pickup ~5:30pm on regular days needs Shawn or Patty free —
       check Patty's calendar before assuming Shawn is the one blocked
     - no work meetings before 9:00am
     - 15 min buffer between meetings; travel time for in-person
3. **Compute free windows** in America/Los_Angeles. A slot is free only if
   it clears every calendar above plus the rules.
4. **Respond**:
   - To **Shawn/Patty**: full detail is fine — show the windows and what's
     bounding them ("free 2–4, then Alina pickup").
   - To **anyone else**: free/busy only. NEVER name events, people, health or
     school details. Offer 2–3 concrete slots: "Shawn could do Thu 2:00–2:30
     or Fri 10:00–11:00 (Pacific). Which works?"
5. **On agreement**: create a tentative event (`cal_create_event`) on the
   right calendar — after confirmation, per write-gating — and tell Shawn
   which slot was booked and with whom.

## Guardrails

- If asked about someone else's schedule (Patty's, Alina's) by a third
  party: decline and notify Shawn.
- If calendars are unreachable, say so — never guess availability.
- If the request smells like scheduling with money/commitment attached
  (client engagement, paid gig), propose the slot but flag Shawn before
  confirming.
