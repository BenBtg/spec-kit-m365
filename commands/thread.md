---
description: "Pull a Teams channel thread, summarize decisions and requirements, and generate a Spec Kit specification"
---

# Teams Thread to Specification

You are converting a Microsoft Teams channel conversation into a structured Spec Kit feature specification. Follow each step sequentially. Do not skip steps. Wait for user input when indicated.

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

Check if a configuration file exists at `.specify/extensions/m365/m365-config.yml` in the project root. If it exists, read it and use its values as defaults for any parameters the user did not provide. If it does not exist, that is fine — proceed with user-supplied parameters only.

---

## Step 1 — Resolve Target Team, Channel, and Thread

The user may have provided parameters inline with the command invocation. Parse the user's input for any of these:

| Parameter | Description |
|-----------|-------------|
| `teamId` | The GUID of the Teams team |
| `teamName` | The display name of the team (used to look up the ID) |
| `channelId` | The ID of the channel (e.g., `19:...@thread.tacv2`) |
| `channelName` | The display name of the channel (used to look up the ID) |
| `messageId` | The ID of a specific thread root message (to pull that thread only) |
| `since` | ISO 8601 date — only pull messages created/modified after this date |

### 1a. Resolve the Team

**If `teamId` is provided:** use it directly.

**If `teamName` is provided but not `teamId`:** look up the team:
```bash
m365 teams team list --joined -o json
```
Find the team whose `displayName` matches `teamName`. If no match, show the list of joined teams and ask the user to pick one.

**If neither is provided:** check the config file for `defaults.teamId` or `defaults.teamName`. If still missing, list teams and ask:
```bash
m365 teams team list --joined -o json
```
Display the team names and IDs, then ask:
> Which team should I pull messages from? Provide the team name or ID.

### 1b. Resolve the Channel

**If `channelId` is provided:** use it directly.

**If `channelName` is provided but not `channelId`:** look up the channel:
```bash
m365 teams channel list --teamId <TEAM_ID> -o json
```
Find the channel whose `displayName` matches `channelName`. If no match, show the list and ask.

**If neither is provided:** check config for `defaults.channelId` or `defaults.channelName`. If still missing, list channels and ask:
```bash
m365 teams channel list --teamId <TEAM_ID> -o json
```
Display the channel names and IDs, then ask:
> Which channel should I pull messages from? Provide the channel name or ID.

### 1c. Resolve the time filter

**If `since` is provided:** use it directly.

**If not provided:** check config for `defaults.since`. If the config value is a relative duration like `7d`, compute the ISO 8601 date by subtracting that duration from today's date.

**If still not set:** default to 7 days ago from today.

### 1d. Confirm with the user

Display the resolved parameters:
> **Target:** Team `<teamName>` → Channel `<channelName>`
> **Thread:** `<messageId>` (or "All recent messages")
> **Since:** `<since date>`
>
> Proceed?

Wait for the user to confirm before continuing to Step 2.

---

## Step 2 — Fetch Messages

### 2a. Pull channel messages

```bash
m365 teams message list --teamId <TEAM_ID> --channelId <CHANNEL_ID> --since <SINCE_DATE> -o json
```

### 2b. If a specific `messageId` was provided, also fetch its replies

```bash
m365 teams message reply list --teamId <TEAM_ID> --channelId <CHANNEL_ID> --messageId <MESSAGE_ID> -o json
```

When a specific `messageId` is given, filter the channel messages to include only the root message matching that ID, plus all its replies from the reply list. When no `messageId` is given, use all messages from Step 2a.

### 2c. Flatten into a conversation

From the JSON results, extract and sort chronologically:

| Field | Source |
|-------|--------|
| Timestamp | `createdDateTime` |
| Sender | `from.user.displayName` |
| Content | `body.content` |
| Message ID | `id` |
| Is Reply | `replyToId` is not null |
| Importance | `importance` |

Produce a clean, readable conversation log. Strip HTML tags from `body.content` if `body.contentType` is `"html"`. Preserve @mentions.

If zero messages are returned, stop and tell the user:
> No messages found for the specified channel and time range. Try a broader `--since` date or check that you have access to this channel.

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

## Step 4 — Analyze and Summarize the Conversation

Using the conversation log from Step 2, perform a thorough analysis. This is the critical summarization step.

### 4a. Identify the Feature

Determine the primary topic or feature being discussed. Derive a concise feature name suitable for a spec title and a kebab-case branch name (e.g., `user-authentication-flow`).

### 4b. Extract Final Decisions

Carefully distinguish between:
- **Final decisions** — statements where participants agreed, confirmed, or concluded something. Look for phrases like "let's go with", "agreed", "decided", "we'll do", "final answer", "confirmed".
- **Exploratory discussion** — brainstorming, questions, alternatives considered but not chosen. These should be noted as context but NOT treated as requirements.

**Weighting rule:** Later messages in the conversation carry more weight than earlier ones. If an earlier decision was revisited and changed later, use the later decision.

### 4c. Extract Requirements

From the final decisions, derive:
- **Functional requirements** (FR-001, FR-002, ...): what the system must do
- **Non-functional requirements** (if discussed): performance, security, scalability constraints

### 4d. Derive User Stories

Identify user stories implied in the conversation. Structure each as:
> **As a** [role], **I want** [action], **so that** [outcome].

Assign priorities based on how emphatically or frequently they were discussed:
- **P1** — Explicitly agreed as must-have or discussed as core
- **P2** — Mentioned as important but not critical-path
- **P3** — Nice-to-have, briefly mentioned

### 4e. Derive Acceptance Criteria

For each user story, extract or infer acceptance criteria in Given/When/Then format from the conversation. If the conversation didn't specify criteria explicitly, derive reasonable ones from the requirements discussed.

### 4f. Flag Open Questions

Identify any topics where:
- A question was asked but never answered
- Participants disagreed and no resolution was reached
- Something was deferred ("we'll figure that out later")
- Ambiguity remains

List these as open questions in a dedicated section.

### 4g. Identify Participants

List all unique participants from the conversation with:
- Display name
- Number of messages they sent
- Their apparent role (if discernible from context — e.g., "PM", "dev lead", "designer")

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
  type: teams-thread
  team: "<team name>"
  channel: "<channel name>"
  threadId: "<messageId or 'channel-wide'>"
  dateRange: "<since date> to <latest message date>"
  participants:
    - "<name 1>"
    - "<name 2>"
  messageCount: <number>
  extractedAt: "<current timestamp>"
---
```

### 5c. Populate the spec content

Fill in ALL sections of the spec template:
- Feature name as the title
- User stories with priorities, acceptance scenarios, and edge cases
- Functional requirements with IDs (FR-001, FR-002, ...)
- Key entities mentioned in the conversation
- Success criteria derived from the decisions
- Assumptions flagged from the analysis

Add an additional section at the end:

```markdown
## Open Questions & Unresolved Items

<!-- Items flagged from the source conversation that need clarification -->
- [ ] [Open question 1]
- [ ] [Open question 2]

## Conversation Context

<!-- Key discussion points that informed this spec but are not requirements -->
- [Context point 1]
- [Context point 2]
```

---

## Step 6 — Output Summary

After generating the spec, display a summary:

```
✅ Specification generated successfully

📄 File: .specify/specs/<NNN>-<feature-name>/spec.md

📊 Extraction Summary:
   • Feature: <feature name>
   • User Stories: <count> (P1: <n>, P2: <n>, P3: <n>)
   • Requirements: <count> functional requirements
   • Decisions Captured: <count>
   • Open Questions: <count>
   • Source Messages: <count> messages from <participant count> participants

⚠️  Items Needing Attention:
   • <any flagged ambiguities or unresolved items>
```

If there are open questions, recommend the user run `/speckit.clarify` as the next step to resolve them.

---

## Error Handling

- **M365 CLI not found:** Direct user to install with `npm install -g @pnp/cli-microsoft365`
- **Not authenticated:** Direct user to run `m365 login`
- **No messages returned:** Suggest broadening the `--since` date or checking channel access
- **Rate limiting:** If M365 CLI returns a throttling error, wait 30 seconds and retry once
- **Permission denied:** Explain that the user must be a member of the team to access messages

---

## Notes

- The M365 CLI message history is limited to the last 8 months. Messages older than that cannot be retrieved.
- HTML content in message bodies is stripped to plain text for analysis. Attachments and images are noted but not retrieved.
- This command generates a **draft** specification. The user should review and refine it, then run `/speckit.clarify` to resolve open questions and `/speckit.plan` to create an implementation plan.
