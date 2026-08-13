---
name: vantedge-usage
description: Reference for the vantedge-cli usage command family — see LLM spend, tokens, and error rates for an app, workspace, or org. Load this whenever the user asks about cost, token counts, LLM usage, spend attribution, or "why is our bill high," or when triaging a failed run's LLM-side signals.
---

# vantedge-cli usage — LLM cost & token attribution

`vantedge-cli usage` surfaces the LLM call log with token counts, cost estimates, and error rates. Every LLM call an agent makes routes through `gateway.llm()`, which stamps the call with `app_id`, `workflow_id`, `workflow_type`, and `activity_attempt` so spend attributes back to the app that made it.

**Three read commands:**

| Command | Purpose |
|---|---|
| `vantedge-cli usage [--app \| --workspace \| --org]` | Aggregate spend for one scope |
| `vantedge-cli usage top` | Cost-spike leaderboard — which app / model / user is driving spend right now |
| `vantedge-cli run <wf_id> [--llm-calls]` | Per-run detail — always carries a top-level `usage` summary; `--llm-calls` adds per-call rows |

**Default format is JSON** on `usage` / `usage top` (machine-parseable so drivers can pipe through `jq`). Add `--format table` for a human-readable summary. `run` also stays JSON by default — flipping that would be a breaking change.

## Scope selection

`--app`, `--workspace`, and `--org` are mutually exclusive. When you pass none, the CLI falls back to the caller's default workspace (via `whoami`), matching `sources` / `runs`.

```bash
# App-scoped: which model + workflow_type is driving this app's spend?
vantedge-cli usage --app email-triage --since 7d --group-by workflow

# Workspace-scoped (defaults to the caller's workspace if omitted).
vantedge-cli usage --workspace 37 --since 24h

# Org-scoped: aggregate across every workspace in the caller's org.
vantedge-cli usage --org --since 30d --group-by app
```

## Time windows

Both `--since` and `--until` accept either a relative duration or an ISO 8601 timestamp — the string is passed through to the backend which parses it:

- Relative: `30s`, `5m`, `1h`, `24h`, `7d`, `30d`
- ISO 8601: `2026-08-01T00:00:00Z`

Omitting `--until` defaults to "now."

## `--group-by` — server-side pivot

Ask for the aggregate broken down along one axis. The backend returns a `groups` array in addition to the flat totals.

| `--group-by` | Question it answers |
|---|---|
| `day` | Daily spend curve. Timeline view. |
| `model` | Which model is costing the most. `claude-sonnet-4-5` vs `claude-opus-4-7`. |
| `workflow` | Which workflow_type inside an app is driving spend (only meaningful with `--app`). |
| `app` | Which app is spending — pass with `--org` for a leaderboard-style pivot. |
| `user` | Per-user spend — org-level attribution when workspaces are shared. |

## `usage top` — cost-spike leaderboard

The fastest way to answer "why is spend high right now" without loading a dashboard:

```bash
# Top 10 spenders in the last hour (default window).
vantedge-cli usage top

# Rank by tokens or error count instead of cost.
vantedge-cli usage top --by tokens
vantedge-cli usage top --by errors

# Longer window, wider fanout.
vantedge-cli usage top --window 24h --limit 20
```

`--by` picks the ranking metric — `cost` (default), `tokens`, or `errors`. `--window` accepts the same duration syntax as `--since`.

## `run <wf_id>` — always includes a `usage` summary

The `run` command's JSON response ALWAYS carries a top-level `usage` key, even when the workflow made zero LLM calls:

```json
{
  "workflow_id": "email-triage-abc123",
  "status": "FAILED",
  "usage": {
    "llm_calls_count": 3,
    "llm_error_count": 1,
    "total_tokens": 4820,
    "cost_usd_estimate": 0.019,
    "error_kind_summary": { "context_overflow": 1 }
  }
}
```

**Why the empty-shape zero fallback matters.** An AI driver diagnosing a failed run needs to distinguish three states without a second call:

1. `llm_calls_count == 0` → the run failed before invoking a model. Check `history` + durable logs.
2. `llm_error_count > 0` → the LLM caused the failure. Look at `error_kind_summary`.
3. `llm_calls_count > 0` AND `llm_error_count == 0` → the LLM part completed; the failure is downstream (parsing, connector write, output delivery).

The three-way branch above is the whole `/diagnose-failed-run-cost` recipe. See `diagnose-failed-run-cost.md`.

### `--llm-calls` — itemized per-call breakdown

Add `--llm-calls` to embed a `llm_calls` array in the response with one row per LLM call attempt:

```bash
vantedge-cli run email-triage-abc123 --llm-calls
```

Each row carries: `call_id`, `served_model`, `prompt_tokens`, `completion_tokens`, `cache_read_tokens`, `cache_creation_tokens`, `duration_ms`, `ok`, and — on failed calls — `error_kind` (`auth | rate_limit | timeout | oom | user_code | context_overflow`). Rows are grouped by `call_id`, and each row's `attempt_history` shows chain fallover as sub-rows so you can see when the platform automatically retried on a fallback model.

## `error_kind` — the diagnostic tag

Failed LLM calls carry an `error_kind` label. Each maps to a different fix path:

| `error_kind` | Meaning | Fix |
|---|---|---|
| `auth` | API key rejected upstream | Escalate — platform owner rotates the key |
| `rate_limit` | Provider throttled | Retry with backoff; if persistent, escalate for higher tier |
| `timeout` | Chain timeout exceeded | If prompt is legitimately huge, escalate to raise `request_timeout`; otherwise trim prompt |
| `oom` | Worker pod out of memory | App-side — reduce batch size / cap rows |
| `user_code` | Uncaught exception in the app | Fix in `app.py` (see `troubleshooting.md`) |
| `context_overflow` | Prompt exceeded the model's context window | App-side — trim or truncate |

## Cache-hit ratio

`cache_read_tokens` and `cache_creation_tokens` are separate — Anthropic prompt caching bills two rates. A high `cache_read_tokens / prompt_tokens` ratio means prompt caching is doing its job; a low ratio on a repetitive-prompt workload means the cache-control markers aren't landing where they should. Check the SDK's cache-block conventions.

## Cost display

The `cost_usd_estimate` field is computed per-row from a published rate table. **Estimated — actual invoices may vary.** Cross-provider totals (`total_cost_usd_estimate`) are safe to sum; raw `total_tokens` across providers is not (different $/token).

## What the "Unattributed" bucket means

Rows can land with `app_id = null` for three legitimate reasons — dashboards and `usage --org --group-by app` show them as three distinct labels:

- **Platform** — `dashboard-agent` or `context-router` traffic. Never was app-scoped; the source is known via `profile_name`.
- **Legacy SDK** — app on an old base image without the attribution headers. Fix: bump `base_image:` in `vantedge.yaml` and redeploy.
- **Historical** — pre-Phase-1 SDK traffic. Cannot be re-attributed.

If your org shows unusually large "Legacy SDK" spend, redeploying apps against the current base image is the fix.

## Common flows

### "Why did our LLM bill spike this morning?"

```bash
# What blew up in the last 2 hours?
vantedge-cli usage top --window 2h --by cost --limit 10

# Zoom in on the offender.
vantedge-cli usage --app <slug> --since 2h --group-by workflow

# Look at one failing run to see if a specific attempt loop is churning.
vantedge-cli runs --app <slug> --status FAILED --limit 5
vantedge-cli run <wf_id> --llm-calls
```

### "How much did app X cost last week?"

```bash
vantedge-cli usage --app x --since 7d --format table
```

### "Which model is driving most of our tokens?"

```bash
vantedge-cli usage --org --since 7d --group-by model
```

### "Is our prompt cache working?"

```bash
vantedge-cli usage --app x --since 24h --format json \
  | jq '{ prompt: .prompt_tokens, cache_read: .cache_read_tokens }'
# cache_read / prompt near 1.0 → caching is landing
# cache_read near 0 → cache_control markers aren't hitting
```

## Gotchas

- **Rollup lag.** The aggregate endpoints hit a rolled-up table for speed; typical lag 60–120 seconds. For a run that JUST finished, add `--llm-calls` on `run` (that path bypasses the rollup with `live=true` server-side).
- **Cross-provider sums.** `total_tokens` across mixed providers is nonsensical for billing — always prefer `cost_usd_estimate` when comparing spend across models. The `token_totals_by_provider` array in the response gives the per-provider split.
- **`--org` needs an active org.** Users without a Clerk-active-org will get a resolver error; run `vantedge-cli whoami` to confirm your org association.
- **Retry loops inflate `llm_calls_count`.** One HTTP-level LLM call can produce 2–5 rows when the chain fell over — that's still ONE `call_id`. Group by `call_id` before comparing to what your app "did."

## Where to look next

- Full `vantedge-cli` reference — `cli.md`
- Failed-run diagnostic recipe — `diagnose-failed-run-cost.md`
- Cost-spike triage recipe — `diagnose-cost-spike.md`
- Troubleshooting playbook (history / durable logs) — `troubleshooting.md`
