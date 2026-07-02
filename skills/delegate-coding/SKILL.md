---
name: delegate-coding
description: Hand implementation work to Claude Code instead of doing it inline. Three lanes - GitHub issue mention (cloud, zero infra), headless claude -p over SSH on Shawn's laptop, or a scheduled Claude Code web session. Use when Shawn asks to build/fix/change code, or a cron job detects work needing implementation.
---

# Delegate coding work to Claude Code

You (Hermes) coordinate; Claude Code implements. Pick the lane by where the
work lives and how heavy it is.

## Lane A — GitHub mention (default for repo work)

Zero infrastructure: the Claude GitHub Action watches Shawn's repos.

1. Write a self-contained brief: goal, acceptance criteria, constraints,
   affected files if known. Assume the reader has no chat context.
2. Via terminal: `gh issue create --repo myacaexpress/<repo> --title "..."
   --body "...

@claude please implement this"` (the @claude mention triggers the action).
3. Claude Code implements on a branch and opens a PR. Track it: note the
   issue/PR in MEMORY.md `[loops]`, check status in later heartbeats, and
   tell Shawn when the PR is ready for review.

## Lane B — Headless Claude Code locally (local/uncommitted work)

Hermes runs on Shawn's laptop, so this is just the local terminal backend —
no SSH needed. Requires the `claude` CLI installed. For work on Shawn's
machine: Apps Script push (`clasp`), local scripts, anything not repo-hosted.

```bash
cd ~/projects/<dir> && claude -p "<self-contained brief>" --output-format json
```

(If the gateway ever moves off the laptop, this lane becomes the same
command over SSH/Tailscale with a dedicated restricted key.)

- Keep briefs bounded: one task per invocation, state what "done" means.
- Capture the JSON result; summarize outcome + any failures to Shawn.
- NEVER run destructive commands over this channel yourself — the brief may
  ask Claude Code to change files in the project dir, nothing outside it.

## Lane C — Scheduled Claude Code web sessions

For recurring implementation chores (dependency bumps, report generation),
Shawn can create a Claude Code routine (cloud session on a schedule) from an
existing session. You don't control these directly — propose the routine
text to Shawn and he approves it once; thereafter it runs itself.

## Rules

- Delegate, don't disappear: every delegation gets a `[loops]` memory entry
  with where it's tracked (issue #, PR #, or laptop task) and gets followed
  up in the next briefing.
- Code review still belongs to Shawn: PRs get summarized, never auto-merged.
- If a delegated task fails twice, stop retrying and escalate to Shawn with
  the error.
