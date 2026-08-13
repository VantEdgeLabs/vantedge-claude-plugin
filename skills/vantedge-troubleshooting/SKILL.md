---
name: vantedge-troubleshooting
description: Playbook for diagnosing failed agent-app workflows on VantEdge. Load this when a workflow returns FAILED or an activity throws — walks you through history events, durable logs, and the fix-vs-escalate decision.
---

# vantedge — troubleshooting a failed run

When a workflow lands in `FAILED` (or an activity throws), you have three
signals to work with: the workflow's Temporal event **history**, the app's
**durable logs** (survives pod restart), and the **run summary**. This skill
walks the standard diagnostic loop, tells you how to read each signal, and
gives you a table that maps common error strings to fix-here vs escalate.

**Prerequisites you should verify first:**

- App image is on `vantedge-app-base:0.4.0` or later — that's the version that
  auto-captures activity exceptions into durable logs. Older images (0.3.0)
  give you an event history but leave the pod-side context blank.
- You know the `workflow_id`. If you don't, start with `runs`.

## The 4-step diagnostic loop

Run these in order. Each narrows the search.

### 0. Take the LLM-first fork (fastest branch)

Before opening `history`, look at the run's `usage` summary — it takes one
command and immediately routes you to the right investigation:

```bash
vantedge-cli run <workflow_id> --format json | jq '.usage'
```

The `usage` block is always present, even when the workflow made zero LLM
calls. Read `llm_calls_count` and `llm_error_count`:

- **`llm_calls_count == 0`** — the workflow failed BEFORE any LLM invocation.
  Skip to step 3 (history) + step 4 (durable logs); the LLM path isn't the
  culprit.
- **`llm_error_count > 0`** — the LLM caused the failure. Get the per-call
  breakdown with `vantedge-cli run <wf_id> --llm-calls` and map each row's
  `error_kind` (auth / rate_limit / timeout / oom / user_code /
  context_overflow) using the table in `usage.md`.
- **`llm_calls_count > 0` AND `llm_error_count == 0`** — the LLM part
  completed cleanly; the failure is downstream (parsing, connector write,
  output delivery). Continue with step 3 (history) + step 4 (durable logs)
  as normal.

Full three-way branch and fix table: see `diagnose-failed-run-cost.md`.

### 1. Scope the failures

```bash
vantedge-cli runs --status FAILED --limit 5
# or, if you know the app:
vantedge-cli runs --app <app> --status FAILED --limit 5
```

Pick the workflow_id you're investigating. If **every** recent run failed
with the same shape, that's a systemic problem — hold onto that as evidence
for the escalation decision.

### 2. Get the run summary

```bash
vantedge-cli run <workflow_id>
```

Shows workflow id, run id, task queue, status, start / close / duration,
and — if present — the terminal failure summary. Duration tells you whether
this was a fast blowup (config / import error) or a long-running failure
(timeout, transient upstream).

### 3. Read the event history

```bash
vantedge-cli history <workflow_id>
```

Post-PR1 the history response includes structured failure fields on each
failed event. What to look for:

- **`failure.message`** — the exception message. Match this against the
  fix-vs-escalate table below.
- **`failure.stack_trace`** — Python stack. The frame that lives in
  `/app/app.py` is the user's code — that's where a fix goes.
- **`attempt`** — how many times this activity retried. `attempt >= 3` with
  the same message means the retry policy exhausted; the failure is not
  transient.
- **`retry_state`** — if `RETRY_STATE_MAXIMUM_ATTEMPTS_REACHED`, Temporal
  gave up because the `RetryPolicy(maximum_attempts=...)` cap was hit. Not
  transient — the underlying error needs fixing (or the cap raised).
- **`timeout_type`** — on `ActivityTaskTimedOut`:
  - `TIMEOUT_TYPE_START_TO_CLOSE`: your activity took longer than its
    `start_to_close_timeout=`. Either bump the timeout or fix why the
    activity is slow (usually a hanging LLM/network call).
  - `TIMEOUT_TYPE_SCHEDULE_TO_START`: no worker picked up the task in time
    — the worker is likely down or on the wrong task queue (see
    `sdk-runtime.md` → "Task queues" for the collision trap).
  - `TIMEOUT_TYPE_HEARTBEAT`: the activity stopped heartbeating. Usually
    means the pod OOM'd or crashed mid-activity.

### 4. Pull durable logs for the workflow

```bash
vantedge-cli logs <app> --durable --since 10m --workflow <workflow_id>
```

`--durable` reads the backend's AppLog history (not pod stdout) — it
survives pod restart and filters to just this workflow's rows. Combine with
`--level ERROR` to hide the noise:

```bash
vantedge-cli logs <app> --durable --since 10m --workflow <workflow_id> --level ERROR
```

## Reading durable logs

Each log line is one AppLog row:

```
2026-08-06T14:22:11Z  ERROR  activity=process_item  attempt=3  KeyError: 'customer_id'
  Traceback (most recent call last):
    File "/app/app.py", line 47, in process_item
      cid = row['customer_id']
  KeyError: 'customer_id'
```

The stack trace under an ERROR row is auto-captured from an uncaught
activity exception. This requires `vantedge-app-base:0.4.0` or later.

**Rule.** If `logs --durable` returns **no ERROR entries** but `history`
shows a FAILED activity, the base image is < 0.4.0 and the pod-side context
was never captured. Tell the user to bump `base_image:` in `vantedge.yaml`
and redeploy — after that, retries will land with a real stack trace.

## Fix-vs-escalate playbook

Match `failure.message` (or the log entry) against the leftmost column.
The **Action** column tells you what to do without opening a support ticket
— anything marked "escalate" is out of the app author's control.

| Error text pattern | Meaning | Action |
|---|---|---|
| `httpx.HTTPStatusError: 502 for url '.../api/data/query/'` | Backend data-plane proxy (router) unreachable. | Wait 30s and retry; if persistent, escalate — the router pool is likely restarting. |
| `httpx.HTTPStatusError: 502 for url '.../api/data/llm/'` | LLM proxy failed upstream. | Check gateway health; if persistent, escalate. |
| `Gateway returned 504 for <profile>/<task> after N attempt(s)` | Upstream model provider timed out. | If the request is legitimately long (large prompt, reasoning model), the platform chain's `request_timeout` is likely too short — escalate. Otherwise trim prompt / lower `max_tokens`. |
| `No active model assignment for profile=X task=Y` | The platform has no active model configured for this profile+task. | Escalate — platform owners need to create the assignment. Not fixable from `app.py`. |
| `Unknown connector_type: X. supported_types: [...]` | The user passed a non-canonical connector slug. | Pick one from `supported_types` in the same error. Fixable in `vantedge.yaml` (`data_sources:`) or the `connectors add` invocation. |
| `Only the workspace owner can perform this action` | Permission error — the caller isn't the workspace owner. If the caller IS the owner, the workspace context wasn't propagated (frontend bug). | If the user is genuinely not the owner: expected, no fix. If they are: escalate. |
| `TransportError` against a `.svc.cluster.local` URL | Backend or router service is unreachable from the pod. | Platform infra — escalate. |
| `temporalio.*` errors on run trigger, or connection errors targeting the Temporal address | Temporal cluster is down or unreachable. | Platform infra — escalate. |
| Activity's own exception (`KeyError`, `TypeError`, `AttributeError`, `IndexError`, unhandled `ValueError`) with the top user-code frame in `/app/app.py` | The app's code is wrong. | Fix in `app.py` — do NOT escalate. |

**Notes on reading the table:**

- The 502s target the **backend data-plane proxy**, not Anthropic/OpenAI
  directly — agents never talk to model providers directly, so a bare
  `httpx.ConnectError` to `api.anthropic.com` indicates someone bypassed
  `gateway.llm()`. That's a code fix.
- `Gateway returned 504` comes out of `ai_gateway/views.py` with an attempt
  count baked in — if you see `after 3 attempt(s)`, the platform already
  retried. Bumping app-side retry won't help.
- A frame like `/site-packages/temporalio/...` at the top of a stack is
  NOT the user's code — keep scrolling until you find `/app/app.py`.

## Escalation criteria

Stop debugging and ask the user to open a platform-owner ticket when any of
these hold:

- The error message mentions `temporalio`, resolves to a `.svc.cluster.local`
  host, or the workflow can't be triggered at all.
- An ArgoCD Application's `sync_status` is `OutOfSync` or `health_status` is
  `Degraded` on the affected app.
- `logs --durable` returns NO entries AND `history` shows a FAILED activity
  AND the app is already on base image 0.4.0 or later. (Durable capture
  should be working — if it's not, the capture pipeline is broken.)
- The same error appears across **multiple different apps** in the same
  workspace. That points at a workspace-level or platform-level issue, not
  an app bug.
- The error text doesn't match any row in the playbook table and its stack
  bottoms out in `vantedge/`, `temporalio/`, or `k8s/` (i.e. platform code,
  not user code).

## Common fix patterns (when it IS the app's fault)

Once you've localized to `/app/app.py`, the fix path depends on the shape:

- **`KeyError` in a dict lookup** — the row shape from `context_router.query`
  didn't match what the code expected. Either the schema changed (run
  `vantedge-cli schema <source>`) or the SQL was guessed. Rewrite the query
  or handle the missing key with `.get(...)`.
- **`RestrictedWorkflowAccessError` on import** — module-top import of a
  network-touching module (usually `from vantedge.tools import context_router`
  at file top). Move the import inside the activity. See `sdk-runtime.md` →
  "Sandbox rules".
- **`TIMEOUT_TYPE_START_TO_CLOSE`** — activity slower than expected. Bump
  `start_to_close_timeout=` on the `execute_activity(...)` call. If the
  activity is inherently long, consider splitting it into smaller activities
  so retries don't restart from scratch.
- **`workflow type X is not registered` / `activity function Y is not
  registered`** — two apps sharing a task queue. Give each app its own
  `--task-queue` on `deploy`. See `sdk-runtime.md` → "Task queues".
- **`RuntimeError: VANTEDGE_API_URL is not set`** — worker is running outside
  the pod (local dev) without env. Populate `.env` per `.env.example`.

## Where to look next

- Command reference (`runs`, `run`, `history`, `logs`): see `cli.md`
- Runtime SDK reference (sandbox rules, retry policies, task queues): see
  `sdk-runtime.md`
- Concrete agent recipes to compare a broken app against a known-good
  shape: see `agent-recipes.md`
