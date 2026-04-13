# Changelog

All notable changes to spec-kit-m365 will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-04-13

### Added

- `/speckit.m365.thread` command — pull Teams channel threads and generate Spec Kit specifications
  - Interactive team/channel/thread selection via M365 CLI
  - Per-message analysis distinguishing final decisions from exploratory discussion
  - User story extraction with priority assignment (P1/P2/P3)
  - Functional requirement derivation with IDs (FR-001, FR-002, ...)
  - Open question flagging for unresolved items
  - Source traceability metadata in generated specs
- `/speckit.m365.meeting` command — pull meeting transcripts and generate Spec Kit specifications
  - VTT transcript parsing with speaker attribution
  - Decision extraction with confidence levels (High/Medium/Low)
  - Action item extraction with owner and deadline tracking
  - Calendar query helper for meeting ID discovery
  - Distinction between brainstorming, decisions, and parking lot items
- Configuration template (`m365-config.template.yml`) for setting default teams, channels, and output preferences
- Documentation with setup instructions, usage examples, and M365 CLI prerequisites
- Example outputs demonstrating thread-to-spec and meeting-to-spec conversions
