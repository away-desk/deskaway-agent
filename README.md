# deskaway-agent

The planning service: it turns a plain-language task into an ordered
checklist, classifies how reversible each step is, and replans when a step
fails.

A stateless HTTP service called by `deskaway-relay`. It is the only
component that talks to a model provider, and it never executes anything
itself.

## Status

**Early development, nothing works yet.**

The route and module layout is settled, but every file is empty —
`pyproject.toml` included, so the package has no dependencies, no entrypoint
and no test config. There is no prompt, no tier definition, and no eval
case. Nothing here answers a request today.

## Running locally

Not yet possible: `pyproject.toml` is empty, so there is nothing to install
and no app to serve.

The interpreter version is pinned in `.python-version`. The intended path,
once the manifest and `app/main.py` exist:

```sh
git clone https://github.com/away-desk/deskaway-agent.git
cd deskaway-agent

uv sync                                  # honours .python-version
export DESKAWAY_MODEL_API_KEY=...        # a provider key of your own

uv run uvicorn app.main:app --reload     # http on localhost
uv run pytest                            # unit + integration
```

Target is under ten minutes on a cold clone. This service needs no database
and no local relay to boot — a provider key is the only external dependency,
and the replay recorder is intended to let the test suite run without one.
`docs/runbook.md` will carry the detail; it is currently empty.

## The rest of DeskAway

Cross-repo docs and architecture decisions live in
**[deskaway-docs](https://github.com/away-desk/deskaway-docs)**. All
components are under the **[away-desk](https://github.com/away-desk)** org.

The step and plan shapes this service emits are defined in
[deskaway-protocol](https://github.com/away-desk/deskaway-protocol).
Contributor guidance, including the rule for maintaining this README, is in
[AGENT.md](./AGENT.md).
