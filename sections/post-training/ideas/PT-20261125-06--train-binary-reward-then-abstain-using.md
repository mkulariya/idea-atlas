---
id: PT-20261125-06
title: Train with binary reward, then abstain using a conformal cutoff on answer-tail surprise
tldr: Freeze a reasoning model trained with plain correctness reward under a forced token cap, then submit an answer only when its answer-span tail token surprise falls below a conformal cutoff calibrated on held-out data.
section: post-training
also_in: []
subtopic: reasoning-training
type: recipe
tags: [budget-forcing, conformal-prediction, abstention, binary-reward, token-logprobs, reasoning-rl]
aliases: [Tail-CRC, single-rollout conformal risk control]
status: emerging
scores: {novelty: 3, leverage: 3, evidence: 4, generality: 3, actionability: 4, surprise: 3}
power: 66
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.31065', title: 'Decoupling Solving from Abstention: Single-Rollout Conformal Risk Control for Hard-Budget LLM Reasoning', role: origin}
related: []
superseded_by: null
---

## Summary

Under a hard token cap a reasoning model often never answers, and forcing it to stop thinking recovers answers but also exposes confident mistakes: on Qwen3-8B DAPO wrong submissions rose from 0.4% to 10.6% while correct ones rose from 7.6% to 28.6%. Tail-CRC splits the problem in two. Stage one runs Dr. GRPO in a budget-forcing environment with a terminal correctness reward and no wrong-answer penalty, because a penalty lets a weak solver avoid errors by exhausting the budget without submitting. Stage two freezes the model and scores the answer span by the mean of its largest 10% token surprises, computed from token log-probabilities that hosted APIs already return. Stage three calibrates a cutoff on the j-th smallest score among wrong calibration answers and submits only valid answers below it, giving an expected wrong-submission rate of at most the target risk level under exchangeability. On 8B Omni-MATH wrong submissions fell from 30.3% to 5.8% at a 4K budget and from 41.8% to 6.5% at 8K, and 50 RL updates raised correct submissions 15.0 to 29.6 points over inference-only budget forcing while cutting generation cost 3.5% to 12.6%. Use it when wrong answers cost more than abstentions, you serve under a hard cap, and the API exposes log-probabilities; the main caveat is that its gains over equally calibrated simpler baselines are small at some settings and the study covers two Qwen models on math only.

## Detail

### Problem

With a hard output cap, a reasoning model often runs out of tokens before answering: at a 4K cap, uninterrupted decoding gave no answer on 88.0% to 99.2% of problems for Qwen3.5-4B and Qwen3-8B across two math sets. Forcing the model to stop thinking recovers answers, but it also exposes confident mistakes: on Qwen3-8B DAPO, correct submissions rose from 7.6% to 28.6% while wrong submissions rose from 0.4% to 10.6%.

The usual fixes each cost something: supervised fine-tuning gives no error bound, wrong-answer penalties in RL can teach a weak solver to avoid answering at all, and multi-sample conformal methods multiply inference compute.

### Mechanism

**Budget forcing.** Inject an end-of-thinking delimiter when the token budget runs low so the model writes an answer instead of being truncated (the s1 strategy).

**Dr. GRPO.** A GRPO variant that centers rewards within each group of rollouts ($A_g=r_g-\bar r$) without dividing by the standard deviation.

**Token surprise.** The negative log-probability $\ell_t$ of a sampled token; high means the model found it unlikely.

**Conformal risk control (CRC).** Choose a score cutoff on held-out labeled data so that the expected loss (here, submitting a wrong answer) on a new exchangeable example is at most a chosen level $\alpha$.

**Exchangeability.** Calibration and test problems are drawn so that their order does not matter, which is what makes the finite-sample guarantee hold.

**Stage 1: learn to solve.** Run Dr. GRPO in the budget-forcing environment with terminal reward $r_0(Y)=\mathbb{I}\{Y=R\}$ (correct versus not), with no wrong-answer penalty and no KL term. Reported setup: 50 updates on 400 DAPO problems, 16 prompts with 8 rollouts each per update (6,400 trajectories per seed), rank-32 LoRA, 4K forced budget, sampling at temperature 1, top-p 0.95, top-k 20. Both thinking and answer tokens get the terminal advantage; prompt and injected delimiter tokens are masked.

**Stage 2: freeze and score.** For the answer span (tokens after the first end-of-thinking delimiter, before EOS), sort the $L$ token surprises in descending order $\ell_{(1)}\ge\dots\ge\ell_{(L)}$ and compute the tail score

$$S=\frac1k\sum_{t=1}^{k}\ell_{(t)},\qquad k=\lceil 0.1L\rceil .$$

Invalid or empty answers get $S=+\infty$. This uses only token log-probabilities that hosted APIs return.

**Stage 3: calibrate.** Generate and grade one trajectory per held-out problem ($n$ of them). With target $\alpha$, set $j=\lfloor (n+1)\alpha\rfloor$ and let $S^W_{(j)}$ be the $j$-th smallest score among wrong calibration answers; the cutoff is $\hat\tau=S^W_{(j)}$ (with edge cases for $j=0$ or too few wrong answers). At deployment, submit when the answer is valid and $S<\hat\tau$; otherwise abstain.

**Guarantee (Theorem 1).** Under exchangeability, the expected rate of wrong submissions on a test problem is at most $\lfloor (n+1)\alpha\rfloor/(n+1)\le\alpha$ at any solver accuracy. For $n=300$, $\alpha=0.065$: at most $19/301\approx6.31\%$.

### Why it works

**Why binary reward.** Under a hard cap, a weak solver can avoid a wrong-answer penalty by exhausting the budget without submitting, which starves it of attempts it needs to learn. The authors derive an exploration-starvation boundary in an attempt-solver learning model and show that wrong-answer penalties introduce failure-only comparisons into group-relative RL. Binary reward keeps every attempt informative.

**Why the tail.** A boxed answer can be a single confident token, but the answer span before it exposes the transitions used to build it. Averaging only the largest surprises concentrates on sparse "this step was a guess" signals; the authors give a sparse-signal model and a length-induced ranking reversal argument for why a mean over all tokens can mislead.

My read: splitting solving from abstention is sound design, because calibration can be redone for a new domain or risk target without retraining. The tail-versus-mean choice looks less settled (see limits).

### Evidence

Models: Qwen3.5-4B and Qwen3-8B at 4K and 8K caps; data: DAPO (400 train, 200 dev, 500 calibration, 500 test) and independent Omni-MATH (300 calibration, 300 test, calibration cutoffs locked before test labels were revealed).

- **Training supplies correct answers.** 50 RL updates on 400 problems raised correct-submission rates by 15.0 to 29.6 and 4.3 to 12.8 percentage points over inference-only budget forcing (the two ranges as reported in the abstract) while cutting generation cost by 3.5% to 12.6%.
- **Calibration controls errors.** On 8B Omni-MATH with target-domain calibration, wrong submissions fell from 30.3% to 5.8% at the 4K budget and from 41.8% to 6.5% at 8K.
- **Speed.** 4K generation ran 2.05 times faster for 4B and 2.23 times faster for 8B than unforced 8K base decoding (local amortized batch time).
- **Penalty RL baselines.** Tail-CRC led the three penalty-based RL baselines after matched post-hoc calibration (Table 2).
- **Honest caveats in the paper.** On 8B Omni at $\alpha=0.065$, Tail-CRC's 4K net-score gains over identically calibrated budget forcing and Answer-SFT are only 0.17 and 0.50 points; at 8K it keeps 4.0 points fewer correct answers than calibrated Penalty RL.

### Limitations and open questions

Authors' limits: calibration cannot create correct answers or separation absent from the score; the guarantee is marginal and averages over calibration and test draws, so a realized test panel can exceed the target; the study covers two Qwen generations and two math domains; Binary RL used two seeds and penalty arms one; the 10% tail fraction is not claimed universally optimal; Tail versus Mean NLL splits 18-18 across comparisons. With costly errors and an 8K budget, conservative native decoding stays competitive. Calibration needs labeled references.

My read: gains over equally calibrated simpler baselines are small at some settings, so most of the benefit may come from calibration itself, which is valuable on its own.

### How to apply

Use it when you serve a reasoning model under a hard token cap and wrong answers cost more than abstentions, and your API returns token log-probabilities.

1. If you cannot train, apply budget forcing plus the tail-score CRC step alone: grade 300 held-out problems, compute tail scores, set the cutoff for your risk target.
2. If you can train, run a short binary-reward Dr. GRPO stage in the same forced-budget setup (LoRA rank 32, 50 updates of 16 prompts by 8 rollouts is a small run).
3. Recalibrate on a few hundred labeled examples whenever the domain or risk target changes; the paper shows a source-domain cutoff can admit too many target errors.
4. Report correct, wrong and abstained rates together, not accuracy alone.

**Smallest experiment:** on your current model at your real token cap, force the end of thinking, record answer-span token log-probs for 300 labeled problems, and plot wrong-answer rate against the tail-score cutoff.

Assessment: Try in an experiment. The calibration step is cheap and has a real guarantee, but the training gain and the tail-versus-mean choice rest on two Qwen models and math only.

### Connections

Budget forcing comes from s1 (Muennighoff et al.); conformal risk control from Angelopoulos et al.; the authors' cited conformal LLM methods calibrate candidate sets or aggregate multiple reasoning paths and so spend extra samples. What is new here is the pairing: binary-reward RL inside the budget-forcing environment, an answer-span tail-surprise score computed from one rollout, and CRC on that score, plus the analysis of why penalties starve exploration.

Connections: it shares the "decouple training from a check" move with idea 11, which uses a separate rubric judge. It also contrasts with idea 8, which changes the advantage, not the abstention rule. The cutoff rule is a statistical version of the code-level answer check in idea 10.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [Decoupling Solving from Abstention: Single-Rollout Conformal Risk Control for Hard-Budget LLM Reasoning](https://arxiv.org/abs/2611.31065) — On Qwen3-8B Omni-MATH the calibrated tail-surprise cutoff cut wrong submissions from 30.3% to 5.8% at 4K (41.8% to 6.5% at 8K), while 50 binary-reward RL updates raised correct submissions by up to 29.6 points over inference-only budget forcing.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: created from bulletin 2026-11-25 (#6).
