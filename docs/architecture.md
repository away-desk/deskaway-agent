# Architecture — deskaway-agent

## Responsibility

Turns a task written in plain language into something a machine can be held to: an
ordered checklist of steps, each tagged with how reversible it is and whether a
human must approve it. When a step fails, it produces a revised plan from what
actually happened.

It is a stateless HTTP service with exactly one caller, `deskaway-relay`.

## What it deliberately does not do

- **It does not execute anything.** No shell, no filesystem, no process launching.
  It emits a plan; `deskaway-desktop` decides whether to run it. This is the
  system's most important boundary: the component that talks to a model cannot
  act, and the component that can act does not talk to a model. Collapsing the two
  would make every guard rail elsewhere decorative.
- **It holds no state.** No database, no session, no memory between requests. A
  task's history arrives in the request or it does not exist.
- **It does not decide whether a step is allowed to run.** It classifies and
  recommends. Enforcement happens on the desktop, against the machine as it
  actually is, because only the desktop knows what is really there.
- **It does not talk to the phone, the desktop, or another agent instance.** One
  inbound edge is one thing to authenticate and rate limit.
- **It never hands the provider key to anything.** The key lives here and nowhere
  else. It is not returned in a response, not forwarded to the relay, and never
  reaches a user's machine.
- **It does not replan forever.** Revision is bounded; a task that cannot be
  planned fails as a task rather than looping.
- **It does not loosen the parser to make a model's output fit.** Unparseable
  output is an error, never a silent best guess.

## Internal pieces, and how a message flows

**`api/v1/routes/`** — `plan` (task text → steps), `classify` (step →
reversibility tier), `revise` (failed step + context → new plan for the
remainder), `health`.

**`planning/`** — `context_builder` assembles what the model is told;
`step_parser` turns the answer into structured steps and rejects what it cannot
parse; `revision` handles replanning. Prompts are versioned directories under
`prompt/system/`, never strings in code.

**`policy/`** — `reversibility/tiers.yaml` defines the tiers and `classifier.py`
assigns them; `autonomy/v1_supervised.yaml` states what an autonomy level permits;
`approval_gate.py` combines the two into a yes or no per step. These are YAML so
they can be reviewed by someone who does not read Python.

**`providers/`** — `model_client.py` is the single egress for model calls, with
`retry.py` and `token_accounting.py` attached. **`replay/recorder.py`** captures
each call and response.

### Flow of one plan request

1. `POST /v1/plan` arrives; `api/v1/deps.py` resolves request-scoped dependencies
   and `core/auth.py` authenticates the caller.
2. The route hands the task to `planning/context_builder`, which assembles the
   model context and selects a prompt version from `prompt/system/`.
3. `providers/model_client.py` makes the call — the only place in the service that
   does. `retry.py` governs backoff, `token_accounting.py` records usage, and
   `replay/recorder.py` writes the call and its response.
4. `planning/step_parser.py` parses the output into structured steps. Unparseable
   output fails the request here; nothing downstream sees a partial plan.
5. Each step passes through `policy/reversibility/classifier.py`, which reads
   `tiers.yaml` and assigns a tier.
6. `policy/approval_gate.py` combines each tier with the run's autonomy level from
   `autonomy/*.yaml` and marks the step as requiring approval or not.
7. The response is validated against the step shapes in `deskaway-protocol` before
   it leaves. A plan that does not conform is an error, not a warning.

`revise` follows the same path, entering at step 2 with the failure context added.

## Layering rules

**Every model call goes through `providers/model_client.py`.** No direct SDK call
anywhere else, however convenient.

The reason is that three separate guarantees hang off that single chokepoint:
retries and timeouts, token and cost accounting, and the replay recording that lets
tests run without a live key. A second call path silently breaks all three at once,
and nothing fails loudly when it does — the bill and the eval suite degrade
quietly.

Two more:

- **`policy/` is data first.** Tiers and autonomy levels live in YAML; Python reads
  them and must not hardcode them. Someone reviewing what counts as destructive
  should not have to read code to do it.
- **`planning/` never calls `policy/` to decide whether to plan a step.** Planning
  proposes, policy judges. Keeping them apart is what lets the policy be changed
  without regenerating plans.

## What it talks to, and in which direction

| Direction | Peer | Over |
| --- | --- | --- |
| **inbound** | `deskaway-relay` | HTTP — the only caller |
| **outbound** | model provider | HTTPS, via `providers/model_client.py` only |
| — | `deskaway-protocol` | build-time only; step and plan shapes |

Nothing else. No database, no queue, no cache, and no connection to the phone or
the desktop. That short list is deliberate: it makes this the easiest component to
run locally and the easiest to reason about when it misbehaves.
