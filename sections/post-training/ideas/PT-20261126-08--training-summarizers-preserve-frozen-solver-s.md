---
id: PT-20261126-08
title: Training summarizers to preserve a frozen solver's short-context predictions extends reasoning horizons
tldr: Reward a summarizer by how well a frozen solver's next-token predictions under its summary match those under the full reasoning prefix, extending effective horizon without answers or a verifier.
section: post-training
also_in: []
subtopic: reasoning-training
type: method
tags: [context-folding, long-horizon-agents, reasoning-rl, kl-divergence, cost-reduction]
aliases: [Delta-KL, TruSuCo, truncate-summarize-continue, summary-preserving KL reward]
status: emerging
scores: {actionability: 4, evidence: 4, generality: 3, leverage: 4, novelty: 4, surprise: 3}
power: 76
evidence: B
first_seen: '2026-11-26'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.31621', title: Self-Improving Long-Horizon Reasoning by Learning Summaries from Local Continuations, role: origin}
related:
- {id: PT-20261126-05, rel: alternative-to}
- {id: AG-20261125-07, rel: alternative-to}
superseded_by: null
---

## Summary

Reasoning models degrade as their context grows, so longer traces do not by themselves give longer effective horizons; one remedy is to compress the reasoning so far into a compact state and continue from it. TruSuCo cuts the solver's trace at a 4096-token window, keeps the last 128 tokens verbatim, summarizes the earlier tokens into at most 512 tokens, and appends question plus summary plus tail so the solver reads them as its own reasoning. The summarizer is trained with Delta-KL: sample 8 continuation probes from the frozen solver under the full prefix, then reward a candidate state by how closely the solver's teacher-forced token log-probabilities under the state match those under the full prefix, with positions reweighted by the tail-copy baseline and a penalty at the length cap. Training uses GRPO for 80 steps on about 2.5k math questions with 4k-token prefixes and no answers. With a 4B solver, average math accuracy at a 32k budget rises from 40.4 single to 46.5 trained (AIME 2024-2026: 53.4 to 61.1), and trained summaries beat Single and the untrained summarizer in all nine settings; the gain over the untrained summarizer on math is at most about 2 points. Use it when you run long traces with a smaller model, have no verifier, and context collapse is the failure. The main caveats are that the signal preserves whatever the solver's short-context behavior is, training was 5 to 6 H100-hours with no visible run-to-run variance, and RC or Delethink come within a point in several cells.

## Detail

### Problem

Reasoning models degrade as their context grows, so longer traces do not automatically give longer effective horizons. One remedy is to compress the reasoning so far into a compact state and continue from it. Existing ways to train that state use the final-answer reward, which needs verifiers or answers and does not isolate what the state contributed, or distill it from a stronger model. Neither measures the state's own quality.

An outcome-free signal would let you improve long-horizon reasoning on prompts with no ground truth, using only the model you already have.

### Mechanism

**Solver and summarizer.** The solver is the frozen reasoning model; the summarizer is a separate model that writes a compact summary of the solver's reasoning so far.

**KL divergence.** A measure of how different two probability distributions are, here over next tokens.

**Teacher forcing.** Scoring fixed token sequences by feeding them in and reading the model's log-probabilities, instead of sampling.

**GRPO.** A policy-gradient method that scores a group of samples and uses reward minus group mean as the advantage.

**Markovian state.** A summary is Markovian if, given it, future reasoning is nearly independent of the discarded history.

**avg@32.** Accuracy averaged over 32 sampled runs per problem.

**Inference: TruSuCo (truncate, summarize, continue).** The solver reasons until its local window reaches $K=4096$ tokens. The trace is cut: the last $k=128$ tokens are kept as a verbatim tail, and the earlier $K-k$ tokens are summarized into at most 512 tokens. The solver continues from the question plus summary plus tail, appended to its open assistant turn so it reads them as its own reasoning. In *append* mode all summaries are concatenated; in *roll* mode the summarizer also reads the previous summary and replaces it.

**Training: Delta-KL.** For each question, take the frozen solver's first $K$ reasoning tokens as the window $a$. Sample $J=8$ continuation probes $y^{(j)}\sim\pi_{so}(\cdot\mid Q\oplus a)$ from the full prefix once and reuse them. To score a candidate state $s$ (summary plus tail), teacher-force the probes under both contexts:
$$\Delta_{j,i}(s)=\log\pi_{so}(y^{(j)}_i\mid Q\oplus s\oplus y^{(j)}_{<i})-\log\pi_{so}(y^{(j)}_i\mid Q\oplus a\oplus y^{(j)}_{<i}).$$
Averaging estimates negative KL between full-prefix and state-conditioned continuations, but strongly favors simply copying the tail. So the authors reweight positions by $w_{j,i}=\max(0,-\Delta_{j,i}(s_{tail}))$, which emphasizes positions where the full prefix predicts better than the tail alone; clip positive differences to zero; and normalize per probe. A constant penalty $\lambda$ applies if the summary hits its length cap. The summarizer is updated with GRPO (batch 32 questions, 8 summaries each, 80 steps). Training data: about 2.5k math questions with 4k-token prefixes; no answers are used.

### Why it works

The authors' hypothesis is that a model that degrades over long contexts still behaves reliably in short ones, so the model conditioned on a short context can act as its own teacher. A good state should keep the solver's continuation distribution approximately unchanged.

In a synthetic task of 64 independent multiplications the solver without truncation collapses (below 0.05 accuracy on the last problems), and merely copying the last tokens already helps. Trained summaries come closest to an oracle statement that says which problems are done: 90.0% of trained summaries state which problems are done, against 51.8% of untrained ones. The untrained summarizer often writes a self-contained question, and the solver then solves it in isolation and believes the task has ended.

My read: the reweighting that stops the reward from rewarding tail copying is the crucial design choice; the paper motivates it but I did not locate an ablation of it in the sections I read.

### Evidence

**Setup.** Solvers DeepSeek-R1-Distill-Qwen-1.5B and Qwen3-4B-Instruct-2507; summarizers Qwen3-1.7B and Qwen3-4B-Instruct. Evaluation: 220 math problems (90 AIME 2024-2026, 100 BeyondAIME, 30 HMMT Feb 2025) and Reasoning Gym puzzles (192 per task); 32k solver-token budget, avg@32 with standard errors. Baselines: Single (vanilla), Base (untrained summarizer), Reasoning Cache (RC), Delethink.

| 32k budget, accuracy % | Single | Base | RC | Delethink | Trained |
|---|---|---|---|---|---|
| Math all, 4B solver | 40.4 | 45.0 | 45.9 | 44.6 | 46.5 |
| AIME 24-26, 4B solver | 53.4 | 59.6 | 60.2 | 58.6 | 61.1 |
| Math all, R1-1.5B | 16.1 | 17.8 | 17.9 | 17.3 | 19.0 |
| Futoshiki | 45.2 | 34.2 | 42.4 | 45.9 | 46.1 |
| Sokoban | 37.5 | 34.0 | 39.9 | 35.7 | 41.6 |

Trained beats Single and Base in all nine settings and is best in seven; it is within one point of the best in the other two. Gains reach 7.7 points over Single (AIME 4B) and 12 over Base (Futoshiki). Over Base on math the gain is at most about 2 points. On Sokoban and Futoshiki the untrained summarizer falls below Single.

At a 128k budget, summarizers trained on 4k prefixes exceed Single by 8.0 points on math with the 4B solver and 4.0 with R1; on AIME 2024-2026 the 4B configuration reaches 63.3%, 10.3 above Single. Training takes 5 to 6 hours on three H100 GPUs for 80 steps.

### Limitations and open questions

The authors note that over the untrained summarizer the math gain is at most about 2 points, because that summarizer is already useful on math.

My read: results use two small solvers and two summarizer sizes, I saw no variation across training runs, and RC or Delethink come within one point of Trained in several cells. No code link is stated. The approach assumes the solver's short-context behavior is good; if it is not, the reward preserves a bad distribution. The 4k-token training prefix is short compared with the 128k test horizon, which the evidence supports but only for these tasks.

### How to apply

**When:** you run long reasoning traces with a smaller model, no verifier is available, and context collapse is the failure.

1. Sample unfinished 4k-token reasoning prefixes from the solver on a few thousand prompts and store 8 continuation probes each.
2. Score candidate summaries with the reweighted Delta-KL reward against the tail-copy baseline; add a length-cap penalty.
3. Train a 1.7B-4B summarizer with GRPO for about 80 steps.
4. At inference use windows of 4k tokens, tail of 128, summary of 512, append mode with a sink variant for long budgets.

**Cost:** about 5 to 6 H100-hours at the 4B scale.

**Smallest experiment:** run Single versus untrained-summary TruSuCo versus a summarizer trained for a few dozen steps on 500 prompts, at 16k and 32k budgets.

Assessment: Try in an experiment. A clean outcome-free signal with consistent but modest gains on small solvers; the benefit over untrained summarization on math is small.

### Connections

Closest prior work, named in the paper: InftyThink, the Markovian Thinker (Delethink), Reasoning Cache and Memento, which compress or truncate reasoning and train with final-answer rewards or external teachers. The difference is the training signal: the frozen solver's own short-context predictions on stored continuation probes, requiring no answers, verifiers or stronger model.

**Connections.** Idea 6 (InstructReplay) also keeps a model close to its own distribution. Idea 5 (RLVB) is another outcome-free, per-step signal, but from a simulator. Idea 2 (delegation) and this idea both cut what a costly model must re-read each step; a summarizer could compress subagent histories.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [Self-Improving Long-Horizon Reasoning by Learning Summaries from Local Continuations](https://arxiv.org/abs/2611.31621) — Training a summarizer with a reweighted KL-to-full-prefix reward over the frozen solver's continuation probes raised math accuracy at a 32k budget from 40.4 (single) to 46.5 with a 4B solver and from 16.1 to 19.0 with R1-1.5B, beating Single and the untrained summarizer in all nine settings, with at most about 2 points over the untrained summarizer on math.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: created from bulletin 2026-11-26 (#8).
