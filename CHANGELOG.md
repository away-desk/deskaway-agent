# Changelog

All notable changes to this repository are recorded here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and this repository follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Source-of-truth rule in AGENT.md: the two V1 plan files in the parent
  folder are authoritative, changes land there first, and neither is edited
  without explicit approval.

- SECURITY.md: reporting route corrected. Private vulnerability reporting is a
  public-repository feature and was never enabled, so the file now routes
  reports through a repo issue and carries a checklist to work through before
  any repo goes public.
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

### Changed

- SECURITY.md rewritten for a public repository: vulnerabilities are reported
  through GitHub private vulnerability reporting, and the protections section
  now lists secret scanning, push protection and branch protection. The
  go-public checklist is removed, having been completed. AGENT.md's SECURITY.md
  rule updated to match.

### Removed

- Empty placeholder workflow files, replaced by the CI workflow above. The
  deploy, release, publish and security pipelines will return when there is
  code and credentials to make them real.

Nothing has been released yet and no version has been tagged. Entries
accumulate here until the first release, at which point this heading becomes
that version and a fresh `[Unreleased]` opens above it.
