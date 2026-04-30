# Meeting Intelligence User Walkthroughs

This is a user-facing walkthrough for proving the three common Meeting Intelligence workflows:

1. first-time setup
2. a scheduled Calendar meeting that gets a briefing, starts recording automatically, and shows the end-of-meeting popup
3. an ad hoc recording from the `noted` menubar app, with the current user-facing limitations called out

The guide assumes the default local setup:

- your Calendar app has a calendar named `Calendar`
- `briefing` uses `user_config/settings.toml`
- `noted` is launched as a menubar app
- the default `briefing` Calendar window is still 10 to 45 minutes before the meeting
- the default recording pre-roll is still 90 seconds before the meeting
- the default end-of-meeting popup appears 5 minutes before the scheduled end

Use the existing defaults unless this guide explicitly says otherwise.

## Setup

### 1. Install `briefing`

From the `briefing` repo:

```bash
./scripts/setup.sh
```

Open `briefing/user_config/settings.toml` and check only the user-facing basics:

- `paths.vault_root` points at your Obsidian or Markdown vault
- `paths.meeting_notes_dir` is where meeting notes should be written inside that vault
- `calendar.include_calendar_names` includes `Calendar`
- `[llm]` is set to the LLM CLI you actually use

Do not change the recording timing defaults for this walkthrough.

### 2. Sign in to your LLM CLI

Use whichever provider you selected in `briefing/user_config/settings.toml`:

```bash
claude auth login
```

or:

```bash
codex login
```

or:

```bash
copilot login
```

or configure Gemini with an API key or Vertex credentials.

The selected provider must work without an interactive prompt once setup is complete.

### 3. Build and launch `noted`

From the `noted` repo:

```bash
cd Noted
swift build
cd ..
./scripts/release.sh test
open dist/Noted.app
```

The menubar icon should appear. Launching the app does not request microphone permission; macOS asks for that when the first recording starts.

### 4. Check `noted` settings in the app

Open the `noted` menubar menu and choose `Settings...`.

For this walkthrough, use:

- `Transcription Model`: `Whisper Base` or the default model if you prefer
- confirm the model cache status is either `Cached` or `Built in` before depending on offline use
- `Locale`: `en-AU`
- `Input Microphone`: `System Default` unless you intentionally want a specific microphone
- `Auto-ingest to Briefing`: on
- `Default Directory`: leave the default unless you intentionally want sessions somewhere else

Do not edit `~/Library/Application Support/noted/settings.toml` for this walkthrough.

### 5. Make sure validation passes

From the `briefing` repo:

```bash
uv run briefing validate
```

Expected result:

- Calendar access is granted, or macOS prompts for it
- the selected LLM CLI is found and authenticated
- `noted` is found on `PATH`

If `noted` is not found, add the app executable to your PATH once:

```bash
mkdir -p "$HOME/.local/bin"
ln -sf "/Users/dcroton/Local/git-repos/dev/noted/dist/Noted.app/Contents/MacOS/Noted" "$HOME/.local/bin/noted"
```

`noted` invokes `briefing session-ingest` automatically after a recording finishes when Auto-ingest is on and the `briefing` command is on the PATH visible to the app.

## Config

The scheduled test uses one normal Calendar event and one normal `briefing` series config. You should not need to create scripts or hand-write manifests.

### Calendar notes block

When the guide tells you to paste a `noted config` block into the Calendar event notes, paste this exactly:

```yaml
noted config:
record: true
mode: in_person
participants:
  host_name: Darren
  attendees_expected: 1
transcription:
  language: en-AU
  asr_backend: whisperkit
  diarization_enabled: true
```

This makes the event eligible for recording. It keeps the timing defaults from `briefing` and `noted`, including the end-of-meeting popup.

If you run `briefing watch` on more than one Mac, also configure `location_type` routing in `briefing/user_config/settings.toml` before this walkthrough. For a one-off test that should run from the current Mac, add a matching line under `noted config`, for example `location_type: office` or `location_type: home`.

## User Story 1: First-Time Local Setup

Follow [Setup](#setup), then confirm:

- `noted` is visible in the menubar
- `uv run briefing validate` succeeds
- Calendar permission has been granted
- the LLM CLI is authenticated
- no microphone prompt has appeared yet

Expected result: the system is installed and ready for either a scheduled meeting or an ad hoc recording.

## User Story 2: Scheduled Test Meeting

This proves the full scheduled flow:

1. Calendar event exists
2. `briefing` creates the pre-meeting note
3. `briefing watch` starts `noted` automatically before the meeting
4. `noted` shows the end-of-meeting popup
5. `noted` processes the recording
6. `briefing` writes the post-meeting summary when Auto-ingest is on and `briefing` is available on PATH

### 1. Create the test Calendar event

In Apple Calendar, create a new event:

- Calendar: `Calendar`
- Title: `MI Test Scheduled`
- Start: about 20 minutes from now
- Duration: 10 minutes
- Notes: paste the `noted config` block from [Config](#config)

Why 20 minutes? The default `briefing run` window processes meetings starting 10 to 45 minutes from now. This leaves enough time to create the series config, run the briefing, and start the watcher before recording pre-roll.

### 2. Create the series config

From the `briefing` repo:

```bash
uv run briefing init-series
```

If it lists several upcoming events, find `MI Test Scheduled` in the list and rerun with the shown index:

```bash
uv run briefing init-series --index 3
```

Use the actual index from your output.

Expected result: `briefing` writes a series YAML file under `briefing/user_config/series/`.

### 3. Generate the briefing

From the `briefing` repo:

```bash
uv run briefing run
```

Expected result: a meeting note is created in your configured meeting notes folder. It should contain:

- `## Briefing`
- `## Meeting Notes`

Open the note in Obsidian or your Markdown editor and confirm the briefing text is present.

### 4. Start automatic recording

Keep `noted` running in the menubar.

From the `briefing` repo:

```bash
uv run briefing watch
```

Leave this running. About 90 seconds before the Calendar event starts, `briefing watch` should start `noted`.

Expected result:

- if this is the first recording for this app build, macOS asks for microphone permission
- after permission is granted, `noted` begins recording
- the `noted` menubar icon changes to a recording state

### 5. Confirm the end-of-meeting popup

Because the event is 10 minutes long and the default popup timing is 5 minutes before the scheduled end, the popup should appear about 5 minutes after recording starts.

Expected popup buttons:

- `Stop`
- `+10 min`

For this walkthrough, click `Stop` when the popup appears.

If you click `+10 min` instead, the session should extend and the popup should appear again before the new end time.

### 6. Confirm processing

After stopping, give `noted` time to process the recording. The menubar status should move from recording to processing and then to a completed state.

Open the meeting note again. Expected result for the pre-meeting note:

- the original `## Briefing` block is still present
- your `## Meeting Notes` area is preserved
- a `---` divider and `## Meeting Summary` section have been appended at the end of the note

## User Story 3: Ad Hoc Recording

This proves the immediate recording path from the `noted` menubar app.

Ad hoc recordings do not have a scheduled end time, so they do not show the end-of-meeting popup.

### 1. Start recording

Open the `noted` menubar menu and choose:

```text
Start Ad Hoc Session
```

If macOS asks for microphone permission, grant it.

Speak for at least one minute so there is enough audio to transcribe.

### 2. Stop recording

Open the `noted` menubar menu and choose `Stop Recording`.

Expected result:

- recording stops quickly
- `noted` continues processing in the background
- the menubar status eventually reaches a completed state

### 3. Confirm the ad hoc output

Open the output directory shown in `noted` Settings.

Expected result:

- the latest ad hoc session directory contains recording output, transcript files, and `outputs/completion.json`
- the ad hoc note is written directly under the Default Directory as `<session-id>.md`
- if Auto-ingest is on and `briefing` is available on PATH, the note includes a `---` divider and `## Meeting Summary` section

## Product TODOs

These should be addressed before this walkthrough is copied into the `briefing` or `noted` repos as end-user documentation:

- Add a user-facing way to reveal the latest session directory from the `noted` menubar status panel.
- Add a user-facing way to rerun or trigger `briefing session-ingest` for the latest completed session.
- Make `briefing` setup install or expose a stable `briefing` command that `noted` can call without users creating wrapper scripts.
- Consider a `briefing` command or UI path that creates a test Calendar event and series config without asking users to inspect EventKit indexes.

## Troubleshooting

### `briefing validate` cannot see Calendar

Open System Settings > Privacy & Security > Calendars and grant access to the terminal app you used to run `uv run briefing validate`.

### `init-series` does not show the test event

Check:

- the event is on the `Calendar` calendar
- the event starts in the future
- `calendar.include_calendar_names` in `briefing/user_config/settings.toml` includes `Calendar`

### `briefing run` does nothing

Check:

- the event starts 10 to 45 minutes from now
- the series config was created for `MI Test Scheduled`
- `uv run briefing validate` succeeds
- the LLM CLI is authenticated

### Recording does not start automatically

Check:

- `noted` is running in the menubar
- `uv run briefing watch` is still running
- the event notes contain the `noted config` block
- if multi-Mac routing is configured, the event or default `location_type` matches this Mac
- the event has not already started
- `noted` is available on `PATH`

### The popup does not appear

Check:

- this is a scheduled Calendar recording, not an ad hoc recording
- the Calendar event has a scheduled end time
- `noted` is still running in the menubar
- the event has reached the default popup time, 5 minutes before scheduled end

### The scheduled summary is not written

Check the latest `noted` session output directory for:

- `outputs/completion.json`
- `transcript/transcript.txt`
- `logs/briefing-ingest.stderr.log`

If processing completed but ingest failed, rerun ingest manually from the `briefing` repo:

```bash
uv run briefing session-ingest --session-dir /path/to/session
```
