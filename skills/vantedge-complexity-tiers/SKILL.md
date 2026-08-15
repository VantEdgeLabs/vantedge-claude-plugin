---
name: vantedge-complexity-tiers
description: How to pick the right complexity tier (low, medium, high) for LLM calls in VantEdge agent apps. Load this whenever authoring or reviewing an LLM call site in a VantEdge app.py — the tier picked determines cost and capability. Reuse for every llm() call.
---

# vantedge complexity tiers — picking `low` / `medium` / `high`

Every `gateway.llm(...)` call in an agent app takes a `complexity` argument. The tier the author picks — usually Claude, at the call site — determines which chain the backend routes the call through, and therefore both the cost and the ceiling on what the call can do.

## Why tiers exist

Cost and lifecycle. Tiers give agent code a stable, capability-shaped abstraction so it doesn't have to name a specific model — model names change constantly, capability contracts don't.

## The three tiers — capability contract

- **low** — single-shot pattern matching, extraction from short context, classification, JSON conformance. No multi-step reasoning. Short context (<10k). Example use: parse a webhook payload, tag an email as spam/not-spam.
- **medium** — multi-step reasoning, a few tool calls, coherent generation, moderate context (~30k). Workhorse tier — most tasks land here. Example use: summarize a threaded email conversation into action items, generate a SQL query from natural language.
- **high** — complex tool orchestration (many-step), long-context comprehension (100k+), correctness under adversarial input, deep reasoning. Example use: multi-hop research over a corpus, generate + debug a nontrivial code change, orchestrate a workflow with 10+ tool calls.

## How to specify at the call site

```python
from vantedge.tools.gateway import llm

result = await llm(
    prompt="…",
    complexity="medium",   # required by convention — always specify
)
```

**Explicit rule:** *always* specify `complexity=` at the call site. The default `"medium"` exists only so legacy apps that predate this param continue to work — new code must be explicit.

## Decision guide (when in doubt)

- Extraction / classification / short JSON: `low`
- Reasoning + coherent generation + few tool calls: `medium`
- Long context OR many tool calls OR adversarial correctness: `high`

## How to inspect the current tier → chain mapping

`vantedge-cli models tiers` — see which chain each tier resolves to today.

## How lifecycle works

The platform manages the tier → chain → model mapping. When a model retires, we swap it in the chain — your code doesn't change. That's the whole reason tiers exist as an abstraction over model names.
