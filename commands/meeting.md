---
description: "Pull a Teams meeting transcript, extract decisions and action items, and generate a Spec Kit specification"
---

# Teams Meeting Transcript to Specification

You are converting a Microsoft Teams meeting transcript into a structured Spec Kit feature specification. Follow each step sequentially. Do not skip steps. Wait for user input when indicated.

---

## Step 0 — Pre-flight Checks

Run these checks before proceeding. Stop and report if any check fails.

### 0a. Verify M365 CLI is installed

```bash
m365 version
```

If this fails, stop and tell the user:
> M365 CLI is not installed. Install it with `npm install -g @pnp/cli-microsoft365` and then run `m365 login` to authenticate.

### 0b. Verify the user is authenticated

```bash
m365 status -o json
```

If the status shows the user is not logged in, stop and tell the user:
> You are not logged in to M365 CLI. Run `m365 login` and complete the device-code authentication flow, then retry this command.

### 0c. Load extension configuration

Check if a configuration file exists at `.specify/extensions/m365/m365-config.yml` in the project root. If it exists, read it and use its values as defaults. If it does not exist, proceed with user-supplied parameters only.

---

## Step 1 — Resolve Target Meeting

The user may provide a meeting ID inline with the command invocation. Parse the user's input for:

| Parameter | Description |
|-----------|-------------|
| `meetingId` | The base64-encoded meeting ID from Microsoft Teams |
| `userId` | (Optional) User ID if pulling transcript on behalf of another user |
| `userName` | (Optional) User principal name — alternative to userId |
| `email` | (Optional) User email — alternative to userId |

### 1a. Resolve the Meeting ID

**If `meetingId` is provided:** use it directly.

**If not provided:** Try the following discovery methods in order. Each may fail due to missing permissions or licensing — move to the next if one fails.

**Method 1 — M365 CLI meeting list** (requires `Calendars.Read` + Exchange Online mailbox):

Compute a start date (30 days ago or from the config `since` value) and run:
```bash
m365 teams meeting list --startDateTime <START_DATE_ISO> --endDateTime <END_DATE_ISO> -o json
```

> **Important:** Do NOT use `--email`, `--userId`, or `--userName` flags — these only work with application permissions, not delegated (device code) authentication.

If this returns **"The mailbox is either inactive, soft-deleted, or is hosted on-premise"**, the user's account has no Exchange Online mailbox. This is common in test/developer tenants. Move to Method 2.

If successful, display the list of recent online meetings with their subjects and dates, then ask:
> Which meeting would you like to pull the transcript for?

Extract the meeting ID from the selected meeting's `id` field.

**Method 2 — Manual fallback:**

All meeting discovery methods in the M365 CLI and Microsoft Graph ultimately require an Exchange Online mailbox. If the user's account doesn't have one, tell the user:
> ⚠️ I couldn't discover meetings automatically. Your account does not have an Exchange Online mailbox, which is required for meeting discovery.
>
> **Options:**
> 1. **Provide the meeting ID directly.** You can find it from:
>    - **Teams meeting URL:** The meeting ID is the base64-encoded string after `/meetingjoin/` in the join URL
>    - **Teams app:** Open the meeting → click meeting info/details → copy the join link
>    - **Outlook calendar:** Open the meeting event → look for the join URL in the body
> 2. **Add an Exchange Online license** to your account in the Microsoft 365 admin center, then retry.
> 3. **Use a production tenant** that has Exchange Online included in its licensing.

### 1b. Confirm with the user

Display the resolved parameters:
> **Meeting ID:** `<meetingId>` (truncated for readability)
> **User:** `<userId/userName/email or "current user">`
>
> Proceed?

Wait for the user to confirm.

---

## Step 2 — Fetch the Meeting Transcript

### 2a. List available transcripts

```bash
m365 teams meeting transcript list --meetingId <MEETING_ID> -o json
```

If the user specified a `userId`, `userName`, or `email`, add the corresponding flag:
```bash
m365 teams meeting transcript list --meetingId <MEETING_ID> --userName <UPN> -o json
```

If no transcripts are returned, stop and tell the user:
> No transcripts found for this meeting. Transcripts are only available if recording/transcription was enabled during the meeting. Check that:
> 1. Transcription was turned on during the meeting
> 2. You have permission to access the transcript
> 3. The meeting ID is correct

### 2b. Select the transcript

If multiple transcripts exist, display them with their `createdDateTime` and ask the user which one to use. If only one exists, use it automatically.

### 2c. Download the transcript content

Determine the temp directory:
- If config has `meetings.transcriptTempDir` set to a path, use that
- Otherwise, use the system temp directory

```bash
m365 teams meeting transcript get --meetingId <MEETING_ID> --id <TRANSCRIPT_ID> --outputFile <TEMP_DIR>/transcript-<TRANSCRIPT_ID>.vtt
```

### 2d. Read and parse the VTT file

Read the downloaded `.vtt` file content. Parse the WebVTT format to extract:

| Field | Description |
|-------|-------------|
| Timestamp | Start and end time of each caption block |
| Speaker | The speaker label (usually `<v SPEAKER_NAME>` in VTT) |
| Text | The spoken content |

Produce a clean conversation log, merging consecutive entries from the same speaker into coherent statements. The format should be:

```
[HH:MM:SS] Speaker Name: Spoken text content...
[HH:MM:SS] Another Speaker: Their spoken text...
```

### 2e. Cleanup

If the config has `meetings.cleanupTranscripts` set to `true` (or unset, defaulting to true), delete the temporary VTT file after parsing.

If zero transcript content is extracted, stop and tell the user:
> The transcript file was empty or could not be parsed. The meeting may not have had any transcribed content.

---

## Step 3 — Fetch the Spec Kit Specification Template

Attempt to retrieve the latest spec template. Try these sources in order:

1. **Local project template:** Read `.specify/templates/spec-template.md` from the project root.
2. **Remote template:** If no local template exists, fetch from:
   ```
   https://raw.githubusercontent.com/github/spec-kit/main/templates/spec-template.md
   ```
3. **Built-in fallback:** If both fail, use this minimal structure:

```markdown
# Feature Specification: [FEATURE NAME]

<!-- yaml frontmatter -->
Feature Branch: [###-feature-name]
Created: [DATE]
Status: Draft

## User Scenarios & Testing

### User Story 1: [STORY TITLE]
**Priority:** P1
**As a** [role], **I want** [action], **so that** [outcome].

#### Acceptance Scenarios
- **Given** [context], **When** [action], **Then** [result].

#### Edge Cases
- [Edge case description]

## Requirements

### Functional Requirements
- **FR-001:** [Requirement description]

### Key Entities
- [Entity]: [Description]

## Success Criteria
- [Measurable outcome]

## Assumptions
- [Assumption]
```

---

## Step 4 — Analyze and Summarize the Transcript

Using the parsed transcript from Step 2, perform a thorough analysis. Meeting transcripts have unique characteristics compared to chat messages — speakers talk at length, topics shift organically, and decisions may be stated verbally without formal confirmation.

### 4a. Identify the Feature

Determine the primary topic or feature discussed in the meeting. If the meeting covered multiple features, identify the primary one — or ask the user which topic to focus on if there are clearly distinct, unrelated topics.

Derive a concise feature name for the spec title and a kebab-case branch name (e.g., `payment-processing-redesign`).

### 4b. Extract Decision Points

Scan the transcript for explicit decisions. Look for verbal cues:
- **Strong decision signals:** "let's go with", "we've decided", "the decision is", "agreed", "we'll do", "final call", "that's confirmed", "motion carried", "everyone good with that? ...yes"
- **Weak decision signals:** "I think we should", "sounds good", "makes sense" (these may be individual opinions rather than group decisions — require corroboration from other speakers)
- **Anti-patterns (NOT decisions):** "maybe we could", "what about", "one option is", "let me think about it", "we need to discuss further"

For each decision, note:
- What was decided
- Who proposed it
- Whether it had explicit agreement from others
- Confidence level: **High** (explicit group agreement), **Medium** (proposed and not contested), **Low** (suggested by one person, unclear group buy-in)

### 4c. Extract Action Items

Look for commitments individuals made during the meeting:
- "I'll take care of...", "I'll follow up on...", "Let me handle..."
- "Can you [name] look into...", "[Name], you'll do..."
- "Action item: ..."
- "By [deadline], we need..."

For each action item, capture:
- The action description
- The assigned owner (speaker name)
- The deadline (if mentioned)
- Whether it relates to a specific decision or requirement

### 4d. Extract Requirements

From the decisions and discussion, derive:
- **Functional requirements** (FR-001, FR-002, ...): what the system must do
- **Non-functional requirements** (if discussed): performance, security, scalability constraints

### 4e. Derive User Stories

Identify user stories implied in the meeting discussion. Structure each as:
> **As a** [role], **I want** [action], **so that** [outcome].

Assign priorities:
- **P1** — Explicitly discussed as critical, must-have, or blocking
- **P2** — Discussed as important but not critical-path
- **P3** — Briefly mentioned, nice-to-have

### 4f. Derive Acceptance Criteria

For each user story, extract or infer acceptance criteria in Given/When/Then format. Meeting transcripts often contain richer context for acceptance criteria because speakers describe expected behaviors conversationally.

### 4g. Distinguish Exploration from Conclusions

Meetings often have lengthy exploration phases followed by brief conclusions. Carefully separate:
- **Brainstorming segments** — ideas thrown around, alternatives explored, no commitment made
- **Decision segments** — points where the group converges on an approach
- **Parking lot items** — topics raised but explicitly deferred

Mark brainstorming content as context, not requirements.

### 4h. Flag Open Questions

Identify topics where:
- A question was asked but not answered
- Participants disagreed and the meeting ended without resolution
- Something was deferred to a future meeting
- Action items were assigned to resolve an open question

### 4i. Identify Participants and Roles

List all unique speakers from the transcript with:
- Display name
- Approximate speaking time (based on transcript entries)
- Their apparent role (if discernible — e.g., "product owner", "tech lead", "designer")
- Any action items assigned to them

---

## Step 5 — Generate the Specification

### 5a. Determine the spec number

If the config has `output.specNumber` set to a number, use it. If set to "auto" or not set, scan the `.specify/specs/` directory for existing spec folders (named `NNN-*`) and use the next sequential number, zero-padded to 3 digits.

### 5b. Create the spec directory and file

Create the directory: `.specify/specs/<NNN>-<feature-name>/`

Generate `spec.md` using the template from Step 3, populated with the analysis from Step 4. Include a **Source** metadata block at the top:

```markdown
---
Feature Branch: <NNN>-<feature-name>
Created: <today's date>
Status: Draft
Source:
  type: meeting-transcript
  meetingId: "<meeting ID (truncated)>"
  transcriptId: "<transcript ID>"
  meetingDate: "<meeting date from transcript metadata>"
  duration: "<approximate duration based on transcript timestamps>"
  participants:
    - "<speaker 1>"
    - "<speaker 2>"
  decisionCount: <number of decisions extracted>
  actionItemCount: <number of action items>
  confidenceNote: "Decisions marked with confidence levels — review LOW confidence items"
  extractedAt: "<current timestamp>"
---
```

### 5c. Populate the spec content

Fill in ALL sections of the spec template:
- Feature name as the title
- User stories with priorities, acceptance scenarios, and edge cases
- Functional requirements with IDs (FR-001, FR-002, ...)
- Key entities mentioned in the meeting
- Success criteria derived from the decisions
- Assumptions flagged from the analysis

Add additional sections at the end:

```markdown
## Decisions Log

<!-- Decisions extracted from the meeting transcript with confidence levels -->
| # | Decision | Proposed By | Confidence | Notes |
|---|----------|-------------|------------|-------|
| D1 | [Decision text] | [Speaker] | High/Medium/Low | [Context] |

## Action Items

<!-- Action items assigned during the meeting -->
| # | Action | Owner | Deadline | Status |
|---|--------|-------|----------|--------|
| A1 | [Action description] | [Speaker] | [Date or TBD] | Pending |

## Open Questions & Unresolved Items

<!-- Items from the meeting that need further discussion or clarification -->
- [ ] [Open question 1]
- [ ] [Open question 2]

## Meeting Context

<!-- Key discussion points that informed this spec but are not requirements -->
- [Exploration/brainstorming point 1]
- [Parking lot item 1]
```

---

## Step 6 — Output Summary

After generating the spec, display a summary:

```
✅ Specification generated from meeting transcript

📄 File: .specify/specs/<NNN>-<feature-name>/spec.md

📊 Extraction Summary:
   • Feature: <feature name>
   • User Stories: <count> (P1: <n>, P2: <n>, P3: <n>)
   • Requirements: <count> functional requirements
   • Decisions Captured: <count> (High: <n>, Medium: <n>, Low: <n>)
   • Action Items: <count> assigned to <owner count> people
   • Open Questions: <count>
   • Transcript Duration: ~<duration>
   • Speakers: <count> participants

⚠️  Items Needing Attention:
   • <count> LOW confidence decisions — review these carefully
   • <count> action items with no deadline
   • <count> open questions requiring follow-up
   • <any other flagged items>
```

If there are open questions or low-confidence decisions, recommend:
> Consider running `/speckit.clarify` to resolve open questions, or reconvene with the meeting participants to confirm low-confidence decisions.

---

## Error Handling

- **M365 CLI not found:** Direct user to install with `npm install -g @pnp/cli-microsoft365`
- **Not authenticated:** Direct user to run `m365 login`
- **Calendar 404 (no mailbox):** The user's account does not have an Exchange Online mailbox. This is common in developer/test tenants. Meeting discovery falls back to the Online Meetings API or manual meeting ID entry.
- **Online Meetings 403:** The app registration is missing the `OnlineMeetings.Read` delegated permission. Direct the user to add it in the Azure/Entra portal and grant admin consent.
- **Chat 403:** The app registration is missing the `Chat.Read` delegated permission. This is an optional discovery method — other methods may still work.
- **No transcripts found:** Explain that transcription must be enabled during the meeting
- **Transcript permission denied:** Meeting transcript access requires either:
  - Being the meeting organizer, or
  - Having `OnlineMeetingTranscript.Read.All` delegated permission, or
  - Having application permissions with an application access policy configured by a tenant admin
- **Rate limiting:** If M365 CLI returns a throttling error, wait 30 seconds and retry once
- **VTT parse failure:** If the transcript file format is unexpected, display the raw content and attempt best-effort extraction

---

## Notes

- The meeting transcript API is currently in **preview** in Microsoft Graph. Behavior may change.
- **Exchange Online mailbox required for meeting discovery:** The `m365 teams meeting list` command and all Graph calendar/meeting APIs require the user to have an Exchange Online mailbox. Developer/test tenants without Exchange Online licenses cannot discover meetings automatically — the user must provide the meeting ID manually.
- **Required permissions for meeting discovery:** `Calendars.Read` (delegated). For transcripts: `OnlineMeetingTranscript.Read.All`. The `--email`/`--userId`/`--userName` flags on `m365 teams meeting list` only work with application permissions, not delegated auth.
- Transcript quality depends on audio quality, speaker clarity, and language settings during the meeting. Names may be misattributed if speaker identification was imperfect.
- For meetings with multiple distinct topics, consider running this command multiple times — once per topic — and editing the meeting ID or asking the agent to focus on a specific time range or topic.
- This command generates a **draft** specification. The user should review and refine it, then run `/speckit.clarify` to resolve open questions and `/speckit.plan` to create an implementation plan.
- Action items extracted from the transcript are informational. They are included in the spec for traceability but should be tracked in your team's task management system.
