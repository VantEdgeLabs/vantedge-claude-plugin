---
name: vantedge-sdk-runtime
description: Reference for the vantedge Python SDK — the runtime library agent apps import. Covers the context_router data client, Temporal workflow/activity/schedule patterns, and the sandbox rules that govern what agent code can and cannot do. Use this whenever writing app.py, workflow classes, activity functions, or schedule declarations for a VantEdge agent.
---

# vantedge SDK — runtime reference

The `vantedge` Python package (v0.4.0) is what agent apps `import` from inside their pods. It provides:

- `vantedge.tools.context_router` — the data-plane client (SQL, NL queries, connector actions)
- `vantedge.tools.gateway` — the LLM gateway (`gateway.llm(...)`, `LLMGateway`)
- `vantedge.runtime.worker` — the generic Temporal worker (the base image runs this as its default command)
- `vantedge.runtime.app` — combined worker + web ASGI server (for apps with `web.enabled: true` in `vantedge.yaml`)
- `vantedge.runtime.schedules.ScheduledWorkflow` — declarative scheduled workflows

**0.3.0 restructure.** LLM calls used to hang off `context_router.llm(...)`;
they now live in `vantedge.tools.gateway`. Data access and model calls are
conceptually separate — the router has nothing to do with model routing — so
they no longer share a namespace. All other `context_router` methods are
unchanged.

## Observability — durable logs & activity failures

> For debugging failures, see the `troubleshooting.md` skill.

Nothing to configure. The runtime wires two durable sinks automatically:

- **Structured `logging` calls** — anything the app emits via Python
  `logging` (e.g. `logging.getLogger("myapp").warning("...")`) is forwarded to
  a durable log store and queryable via
  `vantedge-cli logs <app> --durable`. Survives pod restart / suspend /
  redeploy — no more racing the pod's rolling buffer.
- **Uncaught activity exceptions** — any `Exception` that escapes an
  `@activity.defn` function is written to the durable AppLog with the full
  stack trace, activity name, `workflow_id`, `run_id`, and `attempt` number
  **before** Temporal applies its retry policy. Query them via:

  ```bash
  vantedge-cli logs <app> --durable --level ERROR --workflow <wf_id>
  ```

  The exception is re-raised unchanged, so Temporal's retry behavior is
  identical to what it was without capture — you just also get a durable
  record of every attempt's failure, not only the final one.

**Requires `vantedge-app-base:0.4.0` or newer** in `vantedge.yaml`. On older
base images the exception-capture wrapper isn't installed — use plain
`vantedge-cli logs <app>` (no `--durable`) and hope the pod is still alive.

**Tip:** log with structured context — `log.info("processed", extra={"count":
n, "workflow": wf_id})`. The durable store keeps `extra=` fields and
`--durable` prints them.

**Everything the SDK does is async and Temporal-aware.** Activities that make network calls must be `async def`. Workflows are deterministic — no I/O, no random numbers, no clock reads.

## The app-module contract

The runtime loads one module (`app.py` by default, override via `VANTEDGE_APP_MODULE`) and looks for three exports:

```python
# app.py

from temporalio import activity, workflow
from datetime import timedelta

@activity.defn
async def my_activity(...) -> dict:
    ...

@workflow.defn
class MyWorkflow:
    @workflow.run
    async def run(self, ...) -> dict:
        ...

WORKFLOWS = [MyWorkflow]        # required (or ACTIVITIES)
ACTIVITIES = [my_activity]       # required (or WORKFLOWS)
SCHEDULES = []                   # optional
```

Explicit exports are preferred but the runtime will auto-discover `@workflow.defn` classes and `@activity.defn` functions if the lists aren't defined.

## The `context_router` client — full method reference

Every method is async. Import inside your activity, not at module top (see Sandbox rules below):

```python
# INSIDE an @activity.defn — always this way:
async def my_activity():
    from vantedge.tools import context_router     # <-- import HERE
    rows = await context_router.query("SELECT ...")
```

> **`data_sources` — same name, two different meanings.** The token
> `data_sources` appears in two unrelated places and means two different things.
> Don't mix them up.
>
> - **In `vantedge.yaml`** (`data_sources: [office365]`) — a list of
>   **canonical provider names** the app requires. Matches
>   `vantedge-cli connectors list`. Used by publish/subscribe preflight to
>   verify the subscriber's workspace has a matching connector.
> - **In `context_router.query(sql, data_sources=[...])`** — a list of
>   **connector instance names** in the caller's workspace (whatever the user
>   named their connector, e.g. `"office"`, `"my-outlook"`). Used to scope the
>   SQL search_path to that connector's schema.
>
> A user who added an `office365` connector might have named the instance
> `"office"`. Pass the instance name to `data_sources=`, not the provider
> name.

### `query(sql, *, data_sources=None, timeout=None)`

Run verbatim SQL against workspace connectors. Returns `{"rows": [...], "columns": [...]}`.

```python
result = await context_router.query(
    "SELECT customer_id, total FROM orders WHERE created_at > NOW() - INTERVAL '1 day'"
)
for row in result["rows"]:
    print(row["customer_id"], row["total"])
```

**Cross-source joins:** the router figures out which connectors are involved from the SQL. To hint explicitly:

```python
result = await context_router.query(
    "SELECT o.*, u.email FROM orders o JOIN users u ON u.id = o.user_id",
    data_sources=["postgres", "hubspot"],
)
```

### `execute(sql, *, data_sources=None, timeout=None)`

Alias for `query` — use it for writes (INSERT/UPDATE/DELETE) to make intent obvious to readers:

```python
await context_router.execute(
    "INSERT INTO annotations (order_id, note) VALUES ('abc', 'flagged')"
)
```

### `ask(question, *, data_sources=None, timeout=None)`

Natural-language query. Router does NL→SQL, potentially using golden queries under the hood. Returns the same shape as `query`.

```python
result = await context_router.ask(
    "how many high-value orders came in yesterday?"
)
```

**When to use `.ask` vs `.query`:**

- `.ask` — one-shot exploratory questions, prototyping, human-facing summaries
- `.query` — deterministic behavior, tested SQL, agents that need reproducible output

### `sources(*, timeout=None)`

List connectors available to this workspace.

```python
result = await context_router.sources()
# → {"sources": [
#     {"name": "postgres", "type": "postgresql", "status": "connected", "table_count": 14},
#     {"name": "slack", "type": "slack", "status": "connected", "table_count": 6}
#   ]}
```

### `schema(source, *, timeout=None)`

Full table catalog for one connector.

```python
result = await context_router.schema("postgres")
# → {"tables": {
#     "orders": {
#       "columns": [{"name": "id", "type": "uuid", "nullable": false}, ...],
#       "primary_key": ["id"],
#       "foreign_keys": [{"column": "customer_id", "references_table": "customers", "references_column": "id"}],
#       "indexes": ["idx_orders_created_at"]
#     },
#     ...
#   }}
```

### `action(source, name, *, timeout=None, **args)`

Invoke a connector-specific action (stored procedure). Args are connector-defined.

```python
# Slack: post a message
await context_router.action(
    "slack", "send_message",
    channel="#eng-digest",
    text="Daily summary is ready",
)

# Outlook: download an attachment
result = await context_router.action(
    "outlook", "DownloadAttachments",
    MessageId=msg_id,
    AttachmentId=att_id,
)
```

### `mark_as_read(source, message_id, *, timeout=None)`

Convenience wrapper for the common email "mark as read" pattern:

```python
await context_router.mark_as_read("outlook", message_id)
# equivalent to: context_router.action("outlook", "mark_as_read", message_id=message_id)
```

## The `gateway` client — LLM calls

Model calls live in **a separate module** (`vantedge.tools.gateway`) — they
don't touch the data plane and don't belong on `context_router`. Same
import-inside-activity rule applies.

### `gateway.llm(prompt, *, model=None, temperature=None, max_tokens=None, **kwargs)`

Call an LLM through the platform gateway. **This is the ONLY way agents should
call models** — do not import `anthropic` or `openai` directly.

```python
@activity.defn
async def summarize(long_text: str) -> str:
    from vantedge.tools import gateway

    response = await gateway.llm(
        prompt="Summarize this in 2 sentences:\n\n" + long_text,
        complexity="medium",
        temperature=0.0,
        max_tokens=8000,
    )
    return response["text"]
```

Return shape: `{"text": str, "model": str, "usage": {"input_tokens": int, "output_tokens": int}}`.

## Picking complexity for LLM calls

Every `llm()` call takes a `complexity` argument (`"low"`, `"medium"`, or `"high"`) that the backend uses to route the call to a specific chain. Always specify it at the call site — the default `"medium"` exists only so legacy apps keep working, new code should be explicit about what capability tier it needs.

For the full tier-picking guide, see the [[vantedge-complexity-tiers]] skill.

Signature example:

```python
await llm(prompt=…, complexity="medium")
```

### Class-based form: `LLMGateway`

Use when you want to set model / defaults once and reuse:

```python
from vantedge.tools.gateway import LLMGateway

llm = LLMGateway()
resp = await llm.complete(
    prompt="Extract the customer name from this message:\n\n" + msg,
    model="claude-sonnet-5",
    temperature=0.0,
)
```

**Why route through the platform:**

- Zero user-managed API keys (no `ANTHROPIC_API_KEY` in `.env`)
- Workspace-attributed usage tracking and cost logging
- Fallback chains configured at the platform level (Anthropic primary, OpenAI fallback, etc.)
- No `--allow-internet` needed on the deploy (gateway is internal to the platform)

### Migrating from 0.2.x

The `context_router.llm(...)` call path is gone in 0.3.0. Rewrite:

```python
# 0.2.x
from vantedge.tools import context_router
resp = await context_router.llm(prompt="...", temperature=0.0)

# 0.3.0
from vantedge.tools import gateway
resp = await gateway.llm(prompt="...", temperature=0.0)
```

## Idiomatic patterns

### The typical activity shape

```python
@activity.defn
async def query_and_summarize(question: str) -> dict:
    # Imports live inside the activity (sandbox rule — see below)
    from vantedge.tools import context_router, gateway

    # 1. Read data
    result = await context_router.ask(question)

    # 2. Call the LLM to shape it
    summary_response = await gateway.llm(
        prompt=f"Summarize these rows in 2-3 sentences:\n\n{result['rows']}",
        complexity="medium",
    )

    # 3. Optionally write back or take an action
    return {"summary": summary_response["text"], "row_count": len(result["rows"])}
```

### The typical workflow shape

```python
@workflow.defn
class SummaryWorkflow:
    @workflow.run
    async def run(self, question: str) -> dict:
        return await workflow.execute_activity(
            query_and_summarize,
            question,
            start_to_close_timeout=timedelta(seconds=120),
        )

WORKFLOWS = [SummaryWorkflow]
ACTIVITIES = [query_and_summarize]
```

### Retry policies on activities

Temporal retries activities by default (exponential backoff, no cap). To tighten:

```python
from temporalio.common import RetryPolicy

result = await workflow.execute_activity(
    my_activity,
    args=[foo, bar],
    start_to_close_timeout=timedelta(seconds=60),
    retry_policy=RetryPolicy(
        maximum_attempts=3,
        initial_interval=timedelta(seconds=1),
    ),
)
```

### Fan-out (parallel activities)

```python
@workflow.defn
class BatchProcessWorkflow:
    @workflow.run
    async def run(self, item_ids: list[str]) -> list[dict]:
        # Kick off all in parallel, gather at the end
        return await asyncio.gather(*[
            workflow.execute_activity(
                process_item,
                item_id,
                start_to_close_timeout=timedelta(seconds=30),
            )
            for item_id in item_ids
        ])
```

### Parent → child workflows

For long-running fan-out where you want observability per child:

```python
@workflow.defn
class ParentWorkflow:
    @workflow.run
    async def run(self, item_ids: list[str]):
        for item_id in item_ids:
            await workflow.execute_child_workflow(
                ChildWorkflow.run,
                item_id,
                id=f"child-{item_id}",
            )
```

## Scheduled workflows

Declare recurring workflows via a `SCHEDULES` export:

```python
from vantedge.runtime.schedules import ScheduledWorkflow

SCHEDULES = [
    ScheduledWorkflow(
        id="daily-digest",              # stable id (used as schedule and base of started run ids)
        workflow=DailyDigestWorkflow,   # the @workflow.defn class
        every="24h",                    # interval — see below
        arg=None,                       # optional single argument passed to workflow's run()
    ),
]
```

**Interval format:** accepts:
- Short strings: `"60s"`, `"5m"`, `"1h"`, `"2d"`
- Seconds as int/float: `60`, `3.5`
- `datetime.timedelta` objects

**Behavior:**

- Schedules are created idempotently on worker boot — existing ones are left as-is
- **Overlap policy is SKIP** — a slow tick doesn't stack on the next
- Schedules survive worker restarts (they live in Temporal, not in your app)
- Deleting a schedule from your `SCHEDULES` list does NOT delete it from Temporal — remove it manually via Temporal UI/CLI if you truly want it gone

**When to use schedules vs cron-outside-the-worker:** always schedules. Temporal owns the timing, exactly-once-per-tick semantics, and observability. External cron is opaque.

## Temporal best practices & scheduling

The runtime is Temporal under the hood. A few rules keep scheduled agents correct and keep multiple apps from stepping on each other.

### Task queues — one per app (the collision trap)

A Temporal task queue is a shared work pool: **every worker polling a queue must be able to run every workflow and activity type dispatched to it.** The platform defaults an app's queue to `ws-<workspace_id>`. That's fine for one app per workspace — but if you deploy a **second** app into the same workspace on the same default queue, the two workers pick up each other's tasks and fail with `workflow type X is not registered` / `activity function Y is not registered on this worker`.

**Fix: give each app its own task queue** when a workspace hosts more than one app:

```bash
vantedge-cli deploy email-manager 4 --task-queue ws-4-email-manager --allow-internet
# (web UI / secrets / egress allowlist — set web.enabled + secrets: + egress_allowlist: in vantedge.yaml)
```

The scheduled workflow's action targets the worker's queue automatically, and `ensure_schedules` **reconciles** an existing schedule to the current queue (it recreates a schedule whose queue drifted) — so moving an app to a new queue "just works" on the next deploy. Symptoms of a collision to watch for in `logs`: `NotFoundError: Activity function ... is not registered on this worker, available activities: [a different app's activities]`.

### Schedule ids are global

Schedules live in the shared Temporal namespace. **Namespace the id with the app** (`"<app>-poll"`), never a bare `"poll"`, or two apps clobber one schedule. Removing a `ScheduledWorkflow` from `SCHEDULES` does **not** delete the Temporal Schedule — it keeps firing until you delete it (Temporal UI/CLI).

### Workflows are deterministic; activities do the work

- **No I/O, clocks, or randomness in a `@workflow.run` body.** Use `workflow.now()` / `workflow.uuid4()` / `workflow.random()`, and put all network/DB/LLM calls in activities. (This is also why `context_router` is imported *inside* activities — see Sandbox rules below.)
- **Always set `start_to_close_timeout`** on an activity, sized to its real p99. Temporal retries activities forever by default with backoff — add a `RetryPolicy(maximum_attempts=…)` for calls that shouldn't retry indefinitely.
- **Keep a scheduled workflow small:** query a source, then fan out child workflows/activities for the heavy work (the poller→child pattern). Overlap policy is **SKIP**, so a slow tick never stacks on the next.

### Idempotency

Ticks retry and pollers re-see the same rows. Make processing idempotent: use a **deterministic child-workflow id** (e.g. `deal-<message_id>`) with `WorkflowIDReusePolicy`, and dedup by a stable key (a PK in your store). Then a re-tick or a redeploy is a no-op, not a double-send.

### Cadence

Every tick starts a run (and N children for a poller). Start conservative — `60s`–`5m`. Very short intervals on heavy per-tick work pile up cost even with SKIP.

## Sandbox rules — the one thing that bites everyone

Temporal executes workflow code inside a deterministic sandbox that re-imports your module. Anything that touches the network at import time (`urllib`, `httpx`, `requests`) is rejected.

**Import network-touching modules INSIDE activities, never at module top:**

```python
# WRONG — module-top import of urllib-touching code
from vantedge.tools import context_router   # <-- fails in sandbox

@activity.defn
async def my_activity():
    ...

# RIGHT — import inside the activity
@activity.defn
async def my_activity():
    from vantedge.tools import context_router   # <-- OK
    ...
```

**The `vantedge` package itself is passed through the sandbox automatically** (via `SandboxRestrictions.default.with_passthrough_modules("vantedge")`), so `from vantedge.runtime.schedules import ScheduledWorkflow` at module top is fine — the sandbox reuses the already-imported real module.

**Add other passthrough modules** via env var if you need them at module top (e.g., pydantic models):

```
VANTEDGE_PASSTHROUGH_MODULES=pydantic,my_shared_lib
```

**What's safe at module top:**

- Standard library imports that don't touch the network (`os`, `datetime`, `json`, `dataclasses`, `enum`, `typing`)
- `temporalio` imports (`activity`, `workflow`)
- `vantedge.*` imports (passthrough)
- Pure-python business logic modules

**What's NOT safe at module top:**

- `from vantedge.tools import context_router` — put inside activities
- `from vantedge.tools import gateway` — put inside activities
- `import httpx`, `import requests`, `import urllib` — put inside activities
- `from anthropic import Anthropic` — should never appear anyway (use `gateway.llm`)
- Any module that eagerly opens network sockets, files, or subprocesses on import

## What agents can call — capability summary

| Capability | Method | Notes |
|---|---|---|
| Read workspace data (SQL) | `context_router.query(sql, ...)` | Deterministic, use for tested code paths |
| Read workspace data (NL) | `context_router.ask(question, ...)` | Router picks tables, may use goldens |
| Write workspace data | `context_router.execute(sql, ...)` | Same endpoint as query, just semantic clarity |
| List connectors | `context_router.sources()` | Workspace-scoped, chat connectors excluded |
| Get schema | `context_router.schema(source)` | Live schema, not stale |
| Invoke connector action | `context_router.action(source, name, **args)` | Stored-procedure-style |
| Mark email as read | `context_router.mark_as_read(source, id)` | Common enough to warrant its own method |
| Call an LLM | `gateway.llm(prompt=..., model=...)` | **Only way to call models** — do not import anthropic/openai. Separate module from `context_router`. |
| Schedule your own runs | `SCHEDULES = [ScheduledWorkflow(...)]` | Declarative, idempotent |
| Fan out work | `asyncio.gather(*execute_activity(...))` | Standard Temporal pattern |
| Serve a web UI | Use `vantedge.runtime.app` runner + set `web.enabled: true` in `vantedge.yaml` | See recipes for pattern |

## Output shape (what the platform reads)

Whatever your workflow's `run(...)` method returns is what the platform
delivers — to the dashboard, to email (when configured), and to any future
delivery channel. There's no separate "email API"; the platform inspects the
return dict and picks the best representation.

**Key-priority order** (highest wins):

| Priority | Key | Used for | Notes |
|---:|---|---|---|
| 1 | `html` | Email body, dashboard rich view | Sanitized by the platform (see below). Best when you already know what you want the email to look like. |
| 2 | `markdown` | Email body, dashboard rich view | Rendered to HTML server-side. Convenient middle ground. |
| 3 | `briefing` \| `text` \| `output` \| `message` | Email body, dashboard summary | First present key wins; plain string. |
| 4 | JSON fallback | Dashboard code block, plain email body | Anything not matching above renders as pretty-printed JSON. |

Two companion keys the platform reads alongside the priority key above:

| Key | Used for |
|---|---|
| `subject` | Email subject line. Overrides the manifest's `outputs.email.subject`. Supports `{date}` and `{app_name}` placeholders. |
| `data` | Structured rows. When present alongside a briefing/markdown/text key, the email renders `data` as an inline table beneath the body. Ignored by the dashboard (the run's raw output already includes it). |

**The rule of thumb:** the shape you'd want the dashboard to show is exactly
the shape you'd want in email. One return value, both surfaces.

### Examples of each shape

```python
# 1. Briefing + data (the most common shape — recipe scaffold default)
return {
    "briefing": "Yesterday's revenue was $34,921 across 218 orders.",
    "data": rows,      # inline table in email
}

# 2. Rich HTML (analyst / digest agents)
return {
    "html": "<h2>Deal digest</h2><table>...</table>",
    "subject": "Weekly deal digest · {date}",
    "data": rows,
}

# 3. Markdown (alerter / summary agents)
return {
    "markdown": "## High-value order\n\n**$4,120** from Acme Corp...",
    "subject": "[ALERT] High-value order",
}

# 4. Plain string on a common key
return {"text": "Sync complete: 42 rows updated."}

# 5. Anything else — JSON fallback
return {"metrics": {...}, "counts": {...}}
```

### Email-safe HTML guidance

If you return `html`, the platform sanitizes it before delivery. Design for
the greatest-common-denominator inbox (Gmail, Outlook, Apple Mail):

- **Inline styles only.** External stylesheets and `<style>` blocks are stripped by many clients; the platform sanitizer also removes them.
- **Use tables for layout.** `<div>` + flex/grid don't survive Outlook. Nested `<table>` with `cellpadding`/`cellspacing` is the safe pattern.
- **No `<script>`, `<iframe>`, no external CSS.** The sanitizer removes them.
- **No remote images unless served over HTTPS from a stable host.** Data-URI images work but bloat the message.
- **Keep width ≤ 600px.** Fixed-width tables render consistently across clients.
- **Don't rely on classes.** Style via attributes and inline `style=""`.

The rendered HTML is embedded inside a platform email shell (header + footer
with a link back to the run detail page). Your HTML is the body slot — do
not include your own `<html>`/`<head>`/`<body>` scaffolding.

### Subject line

Precedence, highest wins:

1. `subject` key in the return dict (per-run)
2. `outputs.email.subject` in the manifest (per-app)
3. Platform default: `<app_name> · <date>`

`{date}` = the run's completion date (ISO YYYY-MM-DD). `{app_name}` = the
app's display name.

### What the platform does NOT do

- No template variables beyond `{date}` / `{app_name}` in v1. Interpolate
  anything else yourself before returning.
- No per-recipient personalization. All recipients get the same rendered body.
- No attachments in v1 — inline HTML tables via `data` for structured rows.

## Environment variables (injected by platform)

Agents don't set these manually — they're injected at deploy time. For local dev outside a pod, set them in a `.env` file:

| Variable | Purpose |
|---|---|
| `VANTEDGE_API_URL` | Backend base URL |
| `VANTEDGE_WORKSPACE_TOKEN` | `wst_*` token, scopes data-plane calls to the workspace |
| `VANTEDGE_TLS_VERIFY` | Set to `"false"` for local mkcert-signed backends |
| `TEMPORAL_ADDRESS`, `TEMPORAL_NAMESPACE`, `TEMPORAL_TASK_QUEUE` | Temporal cluster location and per-workspace queue |
| `VANTEDGE_APP_MODULE` | Which module the runtime loads (default `"app"`) |
| `VANTEDGE_ACTIVITY_WORKERS` | Thread-pool cap for sync activities (default `16`) |

## Debugging patterns

### Print rich data during development

Log at the activity level, not workflow — the workflow sandbox is deterministic:

```python
import logging
log = logging.getLogger(__name__)

@activity.defn
async def my_activity(item_id: str):
    log.info("processing %s", item_id)
    ...
    log.info("result: %s", result)
```

These land in `vantedge-cli logs <app>`.

### Test locally without deploying

Run the worker against a workspace token in a `.env`:

```bash
# .env
VANTEDGE_API_URL=https://minikube-api.vantedge.run
VANTEDGE_WORKSPACE_TOKEN=wst_test_xxx      # get from `vantedge-cli whoami` or dashboard
VANTEDGE_TLS_VERIFY=false
TEMPORAL_ADDRESS=localhost:7233             # forward Temporal port or point at cluster
TEMPORAL_NAMESPACE=temporal
TEMPORAL_TASK_QUEUE=ws-3

python -m vantedge.runtime.worker
```

Then trigger from another terminal: `vantedge-cli start my-app --workflow MyWorkflow`.

### Common errors and fixes

| Error | Cause | Fix |
|---|---|---|
| `RestrictedWorkflowAccessError` on import | Module-top import of network-touching code | Move import inside the activity |
| `RuntimeError: VANTEDGE_API_URL is not set` | Running worker outside a pod without env | Set env vars in `.env` |
| `RuntimeError: VANTEDGE_WORKSPACE_TOKEN is not set` | Same as above | Same |
| `httpx.HTTPStatusError: 401` from data plane | Invalid or expired workspace token | Redeploy the app to mint a fresh token |
| Workflow times out | Activity slower than `start_to_close_timeout` | Increase the timeout or split into shorter activities |
| Same run keeps restarting | Activity throws unhandled exception | Add try/except + logging inside the activity; check `vantedge-cli logs` |

## What agent authors should NOT do

- **Do not import `anthropic`, `openai`, or any LLM SDK directly.** Use `gateway.llm()` from `vantedge.tools.gateway`.
- **Do not import HTTP clients (`httpx`, `requests`) at module top.** Import inside activities.
- **Do not perform I/O inside workflow methods** (only inside activities). Workflow bodies are deterministic replay code.
- **Do not use `time.time()`, `random.random()`, or `datetime.now()` inside workflow methods.** Use `workflow.time()`, `workflow.random()`, `workflow.now()` instead.
- **Do not store secrets in `app.py`.** Store the value once with `vantedge-cli secrets set <slug>`, then bind it in `vantedge.yaml` under `secrets: [{name: ENV_VAR, from: <slug>}]` — the deploy mounts it as an env var.
- **Do not attempt to reach the router directly.** Always through the backend data-plane proxy (which the `context_router` client handles for you).

## Where to look next

- CLI reference (login, deploy, publish, run inspection): see `cli.md` skill file
- Concrete agent recipes (analyst, scheduled ETL, alerter, dashboard-backed): see `agent-recipes.md` skill file
