---
id: AG-20261126-02
title: Frontier models that delegate legwork to small local models cut cost by two thirds
tldr: Route subagent calls to a small local model and tell the frontier model that its own turns are expensive; coding cost falls to a third with no fewer tasks resolved.
section: agents
also_in: []
subtopic: multi-agent
type: method
tags: [long-horizon-agents, training-free, harness-coupling, budget-forcing, model-routing, cost-reduction]
aliases: [Delegent, role routing, subagent delegation]
status: emerging
scores: {actionability: 4, evidence: 4, generality: 3, leverage: 4, novelty: 3, surprise: 3}
power: 71
evidence: B
first_seen: '2026-11-26'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.33098', title: 'Delegent: A Multi-Agent Harness That Reduces Cost without Sacrificing Performance', role: origin}
related:
- {id: AG-20261127-07, rel: has-variant}
superseded_by: null
---

## Summary

A long-horizon agent re-reads its entire growing context at every step, so a hundred-step coding run pays for a hundred ever-longer prompts, and existing cascades or routers cannot help because they must hand a whole task to one model that can solve it alone. Delegent keeps Claude Code's loop and subagents unmodified and adds a relay that sends calls from the main loop to the frontier model and calls from subagents and the auxiliary model to a small local model served by vLLM, with one prompt-cache key per task and each subagent pinned to one GPU replica. The frontier model briefs the subagents, receives only their final reports (with quoted evidence checked against cited documents) rather than their tool output, and always writes the final patch or answer. A short steering instruction tells it that its own turns are the expensive part, so it should delegate tool-heavy work and keep its own tool use to a few targeted checks. On DeepSWE the frontier cost per task fell from $2.11 alone to $0.70 with Qwen3.8-27B-FP8 subagents, with no fewer tasks resolved, and 694 search questions were answered as accurately at 41% of cost. The ablation is the key result: without the instruction the 2B subagent resolved 22 of 50 DeepSWE tasks (seven fewer than the frontier model alone) at 55% of cost, while with it both SLMs matched the baseline at a third of cost. Use it when the bill is dominated by reading, searching and test-running legwork and a GPU is available for a 2B-30B model; caveats are reliance on Claude Code and AREX-Turbo prompts, small ablations without confidence intervals, a frontier-model search grader, and latency up by about half (38 versus 25 minutes median on DeepSWE).

## Detail

### Problem

A long-horizon agent re-reads its whole context at every step, and the context grows with each tool call. A coding agent that takes a hundred steps pays for a hundred ever longer prompts. Much of that work is easy legwork: finding the relevant code, reproducing a bug, running tests, issuing searches, reading documents. Existing agents bill all of it at frontier prices, including in their subagents.

The usual cost fixes, cascades and routers, give a whole task to one model and so need a small model that can solve the task alone. A 2B model rarely can. This idea splits the task by role instead.

### Mechanism

**Harness.** The loop software that calls the model, runs its tool calls and feeds results back. Here the base is Claude Code.

**Subagent.** A separate model loop, with a fresh context, that the main loop starts for a focused subtask and from which it receives only a final report.

**SLM.** A small language model, here Qwen3.8-27B-FP8 (one workstation GPU) or MiniCPM5-2B (laptop-sized), served locally with vLLM.

**Prompt caching.** Providers charge a much lower price for input tokens that repeat a previous request's prefix. The paper prices cached input at $0.10 per million tokens versus $2 uncached, with output at $10, so cost is driven by how many frontier turns happen, not only by tokens.

**Cascade / router.** Methods that choose which model answers a whole query (FrugalGPT, RouteLLM).

Delegent keeps Claude Code's loop and subagents unmodified and adds four pieces.

1. **Role routing.** A relay that speaks the Anthropic Messages API sits between Claude Code and the models. Calls from the main loop go to the frontier model; calls from subagents and the auxiliary model go to the SLM. Every frontier call in a task carries one prompt-cache key. SLM calls go to vLLM through a proxy that pins each subagent to one GPU replica so its growing prefix stays cached.
2. **Delegation.** For coding, the frontier model starts subagents with Claude Code's Agent tool and writes a brief for each. For search, a Delegate tool runs a focused search subtask with prompts and tools borrowed from AREX-Turbo (a deep-research agent family built around a 4B model); its report holds the answer, quoted evidence and a confidence. The harness checks each quote against the cited document.
3. **Reports only.** The frontier context receives the subagent's final report, not its tool outputs. Subtasks in one turn run concurrently.
4. **Steering instruction.** A short instruction explains the economics: because every step re-reads the context, the cheapest plan has the fewest frontier steps of its own, so delegate tool-heavy work and keep own tool use to a few targeted checks.

The frontier model always writes the final patch or answer; no SLM output is accepted without the frontier model reading it.

### Why it works

The authors' account: the SLM takes over finding and checking, while the frontier model handles the main coding. Per DeepSWE task, Qwen3.8-27B-FP8 subagents issue about 24 code searches, read about 30 files or file parts, run the tests 12 times and run 5 scripts of their own. The frontier model receives findings it can verify cheaply, and tool output reaching it shrinks by 11% to 12% on DeepSWE.

The saving depends on how much legwork the frontier model would otherwise do itself. GPT-6.1-Sol takes 84 turns per DeepSWE task and delegates after it starts editing; Claude Opus 5.5 plans first, takes 18 turns and delegates mostly before its first edit.

My read: the ablation matters most. Roughly three fifths of the saving comes from the instruction that changes the frontier model's behavior, and two fifths from the cheaper subagent model. This is as much prompt design for the leader as it is a model-mixing result.

### Evidence

**Setup.** Frontier model GPT-6.1-Sol at high reasoning effort (ablations also with Claude Opus 5.5 at xhigh); baseline is the same harness with the frontier model alone, with prompt caching. Coding: samples of SWE-bench Verified, SWE-bench Pro and DeepSWE v1.1. Search: 694 questions (200 BrowseComp-Plus plus four CMASS datasets over ClimbMix). Search answers were graded by GPT-6.1-Sol at low effort.

| Result (authors' numbers) | Value |
|---|---|
| DeepSWE frontier cost per task | $2.11 alone, $0.70 with Qwen3.8-27B-FP8; no fewer tasks resolved |
| Search, 694 questions | 663 correct with delegation, as many as alone, at 41% of cost |
| Total DeepSWE cost with the 27B SLM priced at API rates | 38% of baseline (57% on SWE-bench Verified) |
| Median DeepSWE latency | 38 minutes versus 25 |

Savings by how long the task is for the frontier model alone: about 0.62 relative cost under 25 calls (and 11% of those tasks cost more) down to 0.26 above 200 calls.

Ablation on 50 DeepSWE tasks: without the instruction, the 27B SLM resolves 28 at 55% of the cost and the 2B SLM resolves 22, seven fewer than the frontier model alone; with the instruction both match it at a third of the cost. With subagents left on Sol, the instruction alone brings cost to 60%. On 30 DeepSWE tasks the 27B SLM alone resolves 5 and the 2B none. On BrowseComp-Plus over its own corpus, savings are smaller (63% of cost) because grep-style access returns only matching lines.

### Limitations and open questions

The authors state a reliance on Claude Code for coding and on AREX-Turbo's prompts for search, and that delegation adds latency bounded by local hardware.

My read: the 30- and 50-task ablations are small and I saw no confidence intervals. The search grader is itself a frontier model. Savings are largest for frontier models that iterate a lot; a planner-first model saved less (69% of cost). I did not read the appendix with the instruction text, so its wording is not reproduced here. Latency rose by about half on DeepSWE, which matters for interactive use.

### How to apply

**When:** your agent bill is dominated by long loops of reading, searching and test-running, and you have a GPU for a 2B-30B model.

1. Route subagent calls to a local model and keep the lead model for planning and the final answer.
2. Return only final reports with quoted evidence; verify quotes automatically where possible.
3. Add a short instruction telling the lead model that its own turns are the expensive part.
4. Keep one prompt-cache key per task and pin each subagent to one replica.

**Pitfalls:** without the instruction the 2B model lost tasks; a 2B subagent edited source files in 9 of 50 episodes, so restrict write tools.

**Smallest experiment:** run 20 tasks from your own backlog with and without delegation, and compare resolved count and frontier dollars; then repeat once without the instruction to see how much of the saving it carries.

Assessment: Try in an experiment. A training-free harness change with a two-thirds cost cut on long coding tasks, though the ablations are small and latency rises.

### Connections

Cascades (FrugalGPT) and routers (RouteLLM) pick one model per query; Minions has a cloud model decompose long-document tasks for a local model. The difference here is a harness-level role split for long-horizon agent loops, where the frontier model decides what to hand off, and no output is trusted unread.

**Connections.** Idea 3 (harness rehearsal) shows that models differ in how well they operate a given harness, which is why the 2B SLM needs the steering instruction here. Idea 4 (recovery harm) is a caution: subagents that write to external state inherit the same partial-execution risk. Idea 8 (summarizer reward) and this idea both shrink what the expensive model must re-read.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [Delegent: A Multi-Agent Harness That Reduces Cost without Sacrificing Performance](https://arxiv.org/abs/2611.33098) — Delegent routes subagent work to a local SLM while the frontier model plans and answers: DeepSWE cost per task $2.11 to $0.70 with no fewer tasks resolved, 694 search questions answered equally well at 41% of cost, and without the steering instruction the 2B subagent resolved 22 of 50 DeepSWE tasks versus 29 for the frontier model alone.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: +relation has-variant AG-20261127-07.
- 2026-11-26: created from bulletin 2026-11-26 (#2).
