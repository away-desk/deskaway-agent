# Changelog

All notable changes to this repository are recorded here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and this repository follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Pre-commit secret check in AGENT.md, plus CI steps enforcing that no
  credential-shaped file is tracked and that .gitignore blocks key material.
- Pull request template prompt: which doc changed, or why none was needed.
- docs/runbook.md: symptom-based incident entries, each with how to confirm,
  what to check, how to fix and how to roll back.
- Component documentation: docs/architecture.md, docs/local-setup.md and
  docs/adr/ with the ADR format and template. Upkeep rules for each in
  AGENT.md.
- CI workflow: repo hygiene checks that run today, plus stack-specific lint,
  build and test steps that activate once there is code to run them on.
- Contribution workflow rule: every change goes on a branch and through a
  pull request, and any GitHub write needs approval first (AGENT.md,
  CLAUDE.md).
- Repository scaffolding: agreed directory layout and project documentation
  (README, AGENT.md, CONTRIBUTING, SECURITY, LICENSE).

### Removed

- Empty placeholder workflow files, replaced by the CI workflow above. The
  deploy, release, publish and security pipelines will return when there is
  code and credentials to make them real.

Nothing has been released yet and no version has been tagged. Entries
accumulate here until the first release, at which point this heading becomes
that version and a fresh `[Unreleased]` opens above it.
