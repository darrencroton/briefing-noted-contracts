# Meeting Intelligence System — Post-Soak Action Plan

**Audience:** project owner and dev team.
**Purpose:** run one week of real use, capture issues quickly, and decide the Phase 5 hardening work from evidence.
**Source of truth:** Master Implementation Plan wins; Supplemental Implementation Guardrails are the review checklist.

## Current State

Phase 1 through Phase 5A are complete enough for real in-person use:

- `briefing` plans sessions, writes pre-meeting notes, runs `watch`, and ingests completed `noted` sessions.
- `noted` starts from manifests, records, stops quickly, post-processes asynchronously, writes `completion.json`, and exposes scheduled end/extend controls.
- Contracts are pinned and shared by both consumers.

This plan starts from operational trial, not feature bring-up.

## Trial Goal

For one week, use the system on real in-person meetings until the happy path is boring:

1. Calendar event is eligible.
2. `briefing` creates or refreshes the briefing note.
3. `briefing watch` starts `noted` at pre-roll.
4. `noted` captures, prompts, stops, and processes.
5. `completion.json` is written.
6. `briefing session-ingest` writes the managed summary without touching user notes.

Do not broaden scope during the week unless the current flow blocks daily use.

## Daily Trial Checklist

For each recorded meeting, capture:

- event title, start/end, and mode
- whether briefing note existed before start
- whether recording started at pre-roll
- whether popup appeared at the expected time
- stop path: popup Stop, Extend, auto-stop, or manual CLI
- `completion.json` terminal status
- transcript quality: usable / partial / unusable
- diarization quality: useful / noisy / wrong
- whether summary was written
- any user-owned note content changed unexpectedly

If anything fails, keep the session directory and note unchanged until inspected.

## Trial Issue Log

Add one row per issue during the week.

| Date | Meeting | Priority | Symptom | Evidence | Owner | Decision |
| --- | --- | --- | --- | --- | --- | --- |
|  |  | P0/P1/P2/P3 |  | session dir, note path, log line, screenshot |  | fix now / defer / no action |

## Issues And Lessons Learned

These are the first items to validate or fix during the trial.

### User-Facing Setup Gaps

- `noted` Settings does not expose ad hoc note directory.
- `noted` Settings does not expose the `briefing` handoff command.
- `noted` Settings does not expose automatic ingest on/off.
- `briefing` setup does not install a stable user-facing `briefing` command for `noted` to call.

Action: add these to the menubar Settings or a first-run setup flow before publishing end-user docs.

### Ad Hoc UX Gaps

- Menubar can start an ad hoc session but does not expose a normal Stop control for the active ad hoc session.
- Ad hoc output discovery is not user-friendly enough; users need an obvious way to reveal the latest session directory.
- Ad hoc summary ingestion is not fully configurable through the app UI.

Action: treat ad hoc as trial-only until Stop, output reveal, and ingest handoff are user-facing.

### Scheduled Workflow Gaps

- Test setup still relies on `init-series` index selection when multiple Calendar events exist.
- There is no user-facing “create test event / validate series match” path.
- Summary success depends on the `noted` to `briefing` handoff being configured outside the current app Settings UI.

Action: add a simple validation path that confirms “this event will be briefed and recorded” before relying on automation.

### Documentation Lessons

- User docs should use default settings wherever possible.
- Do not ask users to edit hidden app settings files.
- Calendar setup should say what to paste into the event notes, not generate events by script.
- Product TODOs should stay visible until the UI supports the workflow.

Action: keep `Meeting Intelligence User Walkthroughs.md` as the trial doc; copy into component repos only after the gaps above are closed.

## During-Week Triage Rules

Use these priorities:

- **P0:** raw audio missing after a meeting that should have recorded.
- **P1:** recording starts/stops unreliably, `completion.json` missing, or user note content is modified outside managed blocks.
- **P2:** transcript/summary quality problems where recovery still works.
- **P3:** UX friction, docs confusion, noisy logs, or cosmetic issues.

Guardrails that must not regress:

- `briefing` owns Calendar interpretation and manifest contents.
- `noted` owns capture and manifest execution only.
- `noted stop` stays fast and non-blocking.
- `completion.json` remains the sole terminal outcome signal.
- Managed blocks never alter user-owned note content.
- Raw audio is preserved whenever capture succeeds.

## End-Of-Week Review

At the end of the trial, decide:

1. Is in-person scheduled recording reliable enough to keep using?
2. Which failures happened more than once?
3. Which failures blocked recovery?
4. Which UX gaps caused real confusion?
5. Which deferred items below now have enough evidence to schedule?

Only promote a hardening item if it is backed by trial data or blocks continued use.

## Leave Until After A Week Of Real Use

- Crash recovery / `noted resume`: needs real failure states, especially partial audio and missing completion cases.
- Retention enforcement and FLAC compression: keep all raw audio during first soak for recovery.
- Diarization quality improvements: needs real meeting recordings and ratings.
- Online/hybrid mode: defer until in-person end-to-end is boringly reliable.
- Broader operator dashboard commands like `list-sessions` and `tail-log`: implement only if the logs are painful during soak.

## First Likely Post-Soak Work

Expected first tickets, if trial confirms the need:

1. Add missing `noted` Settings fields for handoff and ad hoc paths.
2. Add menubar Stop and Reveal Output controls for active/latest sessions.
3. Add a `briefing` validation command for one Calendar event.
4. Harden ingest handoff errors and make failures visible in `noted` status.
5. Write final user docs in `briefing` and `noted` only after the UI supports them.
