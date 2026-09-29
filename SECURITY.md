# Security Policy

## Supported versions

**None. Do not run DeskAway in production, or on any machine you care about.**

| Version | Supported |
| --- | --- |
| all | :x: — pre-release, unfinished, no security support |

Nothing has been released and no version has been tagged. There is no supported
version to report a vulnerability *against* yet, and no patch would be issued for
one.

This is stated bluntly because of what DeskAway does: it executes shell commands on
a real machine, and those commands originate from a language model reached over a
network. The controls that make that acceptable — scope enforcement, approval
gates, reversibility tiering, command timeouts — are exactly the parts still
unwritten. An unfinished DeskAway is not a limited DeskAway; it is remote command
execution with the safety features missing.

Read the code, run it against a throwaway VM if you must, and do not point it at
anything real until this section says otherwise.

## Reporting a vulnerability

**Do not open a public issue, pull request or discussion for a security
problem.** These repositories are public, so anything posted there is visible to
everyone.

**Report privately through GitHub:** go to this repository's
[Security tab](https://github.com/away-desk/deskaway-agent/security) and choose
**Report a vulnerability**, or open
[a new private advisory](https://github.com/away-desk/deskaway-agent/security/advisories/new)
directly. Only the maintainers can see it.

Useful to include: what an attacker can do, steps to reproduce, which repository
and component it affects, and whether you think it is already known elsewhere. A
proof of concept helps.

Expect a reply within a week. Since there is no released version, the likely
outcome is a tracked fix in the normal course of development rather than a
security release. Once the fix is merged, the advisory is published with credit
to the reporter unless they ask otherwise.

## Scope

Anything in the [away-desk](https://github.com/away-desk) org. If a finding spans
repos — a protocol flaw letting a paired phone escalate on a desktop, for instance
— report it once against the repo where the fix belongs, or against
[deskaway-protocol](https://github.com/away-desk/deskaway-protocol) if the contract
itself is the problem.

Out of scope: the absence of features that are simply not built yet. That is most of
the codebase at present, which is why the supported-versions section says what it
does.

## What protects these repositories today

Stated plainly, because an overstated security posture is its own risk:

- **Secret scanning and push protection** are on. GitHub scans every push and
  blocks one that contains a credential format it recognises. It does not
  recognise everything — a custom token or a password in plain text gets
  through.
- **Branch protection on `main`**: changes arrive only through a pull request,
  CI must pass before merging, and `main` cannot be force-pushed or deleted.
  There is one maintainer, so no second person reviews a change before it
  merges.
- **`.gitignore` blocks key material** in every repo — `.env`, `*.pem`, `*.key`,
  `*.p12`, plus signing material in the client repos and Terraform state and plan
  files in `deskaway-infra`. Verified with `git check-ignore`, and enforced by two
  CI checks so it cannot quietly stop being true.
- **A pre-commit key check** is documented in `AGENT.md` and expected before every
  `git add`. It is the only check that runs before a secret leaves the machine.
- **The full history was checked before going public** (2026-09-29): no
  credential-shaped file was ever tracked, and no commit added secret-shaped
  content, account ids or internal hostnames.

Everything in these repositories, including every past commit, is public.
Anything committed by mistake must be treated as leaked and rotated, even if a
later commit removes it.
