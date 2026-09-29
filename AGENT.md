# AGENT.md — deskaway-agent

The planning service. It turns a task written in plain language into an
ordered checklist of steps, classifies how reversible each step is, and
revises the plan when a step fails. It is a stateless HTTP service and the
only component that talks to a model provider.

It decides what *should* happen; it never executes anything. Execution is
`deskaway-desktop`'s job, and that separation is deliberate — keep it.

## Rule: the two plan files are the source of truth

The main source of truth for DeskAway is these two files, which live one
level up, in the `DeskAway` folder that holds all seven repos:

- `../DeskAway-V1-Implementation-Plan (1).md`
- `../DeskAway-V1-Repo-Structure.md`

When this repo, this file, or any other doc disagrees with them, they win.

Any change goes into those files first. A change to scope, design,
structure or plan is written into the relevant plan file before anything in
this repo is changed to match it.

Never edit either file on your own. If a change seems needed there — or
something in this repo has drifted from them — stop, say what you would
change and why, and wait for a clear yes before touching them. Nothing is
auto-updated.

## Folder structure

```
.github/workflows/
  ci.yml  build-push.yml  deploy.yml  security.yml
app/
  main.py                 # ASGI app factory
  api/v1/
    deps.py               # shared request dependencies
    routes/
      plan.py             # task text -> ordered steps
      classify.py         # step -> reversibility tier
      revise.py           # failed step + context -> revised plan
      health.py
  core/
    config.py  auth.py  errors.py  logging.py
  planning/
    context_builder.py    # assemble model context from a request
    step_parser.py        # model output -> structured steps
    revision.py           # replan after a failure
    prompt/
      system/v1/          # versioned system prompts
      templates/          # reusable fragments
  policy/
    approval_gate.py      # which steps need a human before running
    reversibility/
      tiers.yaml          # tier definitions
      classifier.py       # assign a tier to a step
    autonomy/
      v1_supervised.yaml  # what this autonomy level permits
  providers/
    model_client.py       # provider calls
    retry.py              # backoff, timeouts
    token_accounting.py   # usage per run
  replay/recorder.py      # capture model calls for replay
  schemas/                # request/response models
  metrics/
tests/
  unit/  integration/  contract/  golden/  fixtures/
evals/
  scenarios/              # task inputs with expected plan shapes
  runner/                 # harness
  regression/             # frozen cases that must not regress
docker/
  Dockerfile  .dockerignore
docs/
  architecture.md         # responsibility, boundaries, message flow, layering
  local-setup.md          # running it from nothing, under ten minutes
  runbook.md              # symptoms, checks, fixes, rollbacks
  adr/                    # one file per decision, never edited after accepting
pyproject.toml            # deps, lint and test config
.python-version           # pinned interpreter
```

Directories holding only a `.gitkeep` are agreed structure with no content
yet.

## Conventions

- Prompts are versioned directories, never edited in place. A prompt change
  that alters output shape means a new version under `prompt/system/`.
- `policy/` is data first. Tiers and autonomy levels are YAML so they can be
  reviewed without reading Python; the classifier reads them, it does not
  hardcode them.
- Planning output is validated against `deskaway-protocol` step shapes before
  it leaves the service. A model that returns something unparseable is an
  error, never a silent fallback.
- Every model call goes through `providers/model_client.py`. No direct SDK
  calls elsewhere — `retry.py`, `token_accounting.py` and `replay/` all hang
  off that one chokepoint.
- `evals/regression/` only grows. Removing a case needs a reason in the PR.
- No provider key in the repo, in a test, or in a fixture.

## Rule: keep README.md current

The README is the one file a newcomer is guaranteed to read. Revisit it
whenever this repo's answer to any of the four questions below changes — not
on a schedule.

Every DeskAway README answers four things, in this order:

1. **What this one repo is**, in two lines, and where it sits in the whole
   system.
2. **Its current status**, stated honestly. Right now that is *early
   development, nothing works yet.*
3. **How to run it locally**, aiming for under ten minutes.
4. **A link back** to the org or to `deskaway-docs`, so someone landing here
   can find the rest.

How to apply it:

- Keep those four as the first four sections, in that order. Anything else
  goes after them.
- Status rots fastest. The moment the first thing in this repo actually
  runs, that line changes in the same PR. "Nothing works yet" is honest
  only until it isn't.
- If a setup step breaks, or creeps past ten minutes, fix the README in the
  PR that caused it. A stale run section is worse than no run section.
- Never write intent as if it were fact. Anything not yet true is either
  labelled as planned or left out entirely.
- Two lines means two lines. If section 1 needs a third paragraph, that
  content belongs in `deskaway-docs`.

## Rule: keep CHANGELOG.md current

Add a line the moment you do something notable — not at release time. The
changelog is cheap to maintain one entry at a time and miserable to
reconstruct from five months of git log.

- Format is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
  versioning is [SemVer](https://semver.org/spec/v2.0.0.html).
- Everything lands under `## [Unreleased]`, grouped by `### Added`,
  `Changed`, `Deprecated`, `Removed`, `Fixed`, or `Security`. Create a group
  when you first need it.
- "Notable" means a reader of this repo would want to know: a new capability,
  a behaviour change, a dependency that changes how you run it, a security
  fix. Not: formatting, a typo, an internal rename nobody outside the file
  can see.
- Write for someone who has not read the diff. "Added pairing code
  expiry" beats "updated code-generator.ts".
- On a release, rename `[Unreleased]` to the version with the date, and open
  a fresh empty `[Unreleased]` above it. Never delete history.
- Entries are past tense and one line. If yours needs a paragraph, it is
  probably two entries.

## Rule: keep the pull request template useful

`.github/pull_request_template.md` pre-fills every PR description. While this
project is one person reviewing their own work, it is the self-check that
catches what you were about to skip — so fill it in honestly rather than
deleting the prompts.

- Answer all five. "N/A" is a fine answer; a blank section is not.
- **Which unit of the plan this belongs to** is the one that pays off later.
  In five months this is how you find which PR did what, so name the unit,
  not the file you touched.
- **Anything deliberately left incomplete** is not an admission. An
  acknowledged gap is a decision; an unmentioned one is a bug you will
  rediscover.
- Change the template when a prompt stops earning its place, and keep it at
  five or six. A template long enough to skim past is worse than none.

## Rule: keep CONTRIBUTING.md short

It is currently five lines because there are no outside contributors. Resist
growing it for people who do not exist yet.

- Update it when the real answer changes: the formatter command, the branch
  rule, or the day the project starts accepting outside contributions.
- Anything longer than a few lines is either repo guidance — which belongs in
  this file — or cross-repo process, which belongs in `deskaway-docs`.

## Rule: keep SECURITY.md honest

One line in it will become false, and it is the important one.

- The supported-versions table says *nothing is supported, do not run this*.
  The day a version is tagged, that table changes in the same PR — an
  unsupported-looking project that is actually shipping teaches people to
  ignore the file.
- The reporting route is GitHub private vulnerability reporting, through the
  Security tab. It only exists on a public repository — if a repo is ever made
  private again, it disappears, and SECURITY.md must switch to another route in
  the same change. Never advertise a reporting route that does not exist.
- The "what protects these repositories today" section must stay accurate and
  unflattering. It currently lists secret scanning, push protection and branch
  protection on `main`. If any of them is switched off or weakened, update it
  in the same change. An overstated security posture is its own risk.
- Do not soften the warning about executing model-authored shell commands
  while the scope, approval and timeout controls are still unwritten. It is
  the most accurate sentence in the repo.

## Rule: ask before touching GitHub, and never push to main

This overrides anything else in this file. Abhay approves every action that
leaves this machine.

### Ask first

Stop and ask before running any of these, showing the exact command:

- `git push` — any branch, any remote, including the first push of a new
  branch
- opening, editing, merging or closing a pull request
- creating or deleting a remote branch, tag or release
- any `gh` command that writes: issues, comments, labels, reviews, workflow
  runs, repo or org settings
- anything at all involving `--force`, `--force-with-lease`, or a remote
  delete

Do not batch these up and ask once at the end. Ask at the point of doing it,
and wait for a clear yes.

### No approval needed

Local work is yours to get on with: `status`, `diff`, `log`, `show`, `add`,
`commit`, creating and switching local branches, `stash`, reading anything.
Commit freely — a local commit is not a GitHub action.

### Every feature goes through a branch and a PR

`main` is never pushed to directly. The loop for any change:

1. Branch off current `main`: `feat/`, `fix/`, `docs/`, `chore/` or
   `refactor/` plus a short kebab-case description — `feat/pairing-code-expiry`.
2. Commit locally, as many commits as the work needs.
3. **Ask**, then push the branch.
4. **Ask**, then open the PR against `main`, filling in every prompt of the
   pull request template.
5. Report back: the branch, the PR link, what is in it, and anything left
   incomplete.
6. **Stop there.** Do not merge, do not squash, do not delete the branch, do
   not mark anything ready or draft. Abhay says what happens next.

A rejected or unanswered request is a stop, not a prompt to find another
route to the same result.

## Rule: how the docs in `docs/` are maintained

Three rules govern everything in `docs/`. They matter more than the per-file
guidance that follows.

**1. A decision is not locked until it is written.** An unwritten decision lives in
a chat window, and a chat window is gone. If a pull request settled something, the
ADR belongs in that pull request.

**2. The doc changes in the same pull request as the code.** Not in a follow-up, not
in a cleanup pass. The pull request template asks which doc changed or why none was
needed, and answering that one line honestly catches almost everything.

**3. Docs are written for someone who was not there.** Not for you today — for a
stranger, or for you in March having forgotten all of it. Any sentence that only
makes sense because you remember the conversation needs rewriting. The test: could
someone who has never spoken to you act on this? If not, it is a note to yourself,
not documentation.

### `docs/architecture.md`

One per code repo. Five sections, in this order:

1. **Responsibility** — what this component is accountable for, in three or four
   lines.
2. **What it deliberately does not do** — the boundaries, stated as prohibitions.
   **This section is more useful than the first.** Writing the boundary down is what
   stops you crossing it at 1am six weeks from now, when the shortcut looks
   reasonable and nobody remembers why it was ruled out.
3. **Internal pieces, and how a message flows** — the top-level parts, then one
   concrete end-to-end trace. The trace is what makes the parts make sense.
4. **Layering rules** — each with its reason. A rule without a reason gets
   "temporarily" broken. The desktop's is the model: `Core` compiles with no
   reference to `Transport` or `Execution`, because the logic deciding whether
   something dangerous happens must be testable with no machine to damage.
5. **What it talks to, and in which direction** — every peer, and which side dials.

**When:** write the stub when the repo gets its first real code. Update it when a new
top-level piece appears, a layering rule changes, or a peer is added or removed.
**Not on every commit** — a new file inside an existing piece is not an architecture
change.

### `docs/adr/`

One file per decision. The most valuable directory here and the easiest to write
badly. Format, naming, and the never-edit rule are in
[`docs/adr/README.md`](./adr/README.md); the template is `0000-template.md`.

The short version: five sections — Status, Context, Decision, **Rejected options**,
Consequences. Rejected options is the point; without it a record reads like a
preference rather than engineering. Name them `NNNN-short-slug.md`, date them, and
never edit an accepted one to change its decision — supersede it with a new one.

**When: at the moment of deciding.** Not a week later. A week later you will
remember what you chose and not what you rejected, and the rejected options were the
whole point.

### `docs/local-setup.md`

Getting this repo running from nothing, targeting under ten minutes: exact versions
to install, how to configure it, the command to run it, and **how to tell it is
working** — a real command with real expected output, not "it should start".

Include the recurring annoyances. Every repo has two or three, and they are the
highest-value paragraphs in the file precisely because they are the ones nobody
writes down.

**When:** the first time you set the repo up on a machine, written as you go. Not
afterwards from memory — by then you have forgotten the step that was not obvious,
which is the only step worth documenting. Then again every time a setup step changes
or starts taking longer than ten minutes.

### `docs/runbook.md` — relay and agent only

Different in kind from the others. They explain the system; this one is read by a
panicking person at 3am, and that person is you in four months with no memory of how
any of it works.

So it is **not prose**. It is a list of symptoms — the relay will not start, all
connections are dropping, approvals are not reaching phones, the database is
refusing connections, memory keeps climbing — and each one carries four things: how
to confirm it is really that, what to check first, how to fix it, and how to roll
back if the fix fails.

Two hard rules:

- **Every command is copy-pasteable.** Nobody composes a shell pipeline at 3am. Put
  the environment variables at the top so the commands below can be pasted as-is.
- **Every entry is tested once.** Break the thing on purpose, follow your own entry,
  and see whether it actually works. Untested entries carry `Verified: never`;
  replace it with a date once you have run it. An untested runbook entry is a guess
  with formatting.

**When:** add an entry the first time a failure mode happens. Fix the entry that was
wrong in the same pull request as the fix — that is the only thing that stops this
file drifting into fiction.

## Rule: check for keys before staging or committing

Run this before `git add`, and again before `git commit`. A secret that reaches a
commit is compromised even if the next commit removes it — it stays in the history,
in every clone, and in any fork. Rotating it is then the only real fix, so the
cheap moment to catch it is before it is staged.

### The check

Paste this at the repo root. It reports what *would* be committed, not what is on
disk:

```sh
# 1. What is actually staged?
git diff --cached --name-only

# 2. Would any credential-shaped file be committed?
git diff --cached --name-only | grep -Ei '\.(pem|key|pfx|p12|jks|keystore|snk|ppk|tfstate|tfvars)$|(^|/)\.env($|\.)|(^|/)(id_rsa|id_ed25519|credentials|secrets?\.(ya?ml|json))$'

# 3. Does any staged content look like a secret?
git diff --cached -U0 | grep -nEi '(api[_-]?key|secret|passwo?rd|token|bearer|private[_-]?key|access[_-]?key)[[:space:]]*[:=]|BEGIN [A-Z ]*PRIVATE KEY|sk-[A-Za-z0-9]{16,}|AKIA[0-9A-Z]{16}|gh[pousr]_[A-Za-z0-9]{20,}|eyJ[A-Za-z0-9_-]{20,}'

# 4. Is anything credential-shaped already tracked from an earlier commit?
git ls-files | grep -Ei '\.(pem|key|pfx|p12|jks|keystore|snk|ppk|tfstate|tfvars)$|(^|/)\.env($|\.)'
```

**Checks 2, 3 and 4 must print nothing.** If any of them prints, stop and deal with
it before continuing — do not commit "just to save progress".

### Reading the results

- **A hit in check 2 or 4** — the file must not be tracked. Add the pattern to
  `.gitignore`, then `git rm --cached <file>` to untrack it while keeping your local
  copy.
- **A hit in check 3** — read it before assuming the worst. A variable *name* in a
  config schema, a documented placeholder, or `KEY_HERE` in an example is fine. An
  actual value is not. If you cannot tell, treat it as real.
- **A hit that is genuinely a false positive** — leave it and move on. Do not add a
  suppression; the next person needs to see the same hit and make the same
  judgement.

### If a secret is already committed

Assume it is public from the moment it existed in a commit.

1. **Rotate the credential first.** Before touching git history, before telling
   anyone. A rotated secret in history is a non-event; an unrotated one removed from
   history is still a live secret sitting in someone's clone and in the reflog.
2. Then remove it from the working tree and add the pattern to `.gitignore`.
3. Only then consider rewriting history, and only if it never left this machine.
   Once it is pushed, rewriting is not a fix — the rotation was the fix.
4. Note it in the pull request. A quietly rotated key is a thing nobody can audit.

### What each repo's `.gitignore` must already cover

Keeping these in `.gitignore` is what makes the check above quiet enough to be
worth running:

- **Every repo** — `.env` and `.env.*` with an exception for `.env.example`, plus
  `*.pem` and `*.key`.
- **`deskaway-desktop`** — `*.pfx` and `*.snk`. Code-signing material; treat it as
  more sensitive than a service credential, because it signs software that runs
  commands on people's machines.
- **`deskaway-android`** — `*.jks`, `*.keystore`, `keystore.properties`,
  `signing.properties`. Same reasoning.
- **`deskaway-infra`** — `*.tfstate*`, `*.tfvars`, `*.tfplan`, and `.terraform/`.
  **State and plan files contain resolved secret values in plaintext**, which is why
  they are ignored rather than merely discouraged. Never paste a raw plan into an
  issue or a pull request comment either.

If you add a new kind of credential to a repo, add its pattern to `.gitignore` in
the same pull request that introduces it — not after the first near miss.

### A note on `.env.example`

It is tracked deliberately: it documents which variables exist. It holds **names and
obviously-fake values only**. The moment a real value is pasted in for convenience,
the file stops being a template and becomes a leak with an innocent-looking name.
