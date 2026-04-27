---
description: "Fetch a Teams meeting transcript by meeting ID and save it as Markdown in the spec directory"
---

# M365 Transcript Fetch to Markdown

You are fetching a Microsoft Teams meeting transcript and saving it as local Markdown.
This command performs ingestion only. Do not generate a specification in this command.

Follow each step sequentially. Do not skip steps.

---

## Step 0 — Pre-flight Checks

Run these checks before proceeding. Stop and report if any check fails.

### 0a. Verify M365 CLI is installed

```bash
m365 version
```

### 0b. Verify authentication

```bash
m365 status -o json
```

If not authenticated, stop and instruct the user to run `m365 login`.

### 0c. Load extension configuration

If `.specify/extensions/m365/m365-config.yml` exists, read defaults.

---

## Step 1 — Resolve Meeting and Output Path

Parse user input for:

| Parameter | Description |
|-----------|-------------|
| `meetingId` | Required Teams meeting ID |
| `transcriptId` | Optional transcript ID when multiple transcripts exist |
| `userId` / `userName` / `email` | Optional user selector for transcript APIs |
| `output` | Optional output Markdown path |

### 1a. Resolve meeting ID

If missing, ask the user for `meetingId`.

Optional discovery flow:

```bash
m365 teams meeting list --startDateTime <START_ISO> --endDateTime <END_ISO> -o json
```

If discovery fails due mailbox/licensing, ask the user for meeting ID directly.

### 1b. Resolve output path

If `output` is provided, use it.

Otherwise use:
- `.specify/specs/transcript-<meetingId>.md`

Sanitize filename characters to `a-z`, `A-Z`, `0-9`, `-`, `_`, and `.`.

### 1c. Enforce workspace-safe output

Resolve the absolute output path and verify it stays inside the current workspace root.
If outside workspace, stop and ask for a path under the project.

### 1d. Confirm with the user

Show meeting ID, transcript ID (if provided), and output path.
Ask for confirmation before fetching.

---

## Step 2 — Fetch Transcript Data

### 2a. List transcripts

```bash
m365 teams meeting transcript list --meetingId <MEETING_ID> -o json
```

If `userId`, `userName`, or `email` is specified, include the corresponding flag.

If no transcripts are returned, stop and explain that transcription must have been enabled for that meeting.

### 2b. Select transcript

- If `transcriptId` is provided, use it.
- If one transcript exists, use it.
- If multiple exist and no `transcriptId` is provided, ask the user which one to use.

### 2c. Download transcript VTT

Download to a temp file:

```bash
m365 teams meeting transcript get --meetingId <MEETING_ID> --id <TRANSCRIPT_ID> --outputFile <TEMP_FILE>.vtt
```

### 2d. Parse VTT into readable Markdown blocks

Extract:
- Caption timestamp
- Speaker (`<v SPEAKER>` when available)
- Spoken text

Format transcript entries as:

```text
[HH:MM:SS] Speaker: spoken text
```

Merge adjacent lines from the same speaker where appropriate.

Security check:
- Never include tokens, auth headers, or secret values.
- Output only transcript content and safe metadata.

### 2e. Cleanup temp files

Delete temp VTT when parsing is complete unless user explicitly requested keeping temp artifacts.

---

## Step 3 — Write Markdown Output

Write Markdown with frontmatter:

```markdown
---
Source:
  type: "m365-transcript"
  meetingId: "<meetingId>"
  transcriptId: "<transcriptId>"
  fetchedAt: "<ISO timestamp>"
---
```

Add body:

```markdown
# Meeting Transcript: <meeting title or meetingId>

- Meeting ID: <meetingId>
- Transcript ID: <transcriptId>

## Transcript

[00:00:05] Speaker A: ...
[00:00:12] Speaker B: ...
```

Verify file exists and is non-empty.

---

## Step 4 — Output Summary

Display:

```
✅ Transcript fetched successfully

📄 Output: <output-path>
📊 Summary:
   • Meeting ID: <meetingId>
   • Transcript ID: <transcriptId>
   • Transcript entries: <count>

💡 Next Step:
   Once saved, use this file as input to `speckit specify` to generate a specification with your presets applied.
```

---

## Notes

- Least-privilege delegated permissions: `OnlineMeetingTranscript.Read.All` and `Calendars.Read` (for discovery).
- This command is ingestion-only and must not generate a spec.
