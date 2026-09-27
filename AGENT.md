# AGENT.md — deskaway-agent

The planning service. It turns a task written in plain language into an
ordered checklist of steps, classifies how reversible each step is, and
revises the plan when a step fails. It is a stateless HTTP service and the
only component that talks to a model provider.

It decides what *should* happen; it never executes anything. Execution is
`deskaway-desktop`'s job, and that separation is deliberate — keep it.

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
  runbook.md  adr/
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

- Answer all four. "N/A" is a fine answer; a blank section is not.
- **Which unit of the plan this belongs to** is the one that pays off later.
  In five months this is how you find which PR did what, so name the unit,
  not the file you touched.
- **Anything deliberately left incomplete** is not an admission. An
  acknowledged gap is a decision; an unmentioned one is a bug you will
  rediscover.
- Change the template when a prompt stops earning its place, and keep it at
  four or five. A template long enough to skim past is worse than none.

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
- The reporting route assumes GitHub private vulnerability reporting is
  enabled on the repo. If that is ever turned off, this file needs a real
  contact route the same day, or reports arrive as public issues.
- Do not soften the warning about executing model-authored shell commands
  while the scope, approval and timeout controls are still unwritten. It is
  the most accurate sentence in the repo.
