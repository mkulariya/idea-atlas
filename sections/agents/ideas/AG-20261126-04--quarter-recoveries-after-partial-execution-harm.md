---
id: AG-20261126-04
title: A quarter of recoveries after partial execution harm state; repeat-call monitors catch only 41%
tldr: When a code agent's block fails after some writes landed, about 25% of confirmed recoveries leave external state wrong, and a repeat-call monitor catches only 41% of that harm.
section: agents
also_in: []
subtopic: agent-safety-security
type: finding
tags: [agent-safety, tool-failure, agent-evaluation, fault-injection, idempotency, side-effects]
aliases: [Agent Recovery Gone Wrong, recovery harm, partial execution, Rule B monitor]
status: emerging
scores: {actionability: 4, evidence: 4, generality: 3, leverage: 4, novelty: 4, surprise: 4}
power: 77
evidence: B
first_seen: '2026-11-26'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.31554', title: 'Agent Recovery Gone Wrong: Measuring Harm Beyond Duplicate Calls', role: origin}
related:
- {id: AG-20261125-01, rel: combines-with}
- {id: AG-20261126-07, rel: combines-with}
superseded_by: null
---

## Summary

Agents that write and run code can change external services with no transaction around them: a block may add an item to a cart, create an address and place an order, and if the third call raises, the first two writes have already taken effect; the agent sees only the error, rewrites the block and runs those writes again. The authors mine 13,988 AppWorld runs for failed blocks with at least one completed write before the failing line, obtaining a registry of 1,917 candidates in 70.6% of which the agent rewrote and reissued those writes. They draw a 200-episode stratified sample from ten models and replay each episode with database snapshots to confirm partial execution and to label harm by database state and task statement. Of 143 determinable confirmed cases, 30 are harmful, a weighted estimate of 24.6% for the registry (95% interval 15.6 to 34.4), and 29.5% before 2025 versus 14.6% in 2025-2026. A monitor that fires when the first block after the failure calls the same creating or accumulating API catches 41% of all harm (0.15 in the newer stratum), because duplicates are 55% of harm while the rest are destruction of pre-existing state or wrong first-time writes. A pilot with one local model (17 episodes, temperature 0) finds model-supplied idempotency identifiers do not match the same operation, while content-derived identities would have matched harmful writes in 14 to 18 of the 30 harmful recoveries but none of the wrong first-time writes. Use it when your agent runs multi-write code blocks against stateful services: after a failed block, snapshot or diff external state before letting it retry, derive operation identities in the runtime, and treat deletions and unrequested writes as a separate class needing review or rollback; the caveats are that 136 of 145 confirmed episodes are in the shopping app, labels come from LLM annotators with kappa 0.63, and wrong first-time writes can be ordinary planning errors, so the estimate is harm observed during recovery rather than caused by it.

## Detail

### Problem

Agents that write and run code can change external services with no transaction around them. One block may add an item to a cart, create an address and place an order. If the block raises an error on the third call, the first two writes have already taken effect. The agent sees only the error, rewrites the block and runs it again. This needs no crash or lost response; an ordinary exception in generated code is enough.

Systems that treat the agent-tool boundary as transactional (staging effects, idempotency keys, compensation, crash injection) were evaluated on injected faults. It was unknown how often a *natural* failure in the agent's own code leads to harm, and what kinds. That decides whether you need a cheap duplicate-call guard or something stronger.

### Mechanism

**Partial execution.** A block of code fails after at least one write call has completed.

**Recovery.** What the agent does next: usually rewrite the block and continue.

**Idempotency contract.** Each logical operation carries an identifier; a second call with the same identifier returns the stored result instead of acting again.

**AppWorld.** A benchmark where agents write Python against simulated apps backed by databases; runs can be replayed deterministically, which lets the authors snapshot database state.

**Harm kinds.** Duplication (an effect applied again), destruction (unrelated pre-existing state removed), wrong change (a new write the task did not require).

This is a measurement design more than a method.

1. **Find candidates.** In 13,988 AppWorld runs, static analysis finds 1,897 failed blocks with at least one write call before the failing line (a registry of 1,917 after improving the detector). In 70.6% of these the agent rewrites and issues those writes again.
2. **Sample.** Stratify by model release year (before 2025: 1,403 episodes, 27 configurations; 2025-2026: 514, 8 configurations), draw 100 from each with a fixed seed. The sample covers ten models.
3. **Replay with snapshots.** Re-execute each episode from its recorded code with a hook at the API boundary and store database snapshots A (before the failed block), B (after it) and C (end of trajectory). A candidate is a *confirmed* partial execution if A to B shows a change matching a write the detector counted as completed.
4. **Label.** Each continuation gets two labels: a harmful action (state change the task statement does not allow) and a residual violation in C. Two LLM annotators label independently on shuffled digests without model or stratum (Cohen's $\kappa=0.63$); disagreements are settled from database state and an author checks them.
5. **Compare detectors.** "Rule B" fires if the first block after the failure calls the same creating or accumulating API as a completed write, without comparing arguments. A multiplicity test compares object counts in C and B.
6. **Test idempotency identity.** A pilot with a local model (gpt-oss-20b, temperature 0, 17 episodes) wraps writes as `effects.call(operation_id, api, args)`, and a counterfactual analysis asks whether an identifier derived from call content would have matched harmful writes.

### Why it works

The authors say the kinds of harm depend on the configuration. For gpt-4-turbo, gpt-4o and llama3-70b, 15 of 17 harmful continuations are duplications; for gpt-oss-20b, 7 of 11 are not: recovery empties a pre-existing cart or makes a wrong first-time write such as an address record containing message text. Because duplicates are only part of the harm, a repeat monitor cannot see the rest.

Model-supplied identifiers fail in the pilot because the model issues a new identifier for the same operation, so the contract never matches. My read: the lesson is that idempotency must be enforced by the runtime from content, not delegated to the model, and that even then it cannot catch wrong first-time writes.

### Evidence

The authors report, on the 200-episode stratified sample:

| Quantity | Value |
|---|---|
| Candidates confirmed as partial executions | 145 of 200 |
| Harmful recoveries among 143 determinable confirmed cases | 30 |
| Weighted estimate for the registry | 24.6% (95% interval 15.6 to 34.4, cluster bootstrap) |
| By stratum | 29.5% (18/61) before 2025; 14.6% (12/82) 2025-2026 |
| Harm never repaired | 49 of 51 harmful continuations leave a violation; all 30 confirmed ones |
| Rule B recall on all harm | 0.41 overall; 0.15 in the 2025-2026 stratum |

Two thirds of the harmful recoveries duplicate an effect; the rest delete user data or make a wrong first-time write. Rule B finds 75% of duplications, but duplications are 55% of all harm (25% in the newer stratum); 23 of its 30 misses are destructions and wrong changes. Content-derived identity would have matched every harmful write in 14 to 18 of the 30 harmful recoveries, none of the wrong first-time writes, and would also have matched writes the task requires in 30 episodes. In the pilot, no condition detectably reduced harm: 5 harmful continuations with enforcement versus 7 when direct writes were also rejected.

A check of the instrument: positive controls revealed defects (large tables skipped, phantom deletions), and the authors fixed them.

### Limitations and open questions

The authors note that 136 of 145 confirmed episodes are in the shopping app, the idempotency pilot is small (17 episodes, one local model) and its results are "emerging evidence", and that wrong first-time writes can also be ordinary planning errors, so the estimate is harm observed during recovery, not caused by it.

My read: labels come from LLM annotators with moderate agreement ($\kappa=0.63$; $\kappa=0.95$ on an author-checked subsample of 60). Older models dominate the registry, the 2025-2026 stratum is mostly one configuration (gpt-oss-20b, 86 of 100 episodes) though results hold without it (19 of 66 confirmed episodes harmful), and AppWorld may not resemble your tools.

### How to apply

**When:** your agent runs multi-write code blocks against stateful services.

1. Do not rely on duplicate-call detection alone; it misses about 59% of harm in this sample.
2. After a failed block, snapshot or diff external state before letting the agent retry, and show the diff to the agent.
3. Prefer runtime-derived operation identities over model-supplied ones, and expect to handle legitimately repeated writes.
4. Treat deletions and unrequested writes as a separate class that needs review or a rollback path.

**Cost:** snapshotting state is cheap in a test harness and harder against real services.

**Smallest experiment:** replay 50 failed multi-write blocks from your own agent logs against a sandbox copy, diff state before and after the retry, and count harmful actions by kind.

Assessment: Adopt now. A concrete, state-verified measurement of a common failure with a clear mitigation direction; adopt the audit, treat the 24.6% as AppWorld-specific.

### Connections

Earlier work on the agent-tool boundary stages effects, uses idempotency keys and compensation, verifies ambiguous outcomes, or injects crashes around tool calls, all on injected faults. This paper measures natural failures in recorded agent runs, labels harm by database state and task, and shows that duplicate detection covers only part of it.

**Connections.** Idea 1 (evidence closure) is another case where the tool layer, not the model, decides outcomes. Idea 7 (late-binding GUI actions) concerns actions that become wrong because the world changes before they land; recovery harm is the same family for API writes. Idea 2 (delegation) gives subagents write access in the same loop, so the same exposure applies.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [Agent Recovery Gone Wrong: Measuring Harm Beyond Duplicate Calls](https://arxiv.org/abs/2611.31554) — Replaying a 200-episode stratified sample of AppWorld runs, the authors found 30 harmful recoveries among 143 determinable confirmed partial executions (weighted 24.6%, 95% interval 15.6-34.4), and a repeat-call monitor recalled only 0.41 of that harm (0.15 in the 2025-2026 stratum).

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: +relation combines-with AG-20261126-07.
- 2026-11-26: created from bulletin 2026-11-26 (#4).
