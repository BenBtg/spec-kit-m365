# Copilot Instructions for spec-kit-m365

## Project Overview

This is a **Spec Kit extension** (not a standalone app) that converts Microsoft Teams conversations and meeting transcripts into structured feature specifications. It has no build system, no tests, and no runtime code — the "commands" are markdown instruction files (`commands/*.md`) that AI agents execute step-by-step.

The extension provides two commands via the Spec Kit framework:
- `/speckit.m365.thread` → Teams channel thread → spec
- `/speckit.m365.meeting` → Meeting transcript → spec

## Architecture

- **`extension.yml`** — Extension manifest declaring commands, dependencies, config schema, and required Spec Kit version (>=0.1.0). This is the entry point the Spec Kit framework reads.
- **`commands/thread.md` / `commands/meeting.md`** — AI agent instruction files. Each is a multi-step procedure (pre-flight checks → data retrieval → analysis → spec generation) that an AI agent follows literally. These are the core "source code" of the extension.
- **`m365-config.template.yml`** — Configuration template users copy to `.specify/extensions/m365/m365-config.yml` in their projects.
- **`docs/examples/`** — Sample inputs/outputs for documentation.

## Key Conventions

### Command files are sequential agent instructions
The markdown files in `commands/` are structured as numbered steps (Step 0, Step 1, ...) that an AI agent must follow in order. Each step has explicit bash commands to run, decision trees for parameter resolution, and error handling. When editing these files, preserve this sequential structure and the explicit M365 CLI commands.

### M365 CLI is the data layer
All Microsoft 365 interaction goes through the [CLI for Microsoft 365](https://pnp.github.io/cli-microsoft365/) (`m365` command). Commands always output JSON (`-o json`). The extension never calls Microsoft Graph directly.

### Spec output format
Generated specs follow a specific structure: YAML frontmatter with source traceability metadata, user stories with P1/P2/P3 priorities, functional requirements with FR-NNN IDs, Given/When/Then acceptance criteria, and open questions. The meeting command additionally produces a decisions log (with High/Medium/Low confidence) and action items table.

### Spec template resolution
Both commands resolve the spec template in this order: local `.specify/templates/spec-template.md` → remote GitHub fetch → built-in fallback. This three-tier approach must be preserved.

### Configuration cascading
Parameters resolve as: user-provided inline args → `m365-config.yml` defaults → interactive prompting. Both commands follow this same pattern.

### Spec numbering
Specs are created at `.specify/specs/<NNN>-<feature-name>/spec.md` where NNN is auto-incremented by scanning existing folders, zero-padded to 3 digits.
