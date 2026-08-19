---
name: vantedge-cli
description: Reference for the vantedge-cli command-line tool — auth, scaffolding, deploying, publishing, and observing agent apps on the VantEdge platform. Use this whenever the user asks you to build, deploy, iterate on, or inspect a VantEdge agent app.
---

# vantedge-cli — command reference

`vantedge-cli` is the control-plane tool for the VantEdge Agent App Platform. It authenticates against `dashboard.vantedge.run`, scaffolds new agent apps, builds and deploys them to Kubernetes-hosted Temporal workers, and lets you observe / control runs.

**Auth model:** the CLI holds a `sk_live_*` API key stored at `~/.vantedge/credentials` (mode 0600). Every command uses this to talk to the backend. Deployed apps DO NOT use this key — they get a workspace-scoped `wst_*` token injected at deploy time.

**Everything else** (Temporal, Kubernetes, workspace tokens, LLM keys) is invisible to the user. If a user asks about them, redirect to what they actually want to accomplish.

## Installation

```bash
pip install vantedge-cli
vantedge-cli --version
```

## Environment variables

| Variable | Purpose | When to set |
|---|---|---|
| `VANTEDGE_API_KEY` | Override the stored API key | Rarely — mostly for CI. `login` writes this to the credentials file. |
| `VANTEDGE_PLATFORM_URL` | Override backend base URL | Local minikube dev: `https://minikube-api.vantedge.run`. Prod: unset (default). |
| `VANTEDGE_TLS_CA_BUNDLE` | Path to CA bundle | Local mkcert-signed backends only: `$(mkcert -CAROOT)/rootCA.pem` |
| `VANTEDGE_BASE_IMAGE` | Override the base runtime image the app builds FROM | Rarely — CI or when pinning to a specific base version |

## Commands — quick reference

| Command | Purpose |
|---|---|
| `login` | Browser-based auth, mints an API key, stores it locally |
| `logout` | Clear stored credentials |
| `whoami` | Show current user + default workspace |
| `init <name> [--web]` | Scaffold a new agent app directory |
| `deploy <name>` | Build (server-side) + deploy an app. Web UI / env vars / secrets are declared in `vantedge.yaml`, not flags. |
| `apps` | List apps in your workspace (yours + shared) |
| `sources` | List connected data sources |
| `schema <source> [table]` | Fetch column-level schema for one or all tables |
| `connectors add <type>` | Add a new data source (opens browser) |
| `connectors list [--format=json]` | List supported connector types with descriptions |
| `publish <name> [--description] [--tags] [--docs @file.md]` | Share app to workspace catalog |
| `unpublish <name>` | Make a shared app private again |
| `start <app> --workflow <Type> [--input JSON] [--id ID]` | Trigger a run manually |
| `runs [--app] [--status] [--limit N]` | List recent workflow runs |
| `run <workflow_id>` | Describe a specific run |
| `history <workflow_id>` | Show a run's event history |
| `logs <app> [--tail N] [--durable] [--since T] [--level L] [--workflow WF] [--limit N]` | Print worker pod stdout, or durable AppLog history |
| `terminate <workflow_id> [--reason]` | Force-stop a run |
| `cancel <workflow_id>` | Gracefully cancel a run |
| `action <source> <action> [--arg K=V]` | Invoke a connector stored-procedure action |
| `usage [--app \| --workspace \| --org] [--since T] [--group-by X]` | Aggregate LLM spend for one scope. Default format: JSON. See `usage.md`. |
| `usage top [--window T] [--by cost\|tokens\|errors] [--limit N]` | Cost-spike leaderboard for the caller's org. Default format: JSON. |
| `run <workflow_id> [--llm-calls]` | Run detail with a top-level `usage` summary (always present). `--llm-calls` adds per-call rows. |
| `secrets set <name> --value \| --from-stdin \| --from-env VAR [--description] [--yes]` | Create or overwrite a workspace secret. Write-only — value is never echoed after write. See `secrets.md`. |
| `secrets list` | List all secrets in the workspace (metadata only). |
| `secrets show <name>` | Show metadata for one secret (value is NEVER returned — matches AWS SM console). |
| `secrets delete <name> [--force] [--yes]` | Delete a secret; `--force` bypasses the "in use by apps" guard. |
| `secrets in-use <name>` | List which apps bind this secret before you mutate it. |
| `models tiers` | List the sdk-agents profile's tier → chain mapping. |
| `models tier <name>` | Show one tier's mapping (`low`, `medium`, or `high`). |
| `models list` | List all chains registered on the gateway. |

## Authentication

### First-time login

```bash
vantedge-cli login
```

Opens a browser to `dashboard.vantedge.run/cli-callback?port=<random>`. The user clicks "Authorize" while signed into their Clerk session; the backend mints a new API key and POSTs it back to `localhost:<port>`. The CLI stores it at `~/.vantedge/credentials` (mode 0600) and prints `✓ Logged in as <email>`.

**If the browser doesn't open** (headless SSH, remote dev): the CLI prints a URL + one-time code and polls until the user completes the flow in a browser elsewhere.

### Check status

```bash
vantedge-cli whoami
# → alex@company.com · CompanyName
```

### Sign out

```bash
vantedge-cli logout
# → Credentials cleared from ~/.vantedge/credentials
```

## Scaffolding a new agent app

### `init` — creates a directory with 7 (or 9) files

```bash
vantedge-cli init slack-digest
# → Scaffolded ./slack-digest/ with vantedge.yaml, app.py, Dockerfile, ...

vantedge-cli init slack-digest --web
# same + web.py + store.py for a --web UI app
```

The scaffold gives you:

| File | Purpose |
|---|---|
| `vantedge.yaml` | The app manifest — name, base_image, entry module, web config |
| `app.py` | Stub with `@workflow.defn` + `@activity.defn` + `WORKFLOWS`/`ACTIVITIES` exports |
| `Dockerfile` | `FROM {base_image}` — builds on top of vantedge-app-base (v0.5.5 default, includes SDK ≥0.6.0 with attribution v1 headers) |
| `requirements.txt` | `vantedge[runtime]` (+ `web` if --web) |
| `.env.example` | Documents the env vars for local dev |
| `.dockerignore`, `README.md` | Standard hygiene |
| `web.py`, `store.py` | Only with `--web` — FastAPI + shared in-proc sqlite |

**After `init`, next steps:** describe intent to Claude Code, deploy with `vantedge-cli deploy <name>` (server-side build; no local Docker needed).

## Configuring output delivery (manifest `outputs:` block)

Agent apps can declare that their run output is delivered via email. Delivery
is a **property of the agent**, not a per-run choice — every run (manual,
scheduled, webhook-triggered) fires through the same delivery path.

Add an `outputs:` block to `vantedge.yaml`:

```yaml
name: hubspot-deal-digest
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.5.5
module: app
web:
  enabled: false
triggers: []
data_sources: [hubspot]        # canonical provider names — see agent-recipes.md
egress_allowlist: []           # outbound public domains this app may reach; empty = no egress

outputs:
  email:
    enabled: true
    recipients:
      - "sales-lead@company.com"
      - "revops@company.com"
    sender: deals                       # optional; default = agents@vantedge.run
    subject: "HubSpot deal digest · {date}"   # optional; supports {date} + {app_name}
```

Fields:

| Field | Required | Purpose |
|---|:---:|---|
| `enabled` | yes | Master toggle. `false` = no email even if recipients are configured. |
| `recipients` | yes when `enabled: true` | List of email addresses. Defaults to `[creator_email]` on first deploy if the manifest omits it. |
| `sender` | no | Local part of the from-address (`<sender>@vantedge.run`). Empty = platform default `agents@`. Must be unique per workspace. |
| `subject` | no | Subject template. Supports `{date}` (run completion date, ISO) and `{app_name}` placeholders. Default: `<app_name> · <date>`. |

**Behavior:**

- **Every run gets emailed.** No per-run opt-in — declaring `enabled: true` means all future runs deliver.
- **Manifest deploys overwrite UI edits.** The frontend settings panel can also edit this config; last-write-wins. Redeploy with a stale manifest and any UI edits get stomped (documented in the deploy output).
- **The agent's return value drives the email body.** No SDK helper functions — the platform reads the workflow's return dict and renders it. See `sdk-runtime.md` for the return-shape convention (key priority: `html` → `markdown` → `briefing`/`text`/etc.).
- **Rate limits apply.** 10 emails/hour per user, 50/hour per app, 500/day per workspace. Exceeding the cap logs a skipped-delivery entry visible on the run detail page.

**Removing email delivery:** delete the `outputs:` block (or set `enabled: false`) and redeploy. Recipients are preserved in the backend so re-enabling is one flip.

## Deploying an app

### `deploy` — build (server-side) + provision

```bash
# The common case: server-side build, no local Docker or registry creds needed
vantedge-cli deploy slack-digest

# Same, with jargon-free progress OFF (raw docker logs + full JSON response)
vantedge-cli deploy slack-digest --verbose

# App needs public HTTPS (Twilio, Stripe, etc.) — LLM calls DON'T need this
vantedge-cli deploy slack-digest --allow-internet

# Deploy an already-built image (skip build entirely)
vantedge-cli deploy slack-digest --image myregistry.example.com/slack-digest:v1.2

# Escape hatch: build LOCALLY (needs Docker + registry creds). Rare — CI, offline
vantedge-cli deploy slack-digest --local-build --image myregistry.example.com/slack-digest:v1.2
```

**Web UI, env vars, secrets bindings, egress allowlist, data sources, output delivery — all declared in `vantedge.yaml`, not CLI flags.** See the manifest reference sections below and the schema in `vantedge-cli init`'s scaffolded file.

**Flags:**

| Flag | Purpose |
|---|---|
| _(no flag)_ | Default — server-side build. Platform builds + pushes + deploys. No local Docker or AWS/ECR creds needed. |
| `--image <tag>` | Skip build, deploy an already-built image (repo:tag or repo@digest). |
| `--local-build` | Build+push LOCALLY with docker buildx (needs Docker + registry creds). Requires `--image` as the push target. For offline/air-gapped or CI that already has ECR creds. |
| `--allow-internet` | Open public HTTPS egress. **Rarely needed** — LLM calls route through `gateway.llm()`, not directly to Anthropic/OpenAI. Use only for third-party services (Twilio, Stripe, etc.). |
| `--base-image <ref>` | Override the base runtime image the app Dockerfile builds FROM. Falls back to `VANTEDGE_BASE_IMAGE` env var, then the `base_image:` key in `vantedge.yaml`. |
| `--build-context <dir>` | Build context directory (default: `.`). Applies to server-side and `--local-build` alike. |
| `--push-image <tag>` | Image tag to push when `--local-build` is set; defaults to `--image`. Used only when the push endpoint differs from the pull endpoint (e.g. local minikube-registry vs localhost:5000). |
| `--dry-run` | Mint token + build config only; don't provision |
| `--verbose` / `-v` | Show raw docker build logs, image digests, and the full JSON deploy response. Default is jargon-free milestones (`Preparing your app…` / `Building your app…` / `Publishing your app…` / `Getting it ready to run…`). |

**Deploy is idempotent by `(workspace, name)`** — running `deploy` again on the same name updates the existing app in place (image rolled, ArgoCD Application updated). No dupes, no rename.

**Agents deploy as PRIVATE by default.** Only the creator can invoke or see the app. Explicitly publish to share (see `publish` below).

**Output on success (default, server-side, jargon-free):**

```
Building slack-digest on the platform ...
Waiting for a build slot ...
Preparing your app ...
Building your app ...
Publishing your app ...
Getting it ready to run ...
✓ Deployed

Web URL:   https://dashboard.vantedge.run/apps/3/slack-digest/    (if web.enabled in manifest)
Dashboard: https://dashboard.vantedge.run/apps/slack-digest
```

Pass `--verbose` for raw docker build logs, resolved image digest, and the full JSON deploy response.

## Publishing to the workspace catalog

### `publish` — flip private → shared

```bash
# Guided (opens browser to publish page for metadata form)
vantedge-cli publish slack-digest

# Scripted / CI (all metadata via flags)
vantedge-cli publish slack-digest \
  --description "Reads #engineering daily, posts a summary to #eng-digest" \
  --tags reporting,slack,scheduled \
  --docs @README.md
```

Publish attaches three pieces of metadata to the app row:

| Field | Purpose |
|---|---|
| `description` | Required. One-liner shown on the catalog card (≤150 chars) |
| `tags` | Optional, chip-style tags for filtering |
| `docs` | Optional markdown blurb explaining what the app does and how coworkers should use it |

**Only the creator can publish/unpublish their own app.** Other workspace members get a 403.

### `unpublish` — flip shared → private

```bash
vantedge-cli unpublish slack-digest
```

Instant. App disappears from the catalog; existing subscribers no longer see it. Runs continue for the creator.

### Subscribe & Clone (dashboard-only in v1)

Publishing puts an app on the workspace's **Shared** tab; teammates then
**subscribe** to clone it into their own workspace with their own schedule
and recipients. Subscribe is dashboard-only for v1 (no CLI flow yet) —
subscribers open the app on `/dashboard/apps` and click **Subscribe**, which
runs a preflight (checks their workspace has every `data_sources:` provider
the manifest declares), configures schedule + recipients, and creates a
subscriber clone.

Subscriber runs use the source app's **current** image — a publisher redeploy
takes effect for every subscriber on the next scheduled run. See
`agent-recipes.md` → "publishing an app so teammates can subscribe" for the
end-to-end flow.

Relevant backend endpoints (dashboard uses these; document here so drivers
know they exist):

| Method | Path |
|---|---|
| POST | `/api/apps/{slug}/publish/` — creator/admin flips `is_org_shared=true` |
| POST | `/api/apps/{slug}/unpublish/` |
| POST | `/api/apps/{slug}/subscribe/preflight/` — returns `{missing_connectors: [...]}` |
| POST | `/api/apps/{slug}/subscribe/` — 409s if preflight isn't clean |
| DELETE | `/api/apps/subscriptions/{uuid}/` — unsubscribe |

## Observing runs

### `runs` — list

> For diagnosing failures, see `troubleshooting.md`.

```bash
# All runs in your workspace
vantedge-cli runs

# Filter
vantedge-cli runs --app slack-digest --status FAILED --limit 20
```

Statuses: `RUNNING`, `COMPLETED`, `FAILED`, `TERMINATED`, `CANCELED`, `TIMEDOUT`.

### `run` — describe one

```bash
vantedge-cli run slack-digest-abc123-def456
```

Prints workflow id, run id, task queue, status, start time, close time, duration.

### `history` — full event history

> Diagnosing a failed run? See `troubleshooting.md`.

```bash
vantedge-cli history slack-digest-abc123-def456
```

Every Temporal event on the run — good for debugging state transitions.

### `logs` — pod stdout or durable history

> For the full log-based debugging loop, see `troubleshooting.md`.

Two modes: live pod stdout (default) or the durable AppLog history (`--durable`).

| Mode | Source | Survives pod restart | Filters |
|---|---|---|---|
| default | worker pod stdout | no | `--tail` |
| `--durable` | AppLog history endpoint | yes | `--since` `--level` `--workflow` `--limit` |

Warning: plain `logs` returns nothing useful if the pod restarted after the failure — the stdout ring buffer went with it. Reach for `--durable` whenever the pod might have been recycled, or you need to look back further than what stdout holds.

```bash
vantedge-cli logs slack-digest                       # live stdout, all pods
vantedge-cli logs slack-digest --tail 200            # last 200 lines per pod

vantedge-cli logs slack-digest --durable                          # recent durable rows
vantedge-cli logs slack-digest --durable --since 1h               # last hour
vantedge-cli logs slack-digest --durable --since 2026-08-06T14:00:00Z
vantedge-cli logs slack-digest --durable --level ERROR            # errors only
vantedge-cli logs slack-digest --durable --workflow slack-digest-1-abc123
vantedge-cli logs slack-digest --durable --level ERROR --limit 50
```

Durable output is one row per line: `<ts> [<level>] <logger> :: <message>`. If the row carries a `context.stack_trace`, it prints indented under the message. If the row has a `context.workflow_id` and you didn't pass `--workflow`, the workflow id and attempt are appended inline so you can grep for a specific attempt.

`--since` accepts a relative duration (`30s`, `5m`, `1h`, `1d`) or an ISO 8601 timestamp — server-side parsing, pass the string through. `--level` picks a minimum severity from DEBUG/INFO/WARNING/ERROR/CRITICAL. `--workflow <wf_id>` scopes to a single workflow attempt. Flags other than `--tail` are silently ignored (with a warning) in live-pod mode.

## Starting a run manually

```bash
# Just start it
vantedge-cli start slack-digest --workflow SlackDigestWorkflow

# With an input payload
vantedge-cli start slack-digest \
  --workflow SlackDigestWorkflow \
  --input '{"channel":"engineering","lookback_hours":24}'

# With a specific run ID (for idempotency in CI)
vantedge-cli start slack-digest \
  --workflow SlackDigestWorkflow \
  --id daily-2026-07-28
```

**When to use `start`:** on-demand invocation, smoke-testing after deploy, external cron systems calling the CLI. Scheduled workflows (declared via `SCHEDULES` in `app.py`) fire automatically — no `start` needed.

## Controlling runs

```bash
vantedge-cli terminate slack-digest-abc123-def456 --reason "user request"
vantedge-cli cancel slack-digest-abc123-def456
```

**Difference:**

- `terminate` — hard stop. Activity in-flight is killed. Use for stuck / runaway runs.
- `cancel` — graceful. Workflow gets a cancellation signal; workflow code decides whether to honor it. Use when the workflow has cleanup logic.

## Data source discovery

### `sources` — list connected data sources

```bash
vantedge-cli sources
```

Returns each connector's name, type, connection status, and tables count.

### `schema` — inspect table structure

```bash
# All tables in a connector
vantedge-cli schema postgres

# One specific table
vantedge-cli schema postgres orders
```

Returns column names + types + nullable + primary keys + foreign keys. **Use this before writing SQL** — it's the fastest way to discover what tables and columns exist.

### `connectors add` — add a new data source

```bash
vantedge-cli connectors add slack
vantedge-cli connectors add postgres
vantedge-cli connectors add office365
```

Opens `dashboard.vantedge.run/connectors/new?type=<t>&return_to=cli` in the browser. The user completes OAuth / config there; the CLI polls until the connector appears, then confirms.

`<type>` must be a canonical connector type (e.g. `office365`, not `outlook`; `drive`, not `gdrive`). If you pass an unknown type, the CLI prints the full supported-types catalog inline — canonical name plus description — and exits non-zero. **Read the descriptions and retry in the same turn**; the error output already contains everything `connectors list` would show, so don't call `connectors list` separately when recovering from this error.

**Connector setup is a one-time, workspace-level action.** Once added, every agent every teammate ships gets access to that connector.

### `connectors list` — browse supported types

```bash
vantedge-cli connectors list
vantedge-cli connectors list --format=json
```

Prints the canonical name + description for every supported connector type. Use this when you don't yet know which `--type` to pass to `connectors add`. When recovering from an "unknown connector type" error, you don't need this — the error output already includes the same list.

## Running connector actions

Some connectors expose stored-procedure-style actions (e.g., `mark_as_read` on email, `send_message` on Slack):

```bash
vantedge-cli action slack send_message --arg channel=#eng-digest --arg text="hello"
vantedge-cli action outlook mark_as_read --arg message_id=AAMkAD...
```

Usually agents call actions from inside their workflow code (`context_router.action(...)`), but the CLI is useful for one-off runs or testing.

## Idiomatic workflows

### Ship a new agent from scratch

```bash
vantedge-cli init my-agent
cd my-agent

# Discover available data
vantedge-cli sources
vantedge-cli schema postgres

# Now describe intent to Claude Code (or write app.py yourself)
# ...

vantedge-cli deploy my-agent
# ✓ Deployed as PRIVATE

# Smoke test
vantedge-cli start my-agent --workflow MyWorkflow
vantedge-cli logs my-agent --tail 50

# When ready to share with team
vantedge-cli publish my-agent \
  --description "..." \
  --tags "..." \
  --docs @README.md
```

### Iterate on an existing agent

```bash
cd my-agent
# Edit app.py to fix a bug or add a feature
vantedge-cli deploy my-agent               # idempotent redeploy
vantedge-cli start my-agent --workflow MyWorkflow  # smoke test
vantedge-cli logs my-agent --tail 100      # verify
```

### Debug a failed run

```bash
vantedge-cli runs --app my-agent --status FAILED --limit 5
# → list of workflow_ids

vantedge-cli run my-agent-xyz123           # what happened
vantedge-cli history my-agent-xyz123       # full event trail
vantedge-cli logs my-agent --tail 500      # stdout around the failure
```

### Add a new data source and use it in a fresh agent

```bash
vantedge-cli connectors add hubspot         # browser → OAuth → back
vantedge-cli schema hubspot                 # discover tables
vantedge-cli init hubspot-lead-scorer
# ... write workflow ...
vantedge-cli deploy hubspot-lead-scorer
```

## Gotchas

- **`--allow-internet` is rarely needed.** LLM calls should use `gateway.llm(...)` from `vantedge.tools.gateway`, which routes through the platform (no user-managed API keys, workspace-attributed logging). Use `--allow-internet` only for third-party services not covered by a connector (Twilio, Stripe webhooks). Prefer the manifest's `egress_allowlist:` for granular per-domain access instead of opening full public HTTPS.
- **`deploy` overwrites in place.** Redeploying a shared app immediately affects all subscribers. There's no versioning or rollback UI in v1 — bump the image tag manually if you need to revert (`--image myregistry/my-agent:v1.2.3` for the old tag).
- **Directory rename ≠ app rename.** If you `mv my-agent renamed-agent && vantedge-cli deploy renamed-agent`, you'll create a new app named `renamed-agent` while `my-agent` still exists in the workspace. Use `apps` to see what's actually deployed before deploying under a new name.
- **`schema` returns live schemas, not a stale snapshot.** If the underlying data source's schema changes, the next `schema` call reflects it.
- **Env vars go through workspace secrets + manifest binding, not CLI flags.** Store the value once with `secrets set`, then bind it in `vantedge.yaml` under `secrets:` — deploys mount it into the pod. Rotating a secret's value doesn't require re-editing the manifest. (There is no `--env` flag on `deploy` anymore.)
- **Chat-scoped connectors don't show up in `sources`.** Only workspace-level connectors do. If a user uploaded an Excel file to Atlas as a temporary connector, agents can't reach it.

## What CLI users should NOT do

- Don't call Anthropic / OpenAI directly with a bring-your-own-key. Use `gateway.llm()` (`from vantedge.tools import gateway`) instead — the platform handles keys, tracking, and workspace attribution.
- Don't manually construct workspace tokens or API URLs. `login` + `--platform-url` handle all URL/auth config.
- Don't try to modify a connector after it's added via the CLI. Connectors are managed in the browser flow; the CLI only reads and invokes them.
- Don't hand-edit the ArgoCD Application or K8s manifests for a deployed app. Redeploy through the CLI.

## Failure modes and how to diagnose

| Symptom | Likely cause | Fix |
|---|---|---|
| `login` opens browser but callback never returns | Firewall blocked `localhost:<port>` | Use the device-code fallback: run `login --headless` |
| `deploy --local-build` fails with "docker not found" | No local Docker | Drop `--local-build` to use the default server-side path (no Docker needed), or install Docker |
| `deploy` succeeds but pod never becomes ready | Import error in `app.py` or missing dep in `requirements.txt` | `vantedge-cli logs <app> --tail 100` to see the traceback |
| Runs error with `VANTEDGE_API_URL not set` | Backend env var missing (only happens locally when running `python -m app` outside a pod) | Set env vars in a local `.env` file per `.env.example` |
| `sources` returns empty | No connectors set up yet | `vantedge-cli connectors add <type>` |
| Query fails with "column does not exist" | Claude Code guessed schema | `vantedge-cli schema <source> <table>` first, then rewrite SQL |
| App can't reach external service (Twilio, etc.) | Sandbox egress closed by default | Add the domain(s) to `egress_allowlist:` in `vantedge.yaml` and redeploy, or `deploy --allow-internet` to open public HTTPS wholesale |

## LLM usage & cost attribution

Every LLM call an agent makes routes through `gateway.llm()`, which stamps the call with `app_id` + `workflow_id` + `workflow_type` + `activity_attempt` so spend attributes back to the app that made it. Three commands surface that data:

```bash
# Aggregate spend — scope by app / workspace / org.
vantedge-cli usage --app email-triage --since 7d --group-by day
vantedge-cli usage --org --since 30d --group-by app

# Cost-spike leaderboard.
vantedge-cli usage top --window 1h --by cost --limit 10

# Run detail — always carries a `usage` summary; --llm-calls adds per-call rows.
vantedge-cli run <workflow_id> --llm-calls
```

**Key rules:**

- Default `--format` on `usage` / `usage top` is JSON (machine-parseable). `--format table` for humans.
- `run` stays JSON by default — flipping that would break existing consumers.
- `run <wf_id>` output ALWAYS carries a top-level `usage` key with `{llm_calls_count, llm_error_count, total_tokens, cost_usd_estimate, error_kind_summary}` — even when the run made zero LLM calls (the object is zero-shaped). Lets an AI driver take the right diagnostic branch without a second call.
- `--app` / `--workspace` / `--org` are mutually exclusive. When none is passed, `usage` falls back to the caller's default workspace via `whoami`.

Full reference: see `usage.md`. Cost-spike triage recipe: `diagnose-cost-spike.md`. Failed-run triage using the `usage` key: `diagnose-failed-run-cost.md`.

## Workspace secrets — external credentials for deployed apps

Instead of baking a DB URL or API token into your Dockerfile, store it once at the workspace level and reference it from `vantedge.yaml`. On deploy, the platform materializes a K8s Secret in the app's namespace and mounts it via `envFrom` — your app just reads `os.environ["DB_URL"]`.

```bash
# Store a value (safest path — no shell history).
echo -n "postgres://..." | vantedge-cli secrets set prod-db-url --from-stdin

# List what's stored.
vantedge-cli secrets list

# Check which apps bind a secret before rotating.
vantedge-cli secrets in-use prod-db-url
```

Reference it in `vantedge.yaml`:

```yaml
secrets:
  - name: DB_URL              # POSIX env var name inside the pod
    from: prod-db-url         # workspace secret slug (what `secrets set` created)
```

**Write-only.** The platform never returns a secret value after write — no `show --reveal`, no runtime `get`. Lose it, `set` a new one, redeploy binders. Full reference: `secrets.md`.

## Where to look next

- Workspace secrets reference: see `secrets.md`
- LLM usage / cost attribution reference: see `usage.md`
- Runtime SDK reference (`vantedge.tools.context_router`, workflow patterns): see the `sdk-runtime.md` skill file
- Concrete agent recipes (analyst, ETL, alerter, dashboard-backed): see the `agent-recipes.md` skill file
- Runtime developer guide (deeper context): `docs/developer-guide.html` in the CLI repo
