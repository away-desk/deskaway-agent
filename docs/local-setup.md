# Local setup — deskaway-agent

**Nothing runs yet.** `pyproject.toml` is an empty file, so there are no
dependencies to install and no application to serve. This page describes the
setup as it is intended to work, so that the first person to fill in the
manifest knows what they are aiming at. Correct it in the same pull request
that makes it true.

## What you need

- A Python interpreter matching `.python-version` (also empty — pin it when the
  package is set up).
- [uv](https://docs.astral.sh/uv/) for dependency management.
- A model provider API key of your own.

No database, no message broker, no local relay. This service is stateless and
its only external dependency is the model provider, which is deliberate — it
should be the easiest component in DeskAway to run.

## Steps

```sh
git clone https://github.com/away-desk/deskaway-agent.git
cd deskaway-agent

uv sync                  # creates .venv and installs, honouring .python-version
```

Configuration is read by `app/core/config.py`. There is no `.env.example` in
this repo yet; add one alongside the first real config field. At minimum the
service will need a provider key:

```sh
export DESKAWAY_MODEL_API_KEY=...
```

Run it:

```sh
uv run uvicorn app.main:app --reload --port 8000
```

Check it is alive:

```sh
curl localhost:8000/healthz
```

Target for the whole sequence on a cold clone is under ten minutes, most of it
`uv sync`.

## Tests

```sh
uv run pytest -q                    # unit and integration
uv run pytest tests/unit -q         # the fast subset
uv run ruff format . && uv run ruff check .
```

The test suite should not need a provider key. `replay/recorder.py` exists so
that recorded model calls can be replayed in tests — if running the suite ever
starts requiring a live key and a network, that is a regression worth fixing
rather than documenting.

## Working on prompts

Prompts live in `app/planning/prompt/system/v1/` as versioned directories. Do
not edit a prompt in place once it is in use; add a new version. Evals under
`evals/` are how you tell whether a prompt change helped, and
`evals/regression/` holds the cases that must not get worse.

## When it will not start

- **Import errors after pulling** — run `uv sync` again; a dependency changed.
- **Provider authentication failures** — check the key is exported in the shell
  you actually ran the server from, not a different tab.
- **Plans come back unparseable** — that is `planning/step_parser.py` rejecting
  model output, which is correct behaviour. Look at the recorded call in
  `replay/` rather than loosening the parser.

Once the service is deployed, production failure modes are in
[runbook.md](./runbook.md).
