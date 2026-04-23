# Session Directory Layout (v1.0)

**Authoritative source:** Master Plan §11. This document fixes the on-disk contract that `briefing` and `noted` both depend on.

## Canonical layout

```
sessions/
  <session_id>/
    manifest.json
    runtime/
      status.json
      ui_state.json
    audio/
      raw_room.wav        (in_person)
      raw_mic.wav         (online / hybrid)
      raw_system.wav      (online / hybrid)
    transcript/
      transcript.json
      transcript.txt
      segments.json
    diarization/
      diarization.json
    outputs/
      completion.json
    logs/
      noted.log
      briefing.log
```

One session = one directory. Never share state between sessions. Never write outside the session directory except to the Obsidian note path specified in the manifest (`paths.note_path`).

## File-requirements table

The contract is not that every file always exists — it is that when a given condition has been reached, the listed files must exist.

| Condition              | Files that must exist                                    |
|------------------------|----------------------------------------------------------|
| Session starts         | `manifest.json`, `runtime/status.json`, `logs/noted.log` |
| Audio capture succeeds | At least one file in `audio/`                            |
| Transcript completes   | `transcript/transcript.json`, `transcript/transcript.txt` |
| Any terminal state     | `outputs/completion.json`                                |

Notes:

- `manifest.json` is a direct copy (or serialisation) of the manifest that was passed to `noted start` — never rewritten during the session.
- `runtime/status.json` is rewritten on every state or phase transition (§10.3).
- `runtime/ui_state.json` is ephemeral UI state (popup history, button presses, icon state). `briefing` does not read it; it may be deleted without loss of session integrity (§10.4).
- `outputs/completion.json` is the only authoritative record of session outcome (guardrail 3). `briefing` reads it first; it must never be inferred from file presence or log parsing.

## Transcript outputs

Locked filenames (§26.3):

- `transcript.txt` — plain text, optionally with bracketed speaker labels if diarization succeeded.
- `transcript.json` — structured segments with timestamps, speaker labels, and confidence.
- `segments.json` — optional, used for debugging or downstream diarization analysis.

Speaker labels produced by the diarizer are opaque identifiers (`speaker_0`, `speaker_1`, …). Binding opaque IDs to real names is handled by `briefing` at summarisation time using the `host_name` and `participant_names` hints (§15.4, §16.3–§16.4).

## Audio files by mode

| `mode.type`   | Files under `audio/`                         |
|---------------|----------------------------------------------|
| `in_person`   | `raw_room.wav`                               |
| `online`      | `raw_mic.wav`, `raw_system.wav`              |
| `hybrid`      | `raw_room.wav` (treated as `in_person` unless overridden) |

Default capture format is WAV 16-bit / 48 kHz mono for `in_person`. Retention is governed by the policy in master-plan §27.10 (30-day rolling window, FLAC compressed).

## Stop reason → terminal status mapping

Reproduced from §11.5. This is the contract for what `terminal_status` an ingest consumer should expect given a `stop_reason`.

| `stop_reason`                  | Implies `terminal_status` |
|--------------------------------|---------------------------|
| `manual_stop`                  | `completed` (or `completed_with_warnings` if processing produced any) |
| `scheduled_stop`               | `completed` (or `completed_with_warnings`) |
| `auto_switch_to_next_meeting`  | `completed` (or `completed_with_warnings`) |
| `startup_failure`              | `failed` |
| `capture_failure`              | `failed` |
| `processing_failure`           | `completed_with_warnings` if audio was captured; `failed` otherwise |
| `forced_quit`                  | `completed_with_warnings` if any usable artefacts were flushed; `failed` otherwise |

## Retention invariants

- Raw audio is preserved whenever capture succeeds, even on downstream failure (guardrail 10). `briefing session-reprocess` depends on this.
- Logs remain in `logs/` alongside the session for the lifetime of the session directory.
- `completion.json` is never deleted independently of the session directory.

## Invariants for writers

- `noted` writes everything except `logs/briefing.log` and (for calendar-driven sessions) `manifest.json`.
- `briefing` writes `manifest.json` for calendar-driven sessions, and appends to `logs/briefing.log` during planning and ingestion.
- Only `briefing` writes to the Obsidian note at `paths.note_path`; `noted` never touches it.
