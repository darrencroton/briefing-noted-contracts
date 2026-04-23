# Changelog

All notable changes to the `briefing` ↔ `noted` contracts are recorded here. Versions follow semver on the `briefing-noted-contracts` root repository; `schema_version` strings inside schema files track `<major>.<minor>` of the schema itself.

Rules for bumps and the change-proposal process live in `versioning-policy.md`.

## [1.0.0] — 2026-04-23

Phase 1 (Lock Contracts) of the Master Implementation Plan. First tagged release.

### Added

- `schemas/manifest.v1.json` — JSON Schema for the session manifest (Master Plan §8).
  - One canonical shape for both calendar-driven and ad hoc sessions; ad hoc sessions use nulls in the permitted slots for `meeting.event_id` and `meeting.scheduled_end_time`.
  - `transcription.asr_backend` locked to the three Swift backends: `whisperkit` (default), `fluidaudio-parakeet`, `sfspeech`. Python backends are not accepted.
  - `hooks.completion_callback` reserved and pinned to `null` in v1; completion handoff is performed by `noted` invoking `briefing session-ingest <session-dir>` directly (§27.6 decision (a)).
  - `recording_policy.max_single_extension_minutes` reserved but not required; runtime extension policy is documented in the master plan (§12.4 / §27.12 decision (c): user may keep extending).
- `schemas/completion.v1.json` — JSON Schema for `completion.json` (§11.3). Required: `schema_version`, `session_id`, `manifest_schema_version`, `terminal_status`, `stop_reason`, all three `*_ok` booleans, `warnings`, `errors`, `completed_at`.
- `schemas/runtime-status.v1.json` — JSON Schema for `runtime/status.json` (§10.3). No `schema_version` field; the filename carries the version, matching the master-plan example.
- `cli-contract.md` — `noted` CLI surface from §9: `start`, `stop`, `extend`, `switch-next`, `status`, `validate-manifest`, `version`; optional `wait`; exit codes; JSON stdout shapes.
- `session-directory.md` — canonical layout, file-requirements table, transcript filenames, audio files by mode, and the stop-reason → terminal-status mapping.
- `vocabulary.md` — locked vocabulary from §26: stop reasons, terminal statuses, runtime statuses, runtime phases, mode types, audio strategies, ASR backends, transcript filenames, timezone rule.
- `versioning-policy.md` — compatibility rule (§8.4), bump classification (patch / minor / major), change-proposal process, authorisation.
- `README.md` — purpose, consumption (git submodule pinned to tag; tarball-at-tag alternative), change-proposal summary, non-negotiables.
- `fixtures/` — directory placeholder; fixture content is scheduled for Step 5 of the action plan.

### Decisions reflected

All §27 decisions in the master plan that shape Phase 1 artefacts:

- §27.1 lifecycle model — (c) `briefing run` + `briefing watch`. Reflected indirectly: contracts do not presuppose a lifecycle.
- §27.2 marker vs series — (d) series **or** `noted config` marker. Reflected: manifest carries no gating field; gating is `briefing`'s concern upstream.
- §27.3 subcommand naming — (a) hyphenated. Reflected in `cli-contract.md` and `README.md` references (`briefing session-ingest`, `briefing session-plan`, `briefing session-reprocess`).
- §27.4 summary block placement — (b) appended after `## Meeting Notes`. Not a contract-schema concern; noted here for traceability.
- §27.5 partial-context policy — (b) lenient post-meeting. Not a schema concern.
- §27.6 completion handoff — (a) `noted` invokes `briefing session-ingest`. Reflected: `hooks.completion_callback` fixed to `null` in the manifest schema; `cli-contract.md` documents the out-of-band invocation.
- §27.7 `no_interaction_grace_minutes` default — (a) 5 min. Reflected in the manifest example and in `briefing`'s defaults (not hard-coded in the schema, which only validates shape).
- §27.9 diarization library — (a) FluidAudio. Reflected in the `asr_backend` enum and in the lock against Python backends.
- §27.11 audio device selection hint — (a) settings-only. Reflected: no device field in the manifest.
- §27.12 extension policy — (c) keep extending. Reflected: `max_single_extension_minutes` is reserved but not required; no cap in the schema.

Open items whose resolution is not yet reflected here because they do not affect Phase 1 schemas:

- §27.8 macOS system-audio capture — Phase 5.
- §27.10 retention policy — Phase 5.

[1.0.0]: https://github.com/darrencroton/briefing-noted-contracts/releases/tag/v1.0.0
