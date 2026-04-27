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
