---
id: PT-20261125-09
title: Rubric RL after self-distillation more than doubles long-document strict pass at flat token spend
tldr: Privileged-context self-distillation then two rounds of rubric-reward RL at full context lifts strict pass on long professional documents from 11.0% to 24.5% at flat token spend.
section: post-training
also_in: []
subtopic: rl-algorithms
type: recipe
tags: [agent-rl, reasoning-rl, on-policy-distillation, teacher-student, long-horizon-agents, process-reward, agent-fine-tuning]
aliases: [privileged-context self-distillation plus rubric RL, long-document agent post-training]
status: emerging
scores: {novelty: 3, leverage: 4, evidence: 4, generality: 3, actionability: 3, surprise: 3}
power: 69
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.30090', title: Post-Training Agents for Long-Document Reasoning, role: origin}
related:
- {id: AG-20261123-09, rel: alternative-to}
- {id: ENV-20261125-11, rel: has-variant}
superseded_by: null
---

## Summary

Professional documents such as engineering drawings, benefits tables and legal statutes spread evidence across many pages and mix text with images; on the paper's 100-task GDP.pdf holdout the best public result was 30.7% strict pass, where strict pass means every rubric criterion is met. The recipe starts from Kimi-K2.7 with rank-32 LoRA adapters at its full context window on 1,465 expert-authored tasks: one round of privileged-context self-distillation on 600 tasks, in which the same checkpoint given criterion-targeted feedback from GPT-5.6 Sol on an earlier attempt scores the student's own sampled tokens, then two rounds of rubric-reward RL on a disjoint 865-task pool with 8 samples per prompt, a per-criterion GPT-5.4 Mini judge, group-relative advantages and truncated importance sampling. Strict pass rose from 11.0% to 13.0% after distillation and to 24.5% after round 2 (mean of four runs, each beating base in a paired exact McNemar test at p≤0.003), while mean generated tokens stayed roughly flat (9,803 to 9,733), so strict passes per 100K generated tokens went from 1.12 to 2.52. Gains held across input lengths up to 240K tokens and in all ten domains, and out of domain on 220 tool-using GDPval tasks mean rubric reward rose from 0.571 to 0.746 with zero-credit tasks falling from 54 to 10 and messages per task dropping 18%. Trajectory comparisons show a context-thrashing retry loop of about 110 messages replaced by one up-front check of all sources plus an explicit fallback rule, and the share of tasks with an executable check of the deliverable nearly tripled. The paper has no ablation of the distillation round or of the feedback source, no RL-only comparison, an untested reward judge, and it needs frontier-model feedback plus 64 H200 GPUs on a proprietary task collection. Use it when you have expert-written rubrics for long-document or deliverable tasks and an open-weight model with a long window, and always evaluate with strict pass, criterion pass and tokens rather than accuracy alone.

## Detail

### Problem

Real professional documents (engineering drawings, benefits tables, legal statutes) are long, spread evidence across pages and mix text with images. Frontier models still fail them: on the paper's 100-task GDP.pdf holdout the best public result was 30.7% strict pass, where strict pass means every rubric criterion is met.

Teams with an open-weight model need a recipe that improves it on such work without simply making it write more, and evidence on whether gains carry over to agentic use.

### Mechanism

**Rubric reward.** Instead of a single correct-answer check, each task has a list of atomic criteria written by an expert, and the reward is the weighted share satisfied, judged by an LLM.

**On-policy self-distillation with privileged context.** The model samples its own response, then the *same* network, given extra information the student does not see, scores those exact tokens; the student is pushed toward what the better-informed copy would say.

**LoRA.** Training a small low-rank adapter instead of all weights.

**TIS and asymmetric clipping.** Truncated importance sampling corrects the mismatch between rollout and training probabilities; asymmetric clipping uses different bounds (here 0.20 and 0.28) on the policy ratio.

Base model $M_0$ is Kimi-K2.7, trained with rank-32 adapters at its full context window on 1,465 expert-authored tasks. The collection catalog the tasks come from covers 2,485 profiled tasks with a median of 27 source pages and 70.9% needing two or more reasoning hops.

**Round 1, privileged-context self-distillation (600 tasks).** Each task carries criterion-targeted feedback $z$ written by GPT-5.6 Sol (Max) on an earlier attempt, prepared before training. The student samples $y$ from $(x)$; the same checkpoint, now conditioned on $(x,z)$, scores the sampled tokens. There is no task reward; the token-level advantage is

$$A_t=-\lambda\big(\log\pi_\theta(y_t\mid x,y_{<t})-\mathrm{sg}[\log q_\theta(y_t\mid x,z,y_{<t})]\big),$$

where $\pi_\theta$ is the student, $q_\theta$ is the same network with privileged prompt, and $\mathrm{sg}$ stops gradients through the teacher probabilities. The student's maximum new tokens are capped at the usable context minus the teacher prompt length, truncated samples are rejected, and teacher scoring is a prefill over exact token IDs.

**Rounds 2 and 3, rubric-reward RL (disjoint 865 tasks; round 3 continues on the same pool).** Eight samples per prompt, each rubric criterion judged by GPT-5.4 Mini at medium effort, group-relative advantages with TIS and clipping 0.20/0.28, responses capped at 65,536 tokens, on 64 H200 GPUs. PDFs are rendered as text for training (images transcribed by a separate model) and as extracted text plus up to 16 selected page images for evaluation.

### Why it works

The authors give no ablation separating the rounds' causes. What they observe: the training reward rises from 0.37 in the first tenth of the run to 0.67 in the last; output length stays flat; trajectories change. In paired runs the third round replaced a context-thrashing retry loop (about 110 messages, the same statute re-extracted sixteen times) with one up-front check of all sources and an explicit fallback rule, and verification moved from checking that files exist to checking what they contain.

My read: rubric criteria reward the specific failures that long documents cause (missed footnotes, wrong table scope), so RL against them teaches scope-holding and checking habits. Because the distillation round alone gave a small gain (below), most of the improvement likely comes from the RL rounds, though the paper does not isolate this.

### Evidence

- **In-domain, GDP.pdf holdout (100 tasks, 262,144-token window, Table 3).** Strict pass: 11.0% base, 13.0% after round 1 (self-distillation), 24.5% after round 2 (mean of four evaluation runs: 23, 24, 24, 27). Criterion pass rate 0.660 to 0.765 (single run for round 2). Mean tokens generated 9,803, 10,775, 9,733, so strict passes per 100K generated tokens went from 1.12 to 2.52. All four runs beat base in a paired test (exact McNemar $p\le0.003$). Gains held across input lengths up to 240K tokens and in all ten domains. The GPT-5.6 Sol leaderboard reference is 30.7% at 3,422 tokens.
- **Out of domain, GDPval, 220 tool-using tasks (Table 4).** Mean rubric reward rose from 0.571 to 0.746 after round three; zero-credit tasks fell from 54 to 10; messages per task fell 18%, from 70.9 to 58.2.
- **Behavior.** The share of tasks with an executable check of the written deliverable nearly tripled (Table 6, a separate 100-task replication).

### Limitations and open questions

Authors' notes: strict pass moved by four points across four runs of one checkpoint; the GPT-5.6 Sol reference was not produced under this harness; the round-3 GDPval result comes from the third of three evaluation runs; comparisons across providers conflate document pipelines.

My read: there is no ablation of the distillation round or of the feedback source, no comparison against RL alone, and the reward judge (GPT-5.4 Mini) could be exploited without being tested for it. The recipe needs frontier-model feedback and 64 H200 GPUs, and a mini-model judge across 8 samples per prompt is a cost. The 1,465 tasks are a proprietary expert collection.

### How to apply

Reach for it if you have expert-written rubrics for long-document or deliverable tasks and an open-weight model with a long window.

1. Evaluate your base model on a holdout with strict pass, criterion pass and tokens (or turns) generated; do not rely on accuracy alone.
2. Start with the RL stage: atomic rubrics, 8 samples per prompt, group-relative advantages, truncated importance sampling, drop truncated samples.
3. Add privileged-context self-distillation only if you can cheaply generate per-task feedback from a stronger model; budget teacher context so the response is never truncated to fit.
4. Read paired trajectories before and after training for behavior changes.

**Smallest experiment:** train a LoRA adapter with the RL stage only on a few hundred rubric tasks and track strict pass and tokens on a 100-task holdout; if tokens stay flat and strict pass rises, extend.

Assessment: Watch. Large in-domain gains at flat token spend with a McNemar test, but no ablations of the recipe parts and a heavy infrastructure requirement.

### Connections

Privileged-information self-distillation (Zhao et al., 2026; Penaloza et al., 2026) and on-policy distillation (Agarwal et al., 2024) exist already, as do rubric rewards (Gunjal et al., 2026; Viswanathan et al., 2025). Concurrent work by Wang et al. (2026b) warm-starts rubric RL from a larger teacher that reads the rubric; here round 1 uses the student checkpoint as its own teacher with written feedback on an earlier attempt.

What the paper adds is the combination at full context on real professional documents, with accuracy reported together with tokens and turns, and trajectory evidence of changed behavior.

Connections: idea 11 isolates the rubric-reward design used in rounds 2 and 3. Idea 7 handles context thrashing at the harness level, whereas this paper reports it fading through training. Idea 2 shows judges soften rules judged in bundles; whether the criterion judge here shows that effect was not tested.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [Post-Training Agents for Long-Document Reasoning](https://arxiv.org/abs/2611.30090) — One self-distillation round plus two rubric-reward RL rounds lifted strict pass on a 100-task long-document holdout from 11.0% to 24.5% (four runs, McNemar p≤0.003) with mean generated tokens flat at about 9.7K, and raised GDPval rubric reward from 0.571 to 0.746 while cutting messages per task 18%.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: +relation has-variant ENV-20261125-11.
- 2026-11-25: created from bulletin 2026-11-25 (#9).
