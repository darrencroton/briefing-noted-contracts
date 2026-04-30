# briefing-noted-contracts

Root repository for the Meeting Intelligence System. Tracks project-wide planning documents and the `contracts/` subdirectory, which holds all neutral schemas, specs, and fixtures consumed by `briefing` (Python) and `noted` (Swift) via pinned semver tags.

`briefing/` and `noted/` are sibling repositories managed independently; they are not subdirectories of this repo.

## Current implementation status

Completed as of 2026-04-29:

- `noted` Phase 2, N-01 through N-13: minimal runtime, manifest validation, canonical session directory, fast stop, async post-processing, and completion output.
- `briefing` Phase 4a, B-12 through B-20: completion reader, transcript adapter, post-meeting prompt, managed `## Meeting Summary` writer, and `session-ingest`.
- Step 7 and B-21: real hand-written manifest vertical slice through `noted` capture and `briefing session-ingest`.
- `briefing` Phase 4b, B-01 through B-11: settings, `noted config` parsing, manifest assembly, `session-plan`, `watch`, invalidation, and watcher `launchd`.
- `noted` Phase 3 plus N-25, N-14 through N-20 and N-25: popup, `extend`, `switch-next`, auto-stop/auto-switch, `ui_state`, back-to-back tests, and next-manifest invalidation handling.
- Polish tickets N-21 through N-24 and B-22 through B-25: automatic ingest handoff, ad hoc canonical Start, cross-boundary diagnostics, dry-runs, formal smoke script, and user docs.
- Phase 5A pre-soak hardening: `noted wait`, Meeting Intelligence `validate` preflight, one-week soak runbook, and the summary eval set.
- Current UX polish: centered `noted` end-of-meeting popup, menubar settings for model cache status / input microphone / auto-ingest, noted-owned model cache under Application Support, visible `## Meeting Summary` sections without hidden Markdown comments, and multi-Mac `location_type` routing in `briefing watch`.

Next work:

- Phase 5 hardening should wait until the completed flows have real operational data.

## Planning documents (root)

Read these before making architectural decisions:

- `Meeting Intelligence System — Master Implementation Plan.md` — authoritative system design, schemas, CLI contract, phases, open questions (§27) with recorded decisions.
- `Supplemental Implementation Guardrails.md` — 12 non-negotiable invariants, derivative of the master plan. In any conflict, the master plan wins.
- `AGENTS.md` — guidance for AI agents working in this repo.

Earlier planning documents (`Initial Action Plan`, `ASR Stack Recommendation`) are preserved in `archive/`.

## `contracts/` — schemas, specs, and fixtures

`contracts/` contains the neutral contract surface consumed by `briefing` and
`noted`:

- `schemas/manifest.v1.json`
- `schemas/completion.v1.json`
- `schemas/runtime-status.v1.json`
- `cli-contract.md` — `noted`'s CLI surface
- `session-directory.md` — on-disk layout
- `versioning-policy.md`
- `fixtures/` — valid/invalid manifests, completion examples, smoke-test WAV
- `CHANGELOG.md`

## Decisions already locked (apply when authoring `manifest.v1.json`)

- `transcription.asr_backend` is an enum: **`"fluidaudio-parakeet"`** (default), `"whisperkit"`, `"sfspeech"`. Do **not** use `"faster-whisper"` or `"whisperx"` — the Swift stack was chosen on 2026-04-23. See Master Plan §15.2, §15.4, §27.9.

## Non-negotiables

- Contract changes are made in `contracts/` first, merged and tagged, then picked up deliberately on each side.
- No code in `contracts/`. Schemas, specs, fixtures, and changelog only.
- Breaking schema changes bump the major version and the `schema_version` string used in manifest/completion payloads.
