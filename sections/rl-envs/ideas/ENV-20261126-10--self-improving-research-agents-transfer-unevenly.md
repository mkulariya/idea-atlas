---
id: ENV-20261126-10
title: Self-improving research agents transfer unevenly, and learned rules can backfire on new tasks
tldr: Freezing four self-improving research agents after evolution and testing on independent tasks yields gains from −3.34% to +2.62%, task-level regressions everywhere, and stored anti-overfitting rules that discard inputs new tasks need.
section: rl-envs
also_in: []
subtopic: task-suites-benchmarks
type: finding
tags: [agent-evaluation, distribution-shift, benchmark-quality, harness-coupling]
aliases: [RecurSciBench, RecurSciPipe, RSI transfer benchmark, evolution gain]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 3, generality: 3, actionability: 4, surprise: 4}
power: 73
evidence: C
first_seen: '2026-11-26'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.32508', title: 'RecurSciBench: Benchmarking Recursive Self-Improvement in Data-Driven Scientific Research', role: origin}
related:
- {id: AG-20261126-09, rel: alternative-to}
- {id: AG-20261125-04, rel: combines-with}
- {id: AG-20261127-06, rel: alternative-to}
superseded_by: null
---

## Summary

Agents that accumulate experience — memory, skills, harness edits, even their own improvement procedure — are increasingly claimed to generalize, but a gain measured on the tasks an agent evolved on says little about the next task. The paper builds RecurSciBench: 58 scientific task families in four domains, 38 used for evolution and 20 independent modeling tasks for testing, with executable tasks produced by RecurSciPipe and checked by execution and scientific validity. Each system evolves on the evolution families, its state is selected using only evolution-task information and frozen, and initial and evolved copies both run the test tasks without carrying experience across tasks, scored so a reference solution gets 0.5. Across 24 evolved states from four systems on GLM-5.3-Flash, overall gains ranged from −3.34% to +2.62% with half negative, and every evolved state regressed on at least 8 of 20 test tasks; some states gained overall while regressing on 12. Case studies show the mechanism: HyperAgents stored a rule to submit the simplest candidate within one standard error of the best, which made runs submit a climatology (historical mean) on a half-hourly CO2 flux task and ignore meteorology, while EvoScientist learned forward-extrapolation validation after chemistry curve tasks and improved two physics tasks. Use it when you deploy or evaluate an agent that updates memory, skills or its own harness: freeze the evolved state, compare it to the initial state on independent tasks, and report task-level wins and losses with per-domain gains. The main caveat is scale: 20 test tasks with three runs each, mean scores near 0.50 by construction, task generation by an LLM pipeline and one backbone make many 1% to 2% differences indistinguishable from noise.

## Detail

### Problem

Agents that improve themselves by accumulating experience (memory, skills, harness edits, even their own improvement procedure) are increasingly claimed to generalize. But a gain measured on the tasks an agent evolved on says little about the next task. Evaluating transfer needs tasks that are executable, scientifically verifiable and independent of the source tasks, and it needs the evolved agent compared with its own initial state.

If you ship an agent that learns from experience, "average score went up" can hide systematic regressions on some task types, and rules that were protective on the source tasks can be harmful elsewhere.

### Mechanism

**Recursive self-improvement (RSI).** An agent updates its own persistent components from experience, with model weights here held fixed.

**Agent state.** $A^t=(R_t,M_t,H_t,I_t,\theta_t)$: solutions and code $R$, memory $M$, reusable harness of prompts, tools and workflows $H$, improvement strategy $I$ and model weights $\theta$.

**Within-type, cross-type, cross-domain transfer.** Evolve on tasks of one kind and test on the same kind, a different kind, or a different scientific domain.

**Evolution gain.** Percentage change in normalized task score between the evolved and initial agent, averaged over tasks.

**Frozen evolved state.** The state selected using only source-task information, then evaluated without further updates.

**RecurSciPipe** (using GLM-5.3) builds executable scientific tasks from raw datasets, with execution-based verification and scientific validity checks. **RecurSciBench** has 58 task families in four domains (Physics, Chemistry, Earth and space sciences, Life sciences): 38 families for evolution (G-evolve), each pairing a data analysis task (A) with a modeling task (F), and 20 families of independent modeling tasks for testing (G-test), 96 tasks in total.

**Protocol.** Each system evolves on G-evolve, updating its own subset of $R,M,H,I$. The evolved state $A^*$ is chosen using only evolution-task information and frozen. Independent copies of the initial state $A^0$ and $A^*$ run each G-test task without carrying experience across tasks, up to three valid runs.

**Score.** Per run, $q_{i,r}=\tfrac{e_i^{ref}}{e_i^{ref}+e_{i,r}}$, where $e_i^{ref}$ is the reference solution's gap to the metric optimum and $e_{i,r}$ the run's gap; the reference scores 0.5 and the optimum 1. The evolution gain is $\Delta^*=\tfrac1N\sum_i\tfrac{S_i^*-S_i^0}{S_i^0}\times100\%$.

**Experiment.** Four systems (EvoScientist, SIA-H, InternAgent-1.5, HyperAgents) on GLM-5.3-Flash with thinking and high effort; six source settings each (all F tasks, all A tasks, F tasks of each domain), giving 24 evolved states. Each task gets one A800 GPU and a six-hour limit with no network access.

### Why it works

The authors' explanation for uneven transfer is that source-task optimization makes particular choices persist beyond the conditions where they are useful. They recommend agents accumulate experience while keeping the flexibility to choose and adapt solutions, and that evolved states be evaluated on diverse tasks.

Their case studies show mechanisms. HyperAgents stored a rule after cross-validation overstated accuracy on source tasks: when candidates score within one standard error of the best, submit the simplest. On a half-hourly CO2 flux task with only three training years, a climatology (historical mean) falls within one standard error of the best candidate, so two runs submit it, ignoring meteorology. Conversely EvoScientist, after evolving on chemistry curve tasks, learned that validation should mimic the scored evaluation and validate by forward extrapolation, which helped two physics tasks that extrapolate.

My read: the persistent-rule failure is a general risk for memory-based agents and is invisible in an average.

### Evidence

**Setup.** 20 G-test modeling tasks, five per domain; scores normalized so the reference solution scores 0.5.

The authors report:

| Quantity | Value |
|---|---|
| Overall gain across the 24 evolved states | −3.34% to +2.62%; half negative |
| EvoScientist | positive in all six states (+0.39% to +2.24%) |
| SIA-H | negative in all six (−3.34% to −0.46%) |
| InternAgent-1.5 / HyperAgents | positive in four / two states |
| Regressions | every evolved state regresses on at least 8 of 20 tasks; some positive overall while regressing on 12 |
| Within-type (F-evolved) | EvoScientist +1.42%, InternAgent-1.5 +1.05%, HyperAgents −1.64%, SIA-H −3.34% |
| Cross-type (A-evolved) | EvoScientist +1.13%, HyperAgents +0.55%, InternAgent-1.5 −0.13%, SIA-H −2.74% |

Domain-level gains span −5.56% to +4.83% within type. HyperAgents' Life sciences mean falls from 0.4993±0.0030 to 0.4841±0.0034 within type. After chemistry-only evolution, EvoScientist's Physics score rises from 0.5050±0.0125 to 0.5411±0.0111 while Chemistry stays at 0.5008±0.0086 versus 0.5001±0.0114. EvoScientist's A-evolved harness writes an independent verifier in 38 of 60 test runs versus 2 of 60 for the initial agent, but its mean score stays within test variation (0.5053±0.0058 before, 0.5090±0.0035 after).

### Limitations and open questions

The authors present results as uneven and system-dependent rather than ranking the systems. They note that assessing evolution requires considering the agent system, the source of experience and the evaluation tasks together.

My read: with 20 test tasks and three runs per task, many gains of 1% to 2% are within noise; the overall mean scores sit near 0.50 by construction of the normalization. Tasks were generated by an LLM pipeline, one backbone model is used for all systems, and I did not read the appendix with the per-system evolution procedures. The authors' own conclusion is more cautious than a ranking, which is appropriate.

### How to apply

**When:** you evaluate or deploy an agent that updates memory, skills or its own harness from experience.

1. Freeze the evolved state and evaluate on independent tasks; always compare with the initial state on the same tasks.
2. Report task-level wins and losses and per-domain gains, not only the mean.
3. Test within-type, cross-type and cross-domain transfer separately.
4. Inspect stored rules that fire on the regressed tasks; look for protective rules that remove task-critical inputs.

**Pitfall:** averaging relative gains lets large gains on a few tasks hide losses on most.

**Smallest experiment:** take your agent's memory after a month of use, evaluate it against a fresh agent on 20 tasks it has not seen, and list tasks where it does worse.

Assessment: Watch. Useful evaluation design and cautionary findings, but small differences, few test tasks and one backbone model limit firm conclusions.

### Connections

Prior RSI work (the four systems evaluated here) mostly reports improvement on the tasks evolved on or on a single transfer setting. This benchmark compares initial and frozen evolved states on independent tasks across within-type, cross-type and cross-domain transfer, and reports task-level regressions.

**Connections.** Idea 9 (coding agents as optimizers) measures gains on a held-out split of the *same* benchmark; this idea shows how much weaker transfer to different tasks can be. Idea 1 (evidence closure) is a parallel point about measurement validity. Idea 3 notes that interface-specific training can interfere elsewhere, a similar regression pattern.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [RecurSciBench: Benchmarking Recursive Self-Improvement in Data-Driven Scientific Research](https://arxiv.org/abs/2611.32508) — Freezing four self-improving systems after evolution on 38 scientific task families and testing on 20 independent modeling tasks gave evolution gains from −3.34% to +2.62% (half of 24 evolved states negative), regressions on at least 8 of 20 tasks in every evolved state, and a stored simplicity rule that made runs submit a climatology on a CO2 flux task.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: +relation alternative-to AG-20261127-06.
- 2026-11-26: created from bulletin 2026-11-26 (#10).
