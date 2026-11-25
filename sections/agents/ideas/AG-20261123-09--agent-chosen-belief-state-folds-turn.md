---
id: AG-20261123-09
title: Agent-chosen belief-state folds turn long research runs into rubric-rewarded segments
tldr: Letting a research agent rewrite its context into a four-field belief state at self-chosen checkpoints, then rewarding each resulting segment with an outcome-blind rubric plus validity gate, reaches 0.151 Brier on ForecastBench.
section: agents
also_in: []
subtopic: memory-context
type: method
tags: [context-folding, agent-rl, process-reward, credit-assignment, harness-coupling, long-horizon-agents, verifier]
aliases: [PBM, Proactive Belief-State Management, belief-state folding, SEIPH, segment rubric reward, DeepForecaster]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 3, generality: 3, actionability: 4, surprise: 3}
power: 72
evidence: C
first_seen: '2026-11-23'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.27286', title: Belief-State Management Meets Rubric Rewards for Agentic Forecasting, role: origin}
related:
- {id: PT-20261123-03, rel: alternative-to}
- {id: AG-20261123-06, rel: combines-with}
- {id: AG-20261125-07, rel: alternative-to}
- {id: PT-20261125-09, rel: alternative-to}
superseded_by: null
---

## Summary

Long research agents in a ReAct loop face two coupled problems: the context fills with stale evidence and old guesses, and the outcome reward arrives late and cannot teach process errors — an audit of 150 trajectories found trajectories with fabrication, time drift or misread evidence predicted the right outcome about as often as error-free ones. The paper gives the agent a fold tool: at checkpoints it chooses, the agent rewrites its context into a structured belief state (Foresight and Todo overwritten, Evidence and Search history appended) and the harness discards the raw messages, resetting context to the question plus the belief state. Each fold therefore also cuts the trajectory into segments, and a judge scores every segment on five rubric dimensions (Search, Evidence, Inference, Probability, History), multiplied by a hard validity gate that zeroes out fabrication, belief-state distortion, time drift, outcome leakage and invalid probabilities, with reward G·U−(1−G). Qwen3.6-35B-A3B is SFT'd on fold-style trajectories, then trained with PPO or GRPO on the segment rewards without ever seeing the resolved outcome. PPO reaches Brier 0.151 on ForecastBench (Qwen3.6 base 0.198, GPT-5.5 0.184, Kimi-K2.6 0.175) and 0.147 on Polymarket-Sim (DeepSeek-V4-Pro 0.153), though both remain behind the market at 0.134 with negative alpha. The fold harness alone already helps an untrained model (Polymarket-Sim Brier 0.221 to 0.193). Use it for long investigation agents whose context fills with stale material and whose final answer is a weak or noisy training signal. Caveats: the ReAct RL baselines start from the base model while PBM RL starts from the SFT checkpoint, so part of the gain may be SFT; runs are single with no confidence intervals on 126 and 229 questions; and the rubric judge GLM-5.2 is also a baseline.

## Detail

### Problem

Long research agents (forecasting, deep research, investigations) face two coupled problems. In a ReAct loop the context keeps growing, so stale evidence and old guesses crowd out new information. And the outcome reward arrives late and is noisy: in the authors' manual audit of 150 trajectories over 30 live Polymarket questions, trajectories with process errors (fabrication, time drift, misreading evidence) predicted the right outcome about as often as error-free ones. An outcome-only reward therefore cannot teach the agent to avoid those errors.

This paper solves both with one design: the same agent-decided checkpoints that keep the context clean also cut the trajectory into segments that can each be graded on process quality. For anyone training long-horizon agents where the final answer is a weak signal, it is a concrete recipe for turning one sparse reward into several dense, checkable ones.

### Mechanism

- **ReAct:** the standard agent loop (reason, act with a tool, observe, repeat) in one growing context.
- **Context folding / summarisation:** replacing raw history with a summary to stay within the context window; usually triggered by length, not by the agent.
- **Brier score:** mean squared error of probabilistic forecasts against 0/1 outcomes; lower is better.
- **Alpha (here):** mean over questions of the market's squared error minus the agent's; 0 means equal to the market, higher is better.
- **Rubric reward:** a judge model scores output against explicit criteria; *outcome-blind* means the judge never sees the resolved answer.

**1. Proactive Belief-State Management (PBM), the harness.**
- The agent has a *fold* tool it may call whenever it decides its current exploration stage is done.
- A fold writes a structured belief state $D_k$ with four fields: **Foresight** and **Todo** (overwritten each time: current forecast reasoning and next steps), **Evidence** and **Search history** (appended).
- The harness then discards the raw messages and resets the context to the question plus $D_k$.
- So a trajectory becomes segments $\tau_k=(q,D_{k-1},\xi_k,O_k)$, where $\xi_k$ is the search/visit/reasoning inside the stage and $O_k$ is the new belief state (plus the final probability $p_K$ in the last segment).

**2. Segment-level rubric reward (SEIPH).**
- A judge model scores each segment on five dimensions — **S**earch, **E**vidence, **I**nference, **P**robability, **H**istory — each with 3 criteria scored 0, 1 or 2, normalised to $U_k$.
- A hard validity gate $G_k\in\{0,1\}$ is 0 for fatal errors: fabricated facts, belief-state hallucination or distortion, target-time drift, outcome leakage, invalid probabilities.
- Reward: $r_k=G_kU_k-(1-G_k)$, i.e. the rubric score if valid and $-1$ otherwise.
- Training never uses the resolved outcome $a_i$.

**3. Training.**
- SFT of Qwen3.6-35B-A3B on PBM trajectories (3 epochs, learning rate $5\times10^{-5}$).
- Then PPO or GRPO with segment rewards: 4 trajectories per question, actor learning rate $5\times10^{-7}$, temperature 1.0; judge GLM-5.2.

The authors also give an idealised analysis: fold-stopping converges under a mirror-descent update, and a calibrated local rubric keeps the expected segment gradient while reducing the variance that later outcomes would add.

### Why it works

**Folding helps inference directly.** Rewriting the context into belief, evidence and todo forces the model to consolidate and to drop raw pages that no longer matter, so later reasoning conditions on a compact, current state. This alone improves an untrained model (Polymarket-Sim Brier 0.221 → 0.193).

**Segments make process rewards local.** Each segment starts from a known belief state and ends in a new one, so a judge can assess whether *this* stage searched well, used evidence correctly and updated the probability sensibly, without reading a 40K-token history or knowing the outcome. Local scores reduce the credit assignment problem from "which of 40K tokens caused a good Brier score?" to "was this stage good?".

**The gate targets the audited failures.** Fabrication, time drift and leakage are exactly the errors the audit found invisible to outcome rewards; making them $-1$ regardless of rubric score removes the incentive to cut corners.

My read: the agent choosing *when* to fold is what makes the segments meaningful; fixed-length chunks would cut across reasoning stages and give the judge less coherent units to grade.

### Evidence

**Benchmarks:** FutureX (3 weekly live evaluations), ForecastBench (229 resolved questions), Polymarket-Sim (126 questions, Aug 31-Sep 5, 2026). Baselines GPT-5.5, GLM-5.2, Kimi-K2.6, DeepSeek-V4-Pro/Flash with ReAct and the same tools.

**Main results (Table 1):**

| System | ForecastBench Brier ↓ | Polymarket Brier ↓ | Alpha ↑ |
|---|---|---|---|
| Qwen3.6 base | 0.198 | – | – |
| GPT-5.5 | 0.184 | – | – |
| Kimi-K2.6 | 0.175 | – | – |
| DeepSeek-V4-Pro | – | 0.153 | – |
| DeepForecaster-PPO | 0.151 | 0.147 | −0.012 |
| Human/market | – | 0.134 | 0 |

DeepForecaster-GRPO ranks second on both FutureX metrics. Calibration: all 26 PPO forecasts with ≥0.9 confidence were correct; GRPO 27 of 29.

**Ablations (Table 2, ForecastBench Brier / Polymarket Alpha):** ReAct+PPO (trajectory-level rubric) 0.197 / −0.104; ReAct+GRPO 0.179 / −0.028; PBM-SFT 0.192 / −0.028; PBM+PPO 0.151 / −0.012; PBM+GRPO 0.176 / −0.026. After RL the number of folds rises from 3.71 to 3.95 (PPO) / 4.22 (GRPO) and responses from 41.4K to 47.1K / 51.5K tokens.

### Limitations and open questions

- **Confounded ablation:** ReAct RL runs start from the base model while PBM RL starts from the PBM SFT checkpoint, so part of the PBM+RL advantage may come from SFT.
- **Single runs, small sets, no confidence intervals:** 126 and 229 questions; 0.147 vs 0.153 Brier is likely within noise.
- **Judge overlap:** GLM-5.2 is both the rubric judge and a baseline.
- **Still behind the market:** human/market Brier 0.134, Alpha negative.
- Theory is idealised (exact advantages, mirror-descent update, calibrated rubric).

My read: the PBM harness gain without training is the most transferable result; the RL gain is promising but needs a clean comparison from the same starting checkpoint.

### How to apply

**When:** long research or investigation agents whose context fills with stale material, and RL where outcomes are delayed or noisy.

**Harness first (no training):**
1. Add a `fold` tool with a fixed schema: Foresight/answer-so-far and Todo (overwrite), Evidence and Search history (append).
2. On fold, reset context to task + belief state. Let the agent decide when to call it; prompt it to fold at stage ends.

**Then training:**
3. Write a rubric with a few dimensions × 3 criteria (0/1/2) plus a validity gate listing fatal errors for your domain.
4. Score segments with an outcome-blind judge; reward $G\cdot U-(1-G)$; run PPO/GRPO from an SFT checkpoint on fold-style trajectories.

**Smallest experiment:** run your current agent with and without the fold tool on 50-100 tasks; compare final quality and tokens per task before investing in rubric RL.

Assessment: Try in an experiment. The fold harness alone improved an untrained model, and segment rubric rewards are a sensible next step, but the RL comparison is confounded and the sets are small.

### Connections

Closest prior work: context-folding and summarising agents (folding for context length), outcome-reward RL for forecasting, and rubric/process rewards for reasoning.

New: the agent's own fold decisions serve double duty as memory management and as RL segment boundaries, and an outcome-blind rubric with a hard validity gate rewards each segment. Neither part is new alone; the combination, and evidence that the harness and the training must go together, is.

**Connections.** Idea 3 also replaces sparse outcome rewards with per-step model judgements and is the closest training-side relative; DM-PRM uses one probability per step, PBM a rubric per segment. Idea 6 shows harness and weights co-adapt; here they are co-designed on purpose. Idea 7 warns that rubric graders mostly fail by penalising valid work, which applies to SEIPH too.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-23 · origin · [Belief-State Management Meets Rubric Rewards for Agentic Forecasting](https://arxiv.org/abs/2611.27286) — Agent-chosen belief-state folds cut ForecastBench Brier to 0.151 (Qwen3.6 base 0.198, GPT-5.5 0.184) and Polymarket-Sim to 0.147 with segment rubric RL, and the fold harness alone moved an untrained model from 0.221 to 0.193.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: +relation alternative-to PT-20261125-09.
- 2026-11-25: +relation alternative-to AG-20261125-07.
- 2026-11-23: created from bulletin 2026-11-23 (#9).
