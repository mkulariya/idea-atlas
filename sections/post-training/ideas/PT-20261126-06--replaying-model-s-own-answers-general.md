---
id: PT-20261126-06
title: Replaying a model's own answers to general instructions largely prevents forgetting during factual fine-tuning
tldr: Adding a forward-KL penalty to a frozen starting model on that model's own saved answers to general instructions cuts fine-tuning forgetting from 28.07 to 2.44 points at equal fact acquisition.
section: post-training
also_in: []
subtopic: merging-continual
type: method
tags: [kl-divergence, teacher-student, catastrophic-forgetting, replay, instruction-following, distribution-shift]
aliases: [InstructReplay, self-answer replay, replay-KL]
status: emerging
scores: {novelty: 3, leverage: 4, evidence: 4, generality: 3, actionability: 5, surprise: 3}
power: 73
evidence: B
first_seen: '2026-11-26'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.32918', title: 'InstructReplay: Replaying Assistant Turns to Preserve General Behavior During Factual Fine-Tuning', role: origin}
related: []
superseded_by: null
---

## Summary

Fine-tuning an instruction-tuned model on new facts teaches the facts but collapses general ability, and the damage is concentrated in generation: SmolLM3-3B fine-tuned on biomedical articles gains 21 points on new questions while IFEval falls from 71% to 9%, and generative benchmarks drop 28 to 45 points while likelihood-scored MMLU barely moves. InstructReplay first has the initial model answer 9,600 general instructions inside its chat template and saves each instruction-response pair. Training then mixes domain sequences with replay sequences 1:1 and adds a forward KL between the frozen initial model's full next-token distribution and the student's at every saved response position, with lambda=0.25 for 300 steps. Mean over six model-domain settings, forgetting falls from 28.07 points under naive SFT to 2.44 while acquisition stays level (15.49 versus 15.23), and InstructReplay has the best acquisition-minus-forgetting score in all six settings. Controls show placement and ownership of the replay text both matter: KL on domain-answer prefixes barely helps, and replaying the dataset's own answers forgets more than twice as much as replaying the model's own. Use it when adapting an instruction-tuned model to domain facts and general behavior matters; the buffer is about 3M response tokens, and gold-answer replay is a weaker substitute. The main caveats are that models are 3B to 4B, lambda was tuned on two of the six settings, and acquisition is judged by a model.

## Detail

### Problem

Teams adapt instruction-tuned models to a domain by fine-tuning on question-answer pairs that state new facts. The facts get learned, but general ability collapses. The authors give an example: fine-tuned on recent biomedical articles, SmolLM3-3B answers 21 points more of the new questions correctly, yet its IFEval (instruction-following) accuracy falls from 71% to 9%.

The damage is concentrated in *generation*. On benchmarks that require a generated answer, accuracy drops by 28 to 45 points, while likelihood-scored MMLU barely moves, so a quick likelihood check can miss it. The authors tie the loss to answer length: fine-tuned models at least halve their responses, mimicking the short domain answers they trained on, and the further a response drifts from the initial model's length, the more often it fails.

### Mechanism

**KL divergence (here, forward KL).** A measure of how far the student's next-token distribution is from the frozen reference's. Penalizing it at a position keeps the model's predictions there close to the reference.

**Replay.** Mixing in data from earlier behavior during new training.

**Prefix / context.** The text before a token. A KL penalty only constrains the model on the prefixes where it is applied, so the choice of replay inputs decides which behavior is protected.

**BOS replay.** Replay on text sampled from a beginning-of-sequence token without the chat template.

**IFEval, MATH-500, BBH, HumanEval+, MMLU.** General benchmarks used to measure retained ability. MMLU is scored by the likelihood of answer options; the others require generating answers.

**InstructReplay**:

1. **Build the buffer once.** The initial model $M_0$ answers 9,600 general instructions inside its chat template. Save each (instruction, response).
2. **Fine-tune with two terms.** Each step mixes domain question-answer sequences and replay sequences 1:1. The loss is $\mathcal{L}=\mathcal{L}_D+\lambda\mathcal{L}_R$, where $\mathcal{L}_D$ is the usual domain SFT loss, and $\mathcal{L}_R$ is the forward KL between the frozen $M_0$'s full next-token distribution and the student's at every response position of the replay sequences, on the same prefix (instruction followed by the saved response so far). $\lambda=0.25$ and training runs 300 steps.

The authors isolate three ingredients with controls that keep $\lambda$, the 1:1 ratio and the schedule: **Domain-KL** applies the KL on domain-answer prefixes; **BOS replay** applies it on samples from a BOS token; **Gold replay** uses the same instruction prompts with the dataset's own answers; **Hard** replays $M_0$'s saved tokens with a cross-entropy loss instead of full distributions.

The replay buffer is generated once per model and shared across domains. Compared with seven adaptation methods: Naive SFT, LoRA-SFT, Self-Rewrite, SDFT, DFT, RAFT and Domain-KL.

### Why it works

The authors' account: where the constraint is placed matters. Putting the KL on domain-answer prefixes barely helps because it only constrains behavior on domain prompts. Putting it on instruction contexts with the model's own answers constrains the behavior that fine-tuning was destroying, namely how long and how well the model answers general requests. Replaying the dataset's own answers on the same prompts forgets more than twice as much as replaying $M_0$'s answers (7.5 versus 3.1 points), so *whose* answers are replayed also matters.

For soft versus sampled targets, the authors show the expected sampled-token gradient equals the forward-KL gradient at a fixed prefix; Hard and InstructReplay behave similarly (3.1 versus 2.4 points forgetting).

My read: the length-collapse story is a good diagnostic. Checking median response length against the initial model after any fine-tune is a cheap early warning.

### Evidence

**Setup.** SmolLM3-3B, Qwen3-4B, Phi-4-mini; two domains (CommonCorpus, PMC biomedical articles), six model-domain settings; five general benchmarks; paired cluster bootstrap with Bonferroni correction; three repeated training runs for InstructReplay, Self-Rewrite and RAFT.

Naive SFT versus $M_0$, mean over six settings (points): IFEval -44.9, MATH-500 -28.0, BBH -36.0, HumanEval+ -30.5, MMLU -0.4. Across 60 adapted checkpoints, IFEval change correlates with median response length (Spearman $\rho=0.77$).

| Method (six-setting mean) | Acquisition A | Forgetting F | S = A − F |
|---|---|---|---|
| Naive SFT | 15.23 | 28.07 | −12.8 |
| RAFT (best acquisition) | 16.15 | 13.96 | 2.2 (A − F) |
| InstructReplay | 15.49 | 2.44 | 13.1 |

Replay-location ablation (mean change vs $M_0$ on IFEval, points): Domain-KL −43.3; BOS replay −14.2; Gold replay −7.4; Hard −4.0; InstructReplay −2.6; median IFEval response length relative to $M_0$: 0.11 (Naive SFT) to 0.85 (InstructReplay).

InstructReplay has the highest S in all six settings; its corrected interval against the strongest baseline excludes zero in the three PMC settings (for example +24.11 on SmolLM3), and across all 42 baseline pairs it excludes zero in 37. It beats both repeated baselines in 35 of 36 run-by-run comparisons. Cost: at most 2.22 GPU-hours per model on PMC versus at least 4.83 for RAFT.

### Limitations and open questions

The authors state that $\lambda$ was selected on two settings (so the four others are the fairer test), that acquisition is judged by a model, and that on one baseline comparison (CommonCorpus Qwen3) the margin in S is 1.74 with an interval including zero.

My read: the models are 3B to 4B, the facts are closed-book Q&A, and the replay buffer is about 3M response tokens, roughly an order of magnitude larger than the domain tokens, so memory and throughput grow. I did not see code. Behavior beyond five benchmarks is untested, as is stacking with RL or preference training afterwards.

### How to apply

**When:** you fine-tune an instruction-tuned model on domain facts and care about keeping general behavior.

1. Have the starting model answer a few thousand diverse general instructions with its chat template; save the responses.
2. Fine-tune with domain data plus a 1:1 replay batch, adding forward KL to the frozen starting model on the replay responses with $\lambda\approx0.25$.
3. Track median response length versus the base model and a generative benchmark (IFEval-style), not just MMLU.
4. If memory is tight, try Hard replay (cross-entropy on saved tokens), which forgot slightly more (3.1 versus 2.4).

**Pitfall:** the buffer is only as broad as your instructions; gold-answer replay forgot more.

**Smallest experiment:** one model, one domain: fine-tune with and without replay for 300 steps and compare IFEval and answer length.

Assessment: Adopt now. Cheap, well-controlled across three model families with consistent gains; only the small-model scale remains untested.

### Connections

Replay for forgetting is classic (Learning without Forgetting, rehearsal methods), and prior LLM work already replays BOS-sampled text (Marek et al.), synthetic context-free data, or self-synthesized refined instructions. The new element is a controlled decomposition: the constraint should be applied in the instruction context, on the initial model's own answers, matched at full distributions.

**Connections.** Idea 3 (harness rehearsal) is the agentic analogue: protect the interface behavior while training for task content. Idea 8 (Delta-KL) also uses a frozen model's own distribution as the teacher. Combining this with idea 5's RL-style training is untested.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [InstructReplay: Replaying Assistant Turns to Preserve General Behavior During Factual Fine-Tuning](https://arxiv.org/abs/2611.32918) — Across three model families and two domains, a forward KL to the frozen initial model on its own saved answers to general instructions cut mean forgetting over five benchmarks from 28.07 to 2.44 points at level acquisition (15.49 vs 15.23), and gold-answer replay forgot 7.5 points versus 3.1.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: created from bulletin 2026-11-26 (#6).
