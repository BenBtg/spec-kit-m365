# Copilot Instructions for spec-kit-m365

## Project Overview

This is a **Spec Kit extension** (not a standalone app) that ingests Microsoft 365 content and normalizes it to local Markdown files. It has no build system, no tests, and no runtime code — the "commands" are markdown instruction files (`commands/*.md`) that AI agents execute step-by-step.

The extension provides three ingestion commands via the Spec Kit framework:
- `/speckit.m365.fetch-message` → Teams message thread or email → Markdown
- `/speckit.m365.fetch-file` → SharePoint/OneDrive file → Markdown
- `/speckit.m365.fetch-transcript` → Meeting transcript → Markdown

Spec generation happens separately via `speckit specify` using the saved Markdown file as input.

## Architecture

- **`extension.yml`** — Extension manifest declaring commands, dependencies, config schema, and required Spec Kit version (>=0.1.0). This is the entry point the Spec Kit framework reads.
- **`commands/fetch-message.md` / `commands/fetch-file.md` / `commands/fetch-transcript.md`** — AI agent instruction files. Each is a multi-step procedure (pre-flight checks → data retrieval → markdown output) that an AI agent follows literally. These are the core "source code" of the extension.
- **`m365-config.template.yml`** — Configuration template users copy to `.specify/extensions/m365/m365-config.yml` in their projects.
- **`docs/examples/`** — Sample inputs/outputs for documentation.

## Key Conventions

### Command files are sequential agent instructions
The markdown files in `commands/` are structured as numbered steps (Step 0, Step 1, ...) that an AI agent must follow in order. Each step has explicit bash commands to run, decision trees for parameter resolution, and error handling. When editing these files, preserve this sequential structure and the explicit M365 CLI commands.

### M365 CLI is the data layer
All Microsoft 365 interaction goes through the [CLI for Microsoft 365](https://pnp.github.io/cli-microsoft365/) (`m365` command). Commands always output JSON (`-o json`). The extension never calls Microsoft Graph directly.

### Ingest-first output format
Each command outputs a Markdown artifact with YAML frontmatter source metadata and normalized content body. Commands must end after writing Markdown and reporting the output path.

### No direct spec generation
Commands in this extension do not generate specs and do not apply presets. The next step is always `speckit specify <saved-markdown-file>`.

### Configuration cascading
Parameters resolve as: user-provided inline args → `m365-config.yml` defaults → interactive prompting. All ingestion commands follow this pattern.

### Output safety
Commands must enforce workspace-safe output paths, avoid writing outside the workspace, and never include credentials/tokens in output files.
