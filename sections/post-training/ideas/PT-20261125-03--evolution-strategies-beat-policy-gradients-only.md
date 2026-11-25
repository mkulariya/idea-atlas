---
id: PT-20261125-03
title: Evolution strategies beat policy gradients only when effective horizon is long and skills few
tldr: 'ES and policy gradients estimate nearly the same gradient but pay for different quantities: ES scales with the number of independent skills, PG with reward-relevant tokens, so ES wins only on long, few-skill tasks.'
section: post-training
also_in: []
subtopic: rl-algorithms
type: finding
tags: [credit-assignment, long-horizon-agents, agent-rl, agent-fine-tuning, evolution-strategies]
aliases: [ES versus PG, effective horizon, effective dimension, ES fine-tuning]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 4, actionability: 3, surprise: 4}
power: 78
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.29795', title: 'Dimension Is Free, Horizon Is Not: When Evolution Strategies Rival Policy Gradients for LLM Fine-Tuning', role: origin}
related:
- {id: PT-20261123-03, rel: alternative-to}
- {id: PT-20261125-08, rel: alternative-to}
superseded_by: null
---

## Summary

Teams choosing a post-training method for agent tasks have no rule for when to leave GRPO-style policy gradients, even though terminal-reward episodes of ten thousand tokens have needed token masking, larger rollout groups, stabilized actor-critic training or tool-level credit. This paper argues the two estimators pay for different things: ES variance grows with the effective dimension r·k (independent skills times decisions per skill), while PG variance grows with the effective horizon k + γ(r)βF, the reward-carrying tokens plus filler tokens weighted by score overlap with the reward directions and by how much terminal credit they share. Filler tokens therefore cost PG and not ES, while each extra independent skill costs ES in full. In a multi-skill combination-lock testbed PG is cheaper with no filler, and as filler grows PG cost climbs while ES stays flat until the two cross; the crossover moves to longer horizons as skills increase and is set by effective horizon rather than filler count. The horizon estimate ranks 20 NLP, math and agentic tasks by the measured noise-to-signal ratio of a PG update at rank correlation +0.83 (+0.55 within the 14 tasks where both methods were trained). In equal-sample training on Qwen2.5-7B-Instruct, 14 tasks, both methods tuned and three seeds, PG was ahead or tied everywhere, yet vanilla ES with none of the usual PG engineering reached a higher Terminal-Bench 2.1 success rate than the released Tmax PG checkpoint, on rollouts averaging about 18,000 tokens with a measured effective horizon near 7,300. Use the framework to decide whether an ES pilot is worth running when reward is terminal, the horizon is thousands of tokens and the base model already earns some reward, comparing at matched wall-clock and GPU hours rather than rollout counts. The main caveat is that the long-horizon win rests on one task and one base model against a released checkpoint rather than a matched tuned run, and the skill count cannot be measured on tasks the base model rarely solves.

## Detail

### Problem

Teams choosing a post-training method for agents have no principled rule for when to leave GRPO-style policy gradients. Terminal-style agent tasks with 10,000-token episodes and a single end-of-episode reward are exactly where PG needs heavy engineering: the paper cites Tmax, which needed masking of tokens whose training and inference probabilities disagree plus larger rollout groups, and T1 and TRACE, which needed stabilized actor-critic training or tool-level credit.

This paper gives a variance argument for when a method with no credit assignment at all (ES) should catch up, and tests it. It also gives a way to measure how far a task is from that regime.

### Mechanism

**Policy gradient (PG).** Sample token sequences from the policy, weight each sequence's score function by its reward (or advantage), and average. Noise comes from the *tokens*: every generated token enters the gradient, including those that do not affect the reward.

**Evolution strategies (ES).** Perturb the model's parameters with Gaussian noise, score each perturbed model, and combine the perturbations weighted by returns. It needs only forward passes and no credit assignment. Noise comes from the *parameters*.

**Terminal credit.** A single reward at the end of a rollout, shared by all tokens, as in outcome-reward fine-tuning.

**Effective dimension and horizon.** The paper's two key quantities, defined below.

**Core argument.** At fixed parameters $\theta$, with $D$ parameters, $T$ generated tokens and per-token score energy $\kappa=\mathbb{E}\|u_t\|^2$ (where $u_t$ is the gradient of the log-probability with respect to the output distribution's own parameters), the naive estimator variances scale as

$$\operatorname{tr}\mathrm{Cov}(\hat g_{ES})\propto D,\qquad \operatorname{tr}\mathrm{Cov}(\hat g_{PG})\propto T\,\kappa .$$

For a categorical policy $\kappa=1-\sum_v p_v^2\le 1$ whatever the vocabulary size, so PG noise grows with token count, not vocabulary.

Only noise along directions that change the reward slows learning. That gives:

- ES cost grows with the **effective dimension** $D_{\mathrm{eff}}=r\cdot k$: the number of independent skills $r$ times decisions per skill $k$.
- PG cost grows with the **effective horizon** $T_{\mathrm{eff}}=k+\gamma(r)\,\beta F$: the $k$ reward-carrying tokens plus filler tokens $F$ weighted by how much their score overlaps the reward-relevant directions ($\gamma$) and by how much terminal credit they share ($\beta$; zero under per-token credit).

**Measuring $T_{\mathrm{eff}}$ on real tasks.** Approximate the reward-relevant directions by the span of log-likelihood gradients of reference answers, then count each generated token by how much of its score energy lies in that span.

**Testbed.** A multi-skill combination lock: $r$ independent skills, each with $k$ decision positions and $F$ filler positions, reward equal to the fraction of correct decisions, a shared MLP policy. Both methods use 64 rollouts per update (ES: 32 antithetic pairs with rank-shaped returns), both tuned. Cost is rollouts until expected reward reaches 0.9.

### Why it works

Under terminal credit, every token's score is multiplied by the same advantage, so tokens that do not affect the reward still add variance, but only to the extent their gradients overlap the reward-carrying directions. With per-token credit that overlap term vanishes, which is why better credit assignment removes PG's horizon cost while shorter generations need not.

ES perturbs in parameter space, so its noise does not depend on how many tokens were generated; but isotropic noise fills every direction, so each independent skill costs it in full, while PG's decision-token noise stays in the skill that produced it.

The authors state the two costs have the same form (bounded noise times a count), so neither can be cheaper everywhere. My read: this is a clean, testable variance picture, but "effective dimension" can only be estimated where the base model already earns some reward, as the authors admit.

### Evidence

- **Testbed.** PG is cheaper with no filler; as filler grows, PG cost climbs while ES stays flat until they cross (Figure 5). The crossover moves to longer horizons as skills increase, and is set by $T_{\mathrm{eff}}$ rather than filler token count. With per-token credit PG cost does not rise. REINFORCE, RLOO, GRPO and Dr.GRPO share the same effective horizon (Appendix B) although their total variances differ.
- **Measurement.** Across 20 NLP, math and agentic tasks, the horizon estimate has rank correlation +0.83 with the measured noise-to-signal ratio of a PG update (+0.55 within the 14 tasks where both methods were trained).
- **Equal-sample training, Qwen2.5-7B-Instruct, 14 tasks, each method tuned, three seeds.** PG is ahead or tied on every task. PG with all weights trained (instead of adapters) changes the language-task scores by at most 0.03 except AG News. ES made progress on every agentic task with a population of at most 32.
- **Terminal-Bench 2.1, Qwen3.5-4B.** Rollouts average about 18,000 tokens and the measured effective horizon is about 7,300, roughly sixty times ALFWorld's. Vanilla ES, with none of the usual PG engineering, reached a higher success rate than the released Tmax PG checkpoint (five evaluations each; comparison from Schweighofer et al., 2026). The authors call it preliminary and note the budgets cannot be put on equal terms.

### Limitations and open questions

Authors' stated limits: the regime where ES should win has barely been tested; the Terminal-Bench comparison is preliminary and uses attainable performance because rollout and compute budgets are not comparable; the skill count cannot be measured on long tasks that the base model rarely solves, so the "few skills" condition is unverified exactly where the horizon condition holds; ES needs infrastructure that runs every perturbed model on full multi-turn episodes.

My read: the headline long-horizon evidence is a single task and base model, with the PG side being a released checkpoint rather than a matched, tuned run. The practical message is the diagnostic, not "use ES".

### How to apply

Use the framework to decide whether an ES run is worth a pilot, not as a reason to drop PG.

1. Estimate your task's effective horizon: compute reference-answer gradients, then measure how much of each sampled token's score lies in their span. Compare with ALFWorld-like (short) or Terminal-Bench-like (thousands of tokens) values.
2. Check you have a learning signal: if the base model almost never earns reward, neither method will learn.
3. If horizon is in the thousands of tokens, reward is terminal, and your base model succeeds sometimes, run an ES pilot with a population of 32 antithetic pairs next to your PG baseline.
4. Compare at matched wall-clock and GPU hours, since rollout counts mislead (ES rollouts need only a forward pass).

**Smallest experiment:** run the combination-lock testbed (or a copy) with your policy size to see where PG and ES cross for your $k$ and $r$, before touching a real agent task.

Assessment: Watch. The theory is clean and the short-task results are solid, but the long-horizon win rests on one preliminary comparison.

### Connections

ES at LLM scale was shown competitive by Qiu et al. (2026), and Liang et al. (2026) explained why via a few high-curvature directions; the long-horizon advantage of ES was discussed by Salimans et al. (2017) and Vemula et al. (2019) analyzed dimension, action dimension and horizon. The authors add a controlled separation of weight sharing from credit assignment, a way to place real tasks on one horizon axis, and trained comparisons with both methods tuned in the same environment.

They also flag that another tuned PG run on the same Countdown setup beat both of Qiu et al.'s methods, so earlier ES-versus-PG results depend heavily on baseline tuning.

Connections: it is the algorithmic counterpart to idea 8, which attacks the same horizon problem by adding token-level credit to PG instead of leaving PG. If this paper's picture holds, idea 8 helps in the regime where PG already wins and ES is the fallback beyond it. Idea 6 shows how PG-style training can starve when rewards are sparse, the same signal condition this paper lists.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [Dimension Is Free, Horizon Is Not: When Evolution Strategies Rival Policy Gradients for LLM Fine-Tuning](https://arxiv.org/abs/2611.29795) — A combination-lock testbed shows the PG-versus-ES crossover is set by effective horizon, not filler token count, and the horizon estimate ranks task PG noise-to-signal at +0.83; vanilla ES reached a higher Terminal-Bench 2.1 success rate than the released Tmax checkpoint on roughly 18,000-token rollouts, while equal-sample Qwen2.5-7B runs favored PG on all 14 tasks.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: +relation alternative-to PT-20261125-08.
- 2026-11-25: created from bulletin 2026-11-25 (#3).
