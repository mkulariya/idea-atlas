---
id: AG-20261127-06
title: Evolve harness interventions from success-failure pairs and gate them by replay selectivity
tldr: TransHarness evolves a coding harness by authoring triggered interventions from success-failure pairs, keeping those a replay gate finds selective and a reviewer finds grounded, lifting DeepSWE from 73.2 to 76.4.
section: agents
also_in: []
subtopic: architectures-scaffolds
type: method
tags: [scaffold-search, coding-agents, agent-evaluation, distribution-shift, harness-coupling]
aliases: [TransHarness, replay-gated harness evolution]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 3, actionability: 4, surprise: 3}
power: 76
evidence: B
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.33757', title: 'TransHarness: From Benchmark Overfitting to Transferable Harness Evolution', role: origin}
related:
- {id: ENV-20261126-10, rel: alternative-to}
- {id: AG-20261126-09, rel: alternative-to}
- {id: AG-20261127-08, rel: has-variant}
- {id: AG-20261127-11, rel: alternative-to}
superseded_by: null
---

## Summary

Evolving a harness from past runs can improve an agent without touching weights, but candidates selected on the benchmark they evolve on gain there and lose elsewhere; the authors' reproduction of Meta-Harness improves on its own evolution set yet drops on DeepSWE v1.1. TransHarness represents a policy as a triple of harness location, executable trigger and diagnostic guidance, authored by an LLM from a success-failure pair of the same task under the base harness, with diagnoses that yield no reusable principle declined. A reviewer either refines the wording or requests one rework, and executable checks validate the code and record where it activates on successful and failed source histories. A replay gate then re-runs each trigger over trajectory pairs from other source tasks and keeps a policy only if it does not fire almost everywhere and does not single out successful runs. A separate semantic selector, without the replay statistics, checks that the diagnosis is supported, that the guidance does not depend on the source task's solution, and that corrective actions are grounded in failed checks or explicit task requirements; survivors are composed and frozen before any target-task evaluation. In the same-model comparison on DeepSWE with the base Mini-SWE harness, TransHarness scores 76.4 against 73.2 for the base, 69.0 for Meta-Harness, and 73.5 for Self-Harness and RRSI, with +13.0 recovery against −9.7 regression, and Qwen Code rises from 55.8 to 63.7. Use it if you maintain a coding-agent harness and have a pool of paired past runs, freezing the result and reporting recovery and regression separately on a different benchmark. Caveats: all methods are LLM-driven with method-specific search budgets, scores are means of three runs without variance reporting, part of the evaluation compares against public leaderboard settings, and the semantic selector is an LLM judging the author's own work.

## Detail

### Problem

A harness is the code around a model: how it calls tools, reads feedback and decides it is done. Evolving a harness from past runs can improve an agent without touching weights. The common failure is overfitting: methods that select candidates on the benchmark they evolve on gain there and lose elsewhere. The authors' own reproduction of Meta-Harness improves on their independent evolution set but drops on DeepSWE v1.1.

For anyone tuning an agent scaffold on a fixed task suite, this is the practical risk: the gains may be an artifact of that suite's recurring patterns.

### Mechanism

**Intervention.** A rule that, when a trigger fires on the agent's execution context, injects guidance at a harness location (a model call, a tool, a context update, or completion).

**Recovery and regression.** Over paired runs of the same task, recovery is the share of tasks the base harness failed and the evolved one passes; regression is the share the base passed and the evolved one fails. Net gain is recovery minus regression.

**Replay.** Re-running a candidate intervention's trigger over recorded trajectories from other tasks to see where it would fire, without new agent runs.

Each policy is a triple $u=(s,g,c)$: harness location $s$, executable trigger $g(h_t,e_t)\in\{0,1\}$ on history $h_t$ and event $e_t$, and diagnostic guidance $c$. Evolution runs over 300 source trajectories (a 100-task set outside the test distribution, three runs each).

1. **Author from contrasts.** From a success-failure pair $(\tau^+,\tau^-)$ of the same task under the base harness, an LLM author diagnoses where decisions diverged and derives a reusable principle. Diagnoses with no reusable principle are declined.
2. **Review.** A reviewer refines wording or requests one rework; executable checks validate the code and record activation on successful and failed source histories.
3. **Replay gate.** Over $N$ trajectory pairs from other source tasks, count activation on failed only ($n_-$), successful only ($n_+$), or both ($n_\pm$). Keep a policy only if $n_\pm/N<0.9$ and $n_-\ge n_+$: it must not fire almost everywhere, and must not single out successful runs.
4. **Independent semantic selection.** A separate judge, without the replay statistics, checks three things: the diagnosis is supported, the guidance does not depend on the source task's solution, and corrective actions are grounded in failed checks or explicit task requirements.
5. **Compose and freeze.** Surviving policies are composed in source order per interface and frozen before any target-task evaluation.

An example: after tests pass, the policy prompts the agent to audit requirement coverage (a completion intervention). A first version said "fix any failures", which forced repair of unrelated failures; review changed it to fix failures caused by the current change and report unrelated ones.

### Why it works

The authors give a decomposition. For an intervention $u$ with exposure $a_D=P_D(A_u)$ (the chance it affects a run) and conditional value $v_D$, the source-to-target change is $\Delta_Q-\Delta_P=(a_Q-a_P)v_P+a_Q(v_Q-v_P)$: transfer can fail because activation shifts or because the guidance is wrong in new settings. Replay tests the first, semantic selection the second. They also show the net effect is positive only if $q<\frac{1-p_0}{p_0}r$, where $p_0$ is base success, $r$ the rescue rate and $q$ the regression rate, so a strong baseline tolerates little regression. My read: this is the useful insight; a harness patch must be judged on regressions as much as on recoveries.

### Evidence

**Setup.** Targets: DeepSWE v1.1 (113 tasks) and Terminal-Bench 2.1 (89 tasks); means over three runs; models DeepSeek V4.1 Flash (1017), GLM-5.3 Flash, Qwen3.8-Flash-Next; lightweight harnesses (Mini-SWE, Terminus 2) and native ones (DeepSeek Harness, ZCode, Qwen Code).

**Table 2 (same model, source trajectories, base Mini-SWE harness on DeepSWE).**

| Method | Score | Recovery | Regression |
| --- | --- | --- | --- |
| Base harness | 73.2 | | |
| Meta-Harness | 69.0 | +12.7 | -16.8 |
| Self-Harness | 73.5 | +13.3 | -13.0 |
| RRSI | 73.5 | +11.5 | -11.2 |
| TransHarness | 76.4 | +13.0 | -9.7 |

**Table 1.** Qwen Code on DeepSWE rises from 55.8 to 63.7 (+7.9); ZCode from 60.2 to 61.9 (+1.7); the DeepSeek Harness from 70.8 to 75.2 on DeepSWE and from 89.9 to 95.5 (+5.6) on Terminal-Bench 2.1. The paper's headline scores of 76.4 and 95.5 are 2.4 and 5.6 points above GPT-6 Astra (74.0 and 89.9 under lightweight harnesses).

**Ablations (Table 4, DeepSWE).** Without learned triggers 72.6; without evidence-based reassessment (a prescribed correction instead) 65.5; without both 66.4; without construction review 67.3 (fourteen unreviewed interventions against seven reviewed).

### Limitations and open questions

The evolution set is 100 tasks and 300 trajectories; all methods in the comparison are LLM-driven with method-specific search budgets, which may favor one. Scores are means of three runs with no variance reporting in the tables I saw. Gains per harness are small for some (ZCode +1.7). Part of the evaluation uses public leaderboard scores under their own reported settings. The semantic selector is an LLM judging the author's work, so its errors are not independent of the author's.

My read: the 76.4% state-of-the-art claim compares against public entries under different settings; the same-model Table 2 comparison is the cleaner evidence.

### How to apply

Use it if you maintain a coding-agent harness and have a pool of past runs.

1. Collect paired successes and failures of the same task under the current harness.
2. Draft interventions as (trigger, guidance) pairs; make guidance a question or audit step, not a fixed fix.
3. Replay each trigger on other tasks' trajectories; discard those with $n_\pm/N\ge0.9$ or $n_-<n_+$.
4. Have a second reviewer without the replay numbers check diagnosis, source-independence and grounding.
5. Freeze, then evaluate on a different benchmark, and report recovery and regression separately.

Cost: mainly LLM calls for authoring and review; not stated here. **Smallest experiment:** write three completion-time audit interventions from your own failures, run the replay gate, and measure net gain on a held-out set of tasks.

Assessment: Try in an experiment. A principled way to reduce harness overfitting, with clear ablations, though much of the headline comparison is against public numbers.

### Connections

Closest prior work, named by the authors: Meta-Harness (search over code, scores and traces), Self-Harness (checks regressions), RRSI (regularized selection), ModularRSI (same-task contrasts, module-restricted updates, review), and DGM, which tests cross-benchmark transfer. TransHarness differs by coupling context-sensitive triggers with diagnostic (not prescriptive) guidance, evaluating selectivity by replay before deployment, and evolving on an independent set instead of the target benchmark. In the ablations, removing evidence-based reassessment (-10.9) or construction review (-9.1) costs the most.

**Connections.** Idea 11 (HoMe) is a harness-search method for memory that also relies on failure diagnosis and cheap evaluation; the two could share selection ideas. Idea 4 shows harness behavior, such as compaction timing, depends on the model, so evolved interventions need per-model validation. Idea 1 is a reminder to test selection gains against shuffled labels.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [TransHarness: From Benchmark Overfitting to Transferable Harness Evolution](https://arxiv.org/abs/2611.33757) — Evolving triggered harness interventions on 100 source tasks and gating them by replay selectivity scored 76.4 on DeepSWE with Mini-SWE versus 73.2 base and 69.0 for Meta-Harness, with +13.0 recovery against −9.7 regression.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: +relation alternative-to AG-20261127-11.
- 2026-11-27: +relation has-variant AG-20261127-08.
- 2026-11-27: created from bulletin 2026-11-27 (#6).
