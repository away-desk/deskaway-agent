# Runbook — deskaway-agent

You are reading this at 3am. Start at the symptom that matches, run the confirm
command, then work down.

> **Status: nothing is deployed.** No environment, no dashboard, no alarm. Every
> entry below is **Verified: never** — the commands are shaped correctly but none
> has been run against a real system. Break each thing on purpose, follow the
> entry, and change its marker to `Verified: <date>`. An untested runbook entry is
> a guess with formatting.

## Two things that make this service easier than the relay

- **Nothing is lost when it dies.** No session, no queue, no data. Restarting is
  almost always safe.
- **The blast radius is bounded.** While the agent is down, new tasks cannot be
  planned, but runs already in progress keep going — their plan already exists and
  lives on the desktop.

So the first question is nearly always: **is this us, or the model provider?**

## Fill these in before you need them

```sh
export AWS_REGION=eu-west-1
export ENV=prod
export CLUSTER=deskaway-$ENV
export SERVICE=deskaway-agent-$ENV
export AGENT_URL=https://agent.internal.example.com
```

## First 60 seconds, always

```sh
curl -sS -o /dev/null -w '%{http_code} %{time_total}s\n' "$AGENT_URL/healthz"
aws ecs describe-services --cluster "$CLUSTER" --services "$SERVICE" \
  --query 'services[0].{desired:desiredCount,running:runningCount,rollout:deployments[0].rolloutState}'
aws logs tail "/ecs/$SERVICE" --since 10m | grep -iE 'error|exception' | head -20
```

**Did something just deploy?** Roll back first, diagnose after.

```sh
aws ecs update-service --cluster "$CLUSTER" --service "$SERVICE" \
  --task-definition "$(aws ecs describe-services --cluster "$CLUSTER" --services "$SERVICE" \
    --query 'services[0].deployments[-1].taskDefinition' --output text)" \
  --force-new-deployment
```

---

## The agent will not start

**Confirm:**
```sh
aws ecs describe-tasks --cluster "$CLUSTER" \
  --tasks "$(aws ecs list-tasks --cluster "$CLUSTER" --service-name "$SERVICE" \
    --desired-status STOPPED --query 'taskArns[0]' --output text)" \
  --query 'tasks[0].{stopped:stoppedReason,exit:containers[0].exitCode}'
```

**Check first — configuration.** `app/core/config.py` should fail fast and exit on a
missing variable rather than start half-configured.

```sh
aws logs tail "/ecs/$SERVICE" --since 15m | grep -iE 'config|missing|invalid|key' | head -20
```

The usual cause is a rotated provider key that was never written back.

```sh
aws secretsmanager describe-secret --secret-id "deskaway/$ENV/model-key" \
  --query '{changed:LastChangedDate,rotated:LastRotatedDate}'
```

**Fix:** correct the secret or the task definition in `deskaway-infra`, merge, let
`apply` run. See `deskaway-infra/runbooks/rotate-model-key.md`.

**If the fix fails:** roll back to the last task definition that started.

---

## The model provider is failing

The most likely real incident, and the least under your control.

**Confirm — separate provider failure from our failure:**
```sh
aws logs tail "/ecs/$SERVICE" --since 15m --filter-pattern 'provider' | head -30
aws logs tail "/ecs/$SERVICE" --since 15m | grep -oE '(429|500|502|503|504)' | sort | uniq -c
```

**Check by status:**

- **429, rate limited.** Check whether `providers/retry.py` is backing off or
  hammering. Retries that do not back off turn a brief limit into a sustained one,
  and the spend makes it obvious:
  ```sh
  aws logs tail "/ecs/$SERVICE" --since 30m --filter-pattern 'tokens' | tail -20
  ```
- **401 or 403.** The key is wrong, expired, or revoked. Rotate rather than debug.
- **Elevated latency, no errors.** Check the client timeout. A planning call slow
  enough to hold the relay's request open is worse than one that fails fast.
- **Provider outage.** There is no local fallback, deliberately. Confirm the relay
  degrades cleanly: new tasks refused with a clear error, not hanging.

**Fix:** for rate limits, reduce concurrency before raising it:
```sh
aws ecs update-service --cluster "$CLUSTER" --service "$SERVICE" --desired-count 2
```
For a pinned-model change, pin the version explicitly rather than tracking a
provider default.

**If the fix fails:** the honest action is to stop accepting new tasks and say so.
In-flight runs are unaffected, so this degrades rather than breaks.

---

## Plans come back unparseable

**`planning/step_parser.py` rejecting output is correct behaviour.** A malformed plan
reaching a desktop would be far worse than a failed request. Do not route around it.

**Confirm — one task shape or all of them:**
```sh
aws logs tail "/ecs/$SERVICE" --since 30m --filter-pattern 'parse' | head -30
```

One shape suggests a prompt or a task input. All of them suggests a provider or
model change.

**Check first:** the recorded call, not your memory of what the model returns.
```sh
aws s3 ls "s3://deskaway-recordings-$ENV/model-calls/" --recursive | tail -5
```

**Fix:**
- Prompt drift → add a **new** version under `app/planning/prompt/system/`. Never
  edit the one in use.
- Provider changed a default model version underneath you → pin it.

**Do not loosen the parser under incident pressure.** That converts a loud failure
into a silent one, and the silent version executes commands.

**If the fix fails:** roll back to the previous prompt version and the previous
model version together, then reproduce offline from the recording.

---

## Plan quality has degraded

Nothing is down. Steps are wrong, over-broad, or badly ordered.

**Confirm — reproduce it with the eval suite:**
```sh
uv run python -m evals.runner --suite regression
uv run python -m evals.runner --suite scenarios --compare-to main
```

**Check what changed**, in this order: a prompt version, a policy YAML, a model
version, or the provider's behaviour with none of the above changed.

```sh
git log --oneline -20 -- app/planning/prompt app/policy
```

**Fix:** roll back the prompt version rather than editing forward under pressure. Add
the failing case to `evals/regression/` before shipping anything.

**If the fix fails:** pin the model version explicitly. A provider-side change with
no diff on your side is the remaining explanation.

---

## Reversibility is being misclassified

**Treat this as more serious than an outage.** A destructive step classified as
reversible means the approval gate never fires and the product's main guard rail is
silently off.

**Confirm:**
```sh
git log --oneline -10 -- app/policy/reversibility app/policy/autonomy
uv run python -m evals.runner --suite regression --filter reversibility
```

These are YAML files, so a bad edit passes every type check — which is why the eval
matters more here than anywhere else.

**Check:** `tiers.yaml` for a changed tier, `v1_supervised.yaml` for a widened
autonomy level, and `approval_gate.py` for how it combines them.

**Fix: fail closed.** Classify unknown steps as requiring approval. An unnecessary
approval prompt is an annoyance; a missing one is an incident.

```sh
git revert <commit-that-changed-policy>
```

Add a regression eval for the specific case **before** shipping the fix.

**If the fix fails:** drop the deployed autonomy level so every step requires
approval, and say so plainly to users. Slow and safe beats fast and wrong.

---

## Cost is climbing

**Confirm:**
```sh
aws logs tail "/ecs/$SERVICE" --since 1h --filter-pattern 'tokens' | tail -30
aws ce get-cost-and-usage --time-period "Start=$(date -u -d '2 days ago' +%F),End=$(date -u +%F)" \
  --granularity DAILY --metrics UnblendedCost
```

**Check first, in likelihood order:**

1. **Retry storm.** A backoff bug is the usual cause, and
   `providers/token_accounting.py` exists partly so it shows up as spend.
2. **Context growth.** `planning/context_builder.py` deciding to include more
   history shows up directly as tokens per call.
3. **Replanning loop.** Revision is meant to be bounded; an unbounded one is a
   correctness problem that presents as a bill.

**Fix:** cap concurrency immediately, then fix the cause.

**If the fix fails:** scale to zero. Nothing is lost — the service is stateless, and
in-flight runs continue without it.

---

## Memory keeps climbing

Less likely here than in the relay, since the service is stateless.

**Confirm:**
```sh
aws cloudwatch get-metric-statistics --namespace AWS/ECS \
  --metric-name MemoryUtilization --period 300 --statistics Maximum \
  --start-time "$(date -u -d '6 hours ago' +%FT%TZ)" --end-time "$(date -u +%FT%TZ)" \
  --dimensions Name=ClusterName,Value="$CLUSTER" Name=ServiceName,Value="$SERVICE"
```

**Check:** the replay recorder buffering when its sink fails, and unbounded context
assembly on large tasks. Both hold per-request data that should not outlive the
request.

**Fix:** rolling restart buys time; the leak is the actual fix.
```sh
aws ecs update-service --cluster "$CLUSTER" --service "$SERVICE" --force-new-deployment
```

---

## After any incident

- Write down what actually happened, in the pull request or an issue.
- **Add an eval case if the failure was about plan content.** A failure mode with no
  eval will recur.
- **Fix the entry above that was wrong, in the same pull request as the fix.**
- Update the `Verified:` marker on any entry you actually used.
