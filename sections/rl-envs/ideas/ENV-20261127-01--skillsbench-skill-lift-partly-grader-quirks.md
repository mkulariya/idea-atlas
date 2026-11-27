---
id: ENV-20261127-01
title: SkillsBench skill lift is partly grader quirks, and its per-task losses are within noise
tldr: Permutation tests with FDR control find no per-task skill harms in SkillsBench, grader quirks inflate part of the lift, and label-shuffled replay shows log-fitted skill gates gain 4-7 pp from noise alone.
section: rl-envs
also_in: []
subtopic: task-suites-benchmarks
type: method
tags: [grader-audit, benchmark-quality, agent-evaluation, verifier, statistical-power, placebo-replay, skills]
aliases: [SkillsBench reanalysis, label-permutation test with FDR control, shuffled-label placebo replay, instruction-evaluation gap]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 4, actionability: 5, surprise: 4}
power: 82
evidence: B
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.33943', title: 'What Does Skill Lift Actually Measure? A Source-Code-Level Audit and Noise-Aware Reanalysis of SkillsBench', role: origin}
related:
- {id: ENV-20261123-07, rel: combines-with}
- {id: ENV-20261127-02, rel: combines-with}
- {id: AG-20261127-08, rel: combines-with}
superseded_by: null
---

## Summary

SkillsBench reported that human-curated skills raise pass rates by about 16.6 pp on average but lower them on some tasks, and follow-up work has begun gating or retiring skills on per-task gain. This audit forms 1,421 paired cells from 7,554 public trials, at most three runs per condition, and reads each task's grader, prompt and skill files. Two grader-dependent cases are confirmed: in earthquake-plate-calculation only the projection named in the skill passes all eight assertions although the prompt never names a projection, and in protein-expression-analysis the grader reads cached cell values, passing the same workbook 3 of 6 tests without them and 6 of 6 with them. Those three tasks average 74.6 pp lift; removing them lowers the mean from 18.7 to 16.1 pp, which the authors stress is not a causal estimate. Label-permutation tests (100,000 permutations, 10% FDR across tasks) find 26 significant tasks, all positive, with the smallest two-sided p among the 17 negative tasks at 0.15, and 58% of cells could not arithmetically have shown a decrease because the no-skill baseline was zero. A skill-free-score gate that looks like a 2-3 pp gain on the logs still gains 4-7 pp after condition labels are shuffled, and shows no statistically significant improvement on held-out runs. Non-significance is not proof of no harm: with three runs per condition the per-task tests have low power, and the audit covers one benchmark without inspecting agent trajectories.

## Detail

### Problem

Skills (folders of instructions, reference files and scripts that a coding agent loads on demand) have become a standard way to extend agents. SkillsBench, a benchmark that scores agents with and without skills, reported that human-curated skills raise pass rates by about 16.6 percentage points (pp) on average but lower them on some tasks. Follow-up work has started to act on that second result by gating or retiring skills on their per-task gain.

If those per-task losses are noise, a team that gates or deletes skills on them throws away good skills and builds a rule that only looks good on the logs it was fitted to. If some of the average lift comes from the grader asking for something the prompt never states, "skills help by X pp" overstates what skills teach the agent.

### Mechanism

**Skill lift.** The pass rate with skills minus the pass rate without. A *cell* is one (configuration, task) pair; the audit forms 1,421 paired cells from 7,554 public trials, with at most three runs per condition.

**Instruction-evaluation gap.** The grader (the program that decides pass or fail) checks something the task prompt does not state.

**Permutation test.** To test "skills have no effect", shuffle the with-skill and no-skill labels among a cell's trials and see how often the shuffled lift is as extreme as the observed one. **FDR (false discovery rate) control** with Benjamini-Hochberg limits the expected share of false positives among the tasks you call significant.

**Offline replay.** Evaluating a decision policy (here "load skills or not") on recorded trials by taking the logged score for whichever arm the policy picks.

1. **Source-code audit.** The authors read each task's grader, prompt and skill files, and ran the official grader on hand-built outputs differing in one undeclared requirement. 
2. **Noise-aware reanalysis.** For each of 83 tasks, they test "skills have no effect in any cell" by permuting condition labels within cells (100,000 permutations, two-sided Monte Carlo p-values) and control FDR at 10% across tasks.
3. **Gating replay with a placebo.** They fit per-task gates that decide whether to load skills: load only when the no-skill pass rate $r_0 < 0.8$, or use random forests that predict expected gain and loss probability. They evaluate by replay on the logs, by replay after shuffling condition labels (which removes any true skill effect), and on held-out runs.

The key step is the placebo replay: it destroys the skill effect and shows how much "gain" a selection rule invents from noise alone.

### Why it works

With at most three runs per condition, a cell's measured lift moves in steps of 33 pp.A single unlucky with-skill run can look like a drop. In the 823 cells where every no-skill run failed ($r_0=0$) a loss is arithmetically impossible, and in the 259 cells where every no-skill run passed ($r_0=1$) a gain is impossible. So negative cells can only occur in the 598 cells with a mixed baseline, and the "11.9% of cells are losses" figure is partly a floor and ceiling effect.

A gate fitted to the logs and replayed on the same logs picks lucky with-skill runs, so it gains even when skills do nothing; the shuffled-label replay measures that bias. My read: the placebo replay is the most reusable part.

### Evidence

**Grader and harness findings (source-code audit, Section III).**
- Two controlled-verified cases of grader dependence. In `earthquake-plate-calculation` the grader accepts distances within 0.01 km; only the projection named in the skill (EPSG:4087) passes all eight assertions, while great-circle and Web Mercator distances score 0, and the prompt never mentions a projection. In `protein-expression-analysis` the grader reads cached cell values; the same workbook with identical formulas in all 143 formula cells passes 3 of 6 tests without cached values and 6 of 6 with them, and the skill tells the agent to recompute cached values.
- One more case (`flood-risk-analysis`) is an operational definition gap. The three tasks average 74.6 pp lift and are 4% of cells (63/1,421); removing them lowers the mean lift from 18.7 to 16.1 pp, though the authors stress this is not a causal estimate.

**Noise findings (Section IV).**
- 169 of 1,421 cells (11.9%) show a loss, and 17 tasks have net negative lift. At 10% FDR, 26 tasks are significant and all are positive; the smallest two-sided p among the 17 negative tasks is 0.15. Arithmetically, 58% of cells could not have shown a decrease.
- 8 of the 13 tasks with negative lift in both exports flip sign between exports; `mars-clouds-clustering` goes from +38.1 pp (7 cells) to -28.2 pp (13 cells) with identical skill files.

**Gating (Section V).** The skill-free-score gate appears to improve results by 2-3 pp in the logs, but after shuffling condition labels it still gains 4-7 pp. On held-out runs not used for fitting or decisions, there was no statistically significant improvement.

### Limitations and open questions

The authors did not inspect agent trajectories, so three tasks where skills steer toward a different algorithm than the reference are only hypotheses. The logs come from two exports, with stronger models in the second (no-skill baseline 26.7% vs 31.7%). Non-significance is not proof of no harm: with three runs per condition the per-task tests have low power, which the paper quantifies in Section VI. The audit covers one benchmark.

My read: the strongest claim is the negative one ("the data cannot show harm"), not a claim that skills never hurt. A 1-in-3 run effect size also means any benchmark with three runs per cell has the same blind spot.

### How to apply

Reach for this when you measure the effect of a prompt, skill, tool or harness change per task and then act on per-task results.

1. **Count runs.** With 3 trials per cell, lifts move in steps of 33 pp; do not read single-cell losses as harm.
2. **Mark impossible cells:** an all-fail baseline cannot show a loss, an all-pass baseline cannot show a gain.
3. **Test per task with permutation** and Benjamini-Hochberg across tasks.
4. **Placebo-test any gate.** Fit and replay it on shuffled labels first. If it still "gains" several pp, your offline gain is selection bias.
5. **Audit the grader.** Ask whether every assertion is derivable from the prompt.

Cost: roughly a day for steps 1-4 on existing logs (my estimate). **Smallest experiment:** take your latest A/B of an agent change, compute the share of cells where a loss was arithmetically impossible, run the label-permutation test on your top five "regression" tasks, and see how many survive FDR control.

Assessment: Adopt now. The protocol is cheap, general, and directly protects against gating or retiring skills on noise.

### Connections

SkillsBench reported that curated skills help on average and hurt on some tasks, and later gating or retirement methods used per-task gains as the signal. This paper does not propose a new skill method. Its contribution is a reanalysis protocol: a source-code audit for grader dependence, label-permutation tests with FDR control per task, and a shuffled-label placebo for any gate fitted on logs. The new part is the arithmetic constraint (no loss is possible when the baseline is zero) that many agent papers ignore.

**Connections.** Idea 8 (MedSkill-E2) reports that agents underuse skills, while this audit reports that the measured average benefit is partly grader-driven; read together, "skills help" is real but smaller and harder to measure than headline numbers say. Idea 2 is the same genre of problem on the data side: a label (here "skill lift", there "proof depth") is trusted without checking what the checker actually requires. Idea 10 shares the lesson that a test suite can disagree with the task statement.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [What Does Skill Lift Actually Measure? A Source-Code-Level Audit and Noise-Aware Reanalysis of SkillsBench](https://arxiv.org/abs/2611.33943) — Across 1,421 paired cells from SkillsBench, 11.9% loss cells are partly an arithmetic floor/ceiling effect, no task shows significant negative lift at 10% FDR (26 significant, all positive), and a log-fitted skill gate still gains 4-7 pp under shuffled labels with no significant held-out improvement.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: +relation combines-with AG-20261127-08.
- 2026-11-27: +relation combines-with ENV-20261127-02.
- 2026-11-27: created from bulletin 2026-11-27 (#1).
