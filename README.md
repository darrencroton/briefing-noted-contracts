# briefing-noted-contracts

Neutral contracts repo for the Meeting Intelligence System. Consumed by `briefing` (Python) and `noted` (Swift) via pinned semver tags.

Currently **empty**. Phase 1 (Master Plan §23) populates this repo with:

- `schemas/manifest.v1.json`
- `schemas/completion.v1.json`
- `schemas/runtime-status.v1.json`
- `contracts/cli-contract.md` — `noted`'s CLI surface
- `contracts/session-directory.md` — on-disk layout
- `contracts/versioning-policy.md`
- `fixtures/` — valid/invalid manifests, completion examples, smoke-test WAV
- `CHANGELOG.md`

## Decisions already locked (apply when authoring `manifest.v1.json`)

- `transcription.asr_backend` is an enum: **`"whisperkit"`** (default), `"fluidaudio-parakeet"`, `"sfspeech"`. Do **not** use `"faster-whisper"` or `"whisperx"` — the Swift stack was chosen on 2026-04-23. See `ASR Stack Recommendation — Swift vs Python.md` and Master Plan §15.2, §15.4, §27.9.

## Non-negotiables

- Contract changes are made **here first**, merged, tagged, then picked up deliberately on each side.
- No code in this repo. Schemas, contracts, fixtures, and changelog only.
- Breaking schema changes bump the major version and the `schema_version` string used in manifest/completion payloads.
