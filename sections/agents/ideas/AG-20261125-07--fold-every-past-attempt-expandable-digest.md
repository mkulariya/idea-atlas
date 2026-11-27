---
id: AG-20261125-07
title: Fold every past attempt into an expandable digest instead of summarizing history
tldr: Digesting every past attempt so the agent can still expand any of them, plus deleting skills unused for five iterations, beat recency truncation on MLE-Bench and KernelBench across three backbones.
section: agents
also_in: []
subtopic: memory-context
type: method
tags: [context-folding, long-horizon-agents, harness-coupling, scaffold-search, agent-evaluation]
aliases: [MLEvolver, attempt folding, expandable attempt digests, digest-and-expand, skill library deletion]
status: emerging
scores: {novelty: 3, leverage: 4, evidence: 4, generality: 3, actionability: 5, surprise: 3}
power: 73
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.28244', title: 'Demystifying the Autoresearch Harness: How Context and Skills Shape Self-Evolving Agents', role: origin}
related:
- {id: AG-20261123-09, rel: alternative-to}
- {id: PT-20261126-08, rel: alternative-to}
- {id: AG-20261127-04, rel: has-variant}
superseded_by: null
---

## Summary

A research agent that runs a hundred iterations accumulates more attempt history than fits in context; the default of dropping the oldest attempts, or compressing them into one running summary, loses the specific code, scores and errors of earlier tries, so the agent repeats failed ideas or cannot build on a good one. Folding instead maps every attempt to a compact diagnostic digest via a summarizer and always shows the agent all digests plus the full records of a subset it can expand, up to the history budget, so every attempt stays addressable; compaction keeps only a running summary and a recent tail, and old attempts become unrecoverable. On the skill side the paper compares deleting skills unselected for five consecutive iterations, merging each new skill into its nearest neighbour above 0.8 cosine similarity, and promoting skills that contributed to an improvement into standing rules; the combined agent MLEvolver is folding plus deletion at 32K tokens and 100 iterations per run. Folding beat the recency-truncation baseline in all six model-by-benchmark settings, averaging 8.6% relative, with gpt-oss-120b gold-normalized score rising from 70.5 to 76.1 and KernelBench geometric-mean speedup from 1.17 to 1.29 times, and gpt-6-sol from 93.2 to 94.7 and 2.16 to 2.34 times. Deletion, merging and promotion all beat the baseline, averaging 4.1%, 2.2% and 3.0% relative, but the skill gains shrink for stronger models (5.1% for gpt-oss-120b versus 1.9% and 2.1% for the two stronger backbones) because stronger models retrieve fewer skills. MLEvolver medalled in 16 of 22 MLE-Bench Lite competitions (72.7%, 23.1% above AIDE and 14.3% above OpenEvolve, 10 golds) and reached 94% fast_1 on KernelBench Level 3. Use it when a long-horizon agent's history no longer fits, keeping every full attempt record on disk by ID; caveats are that no variance or seed counts are visible, the ablation covers 10 competitions, and the AIDE/OpenEvolve comparison changes the search strategy as well as the memory. Assessment: the harness change is cheap enough to test immediately, but the 1-2% skill-policy gains cannot be trusted without variance.

## Detail

### Problem

An agent that runs experiments for hundreds of iterations accumulates more history than fits in context. The default is to drop the oldest attempts (recency truncation) or compress them into one running summary. Both lose the specific code, scores and errors of earlier tries, so the agent repeats failed ideas or cannot build on a good one.

A second problem is the skill library: lessons the agent writes after each attempt pile up, become redundant and noisy, and make the right lesson harder to retrieve. Teams have little controlled evidence on which harness choices matter.

### Mechanism

**Autoresearch agent.** A loop that proposes an idea, implements it, runs the experiment, reads the result and decides what to try next, given a task and a grader.

**Harness.** The scaffolding around the model: what context it sees, what it remembers and how it stores lessons.

**MLE-Bench and KernelBench.** MLE-Bench: Kaggle machine-learning competitions, scored here by gold-normalized score (GNS). KernelBench: CUDA kernel optimization, scored by geometric-mean speedup over PyTorch Eager ($\text{fast}_1$ is the share of problems with a correct kernel faster than Eager, $\text{fast}_2$ faster than 2 times).

**Skill library.** Natural-language lessons distilled from earlier attempts, selected by a skill picker when relevant.

All mechanisms run inside one shared agent framework and are compared to a recency-truncation baseline that discards the oldest attempts first.

**Context management** (the attempt history $H_t$):

- *Folding*: a summarizer $\sigma$ maps each attempt $a_i$ to a compact diagnostic digest $\hat a_i$. At iteration $t$ the agent sees all digests $(\hat a_0,\dots,\hat a_{t-1})$ plus the full records $(a_i)_{i\in U_t}$ of a chosen subset $U_t$ that it expands, up to the history budget. Every attempt stays addressable.
- *Compaction*: keep a running summary $m\leftarrow S(m,h)$ of older history $h$ plus a tail of recent attempts; individual old attempts cannot be recovered.

**Skill management** (library $L_t$, empty at task start):

- *Deletion*: remove a skill after $\tau_d=5$ consecutive iterations without being selected.
- *Merging*: embed each new skill (Qwen3-Embedding-0.6B) and merge it into its nearest neighbor if cosine similarity is at least $\theta=0.8$.
- *Promotion*: turn skills that contributed to at least one improvement into standing rules included in every prompt.

**Combined agent (MLEvolver)**: folding plus deletion, with a 32K-token context and 100 iterations per run.

### Why it works

The authors' explanation is that folding lets the agent hold more experimental history in the same budget while still reaching the details of any relevant attempt, so it can choose to refine a promising solution, debug a failure or propose something new. Deletion uses the model's own retrieval behavior as a relevance signal: lessons that are persistently unselected are likely stale.

They also report that skill gains shrink for stronger models, which retrieve fewer skills and favor defensive guidance more. My read: this fits a picture where history access matters across model strengths, but skill-library management matters mainly for weaker models; the paper does not isolate why.

### Evidence

Setup: 10 MLE-Bench competitions (6 low and 4 medium complexity) and 50 KernelBench tasks, three backbones (gpt-oss-120b, gpt-6-luna, gpt-6-sol), each mechanism compared to recency truncation from the same deliberately weak seed solution (Table 1).

- **Folding** was best in all six settings, averaging +8.6% relative over baseline: +1.6% to +7.9% on MLE-Bench and +8.3% to +19.2% on KernelBench. With gpt-oss-120b, GNS rose from 70.5 to 76.1 and geometric-mean speedup from 1.17 to 1.29 times; with gpt-6-sol, from 93.2 to 94.7 and from 2.16 to 2.34 times.
- **Skills.** Deletion, merging and promotion all improved on the baseline for every model and benchmark. Average relative gains: deletion 4.1%, promotion 3.0%, merging 2.2%. Averaged over mechanisms, skill gains were 5.1% (gpt-oss-120b), 1.9% (gpt-6-luna) and 2.1% (gpt-6-sol).
- **MLEvolver versus AIDE and OpenEvolve** (gpt-6-sol, 32K context, 100 iterations): 16 of 22 MLE-Bench Lite competitions medaled (72.7%), a 23.1% relative gain over AIDE and 14.3% over OpenEvolve, with 10 golds. On KernelBench Level 3 (50 problems) $\text{fast}_1$ reached 94%, 20.5% above AIDE and 34.3% above OpenEvolve; 31 of 50 problems exceeded 2 times speedup.

### Limitations and open questions

The paper states that benefits of skill management depend on how a backbone selects skills. I could not see variance, seed counts or the summarizer model's identity in the text I read, so I cannot judge whether differences of 1 to 2 percent are within noise. The final comparison uses one backbone and 100 iterations; the MLE-Bench subset is 10 competitions for the ablation and 22 for the head-to-head.

My read: the gains on MLE-Bench are small in the ablation (1.6% to 7.9%); KernelBench gains are larger and the solution space there is more continuous, so the benefit may depend on task type. The comparison to AIDE and OpenEvolve changes more than memory, since the search strategy also differs.

### How to apply

Reach for this if your long-horizon agent's history no longer fits in context.

1. Replace "drop the oldest" with digests: after each attempt, store a short diagnostic digest (what was tried, the score, the failure signature) and keep the full record on disk by attempt ID.
2. Give the agent a tool to expand a digest by ID, and a budget for how many full records fit in the history window.
3. For a skill library, track consecutive iterations unused per skill and delete after about five.
4. Check how your backbone uses skills: if it rarely retrieves them, expect small gains.

**Smallest experiment:** in an existing agent loop, swap recency truncation for digests plus on-demand expansion, run 30 iterations on 5 tasks from your own benchmark, and compare best-score-so-far curves.

Assessment: Try in an experiment. The design is simple and won every comparison against truncation, but the reported gains are modest and I could not verify variance.

### Connections

Prior harnesses combine several mechanisms: AIDE searches a tree of candidate solutions, OpenEvolve (an open AlphaEvolve implementation) keeps a program database across islands, and MemGPT-style systems page memory (Packer et al.). Reflexion-style work stores lessons. This paper isolates each memory mechanism against one baseline in one framework across three models.

The new finding is that preserving individually accessible records (folding) beats a rolling summary, and that pruning persistently unused lessons beats consolidating or promoting them.

Connections: idea 9 attacks context thrashing through weights (a trained model stopped re-extracting the same source sixteen times), while folding attacks it in the harness; the two are complementary. Idea 4 raises a parallel question for unattended self-improving agents, whether the loop notices its own failures. Idea 1 is a reminder that any digest the agent trusts can hide what was missing, so keep the raw record expandable.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [Demystifying the Autoresearch Harness: How Context and Skills Shape Self-Evolving Agents](https://arxiv.org/abs/2611.28244) — Per-attempt digests with on-demand expansion beat recency truncation in all six settings (mean +8.6% relative; gpt-oss-120b GNS 70.5 to 76.1, KernelBench speedup 1.17 to 1.29 times), and skill deletion averaged +4.1% relative.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: +relation has-variant AG-20261127-04.
- 2026-11-26: +relation alternative-to PT-20261126-08.
- 2026-11-25: created from bulletin 2026-11-25 (#7).
