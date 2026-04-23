# ASR / Diarization Stack Recommendation — Swift vs Python

**Date:** 2026-04-23
**Scope:** Meeting Intelligence System, `noted` capture agent
**Status:** Recommendation — requires decision before Phase 1 contract freeze

---

## 1. What HushScribe gives you out of the box

`noted/` (pre-strip) ships a working, fully on-device Swift stack in `HushScribe/Sources/HushScribe/Transcription/` and `Services/`:

- **ASR — three backends behind an `ASRBackend` protocol:**
  - `FluidAudioASRBackend` — Parakeet-TDT v3 (default)
  - `WhisperKitBackend` — Whisper Base / Large v3
  - `SFSpeechBackend` — Apple Speech framework
- **VAD:** Silero, via FluidAudio, in front of the ASR pass.
- **Diarization:** `OfflineDiarizerManager` (FluidAudio), run **post-session** on the system-audio stream, with a `SpeakerNamingView` that lets the user bind labels to real names.
- **Streaming architecture:** dual `StreamingTranscriber`s over mic (`AVAudioEngine`) and system audio (`ScreenCaptureKit`, filtered to the active conferencing app).
- **Supporting on-device LLM:** `LLMSummaryEngine` via `mlx-swift-lm` (Qwen3 / Gemma 3) — not relevant to us (summaries live in `briefing`) and should be removed in triage.

Net: the ASR + VAD + diarization + speaker-naming pipeline already exists, compiled, and running on Apple Silicon with zero Python in the process.

---

## 2. Python (`whisperx` + `pyannote.audio`) vs Swift (`WhisperKit` + `FluidAudio`)

### Reasons to prefer Python

- **Diarization quality ceiling.** `pyannote.audio` 3.x still leads published DER benchmarks, particularly for overlapped speech and >3-speaker meetings. FluidAudio's diarizer is good but not state-of-the-art.
- **Word-level alignment.** `whisperx` adds forced alignment + word timestamps that WhisperKit does not natively produce at the same fidelity.
- **Language coverage.** pyannote embeddings and Whisper multilingual perform well across many languages; Parakeet-TDT is English-strong, weaker elsewhere.
- **Model cadence.** The Hugging Face ecosystem gets new checkpoints first; WhisperKit and FluidAudio lag.

### Reasons to prefer Swift (stronger for our use case)

- **Process and install simplicity.** `noted` is a signed/notarized menu-bar `.app`. Adding Python means bundling an interpreter + Torch + models, or shelling out to a sidecar daemon — either path breaks the single-binary distribution and the TCC model.
- **Latency and "fast stop."** Guardrail: `noted stop` is fast; ASR/diarization run async post-stop. CoreML/MLX execution on Apple Silicon is materially faster cold-start than Torch-on-CPU/MPS, which matters for back-to-back meetings.
- **Primary language is `en-AU`.** The quality delta where Python wins (overlapped speech, non-English) is not where we live day-to-day.
- **Maintenance surface.** A Python stack inside a Swift app is two toolchains, two dependency resolvers, and two sets of code-signing problems. The Swift stack is already working in `noted`.

### Recommendation

**Keep the Swift stack.** Re-evaluate only if real-world diarization on our data falls below a useful threshold — §27.9 already says to reassess after the first dataset is in hand.

---

## 3. Required edits to the master plan

### §15.2 ASR Backend — change default and enumerate

> Default: `whisperkit` (Whisper Large v3) or `fluidaudio-parakeet` (Parakeet-TDT v3) as configured in `noted`. The manifest's `transcription.asr_backend` field records which was used. Model size / variant is a `noted` setting, not a per-session manifest value.

### §15.4 Diarization — replace the pyannote reference

> Default: FluidAudio offline diarizer, run post-capture on the system-audio stream. May fail cleanly — the pipeline must continue with a diarization-less transcript, and `diarization_ok: false` must be set in `completion.json` with a warning.

### §27.9 Diarization Library — rewrite options and decision

> **Options:**
> - (a) FluidAudio offline diarizer — Swift, in-process, already integrated in `noted`.
> - (b) `pyannote.audio` via Python sidecar — best-in-class quality at the cost of a second runtime.
> - (c) `whisperx` — moot; requires a Python runtime that `noted` does not host.
>
> **Decision: (a).** Keeps `noted` a single signed Swift binary, preserves fast-stop semantics, avoids a Python runtime in the capture agent. Reassess against real data per the existing "re-evaluate after first dataset" clause.

### Cross-reference cleanup

Remove the "wraps `faster-whisper`" framing wherever it appears (§15.2 prose, §27.9, §26 references section), and drop `faster-whisper` from the external-references list unless retained for `briefing session-reprocess`.

---

## 4. Manifest field: does `transcription.asr_backend` change?

**Yes — rename the default value, keep the field.** In schema v1.0 the field should carry the actual backend string used at capture time, drawn from a small enum:

```json
"asr_backend": "whisperkit" | "fluidaudio-parakeet" | "sfspeech"
```

The field itself stays — it is already doing the right job: recording provenance on the transcript. Default in example payloads changes from `"faster-whisper"` to whichever of `whisperkit` / `fluidaudio-parakeet` is picked as the `noted` default. Specifically:

- Master plan §5.3 example (line ~390)
- Master plan §18 example (line ~1097)
- Master plan settings table (line ~1119)

**Recommended default:** `whisperkit`, for multilingual robustness, with `fluidaudio-parakeet` as the selectable fast-path for English-only sessions.

This is a contract change — it belongs in `contracts/` **before** the v1.0 tag is cut, so it lands in the first pinned schema and does not force a migration later.

---

## 5. Summary

| Question | Answer |
|---|---|
| Keep existing Swift stack? | Yes |
| Adopt Python `whisperx`/`pyannote`? | No, not for v1 |
| Change master plan §15.2? | Yes — enumerate Swift backends, change default string |
| Change master plan §15.4? | Yes — default is FluidAudio offline diarizer |
| Change master plan §27.9? | Yes — decision is (a) FluidAudio, not (b) whisperx |
| Keep manifest `transcription.asr_backend` field? | Yes |
| Change its default value in schema v1.0? | Yes — from `"faster-whisper"` to `"whisperkit"` |
| Contract-repo action before v1.0 tag? | Update default + enum before tagging |
