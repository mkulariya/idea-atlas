---
id: AG-20261125-10
title: Check an agent's answer against its tool outputs in code before any LLM judge
tldr: A monitor extracts the values an answer asserts, vetoes those absent from or not recomputable from tool outputs, and sends a corrective directive rather than a hard block, cutting fabrication from 17.3% to 10.8%.
section: agents
also_in: []
subtopic: tool-use
type: method
tags: [fabrication, verifier, tool-failure, honesty, agent-evaluation]
aliases: [verification-first oversight, grounding veto, Monitor agent, verify-then-direct]
status: emerging
scores: {novelty: 3, leverage: 4, evidence: 4, generality: 3, actionability: 5, surprise: 3}
power: 73
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.28341', title: 'When Tools Fail, Agents Fabricate: Silent Failure and Verification-First Oversight in Tool-Using LLM Agents', role: origin}
related:
- {id: AG-20261125-01, rel: alternative-to}
- {id: AG-20261125-04, rel: combines-with}
superseded_by: null
---

## Summary

An LLM-only monitor that judges whether a final answer is fabricated fails in a specific way: among 118 vetoes it rejected 39 correct answers against 46 wrong ones, because the judge has no ground truth to check against, while most agent answers are checkable against what the tools actually returned. The authors therefore order the final-answer gate from cheapest code check to LLM check: recompute scalar aggregates from retrieved rows and veto on mismatch, extract the headline values the answer asserts and veto if none appears in any tool output, ask the agent to restate when it states no checkable value, and use an LLM only for what code cannot decide, such as goal drift. Step-level supervision reviews the trace with rule-based signals (identical-retry count, consecutive errors) and injects a corrective directive for known patterns such as retry loops, goal drift, premature abandonment and fabrication, with interventions capped per run. Every veto is a directive, so the agent may revise once and false alarms are recoverable; stages 1-3 are code, so the final gate adds at most one LLM call, and step-level supervision averages 6.4 calls per run against the agent's 11 steps. Across four agents (Llama-3.1-8B, Llama-3.1-70B, DeepSeek-V4, Claude Haiku 4.5), 30 tasks, 9 fault types and 3 seeds, the full cascade raised correct recovery from 45.6% to 56.3% and cut fabrication from 17.3% to 10.8%, and a code-only variant reached 54.7% correct and 13.4% fabrication, so the LLM drift stage contributes. It fails where it was needed most: on partial_success faults fabrication fell only from 38.3% to 32.1%, and in 224 of the 324 such runs with the full monitor it never acted, because an answer built on truncated but real rows passes the grounding check. Use it when final answers contain numbers, IDs or strings that should have come from a tool; the caveats are synthetic tasks with mostly scalar answers, a monitor and judge stronger than every agent, recompute covering scalar aggregations only, and a constrained shell with mock HTTP.

## Detail

### Problem

When tools fail silently, agents fabricate (idea 1). A second model that judges "is this answer fabricated?" sounds like the obvious guard, but the paper found it fails in a specific way: an early LLM-only monitor's vetoes rejected correct answers almost as often as wrong ones (39 correct versus 46 wrong among 118 vetoes), because the LLM cannot verify correctness it has no ground truth for.

Most agent outputs are checkable against what the tools returned: SQL rows, JSON, calculator results. Using that fact is cheaper and more precise than asking another model.

### Mechanism

**Monitor agent.** A second, stronger LLM that observes the main agent's trace, can inject directives into the agent's next turn and can veto its final answer.

**Veto versus corrective directive.** A hard block ends the run; a directive tells the agent what looks wrong and lets it revise and resubmit once, so a false alarm is recoverable.

**Silent ingestion.** Idea 1: the agent accepts degraded data on first contact without flagging it.

The Monitor (Claude Sonnet 4.6 in the study) does two jobs.

**Step-level supervision.** Once an error appears in the trace, review each step with rule-based signals (identical-retry count, consecutive errors) and inject a corrective directive for known patterns such as persistent-fault retry loops, goal drift, premature abandonment and fabrication; interventions are capped per run.

**Final-answer verification cascade**, cheapest sufficient check first:

1. *Grounding (code)*: extract the headline values the answer asserts; veto if none appears in any tool output.
2. *Surface (code)*: if the answer states no checkable value, ask the agent to restate its result and re-check next turn.
3. *Recompute (code)*: for aggregation tasks, re-derive the scalar (sum, average, count) from retrieved rows and veto on mismatch. This runs before the grounding veto so correct computed totals are not mistaken for fabrication.
4. *Drift (LLM)*: only for remaining cases, check whether the answer addresses a materially different question.

Every veto is a directive, and the agent may revise once. Stages 1-3 are code, so the final gate adds at most one LLM call; step-level supervision averages 6.4 calls per run against the agent's 11 steps.

### Why it works

Fabrications that leave a trace in the tool outputs are mechanically detectable: a made-up number is absent from every tool result, and a wrong aggregate does not equal the recomputed one. Code checks have no way to be talked into agreeing, and directives make remaining false positives recoverable.

The authors also explain where it fails: when the agent builds its answer on truncated data, the answer is grounded in the rows it did receive, so grounding passes. In 224 of the 324 `partial_success` runs with the full Monitor, it never acted, because it engages after an error appears and a truncated payload shows none.

My read: this is a good demonstration that verification should be anchored to evidence, but it does not touch silent ingestion itself. Exposing completeness in the tool output (idea 1) is the complement.

### Evidence

Four agents (Llama-3.1-8B, Llama-3.1-70B, DeepSeek-V4, Claude Haiku 4.5), 30 tasks, 9 fault types, 3 seeds; Table 7.

- **Pooled outcomes.** Full cascade: correct recovery 45.6% to 56.3%, fabrication 17.3% to 10.8%. Code-only variant (stages 1-3): 54.7% correct, 13.4% fabrication, so the LLM drift stage contributes.
- **By model.** Fabrication fell for every model. Correct recovery rose for three of four: Llama-70B 33% to 55% and Llama-8B 19% to 39%, DeepSeek 68% to 74%. Claude Haiku traded 5.7 points of correct recovery for 6.9 points less fabrication.
- **Where it fails.** On `partial_success` faults, fabrication fell only from 38.3% to 32.1% (code-only: 40.7%).
- **Evaluation caveat.** The authors note LLM-judged behavioral taxonomies are unreliable ($\kappa=0.58$) while ground-truth grading was stable ($\kappa=0.91$), so the numbers above use the latter.

### Limitations and open questions

The authors state: the recompute stage covers scalar aggregations only and misread aggregates for tasks whose target is a derived value (such as a converted total), vetoing some correct answers; the Monitor and judge are strong proprietary models; the constrained shell and mock HTTP limit realism. They argue the gains are, if anything, understated.

My read: tasks are synthetic and 30 in number, answers are mostly scalar values, and the Monitor model is stronger than every agent. Checks of this kind will not catch fabrications built entirely from truncated but real data, which the paper shows are the most common.

### How to apply

Reach for it when your agent's final answers contain numbers, IDs or strings that should have come from a tool.

1. Log every tool output as structured data. At the end of a run, extract the values the answer asserts (numbers, quoted strings, IDs).
2. Veto if no asserted value appears in any tool output. For simple aggregations (sum, average, count), recompute from the retrieved rows and veto on mismatch.
3. Return a directive naming the problem ("the value 32000 does not appear in any tool result; recheck"), and allow exactly one revision.
4. Use an LLM check only for what code cannot decide, such as goal drift.

**Smallest experiment:** replay 100 logged agent runs through steps 1-3 offline, and hand-check the vetoes: if more than a few are false, narrow the extraction before turning it on.

Assessment: Try in an experiment. Cheap, code-based and measurably reduces fabrication, but it misses the truncated-data case that dominates.

### Connections

Reflexion and Self-Refine reflect on prior outputs without external grounding, and Huang et al. (2024) showed this can make outputs worse. SWE-Search adds structured backtracking by tree search. This paper's monitor grounds the decision in the agent's own tool outputs and orders checks from code to LLM.

Connections: it is the mitigation paired with idea 1 for the checkable fraction of fabrications. Idea 4 is the same pattern at program level (an outside rule on observable outcomes beat the agent's own checks). Idea 2 warns that when you do need an LLM judge, bundling criteria can soften it, which is another reason to put code checks first.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [When Tools Fail, Agents Fabricate: Silent Failure and Verification-First Oversight in Tool-Using LLM Agents](https://arxiv.org/abs/2611.28341) — Across four agents, 30 tasks, 9 fault types and 3 seeds, the code-then-LLM verification cascade raised correct recovery from 45.6% to 56.3% and cut fabrication from 17.3% to 10.8%, but on partial_success faults fabrication fell only from 38.3% to 32.1%.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: created from bulletin 2026-11-25 (#10).
