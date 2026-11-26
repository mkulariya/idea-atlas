---
id: PT-20261125-08
title: Contrasting SAE features of correct and incorrect rollouts gives GRPO token-level credit
tldr: Score each token by whether its SAE features match correct rather than incorrect rollouts of the same problem, then add that to the outcome advantage in GRPO.
section: post-training
also_in: []
subtopic: rl-algorithms
type: method
tags: [credit-assignment, process-reward, reward-shaping, reasoning-rl, agent-rl, binary-reward]
aliases: [SAEPO, SAE-guided credit assignment, sparse-autoencoder credit assignment]
status: emerging
scores: {novelty: 4, leverage: 3, evidence: 4, generality: 3, actionability: 2, surprise: 3}
power: 67
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.29936', title: 'SAEPO: Sparse-Autoencoder-Guided Credit Assignment for Policy Optimization', role: origin}
related:
- {id: PT-20261123-03, rel: alternative-to}
- {id: PT-20261125-03, rel: alternative-to}
- {id: PT-20261126-05, rel: alternative-to}
superseded_by: null
---

## Summary

In outcome-supervised RLVR one scalar reward per rollout gives every token the same advantage, so a decisive reasoning step and a filler word get identical credit and the update is noisy. SAEPO splits each question into correct and incorrect rollouts, encodes every generated token target-layer hidden state with a frozen BatchTopK SAE, and matches tokens across rollouts by cosine similarity of their sparse features rather than by position. Averaging those matches over the other correct rollouts and over the incorrect ones gives a semantic score, down-weighted when the group is lopsided, which is standardized over the batch and added to the outcome advantage with weight alpha. Everything else (verifier reward, ratio, clipping, KL) is unchanged, and alpha=0 recovers vanilla training. On Qwen3-1.7B-Base the six-benchmark math average rises from 35.56% to 38.68% with GRPO, and gains hold on Qwen3-4B-Base and OLMo-3-7B-Base under both GRPO and GSPO. Replacing SAE features with raw hidden states hurts and flipping the sign of the score degrades sharply, while controlled edits that introduce a local error lower the score and fixes raise it. The main caveat is that it needs a pretrained SAE for each model and layer, and questions the policy always or never solves get no signal.

## Detail

### Problem

In outcome-supervised RL with verifiable rewards (RLVR), one scalar says whether the final answer was right, and every token of the rollout gets the same advantage. A decisive reasoning step and a filler word receive identical credit, so the update is noisy and slow, especially on long reasoning traces.

Existing token-credit methods (the paper compares KTAE, THR and GTPO) try to fix this, but they work from the policy's own statistics or token-level heuristics rather than from what the model represents internally.

### Mechanism

**RLVR and GRPO.** Reinforcement learning against a checkable reward such as a math answer. GRPO samples $G$ rollouts per question and uses the group-normalized reward as the advantage, the same for every token in a rollout: $A^{\text{out}}_{qg}=(r_{qg}-\mu_q)/(\sigma_q+\epsilon)$.

**GSPO.** A variant that uses a length-normalized sequence-level importance ratio; the paper uses its token-wise form so per-token advantages can be substituted.

**Advantage.** How much better than expected an action or sequence was; it scales the policy-gradient push.

**Sparse autoencoder (SAE).** A network trained to reconstruct a hidden activation from a wide, sparse code. Its features are often more interpretable than raw dimensions. Here a BatchTopK SAE is pretrained and kept frozen.

For each question $q$ with $G$ rollouts:

1. Split rollouts into correct set $\mathcal C_q$ and incorrect set $\mathcal I_q$. If either is empty, the question gets no SAE signal that update.
2. Encode each generated token's target-layer hidden state with the SAE: $\phi_{qg,t}=f_{\text{enc}}(H_{qg,t})$.
3. For token $t$ of rollout $g$, compare it with every token $u$ of another rollout $h$ of the same question using a softmax over cosine similarities with temperature $\tau$: $\omega=\mathrm{softmax}_u(\cos(\phi_{qg,t},\phi_{qh,u})/\tau)$ and $m^{(h)}_{qg,t}=\sum_u\omega\cos(\cdot)$. This matches tokens by meaning, not position.
4. Average $m$ over the other correct rollouts to get $s^+$ and over the other incorrect ones to get $s^-$. The semantic score is $S_{qg,t}=\rho_q\,(s^+_{qg,t}-s^-_{qg,t})$ with $\rho_q=2\min(|\mathcal C_q|,|\mathcal I_q|)/(|\mathcal C_q|+|\mathcal I_q|)$, which down-weights lopsided groups.
5. Standardize $S$ over the batch to get $\hat S$ (unscored tokens get 0) and shape the advantage: $A_{qg,t}=A^{\text{out}}_{qg}+\alpha\,\hat S_{qg,t}$.

Everything else (verifier reward, ratio, clipping, KL) is unchanged, and with $\alpha=0$ the update is vanilla. The SAE is used only during training; inference needs nothing extra. Training data: DeepScaleR-Preview (about 40K math problems) and the DeepCoder-Preview code sets; reward is math_verify correctness plus a small boxed-format bonus. I did not find the values of $\alpha$, $\tau$ or the SAE layer in the text I read.

### Why it works

A token that looks like tokens in correct rollouts of the same problem, and unlike tokens in incorrect ones, probably carries a step that matters; sparse features make that comparison cleaner than raw hidden states. The authors' visualization shows high-scoring features tied to question-relevant concepts and low-scoring ones on connectives, punctuation and stop words.

They also show the signal is not a copy of the outcome: inside a correct trace some stop words get negative final advantages, and inside an incorrect trace some relevant tokens get positive ones.

My read: this is a contrastive, within-question estimate of "which tokens separate winners from losers", so it is a heuristic for credit rather than a causal estimate, and the paper's edit experiments are the main check.

### Evidence

Backbones: Qwen3-1.7B-Base, Qwen3-4B-Base, OLMo-3-7B-Base, with GRPO and GSPO; six math benchmarks (MATH-500, GSM8K, AIME25, AMC, Minerva Math, OlympiadBench) and two code benchmarks (LiveCodeBench-v5, LeetCode v0.3.1).

- **Math averages (Table 1).** Qwen3-1.7B-Base: GRPO 35.56% to 38.68%, GSPO 36.63% to 39.55%. Qwen3-4B-Base: GRPO 48.43% to 49.97%, GSPO 47.73% to 50.15%. OLMo-3-7B-Base: SAEPO reaches 41.92% (GRPO) and 43.07% (GSPO) versus 38.75% and 40.13% for the strongest credit baseline. The paper reports gains up to 6.63% (math) and 5.57% (code) and faster early training accuracy on Qwen3-1.7B-Base.
- **Ablations (Table 2, Qwen3-1.7B-Base, GRPO).** Replacing SAE features with raw hidden states hurt; flipping the sign of the score caused a large degradation.
- **Controlled edits.** Over 976 problems and 1,864 rollouts (904 correct, 960 incorrect), edits that introduce a local error lowered the semantic score, and fixes raised it (confidence intervals exclude zero). Continuation success after an introduced error fell by 0.57 on average (mean from 0.74 to 0.17, success lowered in 95% of edits); after a fix it rose by 0.17 (0.15 to 0.32, raised in 62%).

### Limitations and open questions

Not stated as limits by the authors in the text I read, so this is my read: SAEPO needs a pretrained SAE for each model and layer, which exists for few models; it needs both correct and incorrect rollouts per question, so questions the policy always solves or never solves get no signal; it compares all token pairs across rollouts, which adds compute; and the models are small (1.7B to 7B). The full ablation was done on the smallest backbone only. Seeds and variance were not visible in the text I read. The controlled-edit experiments use minimal edits on rollouts filtered for clear step boundaries.

The sign-flip ablation shows the direction matters, which is reassuring, but it does not show the signal beats a simple length or position prior.

### How to apply

Reach for it only if you already have, or can train, an SAE for your policy and you are bottlenecked by sample efficiency on reasoning RL.

1. Train or obtain a BatchTopK SAE for one mid-to-late layer of your policy; freeze it.
2. In your GRPO or GSPO loop, after grading rollouts, compute the semantic score for questions with both outcomes and add $\alpha\hat S$ to the advantage.
3. Start with small $\alpha$ and monitor whether the reward curve speeds up early; the paper reports faster early gains.
4. Keep an ablation with the sign flipped or with random scores to confirm the signal is real in your setting.

**Smallest experiment:** run your current GRPO recipe on a 1.7B base model for 200 steps with and without SAEPO, three seeds each, and compare the six-benchmark average.

Assessment: Watch. Gains hold across three backbones and beat three credit baselines, but the method needs an SAE per model and I could not verify variance or key hyperparameters.

### Connections

The paper's closest prior work is token-level credit methods KTAE (Sun et al., 2025), THR (Deng et al., 2026) and GTPO (Tan and Pan, 2025), plus process-reward approaches; all give finer credit than GRPO's broadcast advantage. The difference claimed here is the source of the signal: a frozen SAE's feature space compared across correct and incorrect rollouts of the same question, with no learned value model or process-reward model.

Connections: idea 3 argues that credit-assignment cost is what limits PG on long tasks; SAEPO is one way to cut it inside PG. It competes with the unmodified-PG-plus-engineering route cited there. It resembles idea 9 only loosely: both add a signal beyond one scalar outcome, one internal and one from a rubric.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [SAEPO: Sparse-Autoencoder-Guided Credit Assignment for Policy Optimization](https://arxiv.org/abs/2611.29936) — Contrasting frozen SAE features of correct and incorrect rollouts and adding the standardized score to the GRPO advantage lifts the six-math-benchmark average on Qwen3-1.7B-Base from 35.56% to 38.68% with GRPO and 36.63% to 39.55% with GSPO, while raw-hidden-state and sign-flip ablations both hurt and controlled edits move the score in the expected direction.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: +relation alternative-to PT-20261126-05.
- 2026-11-25: created from bulletin 2026-11-25 (#8).
