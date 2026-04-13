# spec-kit-m365

**Microsoft 365 Integration for [Spec Kit](https://github.com/github/spec-kit)** — Convert Microsoft Teams conversations and meeting transcripts into structured feature specifications for Spec-Driven Development.

## What It Does

This extension provides two commands that bridge the gap between team collaboration in Microsoft Teams and structured specification documents:

| Command | Description |
|---------|-------------|
| `/speckit.m365.thread` | Pull a Teams channel thread, summarize decisions and requirements, generate a spec |
| `/speckit.m365.meeting` | Pull a meeting transcript, extract decisions and action items, generate a spec |

Both commands use the [CLI for Microsoft 365](https://pnp.github.io/cli-microsoft365/) to retrieve data and rely on the AI agent executing the command to perform intelligent summarization — no external AI API required.

### Key Features

- **Decision extraction** — Distinguishes final decisions from exploratory discussion, with confidence levels for meeting transcripts
- **Requirement derivation** — Automatically structures decisions into functional requirements and user stories
- **Open question flagging** — Identifies unresolved items and ambiguities for follow-up
- **Spec Kit native output** — Generates specs in the standard Spec Kit template format, ready for `/speckit.clarify` → `/speckit.plan` → `/speckit.tasks`
- **Source traceability** — Every spec includes metadata linking back to the original Teams conversation or meeting

---

## Prerequisites

### 1. CLI for Microsoft 365 (M365 CLI)

Install the M365 CLI globally:

```bash
npm install -g @pnp/cli-microsoft365
```

Verify the installation:

```bash
m365 version
```

### 2. Authentication

Log in to your Microsoft 365 tenant:

```bash
m365 login
```

This starts a device-code authentication flow:
1. The CLI displays a URL and a code
2. Open the URL in your browser (`https://microsoft.com/devicelogin`)
3. Enter the code and sign in with your Microsoft 365 account
4. Return to the terminal — you should see a success message

Verify you're connected:

```bash
m365 status
```

### 3. Permissions

| Command | Required Access |
|---------|----------------|
| Thread command | You must be a **member** of the Teams team whose channel you want to read |
| Meeting command | You must be the **meeting organizer**, or your tenant admin must have configured an [application access policy](https://learn.microsoft.com/graph/cloud-communication-online-meeting-application-access-policy) |

### 4. Spec Kit

This extension requires [Spec Kit](https://github.com/github/spec-kit) v0.1.0 or later.

---

## Installation

### From the community catalog

```bash
specify extension add m365
```

### From a local path (for development/testing)

```bash
specify extension add ./path/to/spec-kit-m365
```

---

## Configuration

After installation, optionally create a configuration file to set defaults:

```bash
cp .specify/extensions/m365/m365-config.template.yml .specify/extensions/m365/m365-config.yml
```

Edit `.specify/extensions/m365/m365-config.yml`:

```yaml
defaults:
  teamId: "your-team-guid-here"
  teamName: "My Team"
  channelId: "19:abc123@thread.tacv2"
  channelName: "General"
  since: "7d"    # Pull messages from the last 7 days

output:
  specDirectory: ".specify/specs/"
  specNumber: "auto"

meetings:
  transcriptTempDir: "system"
  cleanupTranscripts: true
```

If no config file is present, the commands will prompt you interactively.

---

## Usage

### Thread Command

Pull a Teams channel conversation and generate a spec:

```
/speckit.m365.thread
```

The command will interactively walk you through selecting a team, channel, and time range.

#### With parameters

```
/speckit.m365.thread teamName="Engineering" channelName="Backend" since="2026-04-01T00:00:00Z"
```

#### Targeting a specific thread

```
/speckit.m365.thread teamId="fce9e580-..." channelId="19:eb30973b..." messageId="1540747442203"
```

### Meeting Command

Pull a meeting transcript and generate a spec:

```
/speckit.m365.meeting meetingId="MSo1N2Y5ZGFjYy03MWJm..."
```

If you don't know the meeting ID, the command can query your recent calendar events:

```
/speckit.m365.meeting
```

> It will list your recent online meetings and let you pick one.

#### Finding a Meeting ID Manually

The meeting ID is embedded in the Teams meeting join URL. It's the base64-encoded string in the URL path after `/meetingjoin/`. You can also find it in the meeting details in your Teams calendar.

---

## Output

Both commands generate a spec at `.specify/specs/<NNN>-<feature-name>/spec.md` containing:

- **Source metadata** — team/channel/meeting info, participants, timestamps
- **User stories** with priorities (P1/P2/P3) and acceptance scenarios (Given/When/Then)
- **Functional requirements** (FR-001, FR-002, ...)
- **Success criteria** and assumptions
- **Open questions** — unresolved items flagged for follow-up
- **Decisions log** (meeting command) — with confidence levels
- **Action items** (meeting command) — with owners and deadlines

### Next Steps After Generation

The generated spec is a **draft**. Follow the standard Spec Kit workflow:

1. **Review** the spec and correct any AI misinterpretations
2. `/speckit.clarify` — Resolve open questions
3. `/speckit.plan` — Create a technical implementation plan
4. `/speckit.tasks` — Generate actionable task breakdown
5. `/speckit.implement` — Execute the implementation

---

## Examples

See the [docs/examples/](docs/examples/) directory for sample inputs and outputs:

- [Thread example](docs/examples/thread-input-example.md) — sample Teams conversation → spec
- [Meeting example](docs/examples/meeting-input-example.md) — sample meeting transcript → spec

---

## Known Limitations

| Limitation | Details |
|------------|---------|
| **8-month history cap** | The M365 CLI can only retrieve Teams messages from the last 8 months |
| **Transcript API in preview** | The meeting transcript API is based on a Microsoft Graph beta endpoint and may change |
| **Speaker attribution** | Transcript speaker names depend on meeting audio quality and Teams speaker identification |
| **No 1:1/group chat support** | Only Teams channel messages are supported (not private chats) |
| **Meeting ID required** | The M365 CLI has no "list all meetings" command — a meeting ID must be provided or discovered via calendar query |
| **HTML content** | Rich formatting in Teams messages is stripped to plain text for analysis |

---

## Troubleshooting

### "M365 CLI is not installed"

```bash
npm install -g @pnp/cli-microsoft365
```

### "Not logged in"

```bash
m365 login
```

### "No messages found"

- Verify you are a member of the team
- Try a broader `--since` date (messages older than 8 months are unavailable)
- Check the team and channel IDs are correct

### "No transcripts found"

- Transcription must have been enabled **during** the meeting
- You must be the meeting organizer or have application access permissions
- The meeting ID must be correct — try the calendar query approach

### "Permission denied" for transcripts

Tenant administrators must create an application access policy for transcript access with application permissions. See [Microsoft's documentation](https://learn.microsoft.com/graph/cloud-communication-online-meeting-application-access-policy).

---

## Development

### Project Structure

```
spec-kit-m365/
├── extension.yml                 # Extension manifest
├── m365-config.template.yml      # Configuration template
├── commands/
│   ├── thread.md                 # /speckit.m365.thread command
│   └── meeting.md                # /speckit.m365.meeting command
├── docs/
│   └── examples/
│       ├── thread-input-example.md
│       └── meeting-input-example.md
├── README.md
├── CHANGELOG.md
└── LICENSE
```

### Testing locally

1. Clone this repository
2. Install M365 CLI and authenticate (`m365 login`)
3. Install the extension locally: `specify extension add ./spec-kit-m365`
4. Run `/speckit.m365.thread` or `/speckit.m365.meeting` in your AI agent

---

## License

[MIT](LICENSE)
