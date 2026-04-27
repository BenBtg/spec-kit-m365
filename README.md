# spec-kit-m365

**Microsoft 365 Content Ingestion for [Spec Kit](https://github.com/github/spec-kit)** — Fetch Teams messages, meeting transcripts, and M365 files as local Markdown for downstream spec generation.

## What It Does

This extension is ingestion-first. It fetches M365 artifacts and saves normalized Markdown files in your spec workspace.

| Command | Description |
|---------|-------------|
| `/speckit.m365.fetch-message` | Fetch a Teams message thread or email by ID and save as Markdown |
| `/speckit.m365.fetch-file` | Fetch a SharePoint/OneDrive file, convert to Markdown when needed, and save locally |
| `/speckit.m365.fetch-transcript` | Fetch a meeting transcript by meeting ID and save as Markdown |

### Why This Separation Exists

- **Presets apply at spec time**: customer presets belong in `speckit specify`, not ingestion.
- **Human review first**: fetched Markdown can be reviewed and edited before specification.
- **Reusable source artifact**: the same Markdown can drive multiple spec iterations.

---

## Two-Step Workflow

### Step 1: Fetch M365 content as Markdown

```text
speckit m365 fetch-message --id <id>
```

This saves a Markdown artifact (for example `.specify/specs/teams-message-<id>.md`).

### Step 2: Generate a spec from the saved Markdown

```text
speckit specify .specify/specs/teams-message-<id>.md
```

This is where presets are applied and spec generation occurs.

---

## Entra Permissions

Before using any fetch command, your Entra app registration must have delegated permissions to read M365 content. The exact permissions depend on which commands you plan to use.

### Required Permissions by Command

#### Fetch Message (Teams & Email)

Required delegated permissions:
- `User.Read` — read basic user profile
- `Team.ReadBasic.All` — discover and read teams you're a member of
- `Channel.ReadBasic.All` — read channel metadata
- `ChannelMessage.Read.All` — read Teams channel messages
- `TeamMember.Read.All` — read team member list

**Minimum scope (Teams only):** `User.Read`, `Team.ReadBasic.All`, `Channel.ReadBasic.All`, `ChannelMessage.Read.All`, `TeamMember.Read.All`

**For email support, add:**
- `Mail.Read` — read email from Outlook mailbox (requires Exchange Online license)

#### Fetch Transcript

All permissions from **Fetch Message**, plus:
- `Calendars.Read` — read calendar to discover meetings
- `OnlineMeetingTranscript.Read.All` — read Teams meeting transcripts

#### Fetch File

All permissions from **Fetch Message**, plus:
- `Files.Read` — read files from OneDrive
- `Sites.Read.All` — read files from SharePoint sites

### Adding Permissions to Your App

#### Step 1: Sign in to Azure/Entra portal

1. Go to https://portal.azure.com/
2. Search for **"App registrations"** and open it
3. Find your app (for CLI for Microsoft 365, search for "CLI for Microsoft 365" or use application ID `31359c7f-bd7e-475c-86db-fdb8c937548e` if using the default M365 CLI app; or your custom app ID if you created one)

#### Step 2: Add API permissions

1. Click on your app → **API permissions**
2. Click **Add a permission**
3. Select **Microsoft Graph** → **Delegated permissions**
4. Search for each permission and add it:
   - Start with `User.Read` (this is often pre-configured)
   - Add the others according to your command needs (see table above)
5. After adding all permissions, click **Grant admin consent for [Tenant Name]**
   - ⚠️ **Important:** This step requires admin approval. If you can't do it yourself, ask your tenant admin.

#### Step 3: Refresh your M365 CLI token

After granting permissions, log out and log back in to pick up the new scopes:

```bash
m365 logout
```

Then log in again with your same credentials:

```bash
m365 login
```

(If using a custom app ID and tenant ID, include those flags again.)

### Troubleshooting Permissions

#### "Access is denied" errors

**Symptom:** Commands fail with `Access is denied` or `Missing delegated permission` messages.

**Causes:**
- Required permission has not been added to the app
- Permission has been added but admin consent was not granted
- Permission was granted but you have not re-logged in since

**Solution:**
1. Verify the permission is listed in your app's **API permissions** page
2. If it's there, check the status: it should show a green checkmark under "Consent status"
3. If it's missing or shows a warning icon, add it and grant admin consent (see Step 2 above)
4. After any permission change, always `m365 logout` and `m365 login` again

#### Mail.Read not working despite being added

**Symptom:** `fetch-message` works for Teams but fails for email with "Permission denied."

**Possible causes:**
- User account does not have Exchange Online mailbox license
- Dev/test tenant may not have Exchange Online enabled
- Multi-factor authentication may be blocking token acquisition

**Solution:**
- Contact your Microsoft 365 admin to enable Exchange Online for your user account
- For test/dev scenarios, stick with Teams content (`fetch-message source="teams"`)

#### Missing meeting transcripts

**Symptom:** `fetch-transcript` lists meetings but all transcripts return empty or unavailable.

**Possible causes:**
- Meeting recording/transcription was not enabled when the meeting occurred
- Transcript is still being processed (can take minutes to hours after meeting ends)
- Insufficient `OnlineMeetingTranscript.Read.All` permissions

**Solution:**
- Ensure meetings you want to fetch have transcription enabled before they start
- Wait a few minutes after a meeting ends before fetching the transcript
- Verify `OnlineMeetingTranscript.Read.All` permission is granted and consented

---

## Prerequisites

### 1. CLI for Microsoft 365

```bash
npm install -g @pnp/cli-microsoft365
```

```bash
m365 version
```

### 2. Authentication

```bash
m365 login
```

```bash
m365 status
```

### 3. Optional: markitdown (for binary file conversion)

Required by `fetch-file` when converting binary formats such as PDF or DOCX:

```bash
pip install 'markitdown[all]'
```

### 4. Spec Kit

Requires Spec Kit `>=0.1.0`.

---

## Installation

```bash
specify extension add m365
```

For local development:

```bash
specify extension add ./path/to/spec-kit-m365
```

---

## Configuration

Optional defaults file:

```bash
cp .specify/extensions/m365/m365-config.template.yml .specify/extensions/m365/m365-config.yml
```

Example:

```yaml
defaults:
  teamName: "My Team"
  channelName: "General"
  since: "7d"

output:
  specDirectory: ".specify/specs/"

meetings:
  transcriptTempDir: "system"
  cleanupTranscripts: true
```

---

## Usage

### Fetch a Teams thread or email

```text
/speckit.m365.fetch-message source="teams" teamName="Engineering" channelName="Backend" messageId="1712900400000"
```

```text
/speckit.m365.fetch-message source="email" messageId="AAMkAG..."
```

### Fetch a SharePoint/OneDrive file

```text
/speckit.m365.fetch-file source="sharepoint" fileWebUrl="https://contoso.sharepoint.com/sites/eng/Shared%20Documents/requirements.docx"
```

```text
/speckit.m365.fetch-file source="onedrive" fileId="01ABCDEF..."
```

### Fetch a meeting transcript

```text
/speckit.m365.fetch-transcript meetingId="MSo1N2Y5ZGFjYy03MWJm..."
```

---

## Output

All fetch commands write local Markdown files (default: `.specify/specs/`) and print the saved path.

Each output file includes source frontmatter, for example:

```yaml
---
Source:
  type: "m365-message"
  messageSource: "teams"
  messageId: "1712900400000"
  fetchedAt: "2026-04-27T12:30:00Z"
---
```

### Next step

Once saved, use that Markdown file as input to `speckit specify` to generate a specification with presets applied.

---

## Security Model

- **Least privilege**: commands use read-only M365 scopes for fetch operations.
- **No secret leakage**: outputs must not include tokens, auth headers, or credential material.
- **Workspace-safe writes**: output path must resolve inside the current workspace.

---

## Known Limitations

| Limitation | Details |
|------------|---------|
| Teams history cap | M365 CLI Teams message retrieval is limited to the recent history window available via API |
| Transcript availability | Transcripts only exist when recording/transcription was enabled |
| Meeting discovery dependency | Calendar-based meeting discovery requires an Exchange Online mailbox |
| Binary file conversion | `fetch-file` needs `markitdown` for binary formats |

---

## Development

```text
spec-kit-m365/
├── extension.yml
├── m365-config.template.yml
├── commands/
│   ├── setup.md
│   ├── fetch-message.md
│   ├── fetch-file.md
│   └── fetch-transcript.md
├── docs/examples/
├── README.md
├── CHANGELOG.md
└── LICENSE
```

---

## License

[MIT](LICENSE)
