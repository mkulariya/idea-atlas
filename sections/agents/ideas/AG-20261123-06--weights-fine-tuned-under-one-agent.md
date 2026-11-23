---
id: AG-20261123-06
title: Weights fine-tuned under one agent harness collapse under another
tldr: Weights fine-tuned on trajectories from their own evolved harness score 19.8% under that harness but 2.6% under a reference harness, so harness revisions reopen the training decision.
section: agents
also_in: []
subtopic: agent-training
type: finding
tags: [harness-coupling, agent-fine-tuning, scaffold-search, distribution-shift, agent-evaluation]
aliases: [CoEvoBench, harness-weights coupling, cross-harness evaluation, harness overfitting]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 3, actionability: 5, surprise: 4}
power: 79
evidence: B
first_seen: '2026-11-23'
last_updated: '2026-11-23'
sources:
- {arxiv: '2611.27116', title: 'CoEvoBench: Can LLMs Drive Recursive Co-Evolution of Agentic Systems?', role: origin}
related:
- {id: AG-20261123-09, rel: combines-with}
superseded_by: null
---

## Summary

Agent teams typically improve the harness (prompts, tool interfaces, output parsing, control flow) and the weights in parallel, assuming the two gains stack. CoEvoBench gives LLM researchers both levers on one base model and budget, then cross-evaluates the submitted (weights, harness) pairs. Trained weights score 19.8 mean test across five agent benchmarks under their own researcher harness but 2.6 under the reference harness, while the untouched Instruct model under that same reference harness improves from 30.2 to 37.8. The reading is that trajectories are collected through the harness, so fine-tuning teaches its tool names, output formats and context layout rather than the task. In one AppWorld run a training update worth +5 selection tasks became 4 tasks behind Base after a harness revision, then a 4-task gain again after retraining on trajectories from the revised harness. Use this when you fine-tune agents while also editing the scaffold, or ship weights to users with their own harnesses: pin the harness version in every data manifest, re-check trained versus base weights after any harness change, and report a cross-harness number at release. The main caveat is scope: 24-hour LoRA/SFT runs on a 9B base, three runs per cell without confidence intervals, and a reference harness that is not neutral.

## Detail

### Problem

Agent teams usually improve two things in parallel: the harness (prompts, tool interfaces, output parsing, control flow, decoding) and the model weights (SFT or RL on agent trajectories). The common assumption is that the two stack: a better harness plus better weights should be better still, and you can keep iterating either one.

CoEvoBench gives LLM "researchers" both levers on the same base model and budget and then cross-evaluates the results. It finds the gains are strongly coupled: trained weights mostly learn their own harness's conventions. A harness revision can turn a helpful training update into a harmful one. For anyone fine-tuning agents while also editing the scaffold, or shipping a fine-tuned model to customers who use their own harness, this is a direct warning about how gains are measured and how long they last.

### Mechanism

- **Harness:** all the code around the model in an agent: system prompt, tool schemas and docs, how outputs are parsed into actions, control loop, context management, decoding settings.
- **CoEvoBench setup:** an LLM researcher (e.g. GPT-6 Astra, Claude Opus 5) gets base weights (Qwen3.5-9B-Base or MiniCPM5-2B-Base) plus a minimal initial harness, and 24 hours, four NVIDIA H20 GPUs and a USD 100 teacher-model budget. It may edit the harness and/or train the weights, then submits (weights, harness).
- **Splits:** visible tasks (inspectable, usable for training data), selection tasks (score-only feedback), test tasks (used once after submission). Domains: SWE-bench Verified, TB-Lite, AppWorld, τ²-bench Telecom, BFCL-V4.
- **Reference system:** Qwen3.5-9B-Instruct with a reference harness, the target the researchers try to beat.

This is an empirical finding with a simple protocol you can reuse: **cross-evaluate weights and harnesses.**

1. **Collect systems.** From 75 development runs on Qwen3.5-9B-Base, 73 both revise the harness and launch training, and 54 submit trained weights. After excluding runs with integrity issues, 53 systems remain for the contribution analysis.
2. **Decompose the gain** (Table 2a) by evaluating, on the test set: Base weights + initial harness; Base + researcher harness (harness gain); trained weights + researcher harness (training gain on top); and Instruct + researcher harness (how much model headroom remains under that harness).
3. **Swap harnesses** (Table 2b): run the 35 trained-weight systems under the reference harness, and run Instruct under both.
4. **Case-study the dynamics** inside runs: track selection-set scores as harness and weights change in turn.

The paper's reading of the result: training data is collected *through* the harness, so the weights learn its specific formats, tool calling conventions and context layout. A different harness presents a different input distribution, and the fine-tuned weights do worse than the untouched Base model would. The paper's prescription is procedural: after any harness revision, re-collect trajectories under the new harness and re-check whether training still helps.

### Why it works

Fine-tuning a 9B base model on a few hours of trajectories is a narrow distribution shift: it teaches the model to produce outputs the current parser accepts, to use the current tool names and argument formats, and to expect the current context layout. Those are exactly the things a harness revision changes. An instruction-tuned model, trained on broad formats, has no such commitment, which is why Instruct actually *improves* under the better-engineered reference harness (30.2% → 37.8%) while the narrowly trained weights collapse (19.8% → 2.6%).

The within-run case is the cleanest evidence of mechanism. In one Astra AppWorld run, a training update first solves five more selection tasks than Base; after a harness revision, the same weights fall four tasks behind Base; retraining on trajectories collected under the revised harness restores a four-task gain (Figure 5). Same model, same data source, only the harness changed.

My read: this is overfitting to an interface rather than to tasks, and it is likely larger at this scale (short LoRA/SFT runs on a 9B base) than for heavily post-trained frontier models.

### Evidence

**Cross-evaluation (Table 2b, mean test %, Qwen3.5-9B):** trained weights under their own researcher harness 19.8 (SWE-bench Verified 16.0, TB-Lite 14.6, AppWorld 24.1, Telecom 62.8, BFCL 15.7); under the reference harness 2.6. Instruct: 30.2 → 37.8 under the reference harness.

**Contribution (Table 2a, 53 clean systems):**

| Row | SWE-V | TB-Lite | AppWorld | Telecom | BFCL | Avg |
|---|---|---|---|---|---|---|
| + Harness (gain over Base) | 12.8 (+12.4) | 16.3 (+13.8) | 21.8 (+21.8) | 43.5 (+42.9) | 4.0 (+2.2) | 16.9 (+15.7) |
| + Harness + Training (gain) | 18.8 (+6.0) | 17.6 (+1.3) | 25.9 (+4.1) | 45.1 (+1.6) | 14.1 (+10.1) | 21.8 (+4.9) |

Training carries over 80% of the gain on BFCL-V4; the harness carries most of it on the other four, where Instruct under the same harness still scores 6 to 33 points higher.

**Dynamics:** Astra AppWorld: +5 selection tasks from training → −4 vs Base after a harness revision → +4 after retraining. An Opus AppWorld run: harness revisions take test success from 0/51 to 3/51; training then takes it to 12/51.

**Researchers:** average test score Astra 31.23%, Opus 24.19%, GLM-5.3 12.24%, Qwen3.8-max 9.40%, Kimi K3 7.93%. About three quarters of the total gain comes from the first candidate (largest step in 25 of 28 improving audited runs). No researcher used RL.

### Limitations and open questions

- **Short training budgets:** 24 hours on four H20 GPUs, mostly LoRA/SFT; heavier training might produce weights that generalise across harnesses.
- **Small samples per cell:** 3 runs per researcher-benchmark pair, no confidence intervals; the decomposition rows are means across heterogeneous runs.
- **The reference harness is not neutral:** it may be systematically different from all researcher harnesses, so 2.6% partly reflects distance from one particular design.
- **Integrity issues** removed some runs (authors report them).

My read: the direction is robust (35 systems, consistent collapse) and matches practitioner experience; the exact size will vary with how different two harnesses are.

### How to apply

**When:** you fine-tune an agent model while also iterating on its harness, or you ship fine-tuned weights to users with their own scaffolds.

**Practices:**
1. Pin the harness version in every training-data manifest; treat (weights, harness) as one artefact.
2. After any harness change, re-run eval for trained weights *and* the base/instruct model under the new harness; if the trained model's advantage shrinks, re-collect trajectories and retrain.
3. Add a cross-harness eval to release checks: run trained weights under at least one alternative harness (e.g. a vendor-neutral reference). Report both numbers.
4. To reduce coupling, train on trajectories from several harness variants (paraphrased prompts, alternative tool schemas).

**Smallest experiment:** take your latest fine-tuned agent checkpoint and its base/instruct model, evaluate both under your current harness and one simpler alternative; the gap change tells you how much of your training gain is harness-specific.

Assessment: Adopt now. Cross-harness evaluation and retraining after harness changes are cheap safeguards, and 35 systems falling from 19.8% to 2.6% shows the risk is large.

### Connections

Closest prior work: harness-optimisation and scaffold-search work (automated prompt/scaffold design), agent fine-tuning on self-collected trajectories, and "AI scientist"/ML-engineering benchmarks that give agents only one lever (code or training).

New: a benchmark where one agent controls *both* levers under a shared budget, and a cross-evaluation that measures how much of a trained model's gain depends on its own harness. The 19.8% → 2.6% result is the clearest large-sample measurement of harness-weights coupling for agents that I know of.

**Connections.** Idea 9 co-designs the harness (belief-state folding) with RL training and gets gains only from the combination, a positive case of the same coupling. Idea 4 is a training-side analogue: a mismatch in output conventions (the EOS token) between teacher and student breaks distillation. Ideas 7 and 10 concern the other half of agent evaluation: whether the grader can be trusted.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-23 · origin · [CoEvoBench: Can LLMs Drive Recursive Co-Evolution of Agentic Systems?](https://arxiv.org/abs/2611.27116) — Cross-evaluating 35 agent systems trained through their own evolved harness, mean test score fell from 19.8% under that harness to 2.6% under the reference harness, while Instruct rose from 30.2% to 37.8%.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-23: +relation combines-with AG-20261123-09.
- 2026-11-23: created from bulletin 2026-11-23 (#6).
