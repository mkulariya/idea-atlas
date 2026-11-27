---
id: AG-20261127-11
title: Version memory per session and diagnose failures by tracing them to harness components
tldr: Committing the memory state after each session lets an optimizer trace each failed query to construction, retrieval or answering, and cheap sampled evaluation cuts query tokens 43.9K to 17.9K while lifting LongMemEval-S from 0.888 to 0.916.
section: agents
also_in: []
subtopic: memory-context
type: method
tags: [scaffold-search, agent-evaluation, cost-reduction, harness-coupling, memory-systems]
aliases: [HoMe, session-level memory versioning, trajectory-guided failure attribution]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 3, generality: 3, actionability: 4, surprise: 3}
power: 72
evidence: C
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.34624', title: 'HoMe: Toward Practical Harness Optimization of Memory Systems for AI Assistants', role: origin}
related:
- {id: AG-20261127-06, rel: alternative-to}
- {id: AG-20261126-09, rel: alternative-to}
superseded_by: null
---

## Summary

A memory assistant's behavior depends on a harness of code, prompts and configuration, but a low answer score does not say whether the information was never written, was later overwritten, was not retrieved, or was retrieved and misused, and evaluating one candidate means replaying long histories (115K tokens per case on LongMemEval-S, up to 1.5M on BEAM). HoMe commits the memory files of every case after every session into a git harness repository during evaluation, then works backwards from a low-scoring query: if the retrieved context contains the answer the failure is in answering, if only the final memory contains it the failure is retrieval, otherwise construction, and construction failures are traced to source sessions by diffing consecutive memory commits. Case Study Agents run in parallel and a Diagnoser Agent groups their diagnoses by mechanism so the mechanism covering the most failures is fixed first; an Optimizer Agent may pick any previously evaluated harness as the parent of the next candidate. Evaluation budget is protected by a smoke test, by a sampled subset that favors historically difficult units, and by reusing constructed memory from the nearest evaluated ancestor with an equivalent construction process at probability 0.6, with six formal evaluations per search. On LongMemEval-S the answer score rises from 0.888 to 0.916 while query tokens fall from 43.9K to 17.9K; on BEAM-1M from 0.650 to 0.704 with 101.8K tokens down to 60.4K, and against MemPro the score is 10.4 pp higher on LongMemEval-S with 81.5% fewer query-stage tokens. A harness optimized on BEAM-100K reaches 0.908 on LongMemEval-S at 26.6K tokens (MemPro 0.800 at 42.2K), and the reverse direction reaches 0.639 on BEAM-1M at 39.0K tokens, which the authors read as evidence that the edits are structural rather than tuned constants. Use it if you maintain a memory or retrieval harness with a benchmark of long-history questions, starting with per-session commits and diagnosis of about 30 failed queries. Caveats: one memory system (ReMe), two benchmarks, one model family (Qwen), LLM-judged scores, scores that decline in several categories, sampled evaluations that are not directly comparable across candidates, and no variance reported across search runs apart from a second BEAM search.

## Detail

### Problem

The behavior of an assistant's memory depends on a harness: the code, prompts and configuration for building memory from conversations, maintaining it, retrieving from it and answering. Tuning this by hand is slow, and automated tuning has two problems. A low answer score does not say whether the information was never written, was later overwritten, was not retrieved, or was retrieved and misused. And evaluating one candidate means replaying long histories (115K tokens per case on LongMemEval-S, up to 1.5M on BEAM), which is expensive.

### Mechanism

**Memory harness.** The editable parts of a memory system (here the file-native system ReMe): construction, maintenance, retrieval and answering. **LongMemEval-S** has 500 single-query cases with about 115K history tokens. **BEAM** variants have 20 queries per case and histories of about 131K (BEAM-100K) and 1.04M tokens (BEAM-1M).

**Formal evaluation.** One run of a candidate counted against the budget $B$. **Pareto frontier.** Candidates not beaten on both score and token cost.

Three components.

1. **Developer agents.** A Diagnoser Agent analyzes failures of the latest evaluated harness; an Optimizer Agent picks any previously evaluated harness as the parent of the next candidate (refine a branch, start a new branch from an earlier commit, or merge two branches) and edits code, prompts or configuration. Candidates live as commits in a git "harness repository"; each branch tests one hypothesis.
2. **Failure diagnosis.** During evaluation, the memory files of each case are committed after every session. For a low-scoring query the diagnoser works backward: if the retrieved information contains the answer, the failure is in answering; if only the final memory contains it, retrieval; otherwise construction. For construction failures it finds the source sessions and reads the diffs between consecutive memory commits to see whether the fact was written and later corrupted or deleted. Case Study Agents run in parallel (map); the Diagnoser then groups diagnoses by mechanism and prioritizes mechanisms supported by the most cases (reduce).
3. **Experiment manager.** A candidate must pass a smoke test before it consumes budget. Admitted candidates are evaluated on a sampled subset that favors historically difficult units, and constructed memory is reused from the nearest evaluated ancestor with an equivalent construction process (reuse probability 0.6). Hyperparameters: $B=6$ formal evaluations, concurrency 28, $\alpha=0.3$, $\epsilon=0.2$; 28 cases per iteration on LongMemEval-S, 10 cases and 100 queries on BEAM.

### Why it works

Structural improvements need evidence about causes. Session-level versions make intermediate memory states inspectable cheaply, so the diagnoser sees where information was lost instead of guessing from the final score; grouping diagnoses by cause across cases filters out case-specific noise. Cheap evaluation lets the search try more candidates per unit of compute, and the git history gives each iteration its provenance. The authors point to cross-benchmark transfer as indirect evidence that the edits are structural: changes to source allocation, evidence selection and cross-session indexing, not tuned constants. My read: the diagnosis procedure is the part most likely to transfer to other agent systems.

### Evidence

**Setup.** LongMemEval-S is split equally into train and test; BEAM trains on BEAM-100K and tests on BEAM-1M. Baselines: A-MEM, SimpleMem, ReMe (manually tuned per benchmark), and MemPro (automated harness optimization, up to eight formal evaluations). Memory construction uses qwen3.6-flash; the other LLM calls use qwen3.7-max; scoring uses an LLM judge. Token counts do not deduct KV-cache hits.

**Table 1.**

| Benchmark | ReMe start | HoMe | Query tokens (start to HoMe) |
| --- | --- | --- | --- |
| LongMemEval-S | 0.888 | 0.916 | 43.9K to 17.9K |
| BEAM-1M | 0.650 | 0.704 | 101.8K to 60.4K |

LLM calls per query on LongMemEval-S fall from 4.7 to 3.0. Against MemPro, HoMe's answer score is higher by 10.4 pp on LongMemEval-S and 1.1 pp on BEAM, with 81.5% and 7.6% fewer query-stage tokens. A second independent BEAM search reaches 0.720 at 68.3K tokens.

**Transfer (Table 2).** The harness optimized on BEAM-100K reaches 0.908 on LongMemEval-S at 26.6K tokens (MemPro 0.800 at 42.2K; start 0.888 at 43.9K). The harness optimized on LongMemEval-S reaches 0.639 on BEAM-1M at 39.0K tokens (MemPro 0.617 at 113.1K).

**Efficiency.** BEAM optimization takes 10.4 hours against 11 hours 23 minutes for MemPro, and 185.2M tokens against 331.2M (-44.1%). Parallelism cuts evaluation time by over 90%, and reuse saves 94% on LongMemEval-S. Sampled and full-set token reductions correlate at $r=0.870$ across six BEAM candidates. HoMe reaches the BEAM Pareto frontier after 6.3 hours. Gains are largest on single-session preference (LongMemEval-S) and contradiction resolution (BEAM); scores decline on LongMemEval-S multi-session and on BEAM knowledge update and information extraction.

### Limitations and open questions

Authors' hyperparameters for SimpleMem and A-MEM were tuned on the test sets, which favors those baselines; token counts do not deduct KV-cache hits and ReMe-based methods have agentic loops with more prefix reuse. One memory system (ReMe), two benchmarks, one model family (Qwen), LLM-judged scores. Scores decline in several categories, so gains are not uniform. HoMe's sampled evaluations are not directly comparable across candidates, so the Optimizer Agent judges quality from accumulated evidence, which adds subjectivity. Search was six formal evaluations; behavior with larger budgets is not shown.

My read: the transfer results (BEAM-100K to LongMemEval-S and back) are the strongest evidence that the edits are structural. I did not see variance across search runs, apart from one second BEAM search.

### How to apply

Use it if you maintain a memory or retrieval harness with a benchmark of long-history questions.

1. Commit the memory directory after every session during evaluation, and keep the retrieval trace per query.
2. For failed queries, check in order: was the answer in the retrieved context, in the final memory, or neither; if neither, diff the session commits.
3. Group diagnoses by mechanism and fix the mechanism that covers the most failures.
4. Add a smoke test and a cheap sampled evaluation before any full run; reuse constructed memory when only retrieval or answering changed.
5. Evaluate the final harness on a second benchmark and report tokens as well as score.

Cost: about 10 hours and 185M tokens for one BEAM search in the paper. **Smallest experiment:** instrument your memory system with per-session commits, run diagnosis on 30 failed queries, and see how many fall in construction, retrieval and answering.

Assessment: Watch. A useful diagnosis recipe with good transfer results, but one memory system and Qwen-only models, with baselines tuned on test sets.

### Connections

Prior work named by the authors: MemPro, an automated harness optimizer whose candidates only modify prompts or parameters (its training-selected harness scored 0.812 on the LongMemEval-S test against 0.888 for its start), and manually engineered memory systems (ReMe, A-MEM, SimpleMem). HoMe adds session-level memory versioning with trajectory-guided attribution, branching candidate search from any parent, and evaluation shortcuts (smoke tests, adaptive sampling, memory reuse).

**Connections.** Idea 6 (TransHarness) evolves harness interventions and uses a replay gate for selectivity; HoMe uses diagnosis and cheap evaluation, and in the sections I read has no replay-style selectivity gate (my read). Idea 7 profiles subtasks from traces; HoMe attributes failures to components. Idea 1 is the reminder to keep a held-out set separate from the one used for selection. Idea 4 notes that compaction timing is itself a harness parameter.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [HoMe: Toward Practical Harness Optimization of Memory Systems for AI Assistants](https://arxiv.org/abs/2611.34624) — Committing memory state after every session and diagnosing failures by construction/retrieval/answering attribution lifted LongMemEval-S from 0.888 to 0.916 while cutting query tokens from 43.9K to 17.9K and BEAM-1M from 0.650 to 0.704 at 60.4K tokens, with a BEAM-100K harness transferring to 0.908 on LongMemEval-S.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: created from bulletin 2026-11-27 (#11).
