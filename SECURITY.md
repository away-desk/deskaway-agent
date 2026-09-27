# Security Policy

## Supported versions

**None. Do not run DeskAway in production, or on any machine you care
about.**

| Version | Supported |
| --- | --- |
| all | :x: — pre-release, unfinished, no security support |

Nothing has been released and no version has been tagged. There is no
supported version to report a vulnerability *against* yet, and no patch
would be issued for one.

This is stated bluntly because of what DeskAway does: it executes shell
commands on a real machine, and those commands originate from a language
model reached over a network. The controls that make that acceptable — scope
enforcement, approval gates, reversibility tiering, command timeouts — are
exactly the parts still unwritten. An unfinished DeskAway is not a limited
DeskAway; it is remote command execution with the safety features missing.

Read the code, run it against a throwaway VM if you must, and do not point
it at anything real until this section says otherwise.

## Reporting a vulnerability

**Report privately, not in a public issue.** A public issue tells everyone
about the hole before there is a fix.

Use GitHub's private vulnerability reporting: open the **Security** tab of
this repository and choose **Report a vulnerability**
([direct link](../../security/advisories/new)). That opens a private advisory
visible only to the maintainers. It is enabled on every repository in the
[away-desk](https://github.com/away-desk) org, so no email address is needed.

Useful to include: what an attacker can do, the steps to reproduce it, and
which repository and component it affects. A proof of concept helps; so does
telling us if you think it is already public.

Expect a reply within a week. Since there is no released version, the likely
outcome is a tracked issue and a fix in the normal course of development
rather than a security release — but report it privately anyway, and let the
maintainers make that call.

## Scope

Anything in the [away-desk](https://github.com/away-desk) org. If a finding
spans repos — a protocol flaw that lets a paired phone escalate on a desktop,
for instance — file it once against the repo where the fix belongs, or against
[deskaway-protocol](https://github.com/away-desk/deskaway-protocol) if the
contract itself is the problem.

Out of scope: the absence of features that are simply not built yet. That is
the whole repository at present, which is why the supported-versions section
above says what it does.
