---
name: vantedge-secrets
description: Reference for the vantedge-cli secrets command family — workspace-scoped credential storage. Store an external DB URL or API token once, reference it from vantedge.yaml, and every deployed app gets it injected as an env var. Use this whenever the user asks about secrets, credentials, passwords, tokens, API keys, database URLs, connection strings, or "how do I stop baking creds into my Dockerfile."
---

# vantedge-cli secrets — workspace-scoped credential management

`vantedge-cli secrets` stores credentials at the workspace level and injects them into deployed apps as environment variables. This replaces the pre-Phase-6 pattern of baking credentials into a Dockerfile or a committed `.env`, which leaks in git history and can't be rotated.

**Write-only surface.** Once a value is stored, the platform NEVER returns it — not through `show`, not through `list`, not through any read endpoint. This matches the AWS Secrets Manager console model. If you lose a value, delete and recreate. Value reveal is deferred to a later phase (behind a role check + audit trail).

## The one-paragraph mental model

1. `vantedge-cli secrets set` uploads a value to the workspace.
2. Your `vantedge.yaml` declares which secrets the app needs and what env-var name to bind each one to.
3. On `vantedge-cli deploy`, the platform materializes a Kubernetes Secret in the app's namespace and mounts it via `envFrom`.
4. Your app code reads `os.environ["DB_URL"]` (or whatever env-var name you chose) — no runtime SDK call needed.

## Commands — quick reference

| Command | Purpose |
|---|---|
| `secrets set <name> --value <val> \| --from-stdin \| --from-env VAR` | Create or overwrite a secret |
| `secrets list` | List all secrets in the workspace (metadata only) |
| `secrets show <name>` | Show metadata for one secret (value is never returned) |
| `secrets delete <name> [--force]` | Delete a secret; `--force` bypasses "in use by apps" guard |
| `secrets in-use <name>` | List which apps bind this secret before you mutate it |

**Default format is JSON** on every subcommand (machine-parseable so drivers can pipe through `jq`). Add `--format table` for a human-readable summary.

## `secrets set` — create or overwrite

Three ways to pass the value, ranked by leak risk:

```bash
# Safest — no shell history entry, no env-var visible via `ps -eE`.
echo -n "$DB_URL" | vantedge-cli secrets set prod-db-url --from-stdin --description "Prod Postgres"

# Second-best — reads from a local env var. VAR itself is briefly visible
# via `ps -eE` on shared systems.
export MY_DB_URL="postgres://..."
vantedge-cli secrets set prod-db-url --from-env MY_DB_URL

# Convenience — direct --value flag. WARNING: value lands in your shell
# history file (~/.bash_history / ~/.zsh_history). Fine on your laptop;
# avoid on shared machines and in CI logs.
vantedge-cli secrets set prod-db-url --value "postgres://..."
```

**Overwrite behavior.** If a secret with the same name already exists, `set` prompts for confirmation before overwriting:

```
Overwrite existing 'prod-db-url'? [y/N]
```

Pass `--yes` to skip the prompt (safe when scripting or when the driver knows the secret should be replaced). The backend upserts on `name` — a second `set` becomes the new current value.

**MVP consequence** — updating a secret does NOT auto-patch already-deployed apps. Users must re-run `vantedge-cli deploy <app>` to pick up the new value. Auto-rotation with rolling restart is a later phase.

### Name rules

- Lowercase alphanumeric plus hyphens (`^[a-z][a-z0-9-]*$`)
- Reserved prefixes rejected server-side: `VANTEDGE_`, `TEMPORAL_`, `K8S_` — these are platform-injected env vars and a user secret with the same name would silently clobber them
- Value size cap: 4KB (server-side)
- Workspace cap: 200 secrets

## `secrets list` — the workspace catalog

```bash
vantedge-cli secrets list --format json
```

Returns metadata rows — never a value. Response shape:

```json
[
  {
    "name": "prod-db-url",
    "description": "Prod Postgres — read-replica",
    "updated_at": "2026-08-09T14:22:00Z",
    "created_by": "alex@company.com",
    "in_use_by": ["email-triage", "morning-briefing"]
  }
]
```

`in_use_by` is the chip list for the dashboard — flat list of app slugs that declare this secret in their `vantedge.yaml`. Apps deployed BEFORE Phase 6 shipped will show empty `in_use_by` even if they reference a manually-created K8s Secret; the platform tracks only `vantedge.yaml`-declared bindings.

## `secrets show <name>` — metadata for one secret

```bash
vantedge-cli secrets show prod-db-url --format json
```

Response shape (identical to a single row from `list`):

```json
{
  "name": "prod-db-url",
  "description": "Prod Postgres — read-replica",
  "updated_at": "2026-08-09T14:22:00Z",
  "created_by": "alex@company.com",
  "in_use_by": ["email-triage", "morning-briefing"]
}
```

**No value.** Ever. This is the AWS Secrets Manager console model — you can see the secret exists, but not read it back. If you lose the value, delete and recreate.

## `secrets in-use <name>` — pre-mutation blast-radius check

Before you delete or rotate a secret, ask which apps bind it and what env-var name each app uses:

```bash
vantedge-cli secrets in-use prod-db-url --format json
```

Response shape:

```json
{
  "in_use_by": [
    {"app_slug": "email-triage",    "env_var_name": "DB_URL",     "bound_at": "2026-08-01T09:14:00Z"},
    {"app_slug": "morning-briefing","env_var_name": "POSTGRES_URL","bound_at": "2026-08-05T12:00:00Z"}
  ]
}
```

Use this before `delete --force` or when a driver is about to `set` a new value — knowing which apps will need `vantedge-cli deploy` afterwards is the whole point.

## `secrets delete <name>` — blocks on bindings by default

Without `--force`, the backend refuses to delete a secret that is bound to at least one app and returns a 409 with the blast radius:

```
✗ prod-db-url is bound to 2 app(s): email-triage, morning-briefing
Deleting requires --force. Those apps keep the old value until redeployed.
```

With `--force`, the CLI still prompts for a type-to-confirm on the secret name (unless `--yes`):

```
Type 'prod-db-url' to confirm delete:
```

After a forced delete, bound apps continue running with the old value cached in their K8s Secret until their next `vantedge-cli deploy`. To fully rotate:

1. `vantedge-cli secrets in-use <name>` — get the app list
2. `vantedge-cli secrets delete <name> --force`
3. `vantedge-cli secrets set <name> --from-stdin` (with the new value)
4. `vantedge-cli deploy <app>` for each app in step 1

## `vantedge.yaml` — declaring which secrets an app needs

Add a `secrets:` block to the manifest:

```yaml
name: email-triage
base_image: 997334016349.dkr.ecr.us-east-1.amazonaws.com/vantedge-app-base:0.3.0
module: app

secrets:
  - name: prod-db-url         # matches the name passed to `secrets set`
    env: DB_URL               # env var your app.py reads via os.environ
  - name: hubspot-api-key
    env: HUBSPOT_TOKEN
```

Each entry binds one workspace secret to one env-var name inside the app container. On `vantedge-cli deploy`, the platform:

1. Reads the manifest's `secrets:` block
2. Fetches the value for each named secret from the workspace catalog
3. Writes a K8s Secret in the app's namespace
4. Appends `envFrom: - secretRef: <slug>-secrets` to the Deployment
5. Records an `AppSecretBinding` row so `in_use_by` reflects the deploy

Then your app just reads `os.environ["DB_URL"]`. No SDK call. No runtime fetch.

## `--format json` — response shapes

Every subcommand emits JSON by default so an AI driver can `jq` its way through the response without regex-parsing a table.

| Subcommand | JSON shape |
|---|---|
| `set` | `{name, description, updated_at, created_by, in_use_by: []}` — value never present |
| `list` | `[{name, description, updated_at, created_by, in_use_by: [app_slug, ...]}]` |
| `show <name>` | Single row identical to `list`'s element shape |
| `delete <name>` | `{deleted: true, name}` on success; `{deleted: false, name, blocked_by: [app_slug, ...]}` on 409 |
| `in-use <name>` | `{in_use_by: [{app_slug, env_var_name, bound_at}, ...]}` |

## Common flows

### First-time setup — replace a baked-in credential

```bash
# 1. Store the value (safest path — no shell history).
echo -n "postgres://user:pass@host:5432/db" | vantedge-cli secrets set prod-db-url --from-stdin

# 2. Reference it in vantedge.yaml:
#    secrets:
#      - name: prod-db-url
#        env: DB_URL

# 3. Delete the hardcoded value from your Dockerfile / .env.

# 4. Redeploy.
vantedge-cli deploy email-triage

# 5. Verify the app is reading from the new source.
vantedge-cli logs email-triage --tail 50
```

### Rotate a leaked credential

```bash
# See who will be affected.
vantedge-cli secrets in-use hubspot-api-key --format json

# Overwrite with the new value (--yes skips the confirm prompt).
echo -n "$NEW_KEY" | vantedge-cli secrets set hubspot-api-key --from-stdin --yes

# Redeploy every binder to pick up the new value.
for app in $(vantedge-cli secrets in-use hubspot-api-key --format json | jq -r '.in_use_by[].app_slug'); do
  vantedge-cli deploy "$app"
done
```

### Clean up an unused secret

```bash
# Confirm no apps depend on it.
vantedge-cli secrets in-use old-token --format json
# → {"in_use_by": []}

vantedge-cli secrets delete old-token
```

## Gotchas

- **Value cannot be retrieved.** No `show --reveal`. No `get`. If you lose it, `set` a new one and redeploy every binder.
- **Overwrite is silent to deployed apps.** `set` on an existing name updates the workspace catalog but does NOT patch already-deployed pods. You must `vantedge-cli deploy <app>` for the new value to take effect. Auto-rotation is a later phase.
- **Reserved prefixes.** Secrets named `VANTEDGE_*`, `TEMPORAL_*`, or `K8S_*` are rejected — those namespaces belong to the platform.
- **Empty values refused.** `secrets set foo --value ""` is treated as a footgun and rejected. If you actually want an empty value, that's almost certainly a bug in the caller.
- **Trailing newlines.** `echo "value" | secrets set ... --from-stdin` — the CLI strips ONE trailing newline. `echo -n "value"` is safer for tokens where any newline breaks the consumer.
- **Ghost bindings on pre-Phase-6 apps.** Apps deployed before Phase 6 shipped have zero `AppSecretBinding` rows. Their entries on the dashboard show "In use by: nothing" even if a manually-configured K8s Secret exists. Redeploy the app to pick up tracking.
- **Concurrent deploys.** Two `deploy` runs on the same app race on the K8s Secret write. Last write wins; acceptable for MVP.

## What secrets are NOT

- **Not** a runtime SDK — `await vantedge.secrets.get("name")` doesn't exist yet. Access secrets via env vars only. Runtime fetch is a later phase.
- **Not** org-scoped — every secret belongs to exactly one workspace. Cross-workspace sharing is a later phase.
- **Not** an audit log — access history isn't recorded yet. That's a later phase.
- **Not** for platform secrets — Resend, Clerk, etc. go through AWS Secrets Manager + External Secrets Operator, not this surface.

## Where to look next

- Full `vantedge-cli` reference — `cli.md`
- Runtime SDK reference (workflow patterns, `gateway.llm`, etc.) — `sdk-runtime.md`
- Troubleshooting a failing app (env var missing, secret not injected) — `troubleshooting.md`
