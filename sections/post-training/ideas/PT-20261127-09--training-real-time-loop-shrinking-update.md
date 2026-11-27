---
id: PT-20261127-09
title: Training in a real-time loop with shrinking update intervals teaches agents when to think
tldr: Rollouts in a loop that keeps advancing with a [NULL] action when reasoning overruns, decoupled observation insertion and a shrinking update interval beat synchronous, budget-forced and dual-thread baselines by 33%.
section: post-training
also_in: []
subtopic: reasoning-training
type: method
tags: [agent-rl, reasoning-rl, long-horizon-agents, distribution-shift]
aliases: [RRT, real-time reasoning training, decoupled acting and observing, time-pressure curriculum]
status: emerging
scores: {actionability: 3, evidence: 4, generality: 3, leverage: 4, novelty: 4, surprise: 3}
power: 74
evidence: B
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.33513', title: Learning to Reason in Real-Time, role: origin}
related:
- {id: AG-20261126-07, rel: alternative-to}
superseded_by: null
---

## Summary

Agent training usually assumes a synchronous environment that waits while the model writes a long chain of thought; in real time the world advances on its own schedule, so a slow reasoner misses its moment while a forcibly short one reasons badly. The authors train on real-time trajectories: the policy is sampled asynchronously and, if the action is not ready within the environment's update interval, a [NULL] action is applied. Context is decoupled from action, so each new state is inserted even when the preceding action was [NULL], which stops the model from reasoning about a stale state. A curriculum starts at a long update interval of 1600 tokens and shrinks it geometrically every 40 steps, so the policy learns strategies under low pressure before pressure rises. Trained with GRPO on Qwen3-4B, the policy beats the strongest non-real-time baseline by 33% reward in-distribution and 55% out-of-distribution, and when the interval falls from 400 to 50 tokens it loses 18% of reward against 43% for S1 budget forcing. On Snake, the three components add about 48, 10 and 17 points over the synchronous baseline, summing to about 75. Use it when the environment cannot be paused and you control how often it updates. The main caveat is scope: one 4B model on two 2D games with discrete actions, with time measured in generated tokens rather than wall clock.

## Detail

### Problem

Most agent training uses synchronous environments: the world waits while the model generates a long chain of thought. In real-time settings the world changes on its own schedule. A model trained synchronously either reasons too long and misses its moment or, when forced to answer quickly, reasons badly. Existing fixes either wrap a fast and a slow model with a fixed ratio of thinking to acting, or cut the thinking budget by a rule, both ignoring that the right amount of thinking depends on the situation.

### Mechanism

**Real-time environment.** The environment advances every $\Delta_{\mathcal E}$ (here measured in generated tokens) regardless of whether the agent has produced an action. If the action is not ready, a [NULL] action is applied. Smaller $\Delta_{\mathcal E}$ means more time pressure.

**Reasoning tokens $r_t$ and action $a_t$** come from one sample of the policy.

**GRPO** (group-relative policy optimization) trains by comparing a group of rollouts and needs no value model. **S1 budget forcing** appends "wait" to extend reasoning or truncates at a token budget.

The authors call the class "real-time reasoning training" (RRT) and combine three techniques.

1. **Real-time trajectories.** Collect rollouts in the real-time loop (Algorithm 2), not the synchronous one: the policy is sampled asynchronously; if sampling finishes within the interval its action is applied, otherwise [NULL].
2. **Decoupled acting and observing.** Add each new environment state to the context as it is created, even when the action was [NULL]. Coupled context drops the state after a [NULL]: $C_t=H_t\setminus\{a_i,s_{i+1}:i\in\mathcal N_t\}$. Decoupled context keeps it: $C_t=H_t\setminus\{a_i:i\in\mathcal N_t\}$, where $\mathcal N_t$ is the set of earlier steps with a [NULL] action. This stops the model from reasoning about a stale state.
3. **Curriculum in time.** Start with slow environments and shrink the update interval by a fixed ratio every $\kappa$ steps: $\Delta^n=\Delta^{start}\rho^{\lfloor n/\kappa\rfloor}$ with $\rho=(\Delta^{end}/\Delta^{start})^{\kappa/N}$.

Training details: Qwen3-4B, GRPO with no KL penalty and asymmetric clipping (0.28 high, 0.2 low), 640 steps, batch 256, 8 rollouts per prompt, temperature 1.0, $\Delta^{start}=1600$ tokens, $\kappa=40$, 2×H200 GPUs.

### Why it works

The authors' account: synchronous-trained policies never experience being cut off, so they reason without limit. Real-time rollouts penalize slow reasoning, curriculum lets the policy develop good strategies under low pressure before pressure rises (they find training only at high pressure produces a poorer final policy), and decoupled observations let the policy revise its reasoning when the state changes. Their failure analysis splits errors into logic errors (wrong action) and real-time errors (the environment moved while the agent was idle). My read: the combination teaches adaptive thinking length without a separate controller, which is the clean contribution.

### Evidence

**Setup.** Two real-time games, Frogger and Snake, each with four task variants (one held out as out-of-distribution) and varying $\Delta_{\mathcal E}$. Each point averages 256 held-out seeds with 8 rollouts. Baselines trained on the same base model and settings: Synchronous, S1 budget forcing, Reactive (no reasoning), and AgileThinker (a dual-thread scaffold).

**Headline.** Averaged across time pressures, RRT outperforms the strongest non-RRT baseline by 33% reward on in-distribution tasks and 55% on out-of-distribution tasks.

**Robustness to time pressure.** When $\Delta_{\mathcal E}$ falls from 400 to 50, S1 loses 43% of its reward on in-distribution tasks and 54% out-of-distribution; the RRT agent loses 18% and 31%. Synchronous loses at least 97%; AgileThinker 60% and 52%.

**Additive ablation on Snake (Table 1).** Starting from Synchronous: real-time trajectories add about 48 points, decoupling adds about 10 (24 points at $\Delta=200$, 5 at $\Delta=50$), curriculum adds about 17 (25 at $\Delta=100$, 14 at $\Delta=50$); the sum is about 75 points on average.

**Failure analysis.** During training on in-distribution Frogger tasks (Figure 5), S1 starts with 60% of trajectories ending in a logic error (against 40% for RRT and Synchronous) and still has 50% at step 640, while RRT and Synchronous reach near 0%. On out-of-distribution Frogger (Table 2), Synchronous has real-time errors in 100% of trajectories at $\Delta=50$; S1 has none by construction but more logic errors as pressure rises; RRT wins 45% more often than S1 at the highest pressure.

### Limitations and open questions

Authors: the curriculum needs control over the environment's update interval, unavailable in the physical world; time is measured in tokens, not wall-clock; one 4B model on 2D games with discrete actions, so generalization is unclear. Mine: games are small, I did not see variance across training seeds, and S1 (the closest naive RRT baseline) is almost as good at low pressure, so the benefit is concentrated at high pressure. Real systems also have network and tool latency not captured by token counts.

My read: the headline "33%/55%" is relative to the strongest non-RRT baseline; S1 is itself trained under real-time constraints.

### How to apply

Use it when your agent acts in an environment that does not wait: games, trading, UI automation with animation, voice or robotics-adjacent simulators.

1. Wrap your environment in an asynchronous loop where the clock advances by tokens generated.
2. Insert state updates into the context as they happen; do not wait for the action.
3. Run RL (GRPO) starting at a long update interval and shrink it geometrically.
4. Track logic errors and real-time errors separately.

Cost: the paper used two H200s for a 4B model. Pitfall: if tokens are not a good proxy for latency in your deployment, measure the relationship first. **Smallest experiment:** reproduce Snake or a similar task with a 4B model and compare the three variants at two time pressures.

Assessment: Watch. Clear mechanism and ablations, but only a 4B model on simple games.

### Connections

Closest prior work, from the paper: AgileThinker (dual-thread reasoning plus fast action), which uses a predefined thinking/acting ratio; S1 and length-control methods (LCPO, BudgetThinker), which need the budget a priori; and difficulty-adaptive compute (TALE, Thinkless, AdapThink), which ignore time pressure. The authors claim to be first to use reasoning training to instill real-time awareness natively. The three techniques are simple; novelty lies in the real-time loop with [NULL] actions and the decoupling analysis.

**Connections.** Idea 4 (model-driven context) and idea 9 both give the model more control over its own resources (context and time) instead of a fixed rule. Idea 7 trades time for cost; here time is the scarce resource. Idea 3's per-turn credit could also apply when time and tokens differ between turns.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [Learning to Reason in Real-Time](https://arxiv.org/abs/2611.33513) — Real-time rollouts with [NULL] actions, decoupled observation insertion and a shrinking update interval beat the strongest non-real-time baseline by 33% in-distribution and 55% out-of-distribution reward on Frogger and Snake, and lose 18% instead of 43% (S1) when the update interval drops from 400 to 50 tokens.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: created from bulletin 2026-11-27 (#9).
