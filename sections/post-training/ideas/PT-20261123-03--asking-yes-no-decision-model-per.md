---
id: PT-20261123-03
title: Asking a yes/no decision model per step gives label-free process rewards for agent RL
tldr: Per step of every rollout, ask a probability-returning decision model whether the step made progress, standardise scores within the task's rollout group, and add them to GRPO advantages; ALFWorld success rises from 66.3% to 90.0%.
section: post-training
also_in: []
subtopic: rl-algorithms
type: method
tags: [credit-assignment, process-reward, agent-rl, verifier, reward-shaping]
aliases: [DM-PRM, decision-model process reward, decision-model PRM]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 3, actionability: 5, surprise: 3}
power: 78
evidence: B
first_seen: '2026-11-23'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.27068', title: Decision Models Are General-Purpose Process Reward Models, role: origin}
related:
- {id: AG-20261123-01, rel: alternative-to}
- {id: AG-20261123-09, rel: alternative-to}
- {id: PT-20261125-03, rel: alternative-to}
- {id: PT-20261125-08, rel: alternative-to}
superseded_by: null
---

## Summary

Outcome-rewarded agent RL gives one number per episode, so GRPO credits every step of a rollout equally and learns nothing when a whole group of rollouts fails. This method asks a decision model, once per step and once per rollout, whether that step made progress toward the task, and uses the returned probability as a step reward: scores are standardised across all steps of the task's rollouts and added as a weighted extra term to the GRPO advantage, with no step labels, critic or fine-tuning. On Qwen2.5-1.5B-Instruct, ALFWorld success rises from 66.3% to 90.0% on seen rooms and from 63.7% to 92.9% on unseen rooms over GRPO, with paired 95% intervals well above zero, and it also improves WebShop and stacks on GraphGPO (94.0 / 93.6). When task-completing action kinds score below the mean, a per-kind baseline is needed instead of a pooled one, as on WebShop. Cost is about USD 0.12 and ten seconds per iteration of 128 rollouts. Use it for multi-turn agent RL with sparse outcome rewards, especially when most task groups fail. Caveats: one 1.5B policy on two simple text environments, one proprietary scorer behind an API, and the judge sees the outcome and is not policy-invariant.

## Detail

### Problem

RL from verifiable outcomes (RLVR: rewarding a model only for a checkable final result) gives a multi-step agent one number per episode. GRPO credits every step of a rollout with the same value, so a fifty-step episode cannot tell its useful moves from its wasted ones. Worse, when every rollout of a task fails, the group has no signal at all; in this paper's first training iteration that happened in 9 of 16 task groups on ALFWorld and 14 of 16 on WebShop.

Existing fixes each cost something: PPO trains a critic; GiGPO, HGPO and GraphGPO help only where rollouts revisit identical states; trained process reward models need step labels; LLM judges are slow and return text; hand-written rules are gamed.

### Mechanism

- **Advantage:** in policy-gradient RL, how much better an action was than expected; the gradient pushes up the log-probability of tokens in proportion to it.
- **GRPO (Group Relative Policy Optimization):** samples $G$ rollouts per task and sets each rollout's advantage to its reward minus the group mean, divided by the group standard deviation; no critic is needed.
- **Process reward model (PRM):** a model that scores intermediate steps, not just the final outcome. Usually trained on human or rollout-derived step labels.
- **Decision model:** a model that answers closed questions with probabilities rather than free text (here Jev 1.13 from TypeSafe, called through an API, used as provided).

**Setting.** For task $x$ the policy samples $G$ rollouts; rollout $i$ has steps $s_{i,1},\dots,s_{i,K_i}$ (a step = one model output plus the observation that follows). GRPO gives every step of rollout $i$ the same advantage $A^{\text{RL}}_{i,k}=(R_i-\text{mean}_j R_j)/\text{std}_j R_j$, where $R_i$ is the episode outcome.

**Scoring.** After each iteration's rollouts, send each rollout to the decision model as one request: the task, the first observation, and the log as "action → observation" lines (observations cut at 400 characters; the last line states whether the episode succeeded). For each step $k$, ask one question: *considering the task and everything earlier in the log, did this step make progress toward completing the task?* The reply is a probability $p_{i,k}\in[0,1]$. The ALFWorld question spells out what counts (e.g. opening an unsearched place that could hold a needed object) and what does not (no effect, a repeat, handling an unneeded object).

**Credit.** Standardise against all steps of the task's $G$ rollouts:

$$c_{i,k}=\frac{p_{i,k}-\mu_x}{\max(\sigma_x,\sigma_{\min})},\qquad A_{i,k}=A^{\text{RL}}_{i,k}+w\,c_{i,k}$$

where $\mu_x,\sigma_x$ are the mean and standard deviation of all step scores for task $x$, $\sigma_{\min}=0.1$ prevents inflating near-identical scores, and $w$ weights the credit. The credit is added after the method's own normalisation and exists even when all rollouts fail. $w=0$ recovers the base method.

**Per-kind baseline (when needed).** Before any update, compute the mean score of each kind of action in iteration 1. If the actions that finish a task score *below* the overall mean (WebShop: an untrained agent buys wrong products, so "buy" scores low and "search" high), the pooled baseline would teach the agent to browse and not buy. Then compare each step only with steps of the same kind: $c_{i,k}=(p_{i,k}-\mu_{x,\kappa(i,k)})/\max(\sigma_x,\sigma_{\min})$, where $\kappa$ is (page type, action type). The term then ranks instances within a kind and leaves the choice between kinds to the outcome.

**Settings.** Qwen2.5-1.5B-Instruct, verl-agent, 150 iterations × 16 tasks × 8 rollouts, learning rate $10^{-6}$, KL weight 0.01. ALFWorld: pooled baseline, $w=1$. WebShop: per-kind baseline, $w=0.3$.

### Why it works

The added term is a policy gradient with the step scores as rewards and zero discount: each action is credited only for its own step. Because the signal judges the step itself rather than comparing rollouts that share states, it is defined in every group, including all-fail groups where GRPO learns nothing.

The authors check that scores are not just echoing the outcome (which the judge sees): on ALFWorld the kind of step explains several times more score variance than the outcome, and a no-effect action scores the same in won and lost episodes.

Why a *model* rather than a rule: a rule credit (0 for no effect, repeats or invalid actions, 1 otherwise) is gamed, because "look", "inventory" and "examine" satisfy the rule while doing nothing for the task, so the agent learns to spam them. The decision model scores such steps near zero; idle actions fall from 23% of steps to 3%.

My read: the question text is a natural-language shaped reward, made robust to gaming because the model judges relevance, not surface form.

### Evidence

**Setup.** ALFWorld (420 seen and 420 unseen evaluation episodes, three training runs per main method) and WebShop (500 episodes).

**Main results (Table 1/6, success %, means over runs):**

| Method | ALFWorld seen | ALFWorld unseen | WebShop success |
|---|---|---|---|
| PPO (critic) | 54.4 | – | 51.5 |
| GRPO | 66.3 | 63.7 | 59.1 |
| GiGPO | 86.3 | 86.6 | 66.1 |
| GraphGPO | 88.6 | 88.3 | 73.8 |
| DM-PRM + GRPO | 90.0 | 92.9 | 72.2 |
| DM-PRM + GraphGPO | 94.0 | 93.6 | 75.9 |

Paired differences over episodes: +23.7 points over GRPO on seen rooms (95% interval 17.3 to 30.3) and +29.2 on unseen (23.4 to 35.1). DM-PRM + GRPO reaches 80% validation success at iteration 70, GiGPO at 120, GRPO not within 150. On WebShop test goals never used in training, GRPO 69.2 → 77.0.

**Ablation (Table 2, ALFWorld iteration 60, one run each, seen success):** GRPO 37.1; rule credit 11.4; DM-PRM 72.1; $w=0.3$ 77.1; DM-PRM + GiGPO 80.7.

**Cost:** about USD 0.12 and ten seconds to score an iteration's 128 rollouts; under USD 20 per run and under 2% of training time; ALFWorld runs use over a third fewer environment steps than GRPO because episodes end sooner.

### Limitations and open questions

- **One policy size and two benchmarks:** Qwen2.5-1.5B only; ALFWorld and WebShop are standard but simple text environments.
- **One closed scorer:** "decision models" in the title rests on a single proprietary model (Jev); training depends on an external API.
- **The judge sees the outcome**, so scores are not independent of it; the authors' variance analysis mitigates but does not remove this.
- **Per-environment engineering:** question text, log format, weight and possibly a hand-designed kind taxonomy.
- **Not policy-invariant:** unlike potential-based shaping, the term can change what the policy converges to.

My read: the ALFWorld gains are large and well-supported (three runs, paired intervals, strong baselines). I would expect smaller margins on harder, longer-horizon tasks where "progress" is harder to judge locally.

### How to apply

**When:** multi-turn agent RL with sparse outcome rewards, especially when most groups fail.

**Steps:**
1. Log each rollout as task + first observation + "action → observation" lines (truncate observations).
2. Write one progress question per environment naming what counts and what does not.
3. Score all steps with a probability-returning judge (a decision model, or an LLM's logprob of "yes").
4. At iteration 1, before any update, compute mean score per action kind. If task-completing kinds score below the mean, use per-kind baselines.
5. Add $w\cdot c$ to your advantages after normalisation; start with $w=1$ (pooled) or $w=0.3$ (per-kind), $\sigma_{\min}=0.1$.

**Smallest experiment:** in your existing GRPO agent setup, run 50 iterations with and without the credit on the same task seeds, and compare validation success at iterations 10-50 plus the share of no-op or repeated actions.

Assessment: Try in an experiment. Large, well-controlled gains at negligible cost make it worth a run in any sparse-reward agent RL setup, though shown with one small policy and one proprietary scorer.

### Connections

Closest prior work: GiGPO, HGPO and GraphGPO (step advantages from recurring states), trained agent PRMs, PPO with GAE, and LLM-as-judge step scoring.

The precise difference: a zero-shot, label-free step score from a model that returns calibrated probabilities, added as a separate advantage term, so it plugs into any policy-gradient method and works where no states recur and no rollout succeeds. The diagnostic for choosing the credit baseline before training is also new and generally useful.

**Connections.** Idea 1 (same group, same decision model) uses per-action probabilities to select actions at test time; here they shape learning. Idea 9 also replaces sparse outcome rewards with process scores per segment, using a rubric judge; the two approaches combine naturally. Ideas 7 and 10 warn that automated verifiers fail in specific ways; the rule-credit result here is a small instance of reward gaming.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-23 · origin · [Decision Models Are General-Purpose Process Reward Models](https://arxiv.org/abs/2611.27068) — Zero-shot per-step progress probabilities from a decision model, standardised within the task's rollout group and added to the GRPO advantage, lift Qwen2.5-1.5B ALFWorld success from 66.3% to 90.0% seen and 63.7% to 92.9% unseen, and 72.2% WebShop over GRPO's 59.1%, while a hand-written rule credit instead drops to 37.1% at iteration 60.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: +relation alternative-to PT-20261125-08.
- 2026-11-25: +relation alternative-to PT-20261125-03.
- 2026-11-23: +relation alternative-to AG-20261123-09.
- 2026-11-23: created from bulletin 2026-11-23 (#3).
