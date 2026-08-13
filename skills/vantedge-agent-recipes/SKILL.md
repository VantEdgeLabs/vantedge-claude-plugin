---
name: vantedge-agent-recipes
description: Concrete, runnable templates for the four canonical VantEdge agent shapes — analyst (on-demand NL query), scheduled ETL (recurring background), alerter (poll + notify), and dashboard-backed (--web UI over a shared store). Use this when a user asks you to build an agent that matches one of these patterns, or when you need a starter you can adapt.
---

# vantedge agent recipes

Four canonical shapes cover ~90% of what teams build on VantEdge. Each recipe below is complete: `vantedge.yaml`, `app.py`, deploy command, what it demonstrates, and how to vary it.

**Before writing any recipe from scratch:** call `vantedge-cli sources` to see what connectors the workspace has, then `vantedge-cli schema <source>` for anything you plan to query. Guessing schemas produces broken agents.

**All recipes assume you've already run** `vantedge-cli init <name>` to get a scaffolded directory. These files replace or augment what the scaffold produces.

## Recipe 1 — Analyst agent

**Shape:** takes a natural-language question via input, uses `context_router.ask()` to get an answer, returns a structured result. On-demand — no schedule.

**When to use:** the user wants an agent they can invoke ad-hoc via `vantedge-cli start` or from Slack/Notion via URL trigger. Good for "ask about X" workflows where the exact question varies.

### `vantedge.yaml`

```yaml
name: analyst
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.3.0
module: app
web:
  enabled: false
triggers: []
data_sources: []   # implicit — uses whatever the workspace has
```

### `app.py`

```python
"""Analyst agent — asks the workspace's data a natural-language question and returns
a structured answer."""

from datetime import timedelta
from temporalio import activity, workflow


@activity.defn
async def answer_question(question: str) -> dict:
    from vantedge.tools import context_router, gateway

    # Hit the workspace data plane with NL — router picks tables, uses goldens if any match
    result = await context_router.ask(question)
    rows = result.get("rows", [])

    # Have the LLM summarize the rows in prose the caller can use
    summary_response = await gateway.llm(
        prompt=(
            "You are a data analyst. Summarize these query results in 2-3 sentences "
            "for a business audience. Do not invent numbers.\n\n"
            f"Question: {question}\n\n"
            f"Rows ({len(rows)}):\n{rows[:20]}"    # cap at 20 for the LLM context
        )
    )
    summary = summary_response["text"]

    # Return shape follows the platform's delivery convention (see sdk-runtime.md):
    # `html` wins for rich rendering, `subject` sets the email subject, `data`
    # renders as an inline table beneath the body when email is configured.
    html = (
        f"<h2 style='font:600 18px -apple-system,sans-serif;margin:0 0 12px'>"
        f"{question}</h2>"
        f"<p style='font:14px/1.5 -apple-system,sans-serif;color:#333'>{summary}</p>"
        f"<p style='font:12px -apple-system,sans-serif;color:#888'>"
        f"{len(rows)} row{'s' if len(rows) != 1 else ''} returned.</p>"
    )
    return {
        "html": html,
        "subject": f"Analyst answer · {question[:60]}",
        "data": rows[:100],   # first 100 rows table inline in email
    }


@workflow.defn
class AnalystWorkflow:
    @workflow.run
    async def run(self, question: str) -> dict:
        return await workflow.execute_activity(
            answer_question,
            question,
            start_to_close_timeout=timedelta(seconds=120),
        )


WORKFLOWS = [AnalystWorkflow]
ACTIVITIES = [answer_question]
```

### Deploy + invoke

```bash
vantedge-cli deploy analyst --build

# Invoke with a question
vantedge-cli start analyst \
  --workflow AnalystWorkflow \
  --input '"which customers had the highest total spend last month?"'
```

### What this demonstrates

- The minimum viable agent: one workflow, one activity, one LLM call
- Using `.ask()` for NL query (as opposed to hand-writing SQL)
- Combining `context_router.ask` + `gateway.llm` for a "structured NL analyst" pattern
- Returning a JSON-serializable dict from a workflow

### Variations

- **Add typed input** — accept a Pydantic model instead of a raw string for validation
- **Add cost cap** — reject questions where `.ask` returns more than N rows (protect the LLM context)
- **Fan out** — take a list of questions and run them in parallel via `asyncio.gather`

---

## Recipe 2 — Scheduled ETL agent

**Shape:** runs on a schedule (e.g., hourly). Reads from one connector, transforms, writes back to another. Background — no user interaction.

**When to use:** classic ETL — enriching a table nightly, syncing between two SaaS systems, computing metrics into an aggregation table. The most common shape for internal automations.

### `vantedge.yaml`

```yaml
name: order-enrichment
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.3.0
module: app
web:
  enabled: false
triggers: []
data_sources: []
```

### `app.py`

```python
"""Order enrichment ETL — hourly, pulls new orders from postgres, joins with HubSpot
customer data, writes an enriched view back to postgres."""

from datetime import timedelta
from temporalio import activity, workflow
from vantedge.runtime.schedules import ScheduledWorkflow


@activity.defn
async def enrich_new_orders() -> dict:
    from vantedge.tools import context_router

    # 1. Read new orders (last hour's window)
    orders = await context_router.query(
        "SELECT id, customer_id, total, created_at FROM orders "
        "WHERE created_at > NOW() - INTERVAL '1 hour' "
        "  AND id NOT IN (SELECT order_id FROM enriched_orders)"
    )
    if not orders["rows"]:
        return {"processed": 0}

    # 2. Fetch matching customers from HubSpot
    customer_ids = [row["customer_id"] for row in orders["rows"]]
    hubspot_result = await context_router.query(
        f"SELECT id, email, lifecycle_stage, plan_tier "
        f"FROM hubspot.contacts WHERE id = ANY(ARRAY{customer_ids!r})",
        data_sources=["hubspot"],
    )
    hs_by_id = {row["id"]: row for row in hubspot_result["rows"]}

    # 3. Compute enrichment + upsert
    processed = 0
    for order in orders["rows"]:
        contact = hs_by_id.get(order["customer_id"])
        if not contact:
            continue

        await context_router.execute(
            "INSERT INTO enriched_orders (order_id, customer_email, lifecycle_stage, "
            "plan_tier, total, enriched_at) "
            "VALUES ($1, $2, $3, $4, $5, NOW()) "
            "ON CONFLICT (order_id) DO NOTHING",
            data_sources=["postgres"],
        )
        processed += 1

    # Return shape follows the platform's delivery convention (see sdk-runtime.md).
    # `briefing` becomes the email body when email delivery is configured;
    # the counts are still visible in the dashboard's raw output view.
    return {
        "briefing": (
            f"Enriched {processed} of {len(orders['rows'])} new orders in the last hour."
        ),
        "processed": processed,
        "total_new_orders": len(orders["rows"]),
    }


@workflow.defn
class OrderEnrichmentWorkflow:
    @workflow.run
    async def run(self) -> dict:
        return await workflow.execute_activity(
            enrich_new_orders,
            start_to_close_timeout=timedelta(minutes=5),
        )


WORKFLOWS = [OrderEnrichmentWorkflow]
ACTIVITIES = [enrich_new_orders]

SCHEDULES = [
    ScheduledWorkflow(
        id="order-enrichment-hourly",
        workflow=OrderEnrichmentWorkflow,
        every="1h",
    ),
]
```

### Deploy

```bash
vantedge-cli deploy order-enrichment --build
# Schedule fires automatically on the hour — no manual `start` needed.
```

### What this demonstrates

- `SCHEDULES` export — declarative scheduling, no external cron
- Cross-connector read (postgres + hubspot in one query)
- Read-transform-write flow with idempotency (`NOT IN (SELECT ...)`, `ON CONFLICT DO NOTHING`)
- Windowed reads (`INTERVAL '1 hour'`)
- Returning a summary dict for observability via `vantedge-cli history`

### Variations

- **Multiple schedules** — declare more than one `ScheduledWorkflow` in the same app
- **Backfill support** — accept a `since: datetime` argument and switch to that window if provided; otherwise use `now - 1h`
- **Dead-letter handling** — activities that fail three times in a row send a summary to Slack via `context_router.action`
- **Larger batch** — for tables where hourly is too slow, split into per-hour parent workflows spawning per-order child workflows

### Gotcha

The schedule fires **every hour on the boundary** by default. If your Temporal cluster has drift or the activity takes >60m, the SKIP overlap policy prevents backfilling — you'll miss the tick. For high-volume ETL, either split into finer sub-workflows or run more frequently with tighter windows.

---

## Recipe 3 — Alerter agent

**Shape:** polls a data source for a condition, notifies a channel when triggered. Combines read + LLM + action-write.

**When to use:** monitoring / anomaly detection. High-value orders. Failed logins. Support tickets sitting too long. New signups. Anything where "if X happens, tell someone."

### `vantedge.yaml`

```yaml
name: high-value-alerter
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.3.0
module: app
web:
  enabled: false
triggers: []
data_sources: []
```

### `app.py`

```python
"""High-value order alerter — every 5 min, checks for new orders over $1000, asks
the LLM to draft context on the customer, posts to Slack."""

from datetime import timedelta
from temporalio import activity, workflow
from vantedge.runtime.schedules import ScheduledWorkflow


@activity.defn
async def check_and_alert() -> dict:
    from vantedge.tools import context_router

    # 1. Look for high-value orders we haven't alerted on yet
    result = await context_router.query(
        "SELECT o.id, o.customer_id, o.total, c.name AS customer_name, c.email "
        "FROM orders o JOIN customers c ON c.id = o.customer_id "
        "WHERE o.total > 1000 "
        "  AND o.created_at > NOW() - INTERVAL '5 minutes' "
        "  AND o.id NOT IN (SELECT order_id FROM alerts_sent)"
    )
    high_value_orders = result.get("rows", [])
    if not high_value_orders:
        return {"alerted": 0}

    alerted = 0
    from vantedge.tools import gateway

    for order in high_value_orders:
        # 2. Ask LLM to draft context for this specific customer
        history = await context_router.query(
            f"SELECT COUNT(*) AS order_count, SUM(total) AS lifetime_value "
            f"FROM orders WHERE customer_id = '{order['customer_id']}'"
        )
        stats = history["rows"][0] if history["rows"] else {}

        draft = await gateway.llm(
            prompt=(
                "Draft a 1-sentence Slack alert for a high-value order.\n\n"
                f"Order: ${order['total']} from {order['customer_name']} ({order['email']})\n"
                f"Customer history: {stats.get('order_count', 0)} orders, "
                f"${stats.get('lifetime_value', 0)} lifetime value.\n\n"
                "Style: neutral, actionable, no emojis."
            )
        )
        text = draft["text"]

        # 3. Post to Slack
        await context_router.action(
            "slack", "send_message",
            channel="#sales-alerts",
            text=text,
        )

        # 4. Mark as alerted (idempotency)
        await context_router.execute(
            f"INSERT INTO alerts_sent (order_id, alerted_at) "
            f"VALUES ('{order['id']}', NOW())"
        )
        alerted += 1

    # Return shape follows the platform's delivery convention (see sdk-runtime.md).
    # `subject` + `html` together yield a targeted alert email when delivery is
    # configured. The counts remain visible on the dashboard for observability.
    plural = "s" if alerted != 1 else ""
    return {
        "subject": f"[ALERT] {alerted} high-value order{plural}",
        "html": (
            f"<p style='font:14px/1.5 -apple-system,sans-serif'>"
            f"Slack was notified for <b>{alerted}</b> of "
            f"<b>{len(high_value_orders)}</b> eligible high-value order{plural}."
            f"</p>"
        ),
        "alerted": alerted,
        "eligible": len(high_value_orders),
    }


@workflow.defn
class HighValueAlerterWorkflow:
    @workflow.run
    async def run(self) -> dict:
        return await workflow.execute_activity(
            check_and_alert,
            start_to_close_timeout=timedelta(minutes=3),
        )


WORKFLOWS = [HighValueAlerterWorkflow]
ACTIVITIES = [check_and_alert]

SCHEDULES = [
    ScheduledWorkflow(
        id="high-value-alerter",
        workflow=HighValueAlerterWorkflow,
        every="5m",
    ),
]
```

### Deploy

```bash
vantedge-cli deploy high-value-alerter --build
# Fires every 5 minutes automatically.
```

### What this demonstrates

- Poll → filter → LLM → notify → mark-done cycle
- Idempotency via a "sent" tracking table (`alerts_sent`)
- Two data-plane reads followed by one LLM call and one connector action
- Windowed polling (`INTERVAL '5 minutes'` matching the schedule cadence)

### Variations

- **Snooze / dedupe by customer** — only alert once per customer per day, not once per order
- **Multi-tier alerts** — different Slack channels based on order size ($1K → #sales-alerts, $10K → #sales-vip)
- **Include a link** — put a URL in the Slack message pointing back to the customer's page in HubSpot (`context_router.action("hubspot", "GetCustomerLink", ...)`)
- **Threshold configuration** — read the threshold from a config table so it can be tuned without redeploying

### Gotcha

The `alerts_sent` table needs to exist. Either:
1. Assume someone provisioned it (document this in the agent's publish `docs` field), OR
2. Have the agent auto-create it on first run via `CREATE TABLE IF NOT EXISTS ...` inside the activity

Option 2 is safer for a shareable template — the agent bootstraps its own state.

---

## Recipe 4 — Dashboard-backed agent (`--web`)

**Shape:** background worker writes results to a shared in-process store on a schedule; a FastAPI web UI reads from that store and renders a live dashboard. One pod, one process, two entrypoints.

**When to use:** any time you want a "live view" of what the agent has produced. Leaderboards, status dashboards, incident logs, health monitors.

### `vantedge.yaml`

```yaml
name: leads-dashboard
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.3.0
module: app
web:
  enabled: true
  module: web:app       # ASGI app path — module:attr
  port: 8080
triggers: []
data_sources: []
```

### `store.py`

Shared in-process sqlite that both the worker (writer) and the web (reader) use:

```python
"""Shared in-process results store — worker writes, web reads."""

import sqlite3

_db = sqlite3.connect(":memory:", check_same_thread=False)
_db.execute(
    "CREATE TABLE IF NOT EXISTS leads ("
    "id TEXT PRIMARY KEY, name TEXT, score REAL, source TEXT, updated_at TEXT)"
)
_db.commit()


def upsert_lead(lead_id: str, name: str, score: float, source: str, updated_at: str) -> None:
    _db.execute(
        "INSERT OR REPLACE INTO leads (id, name, score, source, updated_at) "
        "VALUES (?, ?, ?, ?, ?)",
        (lead_id, name, score, source, updated_at),
    )
    _db.commit()


def all_leads() -> list[dict]:
    cur = _db.execute("SELECT id, name, score, source, updated_at FROM leads "
                      "ORDER BY score DESC")
    cols = [c[0] for c in cur.description]
    return [dict(zip(cols, row)) for row in cur.fetchall()]
```

### `app.py` (worker side — scores leads on schedule)

```python
"""Leads scoring worker — every 15 min, scores new leads via LLM, writes to shared store."""

from datetime import timedelta
from temporalio import activity, workflow
from vantedge.runtime.schedules import ScheduledWorkflow


@activity.defn
async def score_new_leads() -> dict:
    import store    # in-process shared store
    from vantedge.tools import context_router, gateway

    # Pull recent leads that we haven't scored yet
    result = await context_router.query(
        "SELECT id, name, email, company, source, message FROM leads "
        "WHERE created_at > NOW() - INTERVAL '15 minutes'"
    )
    leads = result.get("rows", [])
    scored = 0

    for lead in leads:
        # LLM scores 0-100 based on the lead message
        scoring = await gateway.llm(
            prompt=(
                "Rate this inbound lead's fit for our product on a scale of 0-100. "
                "Return only the number.\n\n"
                f"Name: {lead['name']}\n"
                f"Company: {lead['company']}\n"
                f"Message: {lead['message']}"
            )
        )
        try:
            score = float(scoring["text"].strip())
        except ValueError:
            score = 0.0

        store.upsert_lead(
            lead_id=lead["id"],
            name=lead["name"],
            score=score,
            source=lead["source"],
            updated_at=lead["created_at"] if isinstance(lead.get("created_at"), str)
                       else str(lead.get("created_at", "")),
        )
        scored += 1

    return {"scored": scored, "eligible": len(leads)}


@workflow.defn
class LeadsScoringWorkflow:
    @workflow.run
    async def run(self) -> dict:
        return await workflow.execute_activity(
            score_new_leads,
            start_to_close_timeout=timedelta(minutes=10),
        )


WORKFLOWS = [LeadsScoringWorkflow]
ACTIVITIES = [score_new_leads]

SCHEDULES = [
    ScheduledWorkflow(
        id="leads-scoring",
        workflow=LeadsScoringWorkflow,
        every="15m",
    ),
]
```

### `web.py` (web side — renders the dashboard)

```python
"""Leads dashboard — read-only HTML view over the shared in-proc store."""

from fastapi import FastAPI
from fastapi.responses import HTMLResponse

import store


app = FastAPI()


@app.get("/api/leads")
async def leads_api():
    """JSON endpoint for programmatic access."""
    return store.all_leads()


@app.get("/", response_class=HTMLResponse)
async def index():
    leads = store.all_leads()
    rows = "".join(
        f"<tr>"
        f"<td>{lead['name']}</td>"
        f"<td>{lead['source']}</td>"
        f"<td style='text-align:right; font-family:monospace'>{lead['score']:.1f}</td>"
        f"<td style='color:#888'>{lead['updated_at']}</td>"
        f"</tr>"
        for lead in leads
    )
    body = rows or "<tr><td colspan=4>No leads scored yet.</td></tr>"
    return f"""
    <html>
      <head>
        <title>Lead Scores</title>
        <style>
          body {{ font-family: -apple-system, sans-serif; max-width: 720px;
                  margin: 40px auto; }}
          table {{ width: 100%; border-collapse: collapse; }}
          th, td {{ text-align: left; padding: 8px 12px;
                    border-bottom: 1px solid #eee; }}
        </style>
      </head>
      <body>
        <h1>Lead scores</h1>
        <p>Scored by the leads-scoring agent every 15 min.</p>
        <table>
          <tr><th>Name</th><th>Source</th><th>Score</th><th>Updated</th></tr>
          {body}
        </table>
      </body>
    </html>
    """
```

### Deploy

```bash
vantedge-cli deploy leads-dashboard --build --web
# Web URL: https://dashboard.vantedge.run/apps/3/leads-dashboard/
# Worker fires every 15 min automatically.
```

### What this demonstrates

- Worker + web in a single pod via `vantedge.runtime.app` (base image handles this when `web.enabled: true`)
- Shared in-process sqlite as the boundary between the writer (activity) and reader (FastAPI handler) — no volume, no IPC
- FastAPI serving from `/apps/<workspace>/<name>/` behind Traefik forward-auth (only workspace members can view)
- Combining a scheduled worker with a live web view

### Variations

- **Persist across restarts** — change `sqlite3.connect(":memory:")` to a file path on the app's PVC (backend chart supports per-app PVCs when configured)
- **Auto-refresh** — add `<meta http-equiv="refresh" content="30">` to the HTML to poll every 30 seconds, or use HTMX / server-sent events for smoother updates
- **Multiple views** — the FastAPI app can expose multiple routes (`/`, `/customers`, `/api/*`) all reading from the same shared store
- **Consumer actions from the UI** — add POST endpoints that push a message into a `queue` in the shared store; a separate scheduled workflow drains the queue. Careful with concurrency here — sqlite is single-writer

### Gotchas

- `store.py` must be shared by both `app.py` (worker) and `web.py` (FastAPI). They run in the same process via `vantedge.runtime.app`, so a module-level `sqlite3` connection works. If you factor `store.py` differently or add multiprocessing, this breaks.
- The FastAPI app is only reachable while the pod is up. Sleep + Kubernetes will restart it, but any `:memory:` state is lost on restart.
- Traefik forward-auth means the URL requires a valid Clerk session cookie from `.vantedge.run`. If you want unauthenticated public access, the sandbox and network policy prevent it — this is by design (agent apps are internal-facing).

---

## Recipe — declaring `data_sources` in the manifest

> **`data_sources` means two different things depending on where you put it.**
> In `vantedge.yaml` it's a list of **canonical provider names** (matches
> `vantedge-cli connectors list`). In `context_router.query(sql, data_sources=[...])`
> it's a list of **connector instance names** in the caller's workspace (what
> the user named the connector — often different from the provider name).

Recipes so far leave `data_sources: []` in `vantedge.yaml`. For an app you
plan to **publish** to teammates, declare what the app actually requires so
the platform's subscribe/preflight can check the subscriber's workspace has
matching connectors:

```yaml
name: outlook-triage
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.3.0
module: app
data_sources: [office365]        # canonical provider name
```

```python
# app.py — the actual query uses the instance name
result = await context_router.query(
    "SELECT ... FROM users_outlook_email_messages",
    data_sources=["office"],     # instance name — what the user called it
)
```

If the user hasn't set up an `office365` connector yet, the subscribe
preflight returns `missing_connectors: [{provider: "office365", ...}]` and the
dashboard deep-links them into the connector wizard.

## Recipe — adding a connector when you don't know the canonical name

When a user says "hook this up to my Outlook" or "connect to our SharePoint,"
the canonical connector type is `office365` — not `outlook`, not `sharepoint`.
Two ways to resolve it:

**Preferred — just try `connectors add` and read the error:**

```bash
vantedge-cli connectors add outlook
# → Unknown connector type: outlook
#
# Supported types:
#   office365     Outlook, OneDrive, SharePoint, Teams
#   slack         Slack workspaces
#   postgres      Postgres databases
#   hubspot       HubSpot CRM
#   ...
```

Pick the canonical name from the descriptions (they mention colloquial
aliases) and retry — no second `list` call needed, the error already has the
full catalog.

**Alternative — list up front:**

```bash
vantedge-cli connectors list
vantedge-cli connectors list --format=json    # machine-parseable
```

The `list` command hits the backend catalog (`GET /api/connector-types/`)
directly, so it reflects newly registered connector types automatically.

## Recipe — publishing an app so teammates can subscribe

An app deploys as private to its creator. Two flips take it further —
**publish** (creator makes it visible to the org) and **subscribe** (a
teammate clones it into their own workspace).

### End-to-end flow

1. **Author + deploy privately, verify it works.**

   ```bash
   vantedge-cli init hubspot-deal-digest
   # ... write app.py ...
   vantedge-cli deploy hubspot-deal-digest --build
   vantedge-cli start hubspot-deal-digest --workflow DealDigestWorkflow
   ```

2. **Declare required connectors in `vantedge.yaml`** (so subscribers'
   workspaces get checked at subscribe time):

   ```yaml
   data_sources: [hubspot]        # canonical provider name
   ```

3. **Publish from the dashboard.** Open the app detail page and click
   **Publish**. Only the creator or a workspace admin sees the button, and
   only on non-clones. Flipping publish sets `is_org_shared=true` on the app
   and it appears in the **Shared** tab on `/dashboard/apps` for the rest of
   the workspace, attributed as "Published by <you>".

   (CLI equivalents `vantedge-cli publish` / `unpublish` write metadata but
   the publish/subscribe surface itself is dashboard-only in v1.)

4. **Teammate subscribes from the Shared tab.** They open the app, click
   **Subscribe**, and the SubscribeModal walks them through:
   - **Preflight** — the backend calls
     `POST /api/apps/{slug}/subscribe/preflight/` and returns
     `{missing_connectors: [...]}` for their workspace. If anything's
     missing, the modal deep-links into the connector wizard with a
     `return_to=/dashboard/apps/{slug}?subscribe=1` param so it re-opens
     the modal after wizard success.
   - **Configure** — they pick a schedule (`schedule_cron`), recipients
     (`recipients[]`), and optionally a `display_name` for their clone.
   - **Confirm** — `POST /api/apps/{slug}/subscribe/` creates a subscriber
     clone (or 409s if connectors are still missing).

5. **Subscribers see the clone as their own app**, attributed "Subscribed
   from {author}". They can unsubscribe via
   `DELETE /api/apps/subscriptions/{uuid}/`.

### Live-sub semantics

Subscriber runs use **the source app's current image**. When the publisher
redeploys, every subscriber gets the new logic on their next scheduled run —
no versioning, no "opt-in to update" prompt. This is a deliberate v1 tradeoff:
publishers own the code, subscribers own the schedule and recipients.

### What subscribers can and can't override

| Owned by publisher (updates via redeploy) | Owned by subscriber (per clone) |
|---|---|
| `app.py` / workflow logic | Schedule (`schedule_cron`) |
| `Dockerfile` / dependencies | Recipients |
| `data_sources` requirement | Display name |
| Base image version | Their own connector instance names |

## Choosing between recipes

Quick decision tree:

- **User invokes with a specific question / input?** → Recipe 1 (analyst)
- **Runs on a cadence, reads and writes data, no UI?** → Recipe 2 (ETL)
- **Runs on a cadence, notifies a channel when something happens?** → Recipe 3 (alerter)
- **Runs on a cadence AND has a live view / dashboard?** → Recipe 4 (dashboard-backed)
- **Runs on a cadence with a chat-style interaction?** → Not covered; interactive apps deferred to v2

## Compose recipes

Real agents often combine two shapes. Common combinations:

- **Analyst + Alerter** — accepts a question via `start`, runs it, and posts the summary to Slack instead of returning it. Just replace the `return` at the end of the workflow with a `context_router.action("slack", "send_message", ...)`.
- **ETL + Alerter** — nightly ETL that also posts a summary of what it did. Add a Slack action at the end of the activity after the enrichment loop.
- **Alerter + Dashboard** — alerter writes findings both to Slack AND to the shared store; dashboard shows a history view. Add `store.upsert(...)` calls alongside the Slack actions.

Composition is cheap because each capability is a `context_router.*` call and the workflow orchestrates them.

## Debugging failed workflows

When a recipe's workflow lands in `FAILED`, don't grep pod stdout blindly.
Use the standard loop:

```bash
vantedge-cli runs --status FAILED --limit 5             # scope
vantedge-cli run <workflow_id>                          # summary
vantedge-cli history <workflow_id>                      # failing event + stack
vantedge-cli logs <app> --durable --since 10m --workflow <workflow_id>
```

The full playbook — how to read `failure.message`, `attempt`, `retry_state`,
`timeout_type`, and when to fix vs escalate — lives in `troubleshooting.md`.
Reach for it whenever an activity throws or a scheduled run silently stops
producing output.

## Where to look next

- Command reference (`vantedge-cli` commands): see `cli.md` skill file
- Runtime SDK reference (`context_router` methods, sandbox rules): see `sdk-runtime.md` skill file
- LLM usage / cost per-app (once an agent is running): see `usage.md` skill file
- Failed-run triage using the always-present `usage` summary: see `diagnose-failed-run-cost.md`
- Triage an unexpected LLM cost spike: see `diagnose-cost-spike.md`
