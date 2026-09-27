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
