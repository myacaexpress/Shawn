# Hermes — Personal Chief-of-Staff Agent

*Planning document, v0.2 — July 2026*
*(v0.2 re-architected around the Hermes Agent framework by Nous Research instead
of a from-scratch orchestrator.)*

This plan turns the "Shawn AI — A Personal Operating System" concept doc into a
concrete build, grounded in (a) the systems Shawn already runs and (b) the
open-source **Hermes Agent** framework (github.com/NousResearch/hermes-agent,
released Feb 2026), which ships the orchestration/memory/messaging plumbing
out of the box.

> **Verification note:** the official docs
> (hermes-agent.nousresearch.com/docs) were unreachable from the planning
> session's network; details below come from search-indexed doc content and
> third-party writeups. Verify exact config keys, limits, and Photon pricing
> against live docs during Phase 0.

---

## 1. The core problem

> "I basically start over every morning without memory of what is happening or
> what I need to do."

Shawn operates across at least four profiles, each with its own calendars,
email streams, and message threads:

| Profile | What it involves | Existing tooling |
|---|---|---|
| **Alina care** | ABA/therapy schedules, IEP, school (Friday Forward emails), sitter/coverage network, coordination with Patty | Apps Script "Alina Care Assistant" + Google Sheet + SMS webhook |
| **Insurance agency** (myacaexpress) | Carrier emails (Aetna, Ambetter, UHC, …), state licensing, commissions, client follow-ups | Gmail label taxonomy (`Carriers/*`, `States/*`, `@Action Item`, `@AaA:*`), shawn-classifier |
| **Freelance dev** | Client projects, GitHub, Vercel/Linear, invoices | Ad-hoc |
| **Personal/family** | Date nights, family logistics, budget | Nothing systematic |

The failure mode isn't lack of data — it's that nothing **persists and
synthesizes** across days and across profiles.

## 2. What exists today (build on it, don't replace it)

1. **Alina Care Assistant** (Apps Script + Google Sheet, owned by
   nursevidales@gmail.com):
   - `CoverageNetwork` — sitters/helpers with tiers, capabilities
     (`pickup`, `watch_unsupervised`, `aba_supervision`), recurring availability.
   - `SchoolSchedule` — auto-extracted from "Friday Forward" school emails.
   - `PlanningSessions` / `CalendarEvents` — schemas exist, mostly unused.
   - SMS in/out via `doPost` → `handleInboundSMS` → `sendSMS`; the `ErrorLog`
     shows a recurring `substring of undefined` crash in `sendSMS` that has
     been eating real inbound questions — fix regardless.
2. **shawn-classifier** — email classification for visibility/reminders.
3. **Gmail labels** — a de-facto workflow state machine (`@Action Item`,
   `@Waiting`, `@For Follow Up`, `@Done`, `@AaA:*` queues).

These become Hermes' **data sources and skills**, not things to rewrite.

## 3. What Hermes Agent gives us out of the box

Mapping the concept doc's requirements to shipped Hermes features:

| Need (from concept doc) | Hermes Agent feature |
|---|---|
| iMessage as the interface | Two paths: **BlueBubbles** channel (Mac relay, since v0.9.0) or **Photon Spectrum** (v0.17.0+): gRPC-native iMessage with *no Mac* — `hermes photon setup --phone …`, device-code login, managed line pool that assigns a dedicated iMessage line |
| Remembers context over time | Built-in persistent memory: `MEMORY.md` (~2.2k chars) + `USER.md` (~1.4k chars) in `~/.hermes/memories/`, injected into the system prompt each session; agent curates its own entries via a memory tool (bounded — errors instead of silently dropping, agent consolidates) |
| "Did I ever …?" recall | `session_search` tool — all CLI + messaging sessions stored in SQLite with FTS5 full-text search |
| Deeper long-term memory | 8 pluggable external memory providers (one active at a time) alongside built-in memory |
| Proactive, not reactive | Built-in **cron scheduler** (gateway ticks every 60s, jobs run in isolated sessions, results delivered to any connected channel); plain-English job creation; a first-class "heartbeat" primitive is in progress upstream (issue #15400) |
| Reads calendar + provider emails | Bundled **google-workspace skill** (`mail_search`, `mail_get`, `cal_list_calendars`, `cal_list_events`, `cal_create_event`, `drive_search`, Sheets/Docs/Contacts) with OAuth2; write operations require user confirmation. Composio MCP is an alternative path |
| Multi-step workflows | Skills system: Markdown playbooks in `~/.hermes/skills/` (agentskills.io standard, progressive disclosure, auto slash-commands); agent can author its own skills after solving a problem |
| Personality / identity | `SOUL.md` per profile |
| Runs on what we have | Gateway is a FastAPI server (port 8642) + dashboard; terminal backends: local, Docker, SSH, Modal, Daytona, Apptainer. Model-agnostic: Anthropic/OpenRouter/OpenAI/own endpoint, switchable via `hermes model` |
| Safety on memory | Memory entries scanned for prompt-injection/exfiltration patterns and invisible Unicode before acceptance |

**Consequence: we are not building a framework.** We are deploying Hermes and
writing the *Shawn-specific layer*: skills, memory seeds, cron jobs, SOUL.md,
and the bridge to the Alina Care sheet.

## 4. Architecture (v0.2)

```
                    ┌──────────────────────────────────────┐
   Shawn & Patty ──▶│  iMessage (Photon Spectrum line —    │
   sitters/helpers  │  dedicated number, self-identifies   │
                    │  as the assistant)                   │
                    └───────────────┬──────────────────────┘
                                    │
                    ┌───────────────▼──────────────────────┐
                    │  HERMES GATEWAY (single profile)     │
                    │  · SOUL.md — identity & guardrails   │
                    │  · MEMORY.md / USER.md — core facts  │
                    │  · session_search — episodic recall  │
                    │  · cron jobs:                        │
                    │      07:00 morning-briefing          │
                    │      daily  conflict-scan            │
                    │      weekly date-night-check         │
                    │      weekly memory-hygiene           │
                    └───┬───────────────┬──────────────┬───┘
                        │               │              │
              ┌─────────▼───┐   ┌───────▼──────┐  ┌────▼─────────────┐
              │ google-     │   │ custom skills│  │ alina-care bridge│
              │ workspace   │   │ (this repo): │  │ skill: reads/    │
              │ skill:      │   │ briefing,    │  │ writes the Sheet │
              │ Gmail, Cal, │   │ conflict,    │  │ (CoverageNetwork,│
              │ Drive,Sheets│   │ date-night,  │  │ SchoolSchedule,  │
              │ (multi-acct)│   │ outreach,    │  │ PlanningSessions)│
              │             │   │ insurance    │  │                  │
              └─────────────┘   └──────────────┘  └──────────────────┘
```

### 4.1 One profile, not four

Hermes "profiles" are *fully isolated* instances (own config, memory, skills,
sessions). Isolation is exactly wrong for Shawn's core need — **cross-domain
coordination** (a work call colliding with school pickup colliding with
Patty's calendar). So: **one Hermes profile**, with the four life-domains
expressed as:
- a domain-tagging convention inside MEMORY.md entries and skill outputs
  (`[alina]`, `[ins]`, `[dev]`, `[personal]`), and
- one skill per domain encoding its rules (which labels matter, tone,
  escalation thresholds).

A second isolated profile *could* later host a business-only agent (e.g., an
agency bot that talks to clients) without contaminating the family assistant.

### 4.2 Memory layering

Three tiers, cheapest first:
1. **Core memory** (`MEMORY.md`/`USER.md`): the ~30 always-true facts — care
   team, sitter tiers, budget rules, key clients, how Shawn & Patty plan.
   Bounded size forces curation; a weekly `memory-hygiene` cron job reviews
   and consolidates.
2. **Episodic recall** (`session_search`): free — every conversation is
   FTS5-searchable. Answers "did I ever email Stephanie about ABA
   observation?"-class questions if the interaction happened through Hermes.
3. **Structured domain data**: stays in the **Google Sheet** (CoverageNetwork,
   SchoolSchedule, PlanningSessions) accessed via the Sheets tools — keeping
   the existing Apps Script and Patty's visibility intact. The sheet is the
   system of record; Hermes memory holds pointers and distilled preferences.

Evaluate an external memory provider (tier 4) only if these three prove
insufficient — likely candidates once volume grows: restaurant/preference
history, provider-notes corpus.

### 4.3 iMessage transport decision

**Recommendation: Photon Spectrum.**
- No always-on Mac required; Hermes can live on a small VPS/Docker host.
- It assigns a **dedicated iMessage line** — which is actually *better* for
  the concept doc's "clear AI identity" rule: sitters and providers see a
  distinct number that introduces itself as the assistant, rather than
  messages coming from Shawn's personal Apple ID.
- BlueBubbles remains the fallback if Photon's managed line pool has
  pricing/reliability issues (spare Mac + BlueBubbles server + Apple ID).
- The existing Twilio/SMS pipeline stays as-is during transition; Hermes also
  supports plain SMS channels if we ever want to consolidate.

Open items to verify in Phase 0: Photon pricing, deliverability/limits on
outbound to new contacts (sitter outreach), group-chat support (family thread
with Shawn + Patty).

### 4.4 Model + hosting

- **Model**: Anthropic Claude via API (the gateway preserves Anthropic prompt
  cache across turns, so it's a first-class path). Start with Sonnet-tier for
  cost; escalate the briefing/planning jobs to a stronger model if needed —
  switching is `hermes model`, no code changes.
- **Hosting**: Docker on a small VPS (or the Mac if preferred — no longer
  required). Gateway port 8642 + dashboard; back up `~/.hermes/` (memory,
  skills, sessions) nightly.

## 5. What we actually build (this repo)

```
Shawn/
├── docs/HERMES-PLAN.md          # this file
├── soul/SOUL.md                 # identity, tone, guardrails (§6)
├── memory-seeds/                # initial MEMORY.md / USER.md content
├── skills/
│   ├── morning-briefing/        # profile-grouped daily digest
│   ├── conflict-scan/           # calendars × school × Patty × care sessions
│   ├── date-night/              # free evening + sitter + restaurant + budget
│   ├── coverage-outreach/       # tiered sitter contact w/ PlanningSessions
│   ├── alina-care-bridge/       # read/write the Alina Care sheet
│   ├── insurance-inbox/         # @Action Item / @Waiting / carrier deadlines
│   └── school-schedule/         # Friday Forward parsing (port from Apps Script)
├── cron/jobs.md                 # schedule definitions + delivery targets
└── deploy/                      # Dockerfile, .env template, backup script
```

Skills are Markdown playbooks — most of the "code" here is carefully written
procedure + the few scripts a skill shells out to. The Apps Script logic
(Friday Forward parsing, coverage tiers) ports naturally into skill
instructions + Sheets reads.

## 6. Guardrails (unchanged from concept doc — enforced via SOUL.md + skill rules)

1. Every outbound message to third parties self-identifies as the AI
   assistant; first contact includes a short intro.
2. "Human please" / urgency → immediate handoff + notify Shawn.
3. Confirm-before-act for anything that spends money, commits time, or
   messages someone new (Hermes' google-workspace skill already gates writes
   behind confirmation; extend the same rule to outreach).
4. Raw personal-message history is mined locally only; only extracted
   insights enter memory.
5. Memory-injection scanning stays on (Hermes default) — memory is in the
   system prompt.

## 7. Phased build path (v0.2)

### Phase 0 — Stand it up (a weekend)
- Fix the `sendSMS` crash in the existing Apps Script (it's still the
  production assistant until Hermes takes over).
- Provision VPS/Docker, install Hermes, connect Anthropic API key.
- `hermes photon setup` — verify iMessage line, pricing, deliverability.
- Connect google-workspace skill to myacaexpress@gmail.com; **calendar
  homework**: share Patty's / personal / school calendars into the account
  (today it sees only its own calendar — conflict detection is impossible
  until this is done).
- Write SOUL.md + seed MEMORY.md/USER.md from CoverageNetwork and the top
  ~30 facts.

### Phase 1 — Core loop: "Hermes remembers" (concept doc Phase 1)
- Conversational Q&A over Gmail/Calendar/Sheet via iMessage.
- `morning-briefing` cron at 07:00, profile-grouped, delivered to iMessage.
- `alina-care-bridge` + `school-schedule` skills (read-only).
- Memory write-back discipline: briefing and every session update MEMORY.md.

### Phase 2 — Proactive layer (concept doc Phase 2)
- `conflict-scan` cron across all shared calendars + SchoolSchedule.
- Calendar write access (confirm-first).
- `date-night` opportunity detection (free Saturday + CoverageNetwork
  availability + "it's been 3 weeks").
- `insurance-inbox`: stale @Waiting threads, licensing/commission deadlines.

### Phase 3 — Full orchestration (concept doc Phase 3)
- `coverage-outreach`: Hermes messages tier-1 → tier-2 helpers directly on
  its own iMessage line, tracks responses in PlanningSessions, escalates,
  hands off to Shawn on request.
- Restaurant reservations (drafted for one-tap confirm; API booking where
  available).
- End-to-end date-night scenario as a single skill invoking the others.
- Retire the Apps Script SMS path once Hermes has feature parity.

## 8. Open decisions

1. **Photon vs BlueBubbles** — recommend Photon (no Mac, dedicated line);
   confirm pricing/deliverability in Phase 0.
2. **Hosting** — ✅ DECIDED: locally on Shawn's Mac laptop (see SETUP.md
   §1.1 for launchd + anti-sleep setup). Bonus: BlueBubbles becomes a
   localhost option and delegate-coding runs `claude -p` directly with no
   SSH. Escalate to Mac mini/VPS only if laptop sleep proves disruptive.
3. **Model** — Claude Sonnet-tier default, stronger model for planning jobs?
4. **Naming/identity** — framework is Hermes (fitting: it *is* the messenger);
   outward identity on the iMessage line: "Shawn AI" per the doc, or Hermes?
5. **Patty's calendars** — sharing consent + which account topology.
6. **Second business-only profile later?** — defer until family assistant is
   stable.
