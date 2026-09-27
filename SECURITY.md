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

**Every repository in the [away-desk](https://github.com/away-desk) org is
currently private.** If you can read this file, you have been granted access to the
repository — so the audience for this section is a small number of people who
already have a private channel to the maintainers.

**Report by opening an issue in the affected repository.** While the repo is
private, that issue is visible only to people with access to it, which makes it an
adequate private channel. Prefix the title with `[security]`.

If you would rather not use an issue, contact the maintainer directly through
whatever channel you already have with them. Do not post details anywhere public.

Useful to include: what an attacker can do, steps to reproduce, which repository
and component it affects, and whether you think it is already known elsewhere. A
proof of concept helps.

Expect a reply within a week. Since there is no released version, the likely
outcome is a tracked issue and a fix in the normal course of development rather
than a security release.

### What this section will say once a repository is public

GitHub's **private vulnerability reporting** — the "Report a vulnerability" button
on the Security tab — is the right route for a public repository, because it lets a
stranger reach the maintainers without disclosing the issue publicly first.

It is **not available on a private repository**, on any plan. It is a
public-repository feature. So it is deliberately not offered above: pointing people
at a button that does not exist is worse than pointing them at an issue that does.

**Before making any repository in this org public**, work through this list:

- [ ] Enable private vulnerability reporting on it (Settings → Security → *Private
      vulnerability reporting*). It is free for public repos.
- [ ] Enable secret scanning and push protection. Also free for public repos, and
      the only control that catches a committed credential at push time.
- [ ] Rewrite the section above to point at the Security tab, and delete this
      checklist from that repo's copy.
- [ ] Re-run the key check in `AGENT.md` against the **full history**, not just the
      current tip. Going public exposes every commit ever made, not just the latest
      one — a credential removed in a later commit is still there to be found.
- [ ] Confirm nothing in the history assumed the repo was private: internal
      hostnames, account IDs, infrastructure detail, or a `.env.example` that
      quietly acquired a real value.

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

- **`.gitignore` blocks key material** in every repo — `.env`, `*.pem`, `*.key`,
  `*.p12`, plus signing material in the client repos and Terraform state and plan
  files in `deskaway-infra`. Verified with `git check-ignore`, and enforced by two
  CI checks so it cannot quietly stop being true.
- **A pre-commit key check** is documented in `AGENT.md` and expected before every
  `git add`.
- **Nothing more.** Secret scanning, push protection and branch protection are all
  unavailable for private repositories on this org's plan. There is no server-side
  net: if a credential is committed, nothing will stop the push. The checks above
  are the whole defence, and they depend on someone actually running them.
