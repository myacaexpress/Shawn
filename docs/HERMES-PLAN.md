# Hermes — Personal Chief-of-Staff Agent

*Planning document, v0.1 — July 2026*

Hermes is the working name for the assistant described in the "Shawn AI — A Personal
Operating System" concept doc. This document turns that concept into a concrete,
phased architecture grounded in the systems Shawn already runs today.

---

## 1. The core problem

> "I basically start over every morning without memory of what is happening or what
> I need to do."

Shawn operates across at least four distinct profiles, each with its own calendars,
email streams, and message threads:

| Profile | What it involves | Existing tooling |
|---|---|---|
| **Alina care** | ABA/therapy schedules, IEP, school (Friday Forward emails), sitter/coverage network, coordination with Patty | Apps Script "Alina Care Assistant" + Google Sheet + SMS webhook |
| **Insurance agency** (myacaexpress) | Carrier emails (Aetna, Ambetter, UHC, …), state licensing, commissions, client follow-ups | Gmail label taxonomy (`Carriers/*`, `States/*`, `@Action Item`, `@AaA:*`), shawn-classifier |
| **Freelance dev** | Client projects, GitHub, Vercel/Linear, invoices | Ad-hoc |
| **Personal/family** | Date nights, family logistics, budget | Nothing systematic |

The failure mode isn't a lack of data — it's that nothing **persists and synthesizes**
across days and across profiles. Hermes' job is to be the memory and the
proactive layer on top of everything already flowing in.

## 2. What exists today (build on it, don't replace it)

1. **Alina Care Assistant** (Apps Script + Google Sheet, sheet owned by
   nursevidales@gmail.com):
   - `CoverageNetwork` — sitters/helpers with tiers, capabilities
     (`pickup`, `watch_unsupervised`, `aba_supervision`), recurring availability.
   - `SchoolSchedule` — extracted from "Friday Forward" school emails
     (early releases, holidays, events).
   - `PlanningSessions` / `CalendarEvents` — schema exists, mostly unused so far.
   - Inbound/outbound SMS via `doPost` webhook → `handleInboundSMS` → `sendSMS`
     (an `ErrorLog` shows a recurring `substring of undefined` crash in `sendSMS`
     worth fixing regardless).
2. **shawn-classifier** — email classification giving visibility/reminders.
3. **Gmail labels** — a de-facto workflow state machine (`@Action Item`,
   `@Waiting`, `@For Follow Up`, `@Done`, `@AaA:*` assignee queues).

These are Hermes' first **tools and data sources**, not things to rewrite.

## 3. Proposed architecture

Five layers. Hermes is an orchestrator (Claude Agent SDK) with tools, memory,
and scheduled triggers — not a monolith.

```
┌─────────────────────────────────────────────────────────────┐
│  INTERFACE      iMessage (primary) · SMS fallback           │
│                 identity: always "🤖 Hermes / Shawn AI"     │
├─────────────────────────────────────────────────────────────┤
│  ORCHESTRATOR   Claude Agent SDK agent                      │
│                 · conversational loop (inbound messages)    │
│                 · scheduled heartbeats (cron):              │
│                   - 7am daily briefing                      │
│                   - conflict scan (calendars × school ×     │
│                     Patty × work)                           │
│                   - weekly "date night" / open-loop check   │
│                 · subagents per domain when needed          │
├─────────────────────────────────────────────────────────────┤
│  MEMORY         persistent, structured, profile-tagged      │
│                 · people registry (sitters, providers,      │
│                   carriers, clients)                        │
│                 · preferences & facts ("Patty liked that    │
│                   Italian place", budget rules)             │
│                 · open loops (waiting-on, follow-ups)       │
│                 · episodic log (what Hermes did & learned)  │
├─────────────────────────────────────────────────────────────┤
│  INGESTION      Gmail (multi-account) · Google Calendars    │
│                 (multi-account incl. Patty's + school) ·    │
│                 Alina Care Sheet · iMessage history         │
│                 (chat.db, local-only extraction)            │
├─────────────────────────────────────────────────────────────┤
│  ACTIONS        calendar write · sitter outreach (existing  │
│                 SMS pipeline) · email drafts · reservations │
│                 (later) — all confirm-before-act by default │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 The profile router

Every inbound item (email, message, calendar event) gets tagged with a profile:
`family/alina`, `insurance`, `dev`, `personal`. This is exactly what
shawn-classifier already does for email — extend the same idea to all streams.
The router determines which memory namespace, which tone, and which urgency
rules apply. It's also what keeps the morning briefing readable ("3 things for
Alina, 2 for the agency, 1 personal") instead of a firehose.

### 3.2 Memory design (the heart of the system)

The concept doc's key requirement — *starts informed rather than blank* — means
memory is the first thing to get right:

- **Structured store**: extend the existing Google Sheet pattern (or SQLite if
  Hermes runs on a Mac) with tables Hermes reads/writes:
  `People`, `Preferences`, `OpenLoops`, `EpisodicLog`.
  Staying in Sheets keeps the existing Apps Script interoperable.
- **Narrative memory**: markdown files per profile (like CLAUDE.md files) the
  agent loads as context — "what's true about the insurance business",
  "Alina's current care team", "how Shawn & Patty plan things".
- **Extraction, not retention**: per the doc's privacy stance, iMessage history
  is mined locally for *insights* (favorite restaurants, reliable sitters,
  important dates); raw messages never leave the Mac.
- **Write-back discipline**: after every interaction and every heartbeat run,
  Hermes appends what it learned/did. That's what kills the
  "start over every morning" problem.

### 3.3 iMessage — the hard constraint

Apple has no official iMessage API. Real options:

| Option | How | Trade-offs |
|---|---|---|
| **A. Always-on Mac + BlueBubbles** | Mac mini (or the existing Mac) runs BlueBubbles server; Hermes talks to its REST API/webhooks | True blue-bubble iMessage, full send/receive, group chats. Requires a Mac that never sleeps. **Recommended target.** |
| B. AppleScript / Shortcuts on Mac | Script Messages.app directly | Send works; reliable *receive* requires polling `chat.db`. Fragile across macOS updates. |
| C. Keep Twilio SMS (current) | Existing Apps Script pipeline | Works today, cloud-friendly, but green bubble and no group-thread richness. **Good Phase-1 bridge** while the Mac path is set up. |

Pragmatic path: **start on SMS (already wired), move to BlueBubbles when the
always-on Mac is ready.** The orchestrator shouldn't care which transport is
underneath — make "messaging" a swappable tool interface from day one.

### 3.4 Where Hermes runs

Two viable homes, and the doc assumes local-first:

- **On the Mac** (concept doc's assumption): required anyway for iMessage and
  chat.db mining. Claude Agent SDK runs fine locally; cron via launchd.
- **Hybrid (recommended)**: cloud (or Claude Code on the web / a small VPS)
  hosts the orchestrator + Google integrations; the Mac runs only a thin
  iMessage bridge (BlueBubbles). This keeps Hermes alive when the Mac sleeps
  and keeps Google auth in one place — only messaging round-trips touch the Mac.

### 3.5 A note on the doc's "technical reviewer" caveat

The concept leaned on unverified "swarm mode" and "proactive memory framework"
capabilities. As of mid-2026, the Claude Agent SDK verifiably supports the
pieces Hermes needs: subagents (fan-out per domain), scheduled/cron invocation,
tool use against Gmail/Calendar/Drive via MCP, and file-based persistent
memory. Nothing in this plan depends on speculative features.

## 4. Phased build path

### Phase 0 — Consolidate & stabilize (≈ a weekend)
- Fix the `sendSMS` crash in the Apps Script (ErrorLog shows it eating real
  inbound questions).
- Inventory all Google accounts/calendars in play. **Note:** the
  myacaexpress@gmail.com account only exposes its own calendar today — Patty's
  calendar, personal calendar, and school calendar need to be shared into one
  account (or each account connected) before conflict detection can work.
- Define the memory schema (People / Preferences / OpenLoops / EpisodicLog)
  and seed it manually with the top ~20 facts (care team, sitter tiers already
  in CoverageNetwork, key clients, budget rules).

### Phase 1 — Core loop: "Hermes remembers" (doc's Phase 1)
- Orchestrator (Claude Agent SDK) with tools: Gmail read, Calendar read,
  Sheet read/write, send/receive message (SMS transport first).
- **Morning briefing** at 7am: profile-grouped digest — today's events across
  all calendars, school exceptions, @Action Item / @Waiting emails needing
  attention, open loops.
- Conversational Q&A over the same data ("Did I ever email Stephanie about ABA
  observation?" — a real question the current bot crashed on).
- Every run writes back to memory.

### Phase 2 — Proactive layer (doc's Phase 2)
- Conflict scanner: my calendar × Patty's × school schedule × care sessions;
  flags collisions days ahead with concrete options.
- Calendar **write** access (create/move events after confirmation).
- Opportunity detection: free Saturday + sitter availability from
  CoverageNetwork + "it's been 3 weeks" → date-night suggestion.
- Insurance-side nudges: @Waiting threads gone quiet, licensing/commission
  deadlines from carrier emails.

### Phase 3 — Full orchestration (doc's Phase 3)
- Sitter outreach automation using the existing PlanningSessions schema:
  Hermes texts tier-1 → tier-2 helpers, tracks responses, escalates, always
  self-identifying as an AI with human-handoff on request.
- Reservations (OpenTable/Resy where APIs allow, otherwise drafted for
  one-tap confirm).
- Multi-step parallel workflows (the date-night scenario end-to-end:
  availability → sitter → restaurant → budget → one confirmation message).
- iMessage history mining on the Mac to enrich preferences.

## 5. Guardrails (from the concept doc, kept as hard rules)

1. Every outbound message to third parties is labeled as the AI assistant;
   first contact includes a short intro.
2. "Human please" / urgency → immediate handoff + notify Shawn.
3. Confirm-before-act for anything that spends money, commits time, or
   messages someone new. Trust levels can loosen per-action-type over time.
4. Raw iMessage data stays on the Mac; only extracted insights persist.

## 6. Open decisions

1. **Runtime home**: hybrid (cloud brain + Mac iMessage bridge) vs. all-local
   on an always-on Mac. → Recommend hybrid.
2. **Messaging transport for Phase 1**: reuse the Twilio/SMS pipeline now and
   add BlueBubbles later, vs. wait for the Mac bridge. → Recommend SMS now.
3. **Memory store**: Google Sheets (Apps Script interop, Patty can see it) vs.
   SQLite + markdown (faster, more private). → Recommend Sheets for structured
   tables + markdown files in this repo for narrative memory.
4. **Account topology**: share all calendars into one Google account vs.
   connect each account. Which accounts exist, and does Patty consent to
   calendar sharing + Hermes reading her availability?
5. **Naming**: framework = Hermes; the outward-facing messaging identity —
   keep "Shawn AI" (as in the doc) or rebrand to Hermes?

## 7. Repo layout (proposed, this repo)

```
Shawn/
├── docs/HERMES-PLAN.md        # this file
├── hermes/                    # orchestrator (Claude Agent SDK, TypeScript)
│   ├── agents/                # briefing, conflict-scan, outreach subagents
│   ├── tools/                 # gmail, calendar, sheet, messaging (transport-agnostic)
│   └── heartbeats/            # cron entrypoints
├── memory/                    # narrative memory (md per profile) — private repo
│   ├── family-alina.md
│   ├── insurance.md
│   ├── dev.md
│   └── personal.md
└── bridge/                    # Mac-side iMessage bridge config (BlueBubbles)
```
