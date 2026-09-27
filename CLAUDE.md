# CLAUDE.md

## Before you touch GitHub: ask

Abhay approves every action that leaves this machine. **Ask first, showing
the exact command, before any `git push`, before opening or merging a pull
request, and before any `gh` command that writes anything.** Local commits
need no approval.

**`main` is never pushed to directly.** Every change goes on a branch
(`feat/`, `fix/`, `docs/`, `chore/`, `refactor/` + short description) →
commit → *ask* → push the branch → *ask* → open a PR filling in the template
→ report the link and **stop**. Do not merge, do not delete the branch.
Abhay says what happens next.

The full version of this rule, and everything else that matters, is in
**[AGENT.md](./AGENT.md)** — repo layout, conventions, and how each project
file is kept current. If the two disagree, `AGENT.md` wins.

This file is duplicated from `AGENT.md` on purpose: it is the one Claude Code
loads automatically, and the GitHub rule above is too important to sit behind
a second hop. Everything *else* goes in `AGENT.md` only — don't grow this
file.
