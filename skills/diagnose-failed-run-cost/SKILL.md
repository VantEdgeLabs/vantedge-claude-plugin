---
name: diagnose-failed-run-cost
description: Recipe for triaging a failed workflow run by looking at its LLM signals first. Load this when a user says "why did this run fail," "workflow X failed," or shares a workflow_id and asks what happened. Uses vantedge-cli run <wf_id>'s always-present `usage` key to pick the right branch of investigation.
---

# /diagnose-failed-run-cost — triage a failed run

`vantedge-cli run <wf_id>` always returns a top-level `usage` summary — even when the workflow made zero LLM calls. That `usage` block is the fastest fork in the road for figuring out what to look at next.

## The one-liner

```bash
vantedge-cli run <wf_id> --format json | jq '.usage'
```

Read `llm_calls_count` and `llm_error_count`, then take the matching branch.

## Three-way branch

### Branch A — `llm_calls_count == 0`

The workflow failed **before invoking any LLM**. The LLM path is not the culprit. Look at:

```bash
vantedge-cli history <wf_id>
vantedge-cli logs <app> --durable --workflow <wf_id> --level ERROR
```

Typical causes: import error at module top, missing env var, connector query failed, activity timeout before the LLM call could fire. See `troubleshooting.md` → "The 4-step diagnostic loop" starting at step 3.

### Branch B — `llm_error_count > 0`

The LLM caused the failure (fully or partially). Get the per-call breakdown to see which `error_kind` fired:

```bash
vantedge-cli run <wf_id> --llm-calls --format json | jq '.llm_calls[] | select(.ok == false)'
```

Map the `error_kind` to a fix using `usage.md` → "error_kind — the diagnostic tag." Fast lookup:

| `error_kind` | Fix |
|---|---|
| `auth` | Escalate — platform-side key rotation |
| `rate_limit` | App-side backoff; if persistent, escalate for higher tier |
| `timeout` | Escalate to raise `request_timeout`, OR trim prompt if legitimately big |
| `oom` | App-side — reduce batch size / cap rows before prompting |
| `user_code` | Fix in `app.py` — an exception escaped the activity |
| `context_overflow` | App-side — trim prompt before send |

### Branch C — `llm_calls_count > 0` AND `llm_error_count == 0`

The LLM part completed cleanly. Whatever failed happened **downstream** of the model call — parsing the response, writing to a connector, output delivery, or a second activity. Look at:

```bash
vantedge-cli logs <app> --durable --workflow <wf_id> --level ERROR
vantedge-cli history <wf_id>
```

Follow `troubleshooting.md` from step 3 onward — the failure is a normal app-code failure that just happens to be downstream of an LLM call.

## Why three branches, not two

An earlier version of this recipe split just on `llm_error_count`. That misses branch A entirely — a run that never reached the LLM was indistinguishable from a run whose LLM succeeded. Both had `llm_error_count == 0`, but the fix paths are completely different. Reading `llm_calls_count` first (is it zero?) is the discriminator.

## Sanity checks before deep-diving

- **Confirm the app is on base image ≥ 0.4.0.** Older images don't capture durable stack traces — a branch-A investigation with no ERROR log entries means the pod-side context wasn't recorded. Bump `base_image:` in `vantedge.yaml` and redeploy before further digging.
- **Confirm the run actually failed.** `vantedge-cli run <wf_id>` shows `status: FAILED` explicitly. A `TERMINATED` run is different — someone or something killed it.
- **If `usage.total_tokens` is huge relative to what the app should be doing**, the failure might be a runaway loop; jump to `diagnose-cost-spike.md` instead.

## Where to look next

- `usage.md` — full command reference and `error_kind` table
- `troubleshooting.md` — non-LLM failure modes (Temporal, connectors, sandbox rules)
- `diagnose-cost-spike.md` — for the cost-side variant of the same triage
