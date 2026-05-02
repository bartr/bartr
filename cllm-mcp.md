# cLLM Operations MCP Server

`cllm-ops` is a small, bounded [Model Context Protocol](https://modelcontextprotocol.io) server that wraps the cLLM/vLLM inference experimentation environment. It lets an MCP-aware client (Copilot Chat in agent mode, Claude Code, Codex) inspect the fleet, tune synthetic capacity, run reproducible benchmark scenarios, and pull metrics-backed summaries — all through a fixed set of validated tools, never through arbitrary shell or cluster access.

The server lives in [mcp/](https://github.com/bartr/vllm/tree/main/mcp) of the cLLM repo and is implemented as a single FastMCP process (`mcp/server.py`) that talks to the cLLM HTTP control plane and the local Prometheus stack.

## Why it was created

cLLM is a GPU experimentation platform: one real vLLM node on a single GPU calibrates a throughput envelope, and cached synthetic nodes replay that envelope under per-request TPS pacing, prefill simulation, and node-local admission. That stack already had a rich HTTP API, dashboards, and a long-running `ask --bench` driver — but every operator workflow still meant hand-crafting `curl` calls, copying YAML around, and stitching together log files.

`cllm-ops` was built to turn those workflows into safe, repeatable tool calls so an LLM operator could drive the experiment loop end-to-end:

1. **Bounded autonomy.** An LLM gets a fixed contract — list nodes, run a scenario, summarize results — instead of a shell. Mutating tools are audit-logged with before/after node state. The real `vllm` node is protected and never mutated. Deletion is intentionally not exposed.
2. **Reproducibility as data.** Scenarios live as YAML files under `benchmark/scenarios/`. `run_scenario` applies node overrides for the run, restores them on exit (even on error), captures before/after metrics snapshots, parses the per-group logs, and writes a Markdown report to `benchmark/reports/{timestamp}-{scenario}.md`. Every run is a checked-in artifact, not transient operator state.
3. **Operator-grade summaries.** `summarize_experiment` returns a structured evidence bundle (topology + live metrics + bench status) so the LLM's summary is grounded in numbers from Prometheus, not paraphrased from a transcript.
4. **Safe live tuning.** `create_synthetic_node` and `update_node` only mutate synthetic nodes, validate every field against numeric bounds, and refuse when `CLLM_READ_ONLY=true`. Synthetic nodes added during a scenario run are cleaned up automatically.
5. **An FDE-style demo surface.** The same server drives the live demo: an operator can ask Copilot to "add a synthetic `rtx` node and run the high-traffic scenario" and watch traffic rebalance from 50/50 to 33/33/33 on the dashboards while the report writes itself.

In short: `cllm-ops` is the thin, audited, LLM-operable layer that turns an already-instrumented cLLM/vLLM lab into something an agent can run.

## Tools

The server exposes 11 tools, grouped by purpose:

### Inspect (read-only, always safe)

| Tool | Purpose |
|---|---|
| `list_nodes` | Full fleet with `protected` annotations and router policy |
| `get_node(id)` | Single node detail (capacity, degradation, realism, stats) |
| `get_config` | cLLM runtime configuration from `GET /config` |
| `get_cache_status` | Cache capacity, entries, hit/miss counts, key list |
| `get_metrics_snapshot(include_raw=False)` | Live Prometheus snapshot — per-node gauges/counters + cache totals |
| `get_benchmark_status(tail_lines=40)` | Whether `ask --bench` is running + recent log rows (treats blank `total_tok_s` as `warming_up=true`) |

### Mutate (audit-logged; refused when `CLLM_READ_ONLY=true` or target is protected)

| Tool | Purpose |
|---|---|
| `create_synthetic_node(id, …)` | Add a new synthetic node; defaults inherited from the existing cllm node |
| `update_node(id, …)` | Change capacity/realism fields on an existing synthetic node (only provided fields change) |

### Experiment (bounded; produce reports + audit entries)

| Tool | Purpose |
|---|---|
| `run_benchmark_window(duration_seconds=120, concurrency=120)` | Ad-hoc bounded `ask --bench` run with before/after metrics deltas |
| `run_scenario(name)` | Execute a YAML scenario from scenarios, restore overrides on exit, write a Markdown report under reports |
| `summarize_experiment(question="")` | Structured evidence bundle (topology + metrics + bench status) for natural-language summaries |

Notably **not exposed**: `delete_node` — destruction is intentionally withheld. Ephemeral nodes created by `run_scenario` are cleaned up by the runner itself (as you saw with `rtx`). The `vllm` node is protected and rejected by both `update_node` and `create_synthetic_node`.

## Guardrails

- **Protected nodes.** `vllm` (the real GPU node) is in a protected set and rejected by every mutating tool.
- **Read-only mode.** `CLLM_READ_ONLY=true` disables all mutating tools for the session.
- **Numeric bounds.** Every capacity and realism field is validated against an explicit `[min, max]` range before any HTTP call.
- **No arbitrary commands.** `run_benchmark_window` and `run_scenario` build the `ask --bench` command from validated fields only — no operator-supplied argv.
- **Audit log.** Every mutating call appends a structured entry (tool, inputs, before/after state) to `CLLM_AUDIT_LOG`.
- **Bounded windows.** Benchmark durations are clamped (30–600s) and warmup rows (~first 15s) are excluded from stats so aggregate `tok/s` isn't reported during the sliding-window warmup.
