---
id: PT-20261126-05
title: Rewarding actions against the simulator's exact posterior teaches agents when to stop searching
tldr: Reward each agent action by the environment's exact posterior-aware teacher's log-probability, comparing turns at the same history, so training stops premature guessing without a critic or judge.
section: post-training
also_in: []
subtopic: rl-algorithms
type: method
tags: [process-reward, credit-assignment, agent-rl, reward-shaping, privileged-information, long-horizon-agents]
aliases: [RLVB, reinforcement learning with verifiable beliefs, posterior-derived process reward, belief-conditioned process reward]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 3, actionability: 3, surprise: 4}
power: 75
evidence: B
first_seen: '2026-11-26'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.32884', title: 'Lazy Agents Answer Against Their Own Beliefs: Reinforcement Learning with Verifiable Beliefs Teaches Them to Act on What They Believe', role: origin}
related:
- {id: PT-20261123-03, rel: alternative-to}
- {id: PT-20261125-08, rel: alternative-to}
- {id: ENV-20261126-01, rel: combines-with}
- {id: PT-20261126-08, rel: alternative-to}
superseded_by: null
---

## Summary

Agents on long evidence-gathering tasks often commit to an answer before the evidence supports it: in the paper's cups task, after two lifts the object could be under either of two cups, yet base models frequently guess. An outcome reward cannot fix this, because a lucky guess earns the same as an evidence-backed answer, while a critic or LLM judge adds per-step credit at the cost of a second model or a call per step and can reward claims the evidence does not yet support. RLVB treats the exact posterior over the hidden answer, which a simulator knowing its own rules, prior and likelihood can compute, as a verifiable quantity: a teacher values each answer by roughly the log of its discounted chance of being right and each evidence action by the expected posterior after depth-n lookahead, and a sampled turn is rewarded by that teacher's log-probability of the chosen action. Credit is per history — each turn is compared only with other turns sampled at the same history, by subtracting their mean reward (or with an EM-style e^{R/alpha} weighting) — with DAgger-style teacher substitution starting at probability 0.5 and decaying, and LoRA adapters for up to 50 updates. Averaged over nine settings, RLVB beats the base model by 17.1% (EM variant 10.4%) and PPO with an LLM-judge belief reward by 9.3% at 1.6 times fewer GPU-hours, while GRPO and PPO on an outcome reward fall 1.0% and 5.1% below base; on Qwen3.5-4B, HiddenGames accuracy goes from 26.5 to 48.5. Use it in a simulator or synthetic environment whose hidden answer and observation model are known, when the failure is stopping too early or too late. The caveat is that it needs that exact posterior, and the analysis shows the gain comes from linking decision control to the agent's own belief line, not from more accurate belief lines or more informative steps; agents also stay far below the teacher (32.4% against 100% on Pro-Worlds).

## Detail

### Problem

Agents on long evidence-gathering tasks often commit to an answer too early; the paper calls this laziness. In the authors' cups task, an agent lifts cups one at a time to find a hidden object. After two lifts the object could still be under either of two cups, yet base models frequently guess, even after writing a belief that names both.

An outcome reward cannot fix this. It scores only the final answer, so a lucky guess earns the same reward as an answer backed by evidence, and a careful search that loses is penalized. A critic or LLM judge adds step credit, but costs a second model or a call per step and can reward a claim the evidence did not yet support.

### Mechanism

**Posterior.** The probability of each possible hidden answer given everything observed so far. A simulator that knows its own rules, the prior over answers and the likelihood of each observation, can compute it exactly.

**Process reward.** A reward given per step instead of only at the end.

**GRPO and PPO.** Policy-gradient methods. GRPO compares samples within a group; PPO trains a separate critic (value model) to estimate returns.

**DAgger.** Imitation learning where the learner's own visited states are labeled by an expert, and the expert's action replaces the learner's with a decaying probability.

**Expectation maximization (EM) learner.** Here, weighting sampled turns by $e^{R/\alpha}$ instead of subtracting a mean.

The method, **RLVB** (reinforcement learning with verifiable beliefs), treats the posterior as a quantity the simulator can verify at every step, just as RLVR verifies the final answer.

1. **Teacher.** At each history, a teacher values an answer $y$ by roughly the log of its discounted chance of being right, and an evidence action by the expected value of the posterior after its observation, using lookahead of depth $n$. The teacher answers with probability $\sigma(\Delta/\tau)$, where $\Delta$ is the best answer's value minus the best evidence action's, and otherwise picks an evidence action by a Boltzmann distribution.
2. **Reward.** $R(h,x)=\log\pi^*(u\mid b)$, the teacher's log-probability of the agent's chosen action $u$ at history $h$ with exact belief $b$. A guess while two answers remain scores about $-7.42$ in the paper's example; lifting another cup scores about 0. The reward never reads the belief line the agent writes before each action, although the update trains that line along with the action.
3. **Update.** Sample $N$ turns at each history. RLVB weights turn $i$ by its reward minus the mean reward of the other turns at the same history; RLVB (EM) weights by $e^{R_i/\alpha}$. A turn gets no positive weight if the teacher's probability for its action is below one twentieth of the teacher's best.
4. **Coverage.** DAgger-style teacher substitution starting at probability 0.5 and decaying.

Training uses LoRA adapters for up to 50 updates.

### Why it works

The key change is the credit assignment: each decision is compared only with alternatives at the *same* history, so a decision is credited against what it was worth under the same evidence, not against the luck of the episode. The reward depends on the evidence gathered, not on which answer happened to be right.

The authors' analysis of mechanism is careful. The trained agent does not write more accurate belief lines on average, and its steps add about as much evidence as the base model's. What changes is that it acts on its own line: it commits at only 0.1% to 2.2% of turns where the evidence is insufficient and its line still names several answers, against 3% to 68% for the base model with the same line. In their words, training mainly links decision control to the agent's own monitoring.

### Evidence

**Setup.** Qwen3-VL-4B-Thinking, Qwen3-VL-8B-Thinking and Qwen3.5-4B; training on HiddenGames, tests also on Pro-Worlds and RoboQuest, never seen in training. Three sampling seeds per test; baselines SFT on teacher actions, GRPO and PPO on outcome reward, and PPO with an LLM-judge belief reward (DeepSeek V4.1 Flash).

The authors report (Table 1, accuracy %, Qwen3.5-4B, base to RLVB): HiddenGames 26.5 to 48.5, Pro-Worlds 20.2 to 32.1, RoboQuest 39.6 to 54.0. Qwen3-VL-4B RoboQuest: 28.4 to 61.2.

| Comparison (average over nine settings) | Result |
|---|---|
| RLVB vs base | +17.1% (EM: +10.4%) |
| GRPO / PPO on outcome reward vs base | -1.0% / -5.1% |
| RLVB vs PPO with LLM-judge process reward | +9.3%, with 1.6 times fewer GPU-hours (4 times with EM) |
| Answers given with sufficient evidence | 51%, 12%, 74% vs 8%, 0.4%, 36% for base |

On HiddenGames, 34.5% to 43.1% of RLVB answers are premature, versus 58.9% to 94.3% for the judge-reward PPO. RLVB or its EM variant is best in eight of nine columns; the exception for the judge baseline is RoboQuest with Qwen3.5-4B. RLVB varies by up to 4% across training runs.

Ablations: adding the belief-line reward term does not raise accuracy; with only the belief term, accuracy equals the base-with-belief (HiddenGames 24.6 vs 24.5; RoboQuest 19.7 vs 34.0), so the action term carries the gain.

### Limitations and open questions

The authors state that the environment must know its rules to compute the posterior; the belief line expresses only uniform sets of answers; wrong answers given with enough evidence rise from 2% to 4% to 10% to 15% of episodes, mostly misreadings in Tabletop tasks; the trained agents' steps are not more informative (a teacher step raises the posterior by 5.9% versus 0.7% to 1.3%); and agents stay far below the teacher (32.4% against 100% on Pro-Worlds).

My read: LoRA with 50 updates and puzzle-style tasks may not carry to open-ended agents where no exact posterior exists. I did not read the appendix with the lookahead derivations.

### How to apply

**When:** you train an agent in a simulator or synthetic environment where the hidden answer and observation model are known, and the failure is stopping too early or too late.

1. Implement the environment's exact posterior over the hidden answer after each step.
2. Define a teacher that answers when the best answer's expected value exceeds continuing; use lookahead of a few steps.
3. Reward each sampled turn by the teacher's log-probability of its action; compare turns at the same history.
4. Mix DAgger-style teacher actions early; keep a belief line in the prompt so the model can act on it.

**Pitfalls:** it does not make the agent pick more informative steps; watch the rise in confident wrong answers.

**Smallest experiment:** on a twenty-questions or hidden-object toy environment, train a small model with outcome reward versus this reward and compare the share of answers given while two or more answers remain possible.

Assessment: Try in an experiment. Clear mechanism and consistent gains in simulated evidence-gathering tasks, but it needs an exact posterior and was tested only on puzzle-style environments.

### Connections

Prior work passes simulator privileged information to a teacher or a critic (asymmetric actor-critic and similar). Process reward models and LLM judges score steps against a learned or judged notion of correctness. The difference here is that the reward is an exact function of the evidence collected, computed by the environment, with no learned component, and credit is assigned per history rather than per episode.

**Connections.** Idea 1 (EviCraft) warns that simulator state must not leak into the agent's tools; this idea uses simulator state only to compute the reward, which is consistent. Idea 3 uses GRPO-style group comparison with an outcome-style reward. Idea 8 (Delta-KL) is another outcome-free per-step signal, built from a frozen model's own predictions instead of a simulator.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [Lazy Agents Answer Against Their Own Beliefs: Reinforcement Learning with Verifiable Beliefs Teaches Them to Act on What They Believe](https://arxiv.org/abs/2611.32884) — On three Qwen policies and three hidden-answer environments, rewarding each turn by an exact-posterior teacher's log-probability and comparing turns at the same history averaged +17.1% accuracy over base versus -1.0% (GRPO) and -5.1% (PPO) for outcome rewards, and +9.3% over PPO with an LLM-judge process reward at 1.6 times fewer GPU-hours, with +17.1% varying by up to 4% across runs.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: +relation alternative-to PT-20261126-08.
- 2026-11-26: created from bulletin 2026-11-26 (#5).
