# cLLM — A GPU Experimentation Platform

**One real GPU. Unlimited experiments.**

cLLM is an open-source platform for running realistic LLM serving experiments without renting a fleet of GPUs. One physical GPU running vLLM calibrates a throughput envelope; cached synthetic nodes then replay that envelope under per-request TPS pacing, prefill simulation, and node-local admission — so scheduling, fairness, routing, and capacity-scaling experiments all run on a single laptop-class GPU.

## Core capabilities

- **Calibrate once, experiment forever.** A `:dsl no-cache` benchmark anchors the real-GPU envelope. Every later experiment runs against that envelope at zero marginal GPU cost.
- **Concurrent multi-target benchmarks.** Real vLLM, pass-through, and synthetic streams run side-by-side in the same time window, on the same host, against the same dashboards — direct cross-validation, no time-stitched runs.
- **Reproducible workloads as data.** Cached responses + prompt files + scenario YAML are checked into the repo. Snapshot the cache, share it, reload it on another machine, and reproduce the same envelope on different hardware.
- **Live, observable scheduling.** A request DSL (`tps`, `prefill`, `stall`, `no-cache`, `re-cache`) lets you script per-request behavior. Every histogram and counter is partitioned by DSL family, tenant, node, and outcome.
- **Three-layer observability.** cLLM (admission/queue), vLLM (KV cache, real tokens), and GPU (utilization, power, thermals) share one time axis and one Grafana stack, tied together by a shared request-correlation ID.
- **Safe LLM-operable control plane.** An MCP server (`cllm-ops`) exposes a fixed set of validated tools — list nodes, run scenarios, summarize results — so an agent like Copilot or Claude can drive the experiment loop. The real GPU node is protected, deletion is withheld, every mutation is audit-logged.
- **Validation as a primitive.** The same `no-cache` mechanism that calibrates the system also re-validates it after a model upgrade, vLLM bump, or hardware swap. One benchmark run is enough.

## Why it matters

GPU time is expensive and serving-system experiments are usually destructive — you can't safely test admission, fairness, or capacity policies on a production lane. cLLM separates the **control plane** (admission, routing, fairness, observability, scheduling) from the **inference plane** (the real GPU), so the control plane can be exercised at full scale on synthetic capacity while the single real GPU keeps the envelope honest.

The result is a workflow where a single benchmark run produces directly comparable evidence across all three observability layers, partitioned by every dimension that matters — and an LLM agent can run the whole loop on your behalf.
