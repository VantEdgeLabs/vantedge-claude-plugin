---
name: diagnose-cost-spike
description: Recipe for triaging an unexpected LLM cost spike. Run this when the user says "spend is up," "bill spiked," "our LLM cost is high," "why are we burning tokens," or asks who / what is driving the current spend. Uses vantedge-cli usage top → drill-down → per-run breakdown.
---

# /diagnose-cost-spike — triage an LLM cost spike

Goal: in three commands, identify (1) which app/model/user is driving the spike, (2) whether it's a runaway retry loop or legitimate volume, and (3) whether the fix is app-side or platform-side.

## Step 1 — leaderboard for the current window

```bash
vantedge-cli usage top --window 1h --by cost --limit 10 --format json
```

The `top` array is ranked descending by cost. What to read:

- **Top row's `cost_usd_estimate` vs baseline.** If one row is >5× the others, that's the offender.
- **`error_count` column.** A row with high cost AND high error count usually means a retry loop — the same call is firing over and over. Also inspect the same window `--by errors`:

  ```bash
  vantedge-cli usage top --window 1h --by errors --limit 10
  ```

  If the same app tops both boards, that's a retry loop confirmed.
- **`total_tokens` vs `cost_usd_estimate`.** High tokens + moderate cost = the app is using a cheaper model heavily (probably legit). Moderate tokens + high cost = the app is hitting a premium model (Opus, GPT-4) on every call — consider profile downgrade.

## Step 2 — zoom into the offender

Once you know the app slug from step 1:

```bash
vantedge-cli usage --app <slug> --since 2h --group-by workflow --format json
```

`groups` array pivots by `workflow_type`. If ONE workflow_type owns the spend, that's the workflow to fix. If spend is spread evenly, the app itself is expensive and needs a model-tier or prompt-size cut.

Also pivot by model:

```bash
vantedge-cli usage --app <slug> --since 2h --group-by model
```

Confirms which model the workflow is landing on.

## Step 3 — inspect a single run to see the retry pattern

Pick a recent run of the offending workflow and read its per-call breakdown:

```bash
vantedge-cli runs --app <slug> --limit 5 --status FAILED
# → pick a workflow_id
vantedge-cli run <workflow_id> --llm-calls --format json
```

Read `usage.llm_calls_count` and count the `llm_calls` array. If they're way higher than the number of `@activity.defn` calls the app makes, the chain is falling over on every attempt — check `error_kind` on each row:

- All `rate_limit` → provider is throttling; escalate for a higher tier or add app-side backoff.
- All `timeout` → chain `request_timeout` is too short for legitimate prompt size; escalate.
- All `context_overflow` → prompt is too big; app-side fix (trim or truncate).
- Mix / `auth` → escalate immediately (key rotation or credential drift).

## Step 4 — decide fix path

| Signal from steps 1-3 | Root cause | Fix path |
|---|---|---|
| One app dominates + high `error_count` + all `context_overflow` | Prompt size drifted past context window | App-side: trim `app.py`, truncate rows before prompting |
| One app dominates + steady `error_count` = 0 + high `total_tokens` | Legitimate volume increase (new users, scheduled runs) | Not a bug — communicate spend, consider caching hits |
| One app dominates + all `timeout` errors | Chain `request_timeout` too short OR upstream is degraded | Escalate to platform owner |
| No single dominant app + broad rise across `--group-by app` | Base-image or platform change increased per-call cost | Escalate; check recent chain assignment / model swap |
| Legacy-SDK bucket is unusually large | Apps on old base image aren't reporting attribution | Rebuild apps against current base image, redeploy |

## Where to look next

- `usage.md` — full command reference
- `diagnose-failed-run-cost.md` — for the failed-run branch of the tree
- `troubleshooting.md` — for non-LLM failure modes
