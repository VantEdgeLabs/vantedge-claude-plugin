# Agent Capabilities Reference

> A guide to what a VantEdge agent can actually DO — the data-plane API surface, the platform primitives it composes with, and worked recipes.
> Companion to the developer guide, which covers scaffolding, deploying, and observing agents.

## The mental model

A VantEdge agent is a **durable Python worker** built on Temporal, packaged as a container, and deployed into a per-workspace Kubernetes namespace by the CLI. The developer guide covers the shape of that package — `vantedge.yaml`, `app.py`, `WORKFLOWS` / `ACTIVITIES` / `SCHEDULES`, the base image, the local dev loop. This document picks up where that leaves off: the data-plane surface the worker calls into, the platform primitives it composes with, and the operational envelope it runs inside.

Two rules make everything else make sense. First, the worker **never holds source credentials or router credentials**. It ships with a workspace token that the platform mints at deploy time and injects as `VANTEDGE_WORKSPACE_TOKEN`; every data-plane call carries that token, and the backend proxy resolves it to exactly one `RouterInstance` and forwards accordingly. Tenant isolation is a server-side lookup, not a client-side convention. Second, the worker's runtime API is deliberately narrow — `context_router.query / ask / execute / sources / schema / action` plus a `SCHEDULES` export — which keeps agent code decoupled from how the platform resolves connectors, applies goldens, or routes queries.

## The context_router client — full reference

`from vantedge.tools import context_router`

Import inside your `@activity.defn`, never at module top — the Temporal workflow sandbox re-imports the module and any eager `urllib` touch trips it (see developer-guide.html §Sandbox constraints, lines 287-292).

Every method attaches `Authorization: Bearer wst_...` from the injected token. `WorkspaceTokenAuthentication` (`data_plane/authentication.py:39-58`) validates it, bumps `last_used_at`, and yields an `AppPrincipal` whose `.workspace` scopes every downstream call. A missing or inactive token returns `401`; a token for a different workspace transparently routes to that workspace — there is no cross-workspace fallback path.

### Method reference

| Method | HTTP | Purpose |
|---|---|---|
| `query(sql, *, data_sources=None, included_tables=None, analyst_guidance=None)` | `POST /api/data/query/` with `verbatim_sql` | Runs SQL as-is through the router's SQL pipeline. |
| `ask(question, *, data_sources=None, included_tables=None, analyst_guidance=None, conversation_context=None)` | `POST /api/data/query/` (NL) | Full TEP pipeline: catalog → DSPy compiler → validation → DAG → synthesis. |
| `execute(sql, *, data_sources=None)` / `write(table, row) -> row_id` | `POST /api/data/query/` (DML) | Same endpoint; DML statement. Prefer `action()` for connector-typed writes. |
| `sources()` | `GET /api/data/sources/` | Lists workspace's provisioned connectors. |
| `schema(source)` | `GET /api/data/sources/?source=<name>` | Returns the cached `discovered_catalog` for one source. |
| `action(source, name, **args)` | `POST /api/data/actions/` | Invokes a proc-action (returns rows) or write-action (returns `[]`). |
| `mark_as_read(source, message_id)` | thin wrapper over `action()` | Typed helper for Office365 / Gmail. |

### `query` / `ask`

Both hit `POST /api/data/query/` (`data_plane/views.py:37-93`). `query` sets `verbatim_sql=True` and the router runs the SQL as given. `ask` submits `{"query": <NL>, "user_id": f"app:{workspace.id}"}` and the router runs the full pipeline.

Return envelope for both:

```json
{
  "answer": "…",
  "data": [{"col": "val", ...}],
  "execution_metadata": {
    "final_gate_passed": true,
    "error": null,
    "sources": [...],
    "row_count": 42,
    "fix_attempts": 0
  },
  "gate_results": [...],
  "reasoning_trace": {...}
}
```

Failure modes to handle: `404 No context router for this workspace`, `409 Router has no endpoint URL yet`, `502 Router request failed` (network), and — the important one — a `200` where `execution_metadata.error` is set (`column_not_found`, `table_not_found`, `syntax_error`, `permission_denied`, `operator_not_supported`, `unknown`) with `data=[]`. Do not degrade a SQL error into a hallucinated "no rows" answer downstream. Hard timeout is 60s.

### `execute` / `write`

Same endpoint as `query`, DML instead of SELECT. Most connectors are not writeable; when they are, the CData driver decides what DML shapes it accepts. For Graph, Salesforce, Slack, and similar, prefer `action()` — the connector exposes typed operations that route through the vendor's real API rather than a DML-to-REST translation.

### `sources` / `schema`

`sources()` (`data_plane/views.py:96-151`) returns the workspace's provisioned connectors, filtered by the default manager (chat-scoped connectors are hidden):

```json
{"sources": [{"name": "warehouse", "type": "postgres", "status": "active",
              "connected": true, "table_count": 14, "tables": ["orders", ...]}]}
```

`schema(source)` returns the cached `discovered_catalog`, **not a live introspection**:

```json
{"name": "warehouse", "type": "postgres", "status": "active", "connected": true,
 "catalog": {"tables": [{"name": "orders", "columns": [...]}]}}
```

If you need live shape (a schema drift check, freshly-added columns), issue a `query()` against the connector — `information_schema` for Postgres, `sys.columns` for SQL Server, `sys_tables` for CData bridges.

### `action`

`POST /api/data/actions/` with `{"source", "action", "args": {...}}` (`data_plane/views.py:154-200`), proxied to the router's `POST {router}/connectors/<source>/actions/<action>` (`endpoints/connectors.py:322-364`). Dispatched by `run_connector_action` (`services/connector_actions.py:74`) in one of two shapes:

- **Proc actions** (`kind="proc"` or undeclared) run `EXEC <proc> <args>` through the CData JDBC bridge and return rows: `{"connector": ..., "action": ..., "rows": [{...}]}`. `DownloadAttachments` yields rows containing a base64 `filedata` field.
- **Write actions** (`kind="write"`) issue DML against the live source and return `{"rows": []}`.

Errors surface as `400` on bad action name / missing declared args, `502` on driver-side EXEC failure. Timeout is 120s (larger than `/query` for MB-scale downloads).

**How to discover actions.** The current developer path is to call the router's connector metadata endpoint or read the router's action manifest for the specific connector type — there is no client-side registry yet. This is called out under Gaps.

### `mark_as_read(source, message_id)`

Documented thin wrapper. Example in developer-guide.html:615.

## What actually happens when you call it

Both `query` and `ask` converge on the router's `POST /query`. There is **no separate `/ask` route** — the two variants differ only in whether the payload carries `verbatim_sql`.

### Backend dispatch (query_data tool path)

For calls originating inside Atlas (via the internal `query_data` tool), the backend runs a full pre-flight before making the HTTP call (`ai_agents/atlas/tools/query_data.py:72`):

1. **Analyst-guidance assembly** — process narrative, framework, source priorities, glossary, connector profiles, prior SQL are folded into an `analyst_guidance` string (lines 124-181).
2. **Process-driven golden force-election** (lines 195-274) — if the active strict-process step is `kind: "golden"`, the step's `golden_query_slug` overrides whatever the caller passed.
3. **Golden resolution** (lines 281-377) — loads the `GoldenQuery` row (`workspace_id + slug + status="active"`), builds `golden_payload = {id, slug, sql_template, params, parameter_schema, mode}`. Presence check only; full validation is router-side.
4. **Strict guardrail pre-flight** (lines 379-428) — always-on `PolicyEngine.evaluate_unit(session_id, "__guardrail__", …)`. Prolog gate. Violated calls short-circuit before the wire.
5. **Unit-aware pre-flight** (lines 430-553) — evaluates pending atlas rule_units against `process_facts`.
6. **Payload construction** (lines 555-725) — assembles the final wire payload with correlation headers.

External SDK / MCP callers hit `POST /api/data/query/` directly and skip the process-aware pre-flight — they get workspace scoping and token auth, nothing more.

### Router-side execution

`POST /query` (`context-router/src/endpoints/core.py:171`) authenticates via the router's own API key (a `RouterInstance` is 1:1 with a workspace and carries its own key). Correlation headers become OTEL span attributes. Then `ContextRouter.process_query()` forks:

**Golden bypass** (`src/tep/compiler/golden_bypass.py:388 build_golden_strict_plan`):

- `_validate_and_coerce_params` — walks declared params, coerces to typed values.
- `_validate_against_catalog` — checks every referenced table/column exists.
- `_substitute_placeholders` — `{name}` → `$N` positional binds; `connector`-typed params become inline identifier substitution.
- Bound SQL flows through `sql_pipeline/pipeline.py` `_process_verbatim`. **No LLM SQL generation, no fix loop, no reviewer.**

**Free-form path** (no `golden_query` payload):

- `CatalogBuilder → SourceCatalog` (OpenMetadata table discovery).
- `QueryCompiler` (DSPy) emits a `TypedExecutionPlan` DAG, with rules as planning context.
- Gate 1 static analysis — non-compliant plans trigger one recompile, then a deterministic rules-aware fallback.
- `PlanOptimizer` → `DAGExecutionEngine` with per-level fact assertion and `$ref` late binding.
- Final Prolog `contract_violation/3` gate whose verdict lands on `execution_metadata.final_gate_passed`.

### What "vetted" means

For a golden, "vetted" means: **the SQL template is stored verbatim in `GoldenQuery.sql_template`, was authored through the golden-authoring wrapper, and runs verbatim at request time.** Only mutations at request time are param coercion, catalog validation, and placeholder substitution. Semantics are trusted because the template is human-authored.

For a free-form query, safety is layered: DSPy `GenerateSQL` + `EXPLAIN`-based validation + fix loop up to seven attempts, `sql_precheck.py` CTE / temp-table detection, Prolog rules-aware Gate 1 (structural), and the final Prolog `contract_violation/3` gate. Both paths get pre/post-flight Prolog guardrails on the backend and structured `SQLExecutionError` surfacing via `result_meta.error`.

### Golden invocation from an agent

Goldens fire **only when the payload contains `golden_query`**. The trigger-matching path against arbitrary NL was removed. There are three election modes today: LLM-elected inside Atlas via the "Available Golden Queries" system prompt section, process-forced under a strict process, or ad-hoc via `target_sources`. From an SDK-facing agent, there is no first-class `context_router.run_golden(slug, **params)` call yet — see Gaps.

## Scheduling & triggers

Export a module-level `SCHEDULES` list from `app.py` alongside `WORKFLOWS` and `ACTIVITIES`. The generic worker registers each entry as a **Temporal Schedule** on boot, idempotent by `id`:

```python
from vantedge.runtime.schedules import ScheduledWorkflow

SCHEDULES = [
    ScheduledWorkflow("myapp-poll", AnalystWorkflow, every="60s"),
    ScheduledWorkflow("tracey-nightly", AnalystWorkflow,
                      cron="*/5 * * * *", arg="SELECT ..."),
]
```

`ScheduledWorkflow(id, workflow_cls, *, every=None, cron=None, arg=None)`:

| Field | Type | Notes |
|---|---|---|
| `id` | `str` | Stable Schedule identifier; also the idempotency key. |
| `workflow_cls` | `@workflow.defn` class | Must appear in `WORKFLOWS`. |
| `every` | `"60s" \| "5m" \| "1h" \| "2d"` \| int \| timedelta | Interval spec. Seconds if int. |
| `cron` | 5-field cron string | **Precedence over `every`** when both set. |
| `arg` | any JSON-serializable | Single positional arg to the workflow's `run()`. |

Idempotency matters because ArgoCD may roll multiple worker replicas that race to register the same `id` — Temporal's upsert semantics converge them on one Schedule. Redeploys never duplicate. Cron uses the standard 5-field expression and interprets timezone from the worker's `TZ` env; set it explicitly at deploy time if you need anything other than UTC.

For dynamic schedules (per-tenant, per-customer, computed at runtime), don't try to mutate `SCHEDULES` — instead export a bootstrap workflow that calls the Temporal Schedule API at startup and reconciles against a config table.

## The web mode

Declare `web.enabled: true` in `vantedge.yaml` (with `web.module: web:app` and `web.port: 8080` as defaults, both overridable in the manifest) and the same pod runs the Temporal worker plus a uvicorn ASGI app (`vantedge.runtime.app` combines them). The chart renders a Service, an Ingress mounted at `https://<VANTEDGE_APP_URL_HOST>/apps/<workspace_id>/<name>/`, and two chained Traefik middlewares:

- **stripprefix** — drops the `/apps/<ws>/<name>` prefix so your FastAPI routes are relative (`ingress.yaml:14-16`).
- **authz forward-auth** — hits `apps/<name>/authz/` (`runs_views.py:105-131`), returns 2xx only if the Clerk-authenticated caller's org owns the app, and injects `X-VantEdge-Workspace` / `X-VantEdge-App` headers on success.

The dashboard's session cookie flows through because the app is served under the same host. Setting `web.public: true` in the manifest disables forward-auth for demos and public dashboards. `ensure_web_tls` copies the wildcard `vantedge-tls` secret into the app's namespace so Ingress TLS terminates without you managing certs.

The **shared in-process store** pattern (`store.py` from the scaffold — a module-scope SQLite instance) is the intended bridge between activities and the ASGI app. It works because worker and web share one process; you get durability of Temporal's activity history plus zero-latency read from a live dashboard. Reach for a PVC or Postgres only when you need cross-process durability.

## Recipes

### Recipe 1 — Analyst Agent

Takes an NL question via workflow input, answers via `.ask()`, returns a structured verdict.

**vantedge.yaml**
```yaml
name: analyst
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.5.5
module: app
web:
  enabled: false
triggers: []
data_sources: [warehouse]
```

**app.py**
```python
from datetime import timedelta
from temporalio import activity, workflow
from temporalio.common import RetryPolicy

@activity.defn
async def ask_router(question: str) -> dict:
    from vantedge.tools import context_router
    reply = await context_router.ask(question)
    return {"answer": reply["answer"], "rows": reply["data"]}

@workflow.defn
class AnalystWorkflow:
    @workflow.run
    async def run(self, question: str) -> dict:
        return await workflow.execute_activity(
            ask_router, question,
            start_to_close_timeout=timedelta(seconds=90),
            retry_policy=RetryPolicy(maximum_attempts=3),
        )

WORKFLOWS = [AnalystWorkflow]
ACTIVITIES = [ask_router]
```

```bash
vantedge-cli deploy analyst <ws>
```

Simplest useful shape: a durable workflow that turns free-text into a governed answer via `context_router.ask()`. The router picks the source, writes the SQL, returns rows plus a summary; the workflow adds Temporal retries so a transient LLM/warehouse blip re-runs cleanly.

### Recipe 2 — Scheduled ETL Agent

Reads Postgres hourly, transforms, writes to a warehouse via `.execute()`.

**vantedge.yaml**
```yaml
name: hourly-rollup
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.5.5
module: app
data_sources: [prod_pg, warehouse]
```

**app.py**
```python
from datetime import timedelta
from temporalio import activity, workflow
from temporalio.common import RetryPolicy
from vantedge.runtime.schedules import ScheduledWorkflow

@activity.defn
async def pull_orders() -> list[dict]:
    from vantedge.tools import context_router
    res = await context_router.query(
        "SELECT region, SUM(amount) AS gross FROM orders "
        "WHERE created_at > now() - interval '1 hour' GROUP BY region",
        data_sources=["prod_pg"],
    )
    return res["data"]

@activity.defn
async def write_rollup(rows: list[dict]) -> int:
    from vantedge.tools import context_router
    if not rows:
        return 0
    values = ", ".join(f"('{r['region']}', {r['gross']}, current_timestamp)"
                       for r in rows)
    await context_router.execute(
        f"INSERT INTO hourly_rollup (region, gross, ts) VALUES {values}",
        data_sources=["warehouse"],
    )
    return len(rows)

@workflow.defn
class RollupWorkflow:
    @workflow.run
    async def run(self) -> int:
        rows = await workflow.execute_activity(
            pull_orders, start_to_close_timeout=timedelta(minutes=2),
            retry_policy=RetryPolicy(maximum_attempts=5))
        return await workflow.execute_activity(
            write_rollup, rows, start_to_close_timeout=timedelta(minutes=2))

WORKFLOWS = [RollupWorkflow]
ACTIVITIES = [pull_orders, write_rollup]
SCHEDULES = [ScheduledWorkflow("hourly-rollup", RollupWorkflow, cron="0 * * * *")]
```

```bash
vantedge-cli deploy hourly-rollup <ws>
```

Cross-connector movement on a Temporal schedule. The `SCHEDULES` export hands cron to Temporal so the platform owns firing (no cron sidecar), and `context_router` enforces which sources the app can read and write — governance travels with the query, not the pod.

### Recipe 3 — Alerter Agent

Polls Slack every 5 minutes, fires a Slack DM via `.action()` when a keyword appears.

**vantedge.yaml**
```yaml
name: mention-alerter
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.5.5
module: app
data_sources: [slack]
```

**app.py**
```python
from datetime import timedelta
from temporalio import activity, workflow
from vantedge.runtime.schedules import ScheduledWorkflow

@activity.defn
async def scan_slack(keyword: str) -> list[dict]:
    from vantedge.tools import context_router
    res = await context_router.query(
        "SELECT Id, ChannelId, Text, UserId FROM Messages "
        "WHERE CreatedTime > dateadd(minute, -5, getdate()) "
        f"  AND Text LIKE '%{keyword}%'",
        data_sources=["slack"],
    )
    return res["data"]

@activity.defn
async def notify(user_id: str, message: str) -> dict:
    from vantedge.tools import context_router
    return await context_router.action(
        source="slack", name="SendMessage",
        ChannelId=user_id, Text=message,
    )

@workflow.defn
class AlerterWorkflow:
    @workflow.run
    async def run(self, params: dict) -> int:
        hits = await workflow.execute_activity(
            scan_slack, params["keyword"],
            start_to_close_timeout=timedelta(seconds=30))
        for h in hits:
            await workflow.execute_activity(
                notify, params["notify_user"],
                f"[{params['keyword']}] {h['Text'][:200]}",
                start_to_close_timeout=timedelta(seconds=15))
        return len(hits)

WORKFLOWS = [AlerterWorkflow]
ACTIVITIES = [scan_slack, notify]
SCHEDULES = [ScheduledWorkflow("mention-alerter", AlerterWorkflow,
                               cron="*/5 * * * *",
                               arg={"keyword": "outage", "notify_user": "U0123ABCD"})]
```

```bash
vantedge-cli deploy mention-alerter <ws>
```

Read-then-act pattern. `.query()` covers observation, `.action()` covers side effects; both go through the same governed connector so the app never handles Slack tokens itself. The router's per-action policy layer decides whether `SendMessage` is allowed for this workspace.

### Recipe 4 — Dashboard-Backed Agent (`web.enabled: true`)

Scheduled activity computes fresh KPIs, writes to the shared in-process store; `web.py` renders them.

**vantedge.yaml**
```yaml
name: kpi-board
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.5.5
module: app
web:
  enabled: true
  module: web:app
  port: 8080
data_sources: [warehouse]
```

**app.py**
```python
from datetime import timedelta
from temporalio import activity, workflow
from vantedge.runtime.schedules import ScheduledWorkflow

@activity.defn
async def analyze(item: str) -> str:
    import json, store
    from vantedge.tools import context_router
    res = await context_router.query(
        "SELECT region, SUM(amount) AS gross FROM orders "
        "WHERE created_at > current_date - 1 GROUP BY region ORDER BY gross DESC",
        data_sources=["warehouse"],
    )
    summary = json.dumps(res["data"])
    store.save(item, summary)
    return summary

@workflow.defn
class AnalysisWorkflow:
    @workflow.run
    async def run(self, item: str) -> str:
        return await workflow.execute_activity(
            analyze, item, start_to_close_timeout=timedelta(seconds=60))

WORKFLOWS = [AnalysisWorkflow]
ACTIVITIES = [analyze]
SCHEDULES = [ScheduledWorkflow("kpi-board", AnalysisWorkflow,
                               cron="*/15 * * * *", arg="revenue_by_region")]
```

`web.py` and `store.py` are the scaffolded defaults from `vantedge-cli init --web` (FastAPI reading `store.all_results()`).

```bash
vantedge-cli deploy kpi-board <ws>
```

Combined runner: worker and ASGI app share one process, so `store.py`'s in-memory SQLite is a straight-through bridge with no PVC and no message bus. The schedule keeps the store fresh; the per-app URL returned by `deploy` (when `web.enabled: true` in the manifest) becomes a live, auth-gated dashboard your team can bookmark.

## Operational reference

### Deploy semantics

`POST /api/apps/deploy/` (`agent_apps/views/apps_views.py:29-88`) is authenticated by `ApiKeyAuthentication`, resolves the workspace inside the caller's org, upserts a `DeployedApp` row, and hands off to `AppDeployService.deploy` (`services/app_deploy_service.py:181-236`). That service, in order:

1. Mints a workspace-scoped token (`WorkspaceToken.objects.create`, line 189-191).
2. Derives `namespace`, `argocd_app_name`, and `task_queue` from `app-{workspace_id}-{name}` / `ws-{workspace_id}` (line 193-198).
3. Builds a Helm `valuesObject` (`build_values`, line 93-136) carrying app identity, image, Temporal address + task queue, backend proxy URL, sandbox flags, persistence, web config, and the token.
4. Wraps it in an ArgoCD `Application` spec on the `vantedge` project pointing at `vantedge-charts/vantedge-app-worker` (line 138-179) with `automated={prune, selfHeal}`.
5. Calls `ArgoCDClient().create_application(spec, upsert=True)` (line 227).

`get_or_create(workspace, name)` makes deploy idempotent per (workspace, app name). `upsert=True` means image bumps, `web.*` manifest changes, and `--allow-internet` toggles all patch the existing Application; ArgoCD reconciles the diff. Redeploy is the update path — there is no separate mutation endpoint.

### Task-queue isolation

Every app polls `ws-<workspace_id>` (`app_deploy_service.py:117, 196`). Workflows submitted from `app_start_run_view` (`runs_views.py:300`) target the deployed app's queue, so one workspace's workers only ever see that workspace's work. `start_run` also tags workflows with `workspace_id` + `app_id` search attributes for later filtering.

### Network policy

The `NetworkPolicy` in `templates/networkpolicy.yaml` is default-deny with egress to DNS and the `vantedge` namespace (Temporal + backend proxy) only. `--allow-internet` (`app_deploy_service.py:107`) adds egress to `0.0.0.0/0:443` excluding RFC1918 (`networkpolicy.yaml:43-60`) — public HTTPS only, no cross-tenant reach.

### Per-app PVC

`persistence.enabled=True` always. Chart mounts a 1Gi RWO PVC at `/data` (`values.yaml:70-74`), and because it's RWO the Deployment strategy flips to `Recreate` (`deployment.yaml:10-14`) so no two pods contend during rollout. Home for file-backed SQLite and saved reports. Cross-replica coordination needs Postgres, not the PVC.

### Observability from outside the CLI

`runs_views.py` accepts both `ApiKeyAuthentication` and `ClerkAuthentication` (line 34) — dashboard and CLI share it:

| Endpoint | Purpose |
|---|---|
| `GET apps/` | List deployed apps in workspace. |
| `GET apps/runs/?workspace=&app=&status=` | Workflow list via Temporal. |
| `GET apps/runs/<wf>/` | Run summary. |
| `GET apps/runs/<wf>/history/` | Event stream. |
| `POST apps/runs/<wf>/terminate/` `/cancel/` | Interrupt. |
| `GET apps/<app>/logs/` | Tail worker pod stdout (`svc.app_pod_logs`). |

ArgoCD UI and `kubectl -n app-<ws>-<name>` remain as operator-level fallbacks.

### Rollback

There is no first-class version history on `DeployedApp` — no image-tag ledger, no `previous_image`. Rollback is redeploying the prior tag; the upsert path patches Helm values and ArgoCD rolls the Deployment. ArgoCD's own rollout history plus `argocd app rollback` on `argocd_app_name` (`deployed_app.py:27`) is the practical undo. `Status` on `DeployedApp` (line 14-17, 38) only tracks last-deploy outcome (`pending` / `deployed` / `failed`), not a version chain.

## Gaps and future work

Honest inventory of what is not exposed yet:

- **First-class golden invocation.** `context_router.run_golden(slug, **params)` does not exist. External agents can only fire goldens by hand-constructing a `golden_query` payload or going through Atlas. A typed SDK-side helper with parameter validation against `GoldenQuery.parameter_schema` is the obvious next step.
- **Connector action discovery.** No client-side registry of available actions per connector type. Developers reverse-engineer from the router action manifest or the connector CLAUDE.md. A `context_router.list_actions(source)` returning declared actions + arg schemas would close the loop.
- **Cross-workspace app grants.** One app runs against exactly one workspace; there is no facility for an app to serve multiple workspaces or for a scheduled job to fan out across workspaces. Multi-tenant SaaS-style apps must run one deployment per workspace today.
- **Connector CRUD from the runtime.** Apps can list `sources()` and read `schema(source)` but cannot provision, rotate credentials on, or remove connectors — that stays in the dashboard / control plane.
- **Structured test doubles.** No official `context_router` mock, no pytest fixtures. Local testing means running the worker on-laptop against a real workspace token, which is fine for smoke but not for CI. A `context_router.fake` that replays fixture rows is a known gap.
- **Rate limiting and quotas.** No documented per-app request budget, no backpressure signal from the router. `.batch_query()` referenced in the README but not yet on the runtime client.
- **Version history on DeployedApp.** As above — rollback works via ArgoCD but there is no image-tag ledger surfaced through the CLI.
- **Golden-authoring from the SDK.** New goldens must be authored through the dashboard wrapper agent. Programmatic authoring would let ETL agents materialize their own vetted query templates.
