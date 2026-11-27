---
id: AG-20261127-07
title: Profile solo traces to tell a lead model which subtasks to delegate
tldr: Profiling both models on past tasks assigns each recurring subtask a delegate, conditional or keep tier; putting that table in the lead's prompt lifts ViBench from 70.2 to 78.5 at 19% lower cost.
section: agents
also_in: []
subtopic: multi-agent
type: method
tags: [model-routing, cost-reduction, training-free, harness-coupling, long-horizon-agents, coding-agents]
aliases: [Partial Routing, trace-tuned delegation, subtask tier table]
status: emerging
scores: {novelty: 3, leverage: 3, evidence: 4, generality: 2, actionability: 3, surprise: 2}
power: 60
evidence: B
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.33402', title: 'Partial Routing: Trace-Tuned Delegation for Resource-Efficient Agentic Workflows', role: origin}
related:
- {id: AG-20261126-02, rel: variant-of}
superseded_by: null
---

## Summary

Task-level routing picks one model for a whole request, so agentic work that mixes easy and hard subtasks either pays frontier prices for routine steps or exposes hard steps to a weaker model. Partial routing profiles both models offline: each solves a set of tasks alone, an LLM annotator labels every step of later rollouts with a subtask, subtask cost is the token cost of its attributed steps, and an anonymized LLM judge gives quality per model per subtask. For each subtask it computes saving and risk (mean and worst-case quality drop) and scores D_k(δ)=δ·Sav−(1−δ)·Risk; above a threshold the subtask is delegate, below it keep, in between conditional, while coordination subtasks like planning stay with the lead. The resulting tier table (identifier, one-line definition, tier) goes into the lead's prompt, and the lead calls a delegate_subtask tool that starts the cheaper worker with a fresh context, the lead's tools and a shared workspace for scoped assignments. At δ=0.75 on ViBench, the lead alone scores 70.2 versus 78.5 at 19% lower cost, while unguided delegation reaches 75.6 and a lead given only the plan 68.9 at 39% lower cost; on Terminal-Bench pass rate goes from 54% to 61% at similar cost. Use it when you run an expensive lead agent and already have logs of two models on the same tasks, starting at δ=0.4 and hand-overriding sensitive subtasks. Caveats: one model pair per benchmark, held-out sets of 10 apps and 23 tasks, only coding and command-line work tested, fixed ρ/τ without a sensitivity sweep, LLM annotator and judge with no human check, no verification that the lead followed its tiers, and delegation trades wall-clock time for cost.

## Detail

### Problem

Agentic tasks mix easy and hard subtasks, but routing usually picks one model for the whole task. That spends frontier prices on routine work, or exposes hard steps to a weaker model's failures. Subagent tools (a lead model spawning workers with fresh contexts) already exist, but what gets delegated is left to the lead's own judgment, with no measurement of where the cheaper model is good enough.

Trained orchestrators learn delegation from outcomes, but they fix the cost-quality trade-off at training time and must be retrained when models change. Partial routing needs only past runs.

### Mechanism

**Task-level routing.** Choosing one model per request (RouteLLM, FrugalGPT-style cascades). **Subtask.** A recurring kind of work inside a rollout (planning, implementation, verification).

**Lead and worker.** The lead runs the task and may call a `delegate_subtask` tool that starts a worker model with the lead's tools, a shared workspace and a fresh context. Each brief gives objective, files, constraints and acceptance criteria; the worker returns a report.

**LLM judge.** A model that scores outputs on a rubric; here it rates both models' work on each subtask under anonymous A/B labels.

Offline, once per model pair:

1. **Solo runs.** Each model solves a set of profiling tasks alone; keep traces, token counts and the final benchmark result.
2. **Subtask annotation.** An LLM annotator seeds a subtask set $\mathcal S$ from the first rollout, then labels every step of later rollouts with an existing subtask or adds one, and a final call merges duplicates. Cost grows linearly with the number of rollouts.
3. **Cost and quality.** The cost of a subtask is the token cost of its attributed steps. The judge gives quality $q\in[0,1]$ per model per subtask.
4. **Compile the policy.** For subtask $k$: saving $\mathrm{Sav}_k=1-\bar c_{W,k}/\bar c_{L,k}$ and risk $\mathrm{Risk}_k=(1-\rho)\,\bar g_k+\rho\, g^{\max}_k$, where $\bar g_k$ is the mean and $g^{\max}_k$ the worst-case quality drop (lead minus worker, floored at 0) over tasks. Score $D_k(\delta)=\delta\,\mathrm{Sav}_k-(1-\delta)\,\mathrm{Risk}_k$. If $D_k\ge\tau$: delegate; if $D_k\le-\tau$: keep; otherwise conditional. Coordination subtasks such as planning and wrap-up always stay with the lead.

Settings: $\rho=0.5$, $\tau=0.15$; $\delta$ (from 0 to 1) controls how much is delegated.

Online: the lead gets the tier table (identifier, one-line definition, tier) in its prompt and decides the scope and timing of each handoff. Tiers are plain text, so a team can override them, for example keeping authentication logic with the lead.

### Why it works

The policy replaces the lead's guess with measurements of where the worker's quality gap is small and the saving is large. The authors suggest the workers' gain on ViBench comes from context management: scoped assignments in fresh contexts with compact reports leave the lead less history to carry. The plan alone mostly cuts cost on ViBench by reducing lead turns. My read: part of the gain may be from having *any* subtask decomposition in the prompt; the "planned lead" baseline isolates this.

### Evidence

**Setup.** ViBench (full-stack apps from product requirement documents; lead GLM 5.2, worker MiniMax M3, list prices $1.40/$4.40 vs $0.30/$1.20 per million input/output tokens) with profiling on five applications, results on ten held-out ones with three runs each. Terminal-Bench 4.0 (lead GPT-6.1 Sol, worker GPT-6 Luna), profiling on 12 tasks, results on 23 held-out tasks with two runs each. Baselines: lead alone, unguided delegation, a task-level oracle router with hindsight, the Jev router, and a lead given only the plan.

**Results at $\delta=0.75$.**
- ViBench: 70.2 (lead alone) to 78.5 at 19% lower cost. Unguided delegation scores 75.6; the planned lead (no worker) 68.9 at 39% lower cost.
- Terminal-Bench: pass rate 54% to 61% at similar cost. Unguided delegation keeps 54% at 9% lower cost; the planned lead reaches 58.7% at 16% lower cost.
- Profiling subtasks: 9 on ViBench from 1,055 steps of ten solo traces; the worker is 61-91% cheaper on every subtask and quality gaps are about two points on backend logic and verification. Across four judge reruns scores vary by at most 0.3 points and all tiers at $\delta=0.75$ are reproduced. Profiling cost about $30 on ViBench.
- Wall-clock on Terminal-Bench: median 69 minutes at $\delta=0.75$ versus 55 for unguided delegation and 14 for the lead alone (Table 11).

### Limitations and open questions

One model pair per benchmark in the main results; the second pair per benchmark ran once against solo models only. Held-out sets are small (10 apps, 23 tasks). The worker is always much cheaper and weaker than the lead, so smaller gaps are untested. Only coding and command-line work was tested. $\rho$, $\tau$ and the number of profiling tasks were fixed, with no sensitivity sweep. The annotator and judge are LLMs, with no check against human labels. The authors did not check whether the lead actually followed its tiers. Delegation trades wall-clock time for cost. A disclosure in the paper: Replit, the source of ViBench's production traces, is a portfolio company of Georgian, the authors' affiliation.

My read: with ten held-out apps, the 8.3-point ViBench gain deserves replication before it is treated as a stable number.

### How to apply

Use it if you already run an expensive lead agent and have logs of two models on the same tasks.

1. Run both models alone on 5-15 representative tasks and log token costs per request.
2. Annotate subtasks with an LLM and have a judge score both models with anonymized labels.
3. Compute Sav, Risk and $D_k(\delta)$; start at $\delta=0.4$ and raise it while score holds.
4. Put the tier table in the lead's prompt, override sensitive subtasks by hand, and add a `delegate_subtask` tool.
5. Check the lead actually delegates the delegate-tier subtasks.

Pitfalls: latency increases (the lead waits for workers). **Smallest experiment:** on one benchmark, compare lead alone, unguided delegation and tiers at $\delta=0.75$ for cost and pass rate.

Assessment: Try in an experiment. Cheap, training-free and controllable, with gains on two benchmarks, but small held-out sets and one model pair.

### Connections

Prior approaches cited: task-level routers (RouteLLM, FrugalGPT, Jev), subtask routing from model cards (Topaz) fixed before execution, unguided subagents (e.g. Claude Code), and trained orchestrators (Puppeteer, Uno-Orchestra). Prompt-optimization methods such as ExpeL and GEPA distill traces into context but, per the authors, do not apply this to routing. The new part is trace-derived, per-subtask tiers with a single control parameter, applied at run time to proprietary models.

**Connections.** Idea 4's mechanism (fresh, compact contexts) is the authors' own hypothesis for the workers' gain. Idea 6 also derives harness guidance from past runs. Idea 11 diagnoses failures by component; partial routing profiles by subtask. They could be combined: choose tiers by failing component.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [Partial Routing: Trace-Tuned Delegation for Resource-Efficient Agentic Workflows](https://arxiv.org/abs/2611.33402) — Trace-profiled per-subtask tiers put in the lead's prompt lift ViBench from 70.2 to 78.5 at 19% lower cost and Terminal-Bench pass rate from 54% to 61% at similar cost at δ=0.75, against 75.6 and 54% for unguided delegation.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: created from bulletin 2026-11-27 (#7).
