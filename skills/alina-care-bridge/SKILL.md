---
name: alina-care-bridge
description: Read/write the "Alina Care Assistant" Google Sheet - the system of record for the coverage network (sitters/helpers), school schedule, and coverage planning sessions. Use for anything involving sitters, pickups, school days, or care coverage.
---

# Alina Care Bridge

Sheet: "Alina Care Assistant" (owner nursevidales@gmail.com)
ID: `1amyzUlHaj3A3IDDl2J0-Vai_z2B9ct5cxxF0ENureII`

An Apps Script assistant also runs against this sheet (SMS webhook). Treat the
sheet as shared state: read fresh before writing, append rather than rewrite.

## Tabs

- **CoverageNetwork** — helpers: Name, Relationship, Phone, Tier (1=first
  call), Capabilities (`pickup`, `watch`, `watch_unsupervised`,
  `aba_supervision`, `tutoring`), Availability, RecurringSchedule (JSON
  day/start/end), Notes, LastContacted.
- **SchoolSchedule** — per-day rows: holidays, `early-release` (changed
  dismissal times), events. Fed by Apps Script parsing "Friday Forward"
  school emails. Read for any availability/pickup question.
- **PlanningSessions** — one row per coverage-planning effort: PlanID,
  EventDescription/Date/times, CoverageNeeds, CoverageStatus,
  OutreachStatus, ConfirmedHelpers, Status, Notes. Hermes owns new rows for
  its own orchestrations (date night, appointment coverage).
- **CalendarEvents** — sync cache; prefer live Calendar API.
- **ErrorLog** — Apps Script errors; mention in briefing if growing.

## Coverage selection logic

1. Filter CoverageNetwork by required capability (unsupervised watch needs
   `watch_unsupervised`).
2. Check RecurringSchedule fit for the target day/time.
3. Order by Tier, then reliability notes, then least-recently-contacted.
4. Record the plan in PlanningSessions before any outreach; update
   OutreachStatus/ConfirmedHelpers as replies arrive; stamp LastContacted.

## Guardrails

- Phone numbers stay in the sheet — never copy them into MEMORY.md or
  messages to third parties.
- Outreach itself (actually messaging a helper) is Phase 3 and
  confirm-first; until then, produce the ranked list + draft message for
  Shawn to send.
