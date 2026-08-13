# VantEdge — Claude Code plugin

Skills and reference material that teach Claude Code how to build, deploy, observe, and troubleshoot **VantEdge agent apps** through the `vantedge-cli`.

The [VantEdge platform](https://vantedge.run) runs your Python agent apps on a managed Temporal + Kubernetes stack. Connect a database once, ship a `vantedge.yaml` + `app.py`, and Claude does the rest.

## Install

```
/plugin marketplace add VantEdgeLabs/vantedge-claude-plugin
/plugin install vantedge
```

Then install the CLI itself (the plugin ships guidance, not the binary):

```
pipx install vantedge-cli
vantedge-cli login
```

## What's in the box

### Skills

| Skill | When Claude loads it |
| --- | --- |
| `vantedge-cli` | Any time you ask to build, deploy, inspect, or iterate on a VantEdge app. |
| `vantedge-sdk-runtime` | Writing `app.py`, workflow/activity classes, schedule declarations. |
| `vantedge-agent-recipes` | Building an analyst, scheduled ETL, alerter, or dashboard-backed agent. |
| `vantedge-secrets` | Storing DB URLs, API tokens, or any credential referenced from `vantedge.yaml`. |
| `vantedge-usage` | Asking about LLM cost, tokens, spend attribution, or "why is our bill high." |
| `vantedge-troubleshooting` | A workflow returned FAILED or an activity threw. |
| `diagnose-cost-spike` | Recipe: triage an unexpected LLM cost spike (`usage top` → drill-down). |
| `diagnose-failed-run-cost` | Recipe: triage a failed workflow via its LLM-side signals. |

### Docs

Reference material Claude will consult when a skill points at it:

- `docs/agent-capabilities.md` — the full capability surface of a VantEdge agent app.
- `docs/developer-guide.html` — end-to-end walkthrough (rendered from the Nextra docs site).

## Prerequisites

The plugin **does not install** `vantedge-cli`. Install it yourself once:

```
pipx install vantedge-cli
```

Then log in via the browser callback flow:

```
vantedge-cli login
```

The CLI stores a workspace-scoped `sk_live_…` key at `~/.vantedge/credentials`. All plugin-driven work reads from there.

## Repo layout

```
vantedge-claude-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin metadata Claude reads on install
│   └── marketplace.json     # entry point for `/plugin marketplace add`
├── skills/
│   ├── vantedge-cli/SKILL.md
│   ├── vantedge-sdk-runtime/SKILL.md
│   ├── vantedge-agent-recipes/SKILL.md
│   ├── vantedge-secrets/SKILL.md
│   ├── vantedge-usage/SKILL.md
│   ├── vantedge-troubleshooting/SKILL.md
│   ├── diagnose-cost-spike/SKILL.md
│   └── diagnose-failed-run-cost/SKILL.md
├── docs/
│   ├── agent-capabilities.md
│   └── developer-guide.html
├── LICENSE
└── README.md
```

## Contributing

Skills live upstream in the [vantedge-client-python](https://github.com/VantEdgeLabs/vantedge-client-python) repo under `skills/`. This plugin repo is the distribution channel — pull requests to update skill content should target `vantedge-client-python` and then get mirrored here.

## License

MIT — see [LICENSE](./LICENSE).
