---
id: AG-20261125-01
title: Agents fabricate by ingesting partial tool output as valid, not by retrying
tldr: Agents accept plausible but truncated tool results as valid on first contact, fabricating 38% of the time on partial-data faults, while agents that retry visibly failing calls usually report failure honestly.
section: agents
also_in: [rl-envs]
subtopic: tool-use
type: finding
tags: [agent-evaluation, benchmark-quality, verifier, tool-failure, fabrication, fault-injection, honesty]
aliases: [RetryRealistic, silent ingestion, silent failure]
status: emerging
scores: {actionability: 5, evidence: 4, generality: 3, leverage: 4, novelty: 4, surprise: 4}
power: 79
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.28341', title: 'When Tools Fail, Agents Fabricate: Silent Failure and Verification-First Oversight in Tool-Using LLM Agents', role: origin}
related:
- {id: AG-20261125-04, rel: alternative-to}
- {id: AG-20261125-10, rel: alternative-to}
superseded_by: null
---

## Summary

Agent-reliability work assumes the dangerous path is a recovery loop — the tool fails, the agent retries, context fills with junk, the budget runs low, and the agent invents a value — so the proposed fix is a better retry policy. The authors instead build RetryRealistic, a fault-injection testbed on a real substrate (OS temp directory with genuine chmod errors, SQLite, a deterministic local HTTP server, a constrained 24-command shell) with 8 tools, 9 fault types and 30 multi-step tasks, run with four ReAct agents over 2,100 faulted runs graded against machine-checkable ground truth. Correct recovery ranged from 18.9% (Llama-8B) to 67.6% (DeepSeek-V4), and 6% to 26% of faulted runs ended in confident fabrication. Fault type governs the failure mode: partial_success, a truncated but plausible payload, produced fabrication on 38.3% of runs, while permission_denied and auth_required produced honest failure on 78.1% and 53.4%, and fault type explained 1.6 times more fabrication variance than model identity. Retries are protective rather than causal: agents retrying a visibly failing call 3 or more times fabricated on 7.0% of runs versus 20.7% at 0-1 retries, and on partial-data faults the agent repeated the faulted call at most once in 554 of 609 runs. A sanitization experiment holding injection point and persistence fixed found 35.2% fabrication under misleading partial data versus 16.0% under an explicit denial. Removing chain-of-thought left correct recovery unchanged but raised fabrication from 17.3% to 25.6%, and an honesty clause cut fabrication to 13.7% overall yet did nothing on partial_success (38% to 40%). Use it when your agent reads data through tools that can truncate, paginate or partially fail, and expose completeness (row counts, truncated flags, next-page tokens) in every tool result; the main caveat is that 30 tasks and four mostly small-or-mid-size agents are a small sample, and few runs repeated the faulted call on partial-data faults.

## Detail

### Problem

A deployed agent that says "the data is unavailable" is annoying; an agent that says "total revenue across all rows: 32000.00" when the true figure is 48,750 is dangerous, because the downstream consumer cannot tell the answer is wrong. The paper's example is exactly this: a truncated read returned half of the rows, and the agent summed them.

Most agent-reliability work assumes the dangerous path is a recovery loop: the tool fails, the agent retries, context fills with junk, the budget runs low, and the agent finally invents something. If that were the mechanism, the fix would be better retry policies. The authors' data point somewhere else, which changes where a team should spend effort.

### Mechanism

**Fault injection.** Deliberately making a tool call fail (or return degraded output) at a chosen point, so you can measure how the agent reacts under controlled conditions.

**Fabrication.** In this paper, a final answer that asserts a wrong or unsupported value as fact. The paper separates three outcomes for every run: *correct* (right result, including valid workarounds), *fabricated*, and *honest failure* (the agent reports that the result is unavailable).

**Persistent versus transient fault.** A transient fault (timeout, network error, locked database) succeeds on retry with probability 0.5 in this setup. A persistent fault (permission denied, auth required, not found, malformed output, partial success, non-zero exit) always fails again.

The authors built **RetryRealistic**, a testbed where agents act on a real substrate rather than canned strings: a real OS temp directory with genuine `chmod` permission errors, SQLite databases, a deterministic local HTTP server, and a constrained shell (24 allow-listed read-and-transform commands, 30-second timeout). There are 8 tools, 9 fault types and 30 multi-step tasks (10 easy, 12 medium, 8 hard). Each task names a `fault_target` (a tool plus an argument substring), the fault fires on the first matching call, and every task has a designed alternative recovery path and machine-checkable ground truth.

Four ReAct agents were run at temperature 0.2 with a 20-step budget: Llama-3.1-8B, Llama-3.1-70B, DeepSeek-V4 (a reasoning model) and Claude Haiku 4.5. Each task ran under each compatible fault plus a clean baseline, with 3 seeds, giving 2,100 faulted runs.

The key analysis steps:

1. Grade each final answer against ground truth (correct / fabricated / honest failure). An LLM-judged behavioral taxonomy proved unreliable (two judges agreed on 67% of runs, $\kappa=0.58$); ground-truth grading agreed on 94% ($\kappa=0.91$).
2. Group outcomes by fault type, then by how often the agent retried the same call.
3. Isolate content from fault type with a *sanitization* experiment that keeps the injection point, tool and persistence fixed and changes only the payload: a misleading partial result, an explicit denial, or a valid-but-empty result.
4. Ablate chain-of-thought and add an "honesty" clause to the prompt.

### Why it works

The authors' explanation is **silent ingestion**: partial or malformed data passes the agent's plausibility filter, so nothing triggers a retry or a flag. A permission error cannot be mistaken for data, so the agent reports it. A truncated JSON payload looks like data, so the agent uses it.

This fits the retry numbers. If fabrication came from retry loops, heavy retriers would fabricate most. The opposite holds: agents that noticed something was wrong and retried mostly ended honest.

My read: this is a *detection* failure, not a *recovery* failure. Instructions and retry budgets act after the agent has decided something is wrong; silent ingestion happens before that decision. That also explains why the honesty prompt helped on explicit errors and did nothing on partial data.

### Evidence

All numbers are from the authors' runs on RetryRealistic.

- **Recovery is unreliable.** Correct recovery ranged from 18.9% (Llama-8B) to 67.6% (DeepSeek-V4) over the 2,100 faulted runs (Table 3). Between 6% and 26% of faulted runs ended in confident fabrication. DeepSeek fabricated least (5.5%); the other three agents fabricated on 19-26%.
- **Fault type governs failure mode.** `partial_success` (truncated but plausible payload) produced fabrication on 38.3% of runs; `permission_denied` and `auth_required` produced honest failure on 78.1% and 53.4% (Table 4). Fault type explained 1.6 times more fabrication variance than model identity ($\eta^2$ 0.061 versus 0.038).
- **Retries protect.** Agents that retried a visibly failing call 3 or more times fabricated on 7.0% of runs and failed honestly on 82.2%; at 0-1 retries fabrication was 20.7% (Table 5). On partial-data faults the agent repeated the faulted call at most once in 554 of 609 runs; 27.8% of those fabricated.
- **Content, not fault type.** With injection point and persistence fixed, fabrication was 35.2% under misleading partial data versus 16.0% under an explicit denial (27 data-target tasks, 4 models, 3 seeds; Table 6). For DeepSeek it was 25% versus 0%; for Llama-8B the gap was small (37% versus 33%).
- **Deliberation helps.** Removing chain-of-thought left correct recovery unchanged (45.6% to 45.5%) but raised fabrication from 17.3% to 25.6% (+8.3 points). An honesty clause cut fabrication from 17.3% to 13.7% at almost no cost to correctness (45.6% to 44.9%), but on `partial_success` fabrication went from 38% to 40%.

### Limitations and open questions

The authors state that RetryRealistic uses a constrained shell and mock HTTP, and that few runs repeated the faulted call on partial-data faults (55 of 609), so whether repeating the call helps is unknown. In 118 of the 2,100 faulted baseline runs the targeted fault never fired.

My read: 30 tasks and four agents is a small sample, and three of the four agents are small or mid-size models. The sanitization experiment is the strongest causal evidence, but it covers 27 tasks. Whether frontier agents with long-horizon tool use show the same ingestion rate is not tested.

### How to apply

Reach for this when your agent reads data through tools that can truncate, paginate or partially fail (file reads, SQL with row limits, HTTP APIs with pagination, shell pipes).

1. Add partial-output faults to your own eval harness: truncate a read to half its rows, drop a page, return a valid-but-incomplete JSON. Count the share of final answers that assert values the full data would contradict.
2. Make completeness visible to the agent: return row counts, `truncated: true`, byte lengths, or next-page tokens in every tool result, because a plausible payload with no completeness signal gives the model nothing to doubt.
3. Keep explicit reasoning before actions, and do not rely on a "never fabricate" prompt for partial data.

**Smallest experiment:** take 20 of your existing agent tasks, truncate the first data-returning tool call to 50% of its output, and compare fabricated versus honest-failure rates with and without a `rows_returned / rows_total` field in the tool result. A few hours of work, no training.

Assessment: Adopt now. The finding is cheap to test and the mitigation (expose completeness in tool outputs, grade against ground truth) costs almost nothing.

### Connections

Agent benchmarks such as WebArena, OSWorld and tau-bench measure task success but, per the authors, do not control fault type or persistence. Fault-injection work such as MAS-FIRE and ReliabilityBench perturbs agents but does not grade fabrication against ground truth or propose a fix. Reflexion and Self-Refine assume reflection helps; Huang et al. (2024) showed that without external grounding self-correction can hurt.

What is new is a controlled demonstration, on a real OS substrate, that the dominant failure is acceptance of degraded data on first contact, and that the payload rather than the fault label drives it.

Connections: the cascade in idea 10 comes from the same paper and is the mitigation for the cases where fabricated values are checkable. Idea 4 is the same pattern one level up: self-improving agents that write checks which stay silent. Idea 5 prices retries, which this paper suggests are protective when they happen.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [When Tools Fail, Agents Fabricate: Silent Failure and Verification-First Oversight in Tool-Using LLM Agents](https://arxiv.org/abs/2611.28341) — In 2,100 faulted runs on RetryRealistic, partial_success faults produced fabrication in 38.3% of runs versus honest failure in 78.1% for permission_denied, and agents retrying 3+ times fabricated on 7.0% versus 20.7% at 0-1 retries.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: +relation alternative-to AG-20261125-10.
- 2026-11-25: +relation alternative-to AG-20261125-04.
- 2026-11-25: created from bulletin 2026-11-25 (#1).
