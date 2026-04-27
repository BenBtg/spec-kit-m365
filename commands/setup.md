---
description: "Set up the spec-kit-m365 extension — verify prerequisites, configure authentication, and create your m365-config.yml"
---

# M365 Extension Setup

You are guiding the user through first-time setup of the spec-kit-m365 extension. Follow each step sequentially. Report issues clearly and help the user resolve them before proceeding.

---

## Step 0 — Check Node.js Version

```bash
node --version
```

Parse the major version number from the output (e.g., `v22.15.0` → 22).

- **If Node.js is not installed:** Stop and tell the user to install Node.js 22 LTS from https://nodejs.org/
- **If major version ≥ 25:** Warn the user:
  > ⚠️ Node.js v25+ has known compatibility issues with M365 CLI. The `open` and `clipboardy` packages crash on this version. Please install and use Node.js 22 LTS.
  >
  > On macOS with Homebrew:
  > ```bash
  > brew install node@22
  > export PATH="/opt/homebrew/opt/node@22/bin:$PATH"
  > ```
  >
  > Add the `export` line to your shell profile (`~/.zshrc` or `~/.bashrc`) to make it permanent.

  Ask the user to fix their Node version before continuing.
- **If major version is 18, 20, or 22:** Continue.

---

## Step 1 — Check M365 CLI Installation

```bash
m365 version
```

- **If the command fails:** Tell the user:
  > M365 CLI is not installed. Install it with:
  > ```bash
  > npm install -g @pnp/cli-microsoft365
  > ```
  > Then re-run this setup command.

  Stop and wait for the user to install it.

- **If the version is below 9.0.0:** Warn the user:
  > Your M365 CLI version is outdated. This extension requires v9.0.0 or higher. Upgrade with:
  > ```bash
  > npm install -g @pnp/cli-microsoft365@latest
  > ```

- **If v9.0.0+:** Continue and display the version.

---

## Step 2 — Configure M365 CLI Settings

Run these to prevent crashes from the `open` and `clipboardy` packages in non-interactive or headless environments:

```bash
m365 cli config set --key autoOpenLinksInBrowser --value false
m365 cli config set --key copyDeviceCodeToClipboard --value false
```

Tell the user:
> ✅ M365 CLI configured — disabled auto-open browser and clipboard copy to prevent crashes in terminal environments.

---

## Step 3 — Create Configuration File

Check if `.specify/extensions/m365/m365-config.yml` already exists.

**If it exists:** Read it and display the current settings. Ask the user:
> A configuration file already exists. Do you want to keep it or overwrite it with fresh defaults?

If the user wants to keep it, skip to Step 4.

**If it does not exist (or user wants to overwrite):**

Ask the user for their organization details. Prompt for each value, explaining what it is and how to find it:

1. **App ID** — "What is your Entra ID app registration's Application (client) ID? You can find this in the Azure/Entra portal under App registrations. Leave blank if using the default M365 CLI app."

2. **Tenant ID** — "What is your Microsoft 365 tenant ID? Find this in the Azure/Entra portal under Azure Active Directory > Overview. Leave blank for multi-tenant apps."

3. **Auth type** — "How would you like to authenticate? Options: `deviceCode` (recommended for terminals), `browser` (opens a browser window). Default: `deviceCode`."

4. **Default team** — "What is the name of the Teams team you'll pull conversations from most often? Leave blank to be prompted each time."

5. **Default channel** — "What is the name of the channel within that team? Leave blank to be prompted each time."

6. **Since duration** — "How far back should the extension look for messages by default? Examples: `7d`, `30d`, `2026-01-01`. Default: `7d`."

Create the directory and file:

```bash
mkdir -p .specify/extensions/m365
```

Write `.specify/extensions/m365/m365-config.yml` with the collected values. Use the template structure from `m365-config.template.yml` but only include non-blank values. Comment out any fields the user left blank.

---

## Step 4 — Authenticate

Check current auth status:

```bash
m365 status -o json
```

**If already logged in:** Display the connected account and ask:
> You're already logged in as `<connectedAs>`. Do you want to keep this session or log in with a different account?

If keeping, skip to Step 5.

**If not logged in (or user wants to re-login):**

Build the login command from the config file values:

- If `auth.appId` is set: include `--appId <appId>`
- If `auth.tenantId` is set: include `--tenant <tenantId>`
- If `auth.authType` is set: include `--authType <authType>`

Run the login command:
```bash
m365 login [--appId <appId>] [--tenant <tenantId>] [--authType <authType>]
```

If using device code flow, display the device code and URL to the user:
> 🔑 To sign in, open https://microsoft.com/devicelogin in your browser and enter the code shown above.

Wait for the user to complete authentication.

After login, verify:
```bash
m365 status -o json
```

Display:
> ✅ Authenticated as `<connectedAs>`.

---

## Step 5 — Verify Permissions

Run a lightweight API call to verify the app has the necessary permissions:

```bash
m365 teams team list --joined -o json
```

- **If successful:** Display the count of teams found and confirm permissions are working.
- **If it fails with a permission error:** Tell the user:
  > ❌ Permission check failed. Your app registration may be missing required delegated permissions.
  >
  > **Fetch message command (Teams minimum):**
  > `User.Read`, `Team.ReadBasic.All`, `Channel.ReadBasic.All`, `ChannelMessage.Read.All`, `TeamMember.Read.All`
  >
  > **Fetch transcript command (additional):**
  > `Calendars.Read`, `OnlineMeetingTranscript.Read.All`
  >
  > **Fetch file command (additional):**
  > `Files.Read` (OneDrive) and `Sites.Read.All` (SharePoint)
  >
  > **Note:** Calendar-based meeting discovery also requires an Exchange Online mailbox on the user's account. Developer/test tenants without Exchange Online licenses may need to provide meeting IDs manually.
  >
  > Grant these in the Azure/Entra portal under your app registration > API permissions, then have an admin click "Grant admin consent."

---

## Step 6 — Summary

Display a setup summary:

```
✅ spec-kit-m365 setup complete

📋 Configuration:
   • Config file: .specify/extensions/m365/m365-config.yml
   • Authenticated as: <connectedAs>
   • Default team: <teamName or "not set — will prompt">
   • Default channel: <channelName or "not set — will prompt">
   • Message lookback: <since>

🚀 You're ready to go! Try:
  • /speckit.m365.fetch-message — Fetch Teams or email content as Markdown
  • /speckit.m365.fetch-file — Fetch SharePoint/OneDrive files as Markdown
  • /speckit.m365.fetch-transcript — Fetch meeting transcripts as Markdown

💡 Next Step:
  After fetch, run `speckit specify <saved-markdown-file>` to generate a spec with presets.
```
