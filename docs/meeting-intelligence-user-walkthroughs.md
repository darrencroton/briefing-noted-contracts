# Meeting Intelligence User Walkthroughs

This guide covers the three common local workflows:

1. first-time setup of `briefing` and `noted`
2. one scheduled test meeting that generates a briefing, starts recording automatically, shows the end-of-meeting popup, and writes a summary
3. one ad hoc `noted` recording that is processed and ingested afterwards

The commands assume the repos live at:

```bash
/Users/dcroton/Local/git-repos/dev/briefing
/Users/dcroton/Local/git-repos/dev/noted
```

Run the commands from Terminal on the Mac that has Calendar, microphone access, and the LLM CLI you plan to use.

## Setup

### 1. Install prerequisites

Confirm these are installed:

- Apple Silicon Mac on macOS 26+
- Xcode 26.3+ command line tools
- Python 3.13+
- `uv`
- one authenticated LLM CLI supported by `briefing`: `claude`, `codex`, `copilot`, or `gemini`

```bash
xcodebuild -version
python3 --version
uv --version
```

### 2. Set common paths

```bash
export DEV_ROOT="/Users/dcroton/Local/git-repos/dev"
export BRIEFING_ROOT="$DEV_ROOT/briefing"
export NOTED_ROOT="$DEV_ROOT/noted"
export PATH="$HOME/.local/bin:$PATH"
```

Add `~/.local/bin` to your shell startup file if it is not already there:

```bash
grep -q 'HOME/.local/bin' "$HOME/.zshrc" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> "$HOME/.zshrc"
```

### 3. Set up `briefing`

```bash
cd "$BRIEFING_ROOT"
./scripts/setup.sh
```

For a deterministic local test, use a local test vault:

```bash
mkdir -p "$BRIEFING_ROOT/tmp/mi-test-vault/Internal/Meetings"
```

Edit `user_config/settings.toml` and make sure these values are set:

```toml
[paths]
vault_root = "/Users/dcroton/Local/git-repos/dev/briefing/tmp/mi-test-vault"
meeting_notes_dir = "Internal/Meetings"

[meeting_intelligence]
sessions_root = "sessions"
noted_command = "noted"
pre_roll_seconds = 60
watch_poll_seconds = 5
watch_lookahead_minutes = 180
default_language = "en-AU"
default_asr_backend = "whisperkit"
default_mode = "in_person"
auto_start = true
auto_stop = true
default_extension_minutes = 5
pre_end_prompt_minutes = 2
no_interaction_grace_minutes = 1

[calendar]
window_min_minutes = 1
window_max_minutes = 30
include_calendar_names = ["Calendar"]
```

Keep your existing `[llm]` provider if it is already authenticated. Otherwise set it now. Examples:

```toml
[llm]
provider = "copilot"
command = ""
model = "claude-sonnet-4.6"
effort = "high"
```

```toml
[llm]
provider = "claude"
command = ""
model = "claude-sonnet-4-5"
effort = "high"
```

Create a `briefing` command that `noted` can call after recording finishes:

```bash
mkdir -p "$HOME/.local/bin"
cat > "$HOME/.local/bin/briefing" <<'EOF'
#!/usr/bin/env bash
cd "/Users/dcroton/Local/git-repos/dev/briefing"
exec uv run briefing "$@"
EOF
chmod +x "$HOME/.local/bin/briefing"
```

### 4. Build and install `noted`

```bash
cd "$NOTED_ROOT"
cd Noted
swift build
cd ..
./scripts/release.sh test
```

Put the app executable on `PATH` as `noted`:

```bash
mkdir -p "$HOME/.local/bin"
ln -sf "$NOTED_ROOT/dist/Noted.app/Contents/MacOS/Noted" "$HOME/.local/bin/noted"
noted version
```

Launch the menubar app:

```bash
open "$NOTED_ROOT/dist/Noted.app"
```

Launching the app starts the menubar controller and end-of-meeting popup watcher. Microphone permission is requested when a recording starts, not merely when the app launches.

### 5. Configure `noted`

If this file does not exist yet, launch `noted` once:

```text
~/Library/Application Support/noted/settings.toml
```

Edit it so these values match the local test setup:

```toml
host_name = "Darren"
language = "en-AU"
asr_backend = "whisperkit"
asr_model_variant = "base"
output_root = "/Users/dcroton/Local/git-repos/dev/noted/sessions"
ad_hoc_note_directory = "/Users/dcroton/Local/git-repos/dev/briefing/tmp/mi-test-vault/Internal/Meetings/Ad Hoc"
briefing_command = "/Users/dcroton/.local/bin/briefing"
ingest_after_completion = true
diarization_enabled = true
default_extension_minutes = 5
pre_end_prompt_minutes = 2
no_interaction_grace_minutes = 1
```

### 6. Validate

```bash
cd "$BRIEFING_ROOT"
uv run briefing validate
```

Expected result:

- Calendar access is granted, or macOS prompts you to grant it.
- The configured LLM CLI is found and authenticated.
- `noted_command_ok` appears because `noted` is on `PATH`.
- Missing Slack or Notion tokens only matter if your series configs use those sources.

## Config

The scheduled walkthrough below uses one isolated series file. It intentionally has no Slack, Notion, file, or email sources, so the only automatic source is `previous_note`.

Set the Calendar name used by your test event. It must be included by `briefing`'s `include_calendar_names` setting.

```bash
export TEST_CALENDAR="Calendar"
```

Create a unique title:

```bash
export TEST_TITLE="MI Test Scheduled $(date +%Y%m%d-%H%M%S)"
```

Create the series file:

```bash
cd "$BRIEFING_ROOT"
mkdir -p user_config/series
cat > user_config/series/mi-test-scheduled.yaml <<EOF
series_id: mi-test-scheduled
display_name: "$TEST_TITLE"
note_slug: mi-test-scheduled
match:
  title_any:
    - "$TEST_TITLE"
  calendar_names_any:
    - "$TEST_CALENDAR"
sources:
  slack:
    channel_refs: []
    dm_conversation_ids: []
    required: false
  notion: []
  files: []
  email: []
recording:
  record: true
  mode: in_person
  participants:
    host_name: Darren
    attendees_expected: 1
  transcription:
    language: en-AU
    asr_backend: whisperkit
    diarization_enabled: true
  recording_policy:
    auto_start: true
    auto_stop: true
    default_extension_minutes: 5
    pre_end_prompt_minutes: 2
    no_interaction_grace_minutes: 1
EOF
```

## User Story 1: First-Time Local Setup

Complete the steps in [Setup](#setup), then run:

```bash
cd "$BRIEFING_ROOT"
uv run briefing validate
noted version
```

Expected result:

- `uv run briefing validate` exits successfully or reports only warnings you understand.
- `noted version` prints one JSON line.
- The `noted` menubar icon is visible.
- No microphone prompt is expected yet. It appears when the first recording starts.

## User Story 2: Scheduled Test Meeting

This test creates a Calendar event that starts 8 minutes from now and lasts 10 minutes. `briefing run` writes the pre-meeting note, then `briefing watch` starts `noted` 60 seconds before the event starts. The popup appears 2 minutes before the scheduled end.

### 1. Create the Calendar event

```bash
EVENT_UID=$(TEST_TITLE="$TEST_TITLE" TEST_CALENDAR="$TEST_CALENDAR" osascript <<'APPLESCRIPT'
set testTitle to system attribute "TEST_TITLE"
set calendarName to system attribute "TEST_CALENDAR"
set eventStart to (current date) + (8 * minutes)
set eventEnd to eventStart + (10 * minutes)
set eventNotes to "noted config:" & linefeed
set eventNotes to eventNotes & "record: true" & linefeed
set eventNotes to eventNotes & "mode: in_person" & linefeed
set eventNotes to eventNotes & "recording_policy:" & linefeed
set eventNotes to eventNotes & "  auto_start: true" & linefeed
set eventNotes to eventNotes & "  auto_stop: true" & linefeed
set eventNotes to eventNotes & "  default_extension_minutes: 5" & linefeed
set eventNotes to eventNotes & "  pre_end_prompt_minutes: 2" & linefeed
set eventNotes to eventNotes & "  no_interaction_grace_minutes: 1" & linefeed
set eventNotes to eventNotes & "transcription:" & linefeed
set eventNotes to eventNotes & "  language: en-AU" & linefeed
set eventNotes to eventNotes & "  asr_backend: whisperkit" & linefeed
set eventNotes to eventNotes & "  diarization_enabled: true"
tell application "Calendar"
  set targetCalendar to first calendar whose name is calendarName
  set newEvent to make new event at end of events of targetCalendar with properties {summary:testTitle, start date:eventStart, end date:eventEnd, description:eventNotes}
  return uid of newEvent
end tell
APPLESCRIPT
)
echo "$EVENT_UID"
```

Expected result: the command prints the Calendar event UID, and Calendar shows a 10-minute event with the `noted config` block in its notes.

### 2. Generate the briefing note

```bash
cd "$BRIEFING_ROOT"
uv run briefing validate
uv run briefing run
```

Expected result: `uv run briefing run` exits 0 and writes a note under:

```text
/Users/dcroton/Local/git-repos/dev/briefing/tmp/mi-test-vault/Internal/Meetings/
```

Confirm the managed briefing block exists:

```bash
rg -n "## Briefing|## Meeting Notes" "$BRIEFING_ROOT/tmp/mi-test-vault/Internal/Meetings"
```

### 3. Start the watcher before pre-roll

Run this while the event is still more than 60 seconds away:

```bash
uv run briefing watch
```

Leave it running through the start of the meeting. At 60 seconds before the event start, `briefing watch` invokes:

```text
noted start --manifest <manifest_path>
```

Expected result:

- If this is the first recording for this app signature, macOS asks for microphone permission. Grant it.
- `noted` begins recording.
- The menubar icon changes to a recording state.
- Session files appear under `briefing/sessions/<session-id>/`.

### 4. Confirm the popup

Two minutes before the scheduled end, the `noted` app should show a floating popup titled `Session ending soon`.

Expected popup buttons for this single scheduled event:

- `Stop`
- `+5 min`

There is no `Next Meeting` button unless the manifest contains a valid next-meeting manifest.

For the walkthrough, click `Stop` when the popup appears. If you click `+5 min`, the prompt should reappear before the extended end.

### 5. Wait for processing and ingest

Get the session id from `noted`'s active capture file before the recording stops, or from the latest registry file afterwards:

```bash
LATEST_REGISTRY=$(ls -t "$HOME/Library/Application Support/noted/sessions/"*.json | head -1)
SESSION_ID=$(/usr/bin/plutil -extract session_id raw -o - "$LATEST_REGISTRY")
SESSION_DIR=$(/usr/bin/plutil -extract session_dir raw -o - "$LATEST_REGISTRY")
echo "$SESSION_ID"
echo "$SESSION_DIR"
```

Wait for completion:

```bash
noted wait --session-id "$SESSION_ID" --timeout-seconds 900
```

Expected result: one JSON line with `"ok": true`, `"terminal_status": "completed"` or `"completed_with_warnings"`, and the `session_dir`.

Confirm the outputs:

```bash
test -f "$SESSION_DIR/outputs/completion.json"
test -f "$SESSION_DIR/transcript/transcript.txt"
test -f "$SESSION_DIR/logs/briefing-ingest.stdout.log"
rg -n "## Meeting Summary" "$BRIEFING_ROOT/tmp/mi-test-vault/Internal/Meetings"
```

Expected result: all `test` commands exit 0, and the note now contains a managed `## Meeting Summary` block.

## User Story 3: Ad Hoc Recording

Use this when you want to record immediately without a Calendar event.

Ad hoc sessions do not have a scheduled end time, so they do not show the end/extend popup. Stop them from the menubar app or the CLI.

### 1. Start the ad hoc session

Make sure the app is running:

```bash
open "$NOTED_ROOT/dist/Noted.app"
```

From the menubar, choose:

```text
Start Ad Hoc Session
```

If this is the first recording for this app signature, grant microphone permission.

### 2. Capture the active session id

```bash
ACTIVE_CAPTURE="$HOME/Library/Application Support/noted/runtime/active-capture.json"
SESSION_ID=$(/usr/bin/plutil -extract session_id raw -o - "$ACTIVE_CAPTURE")
SESSION_DIR=$(/usr/bin/plutil -extract session_dir raw -o - "$ACTIVE_CAPTURE")
echo "$SESSION_ID"
echo "$SESSION_DIR"
```

Expected result: both commands print non-empty values.

### 3. Record a short sample and stop

Speak for at least 60 seconds so the transcript has useful material.

```bash
noted status --session-id "$SESSION_ID"
sleep 60
noted stop --session-id "$SESSION_ID"
```

Expected result: `noted stop` returns quickly after audio is flushed. ASR, diarization, completion writing, and optional `briefing` ingest continue in the background.

### 4. Wait for completion

```bash
noted wait --session-id "$SESSION_ID" --timeout-seconds 900
```

Expected result: one JSON line with `"ok": true`.

Confirm the files:

```bash
test -f "$SESSION_DIR/manifest.json" || test -f "$SESSION_DIR/ad-hoc-manifest.json"
test -f "$SESSION_DIR/outputs/completion.json"
test -f "$SESSION_DIR/transcript/transcript.txt"
test -f "$SESSION_DIR/logs/briefing-ingest.stdout.log"
rg -n "## Meeting Summary" "$BRIEFING_ROOT/tmp/mi-test-vault/Internal/Meetings/Ad Hoc"
```

Expected result: `completion.json` exists, transcript output exists, and the ad hoc note has a managed `## Meeting Summary` block.

## Troubleshooting

### `briefing validate` cannot see Calendar

Open System Settings > Privacy & Security > Calendars and grant access to Terminal, iTerm, or whichever terminal app ran `uv run briefing validate`.

### The Calendar event command cannot find `Calendar`

Your default calendar has a different name. Set `TEST_CALENDAR` to an existing Calendar name, update `include_calendar_names` in `briefing/user_config/settings.toml`, and recreate the test event.

### `briefing run` does nothing

Check these first:

- the test event starts between `window_min_minutes` and `window_max_minutes`
- the event title exactly matches `user_config/series/mi-test-scheduled.yaml`
- the event is on a calendar included by `include_calendar_names`
- `uv run briefing validate` exits successfully

### `briefing watch` does not start recording

Check:

- `watch` was running before the pre-roll time
- `[meeting_intelligence].pre_roll_seconds` is between 60 and 180
- `noted version` works from the same shell
- the event is eligible for recording through either a matched series or a `noted config` marker
- `logs/history.log` contains a `noted_start` boundary entry

### The popup does not appear

Check:

- `dist/Noted.app` is running, not only the CLI
- the session is scheduled, not ad hoc
- the manifest has `meeting.scheduled_end_time`
- `recording_policy.pre_end_prompt_minutes` is greater than 0
- `runtime/pre-end-prompt.json` exists inside the active session directory near the scheduled end

### Ingest does not write a summary

Check:

- `~/Library/Application Support/noted/settings.toml` has `ingest_after_completion = true`
- `briefing_command` points to an executable command
- `logs/briefing-ingest.stderr.log` inside the session directory
- `uv run briefing session-ingest --session-dir "$SESSION_DIR" --dry-run`
