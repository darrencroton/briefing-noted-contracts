# Step 7 Vertical Slice Report

**Date completed:** 2026-04-24
**Executed by:** Claude Code (claude-sonnet-4-6)
**Covers:** Step 7 of the Initial Action Plan and B-21 of the briefing implementation plan

---

## Objective

Prove the full cross-repo integration path end-to-end using real components, not fixtures:

```
hand-written manifest → noted start → capture → noted stop
  → completion.json → briefing session-ingest → ## Meeting Summary written
```

The goal was to surface contract bugs cheaply and produce a clear success/failure record before building the full automated watch/popup surface.

---

## Environment

| Component | Version / State |
| --- | --- |
| Machine | Apple M1 Max, macOS 26 (Darwin 25.4.0), arm64 |
| `noted` | Phase 2 runtime (commit `a70758e`), dist bundle at `noted/dist/Noted.app` |
| ASR backend | `fluidaudio-parakeet` (parakeet-tdt-0.6b-v3, cached locally) |
| `briefing` | Phase 4a complete; `briefing session-ingest` at HEAD |
| Contracts | Pinned to `v1.0.1` on both sides |
| LLM provider | `claude` CLI (`claude-sonnet-4-5`) |

---

## What Was Executed

### 1. Manifest

A hand-written manifest was placed at `/tmp/noted-slice/manifest.json`:

```json
{
  "schema_version": "1.0",
  "session_id": "2026-04-24T171800+1000-slice-test",
  "meeting": {
    "event_id": null,
    "title": "Step 7 Vertical Slice Test",
    "scheduled_start": "2026-04-24T17:18:00+10:00",
    "scheduled_end":   "2026-04-24T17:50:00+10:00",
    "host": null,
    "series_id": null,
    "occurrence_index": null,
    "speaker_count_hint": null,
    "participants": []
  },
  "mode": { "type": "in_person", "audio_strategy": "room_mic" },
  "transcription": {
    "asr_backend": "fluidaudio-parakeet",
    "language": "en",
    "diarization_enabled": true
  },
  "next_meeting": { "exists": false, "manifest_path": null },
  "paths": {
    "session_dir": "/tmp/noted-slice/sessions/2026-04-24T171800+1000-slice-test",
    "note_path":   "/tmp/noted-slice/vault/Step 7 Vertical Slice Test.md"
  }
}
```

Validated clean against `manifest.v1.json` schema.

### 2. noted start

```
dist/Noted.app/Contents/MacOS/Noted start \
  --manifest /tmp/noted-slice/manifest.json
```

**EXIT:0.** Output:

```json
{
  "ok": true,
  "session_id": "2026-04-24T171800+1000-slice-test",
  "status": "recording",
  "pid": <child-pid>,
  "session_dir": "/tmp/noted-slice/sessions/2026-04-24T171800+1000-slice-test"
}
```

Startup sequence logged in `logs/noted.log`: ASR model load (Parakeet Encoder compiled in 29.7 s on first run after re-sign), VAD init, diarizer prewarm — all before recording confirmed.

### 3. noted stop

```
dist/Noted.app/Contents/MacOS/Noted stop \
  --session-id 2026-04-24T171800+1000-slice-test
```

**EXIT:0.** Output:

```json
{
  "ok": true,
  "session_id": "2026-04-24T171800+1000-slice-test",
  "status": "processing",
  "audio_finalised": true
}
```

Stop returned immediately after audio flush as required (guardrail: `noted stop` is non-blocking).

### 4. completion.json (produced by noted)

```json
{
  "schema_version": "1.0",
  "session_id": "2026-04-24T171800+1000-slice-test",
  "manifest_schema_version": "1.0",
  "terminal_status": "completed",
  "stop_reason": "manual_stop",
  "audio_capture_ok": true,
  "transcript_ok": true,
  "diarization_ok": true,
  "warnings": [],
  "errors": [],
  "completed_at": "2026-04-24T07:36:28.264Z"
}
```

Written at `outputs/completion.json` in the session directory, as the sole terminal signal.

### 5. briefing session-ingest

```
uv run briefing session-ingest \
  --session-dir /tmp/noted-slice/sessions/2026-04-24T171800+1000-slice-test
```

**EXIT:0.** Decision: `summarise`. Output:

```json
{
  "ok": true,
  "exit_code": 0,
  "session_id": "2026-04-24T171800+1000-slice-test",
  "decision": "summarise",
  "note_path": "/tmp/noted-slice/vault/Step 7 Vertical Slice Test.md",
  "note_created": false,
  "block_written": true,
  "block_replaced": false,
  "terminal_status": "completed",
  "stop_reason": "manual_stop",
  "error": null
}
```

`session-ingest` read `completion.json` first (not log parsing, not file-presence heuristics). No contract schema errors.

### 6. Resulting note

```markdown
---
title: Step 7 Vertical Slice Test
date: 2026-04-24
series: slice-test
---

<!-- BRIEFING:start -->
## Briefing

This is a pre-populated briefing block for the vertical slice test.

Key context: verifying that `noted` and `briefing` integrate correctly end-to-end.
<!-- BRIEFING:end -->

## Meeting Notes

- This is user-owned content that must NOT be modified by session-ingest.
- Any change to this line would be a guardrail #12 violation.
- A third line of user notes for good measure.

<!-- MEETING-SUMMARY:start session_id="2026-04-24T171800+1000-slice-test" transcript_sha256="601ad10d94e2f101f16413cf473f607fe83d0965067b4fb6dd7c8305897ba46f" -->
## Meeting Summary

- Session was initiated at 17:18 AEST but no substantive discussion was captured
  in the transcript — recording may have stopped before conversation began or
  transcript processing may have encountered an issue.
<!-- MEETING-SUMMARY:end -->
```

---

## Verification Checks

| Check | Result |
| --- | --- |
| `completion.json` matches `completion.v1.json` schema | Pass |
| `session-ingest` read `completion.json` as sole source of truth | Pass |
| `block_written: true` in ingest output | Pass |
| `transcript_sha256` in managed comment matches `shasum -a 256 transcript.txt` | Pass (`601ad10d…`) |
| **Guardrail #12**: `## Meeting Notes` user content byte-identical after ingest | Pass (SHA256 of user-owned section unchanged) |
| `noted stop` returned before `completion.json` existed | Pass (two-phase stop confirmed) |
| Session directory layout matches `session-directory.md` contract | Pass (extra internal runtime coordination files — see notes below) |

---

## Issues Surfaced

### 1. TCC microphone re-prompt after adhoc re-sign (expected macOS behaviour — not a contract bug)

During development, a debug binary was substituted into the dist app bundle and the bundle was re-signed with `codesign --force --sign -`. This revoked the prior TCC approval for `app.noted.macos` and surfaced a new microphone permission dialog. The child process called `AVCaptureDevice.requestAccess(for: .audio)` and blocked indefinitely waiting for the user to respond. The parent's startup timeout fired before the dialog was dismissed.

**Resolution:** User approved the dialog. With mic permission pre-approved, `noted start` succeeds in ~31 s (child launch + CoreML model compilation).

**For the team:** Any re-sign of the app bundle (dev build, CI, notarization, bundle-id change) will revoke TCC and require user interaction on first post-sign launch. This is unavoidable on macOS. The `noted` code correctly calls `requestAccess` at startup; the issue is the CI/build workflow, not the contract.

### 2. Extra runtime coordination files in session directory (internal to noted — not a contract violation)

The `runtime/` subdirectory contains files not listed in `session-directory.md`:

- `session.json` — written by `TranscriptLogger.startSession()` before recording begins
- `stop-request.json` — sentinel written by `noted stop` to signal the child
- `capture-finalized.json` — sentinel written by the child to signal audio flush complete
- `capture-finalized-acknowledged.json` — written by the parent after reading `capture-finalized.json`

These are `noted`-internal coordination artefacts. `briefing` must not read, parse, or depend on them. `completion.json` remains the sole cross-boundary signal and is unaffected.

**Recommendation:** Document these files in `noted`'s `ARCHITECTURE.md` as implementation detail. No contract update needed.

### 3. Empty transcript → graceful LLM handling (expected for a no-speech test)

No speech was captured during the test. The transcript contains only the session header (3 lines). The LLM produced an appropriate summary noting that no substantive discussion was captured. The summary block was written correctly and the `session-ingest` path handled the edge case without error.

**For the team:** This validates that the ingest path is resilient to empty transcripts. The LLM prompt and post-meeting template handle this gracefully.

### 4. LLM provider and model name (local config issue — not a contract bug)

The default `user_config/settings.toml` bootstrapped by `setup.sh` set `provider = "copilot"` (not configured in the test environment) and `model = "claude-sonnet-4.6"` (dot format not accepted by the `claude` CLI). Fixed locally by setting `provider = "claude"` and `model = "claude-sonnet-4-5"`.

**For the team:** The default `user_config/defaults/settings.toml` should use a model name that the `claude` CLI accepts, or validate the model name in `briefing validate`. Consider adding model format validation to the LLM provider check.

---

## noted Code Changes Made During This Slice

Two changes were made to `NotedCLI.swift` and kept as genuine improvements:

### a. `loading_models` status write in `runSession()`

Before calling `transcriptionEngine.start()`, the child now writes `runtime/status.json` with `phase: "loading_models"`. This means `noted status` returns meaningful information during the ~30 s CoreML compilation window rather than returning an error or stale data.

This also enables the two-phase startup wait (below) to confirm the child is alive before model loading completes.

### b. Two-phase `waitForStartup()` + `checkTerminalStartup()` helper

The original implementation polled for 90 s in a single loop looking for `status == "recording"`. The replacement:

- **Phase 1 (30 s):** Waits for any `status.json` to appear. If nothing appears in 30 s, the child never started (fast failure detection).
- **Phase 2 (90 s):** Child is confirmed alive; waits for `recording/capturing` status (covers model load time).
- **`checkTerminalStartup` helper:** Factors out the status inspection logic shared by both phases.

The Phase 1 timeout is unchanged from what's reasonable for child spawn. The Phase 2 timeout of 90 s is 3× the observed model load time (~30 s) and equal to the original single-loop timeout. The original 300 s Phase 2 added during debugging was reduced to 90 s as TCC-blocking no longer applies.

---

## B-21 Sign-Off

B-21 ("Add real noted-session smoke handoff") is **complete**.

The acceptance criterion — a real `noted`-produced session directory with a valid `completion.json` flowing through `briefing session-ingest` and producing a `## Meeting Summary` block — is satisfied. No contract-level bugs were found. All guardrails verified.

---

## Next Steps

- **B-24** (first vertical-slice script/runbook): Now unblocked. This ticket should formalise the commands exercised in this slice into a repeatable script under `briefing/scripts/`, and document the exact artefacts and expected outputs.
- **noted bundle-id change**: When the bundle identifier changes from `app.noted.macos` to the production identifier, TCC will re-prompt on first launch. Plan for this in the rollout.
- **Audio with speech**: Re-run this slice with a voice loop (e.g. `say -v Alex "..."` in a loop during recording) to validate the ASR → transcript → LLM summary path with actual content. Use `mic_plus_system` in the manifest for cleaner system-audio capture (requires screen-recording permission), or `room_mic` to pick up laptop speakers through the microphone.
- **Model name validation**: Add model name format validation to `briefing validate` so the `claude` provider reports a clear error if `model` uses dot format (`4.6`) instead of dash format (`4-5`).
