---
description: "Fetch a Teams message thread or email by ID and save it as Markdown in the spec directory"
---

# M365 Message Fetch to Markdown

You are fetching Microsoft 365 message content and saving it as a local Markdown file.
This command performs ingestion only. Do not generate a specification in this command.

Follow each step sequentially. Do not skip steps.

---

## Step 0 — Pre-flight Checks

Run these checks before proceeding. Stop and report if any check fails.

### 0a. Verify M365 CLI is installed

```bash
m365 version
```

If this fails, stop and tell the user:
> M365 CLI is not installed. Install it with `npm install -g @pnp/cli-microsoft365`, then run `m365 login`.

### 0b. Verify the user is authenticated

```bash
m365 status -o json
```

If not authenticated, stop and tell the user:
> You are not logged in to M365 CLI. Run `m365 login` and retry this command.

### 0c. Load extension configuration

If `.specify/extensions/m365/m365-config.yml` exists, read it and use values as defaults.

---

## Step 1 — Resolve Message Source and Output Path

Parse user input for:

| Parameter | Description |
|-----------|-------------|
| `source` | `teams` or `email` (default: `teams`) |
| `teamId` | Teams team GUID (Teams source only) |
| `teamName` | Team display name (Teams source only) |
| `channelId` | Teams channel ID (Teams source only) |
| `channelName` | Channel display name (Teams source only) |
| `messageId` | Required for root Teams message ID or Outlook email message ID |
| `since` | ISO timestamp filter when discovering Teams messages |
| `output` | Optional output Markdown path |

### 1a. Resolve source and IDs

If `source` is `teams`:
- Resolve team and channel exactly as in setup defaults flow.
- Require `messageId`. If missing, ask the user for the message/thread root ID.

If `source` is `email`:
- Require `messageId` (Outlook message ID).

### 1b. Resolve output path

If `output` is provided, use it.

If `output` is not provided, default to:
1. `.specify/specs/teams-message-<messageId>.md` for Teams
2. `.specify/specs/email-message-<messageId>.md` for email

Sanitize filename characters to `a-z`, `A-Z`, `0-9`, `-`, `_`, and `.`.

### 1c. Enforce workspace-safe output

Resolve the absolute output path and verify it is inside the current workspace root.
If the path resolves outside the workspace, stop and tell the user:
> Output path must be inside the workspace. Please choose a path under this project.

### 1d. Confirm with the user

Show:
- Source type
- Resolved IDs
- Output path

Ask:
> Proceed with message fetch and Markdown save?

Wait for confirmation before continuing.

---

## Step 2 — Fetch Message Content

### 2a. Fetch Teams thread (if `source=teams`)

Fetch root message:

```bash
m365 teams message get --teamId <TEAM_ID> --channelId <CHANNEL_ID> --id <MESSAGE_ID> -o json
```

Fetch replies:

```bash
m365 teams message reply list --teamId <TEAM_ID> --channelId <CHANNEL_ID> --messageId <MESSAGE_ID> -o json
```

Sort root + replies chronologically using `createdDateTime`.

### 2b. Fetch Outlook email (if `source=email`)

```bash
m365 outlook message get --id <MESSAGE_ID> -o json
```

If no content is returned, stop and tell the user no message was found for that ID.

### 2c. Normalize content to Markdown-safe text

For Teams:
- Extract `createdDateTime`, sender display name, message `id`, and `body.content`.
- If `body.contentType` is `html`, strip HTML tags while preserving readable text and mentions.

For email:
- Extract `subject`, `from.emailAddress.address`, `toRecipients`, `ccRecipients`, `receivedDateTime`, and body.
- If body is HTML, strip HTML tags while preserving readable text.

Security check:
- Do not include access tokens, auth headers, tenant secrets, or CLI config secrets.
- Do not include raw command output that contains credential material.

---

## Step 3 — Write Markdown Output

Create parent directory if needed:

```bash
mkdir -p "$(dirname "<OUTPUT_PATH>")"
```

Write Markdown with frontmatter:

```markdown
---
Source:
  type: "m365-message"
  messageSource: "teams|email"
  messageId: "<messageId>"
  fetchedAt: "<ISO timestamp>"
---
```

For Teams, body format:

```markdown
# Teams Thread: <messageId>

- Team: <team name>
- Channel: <channel name>

## Messages

[<timestamp>] <sender>

<content>
```

For email, body format:

```markdown
# Email: <subject>

- Message ID: <messageId>
- From: <from>
- To: <to list>
- Cc: <cc list>
- Received: <timestamp>

## Body

<content>
```

Verify file exists and is non-empty after write.

---

## Step 4 — Output Summary

Display:

```
✅ M365 content fetched successfully

📄 Output: <output-path>
📊 Summary:
   • Source: <teams|email>
   • Message ID: <id>
   • Records: <count>

💡 Next Step:
   Once saved, use this file as input to `speckit specify` to generate a specification with your presets applied.
```

---

## Notes

- Least-privilege delegated permissions for Teams fetch: `User.Read`, `Team.ReadBasic.All`, `Channel.ReadBasic.All`, `ChannelMessage.Read.All`, `TeamMember.Read.All`.
- Least-privilege delegated permission for email fetch: `Mail.Read`.
- This command is ingestion-only and must not generate a spec.
