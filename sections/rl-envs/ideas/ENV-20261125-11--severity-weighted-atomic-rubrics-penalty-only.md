---
id: ENV-20261125-11
title: Severity-weighted atomic rubrics with penalty-only dodged-bullet items make a usable RL reward
tldr: Typing atomic expert-rubric criteria as primary-intent or dodged-bullet and rewarding the severity-weighted share satisfied, with dodged-bullet items penalty-only, gives a trainable reward for open-ended professional deliverables.
section: rl-envs
also_in: []
subtopic: verifiers-rewards
type: method
tags: [verifier, reward-shaping, agent-rl, process-reward, rubric-reward, agent-fine-tuning]
aliases: [severity-weighted atomic rubric reward, dodged-bullet criteria, penalty-only rubric items, typed expert rubric]
status: emerging
scores: {novelty: 3, leverage: 3, evidence: 3, generality: 3, actionability: 4, surprise: 2}
power: 61
evidence: C
first_seen: '2026-11-25'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.30090', title: Post-Training Agents for Long-Document Reasoning, role: origin}
related:
- {id: PT-20261125-09, rel: variant-of}
superseded_by: null
---

## Summary

Open-ended professional work such as reports, workbooks, drawings and memos has no unit test, a single holistic LLM-judge score is coarse and easy to game, and an all-criteria-met binary reward gives almost no signal when the base model passes only 11% of tasks. Each task here carries an expert-written atomic rubric, about 11 criteria on average and at most 30, where every criterion is one auditable statement with subjectiveness and severity metadata, typed as either primary intent (what the prompt leads the user to expect) or dodged bullet (the absence of a likely error the prompt never mentions, such as importing associations from a broader table than the one named). A judge (GPT-5.4 Mini at medium effort) decides each criterion for 8 samples per prompt, and the reward is the severity-weighted fraction satisfied, with primary-intent criteria weighted 1.0, 0.6 or 0.3, dodged-bullet criteria entering only as penalties, and the total floored at zero. Optimization uses group-relative advantages across the 8 samples with truncated importance sampling and asymmetric clipping of 0.20 and 0.28, training a rank-32 LoRA at full context on 865 tasks in rounds two and three of the paper's recipe. The smoothed training reward rose from 0.37 in the first tenth of the run to 0.67 in the last, peaking at 0.74, and strict pass on the 100-task GDP.pdf holdout rose from 11.0% to 24.5% with mean generated tokens flat at 9,803 versus 9,733, so the gain was not bought with longer outputs; paired trajectories show scope errors fixed (selecting the right wiring diagram by serial number, reading a map by its legend fill encoding, defining the applicable case volume before a formula) and the share of tasks with an executable check of the deliverable nearly tripled. Reach for it when you have an open-ended task with expert criteria and no programmatic verifier, and report strict pass, at-most-one-or-two-points-missed, at-least-90% and tokens or turns alongside mean reward. The caveats are that the paper ablates none of these design choices, one LLM judges every criterion with no reported audit of judge error or reward hacking, and with a floor at zero a response with many penalties and few credits gets the same reward as one with no penalties, which blunts the signal on the worst outputs.

## Detail

### Problem

For open-ended professional work (reports, workbooks, drawings, memos) there is no unit test. A single holistic LLM-judge score is coarse and easy to game, and a binary "all criteria met" reward gives almost no learning signal when the base model passes only 11% of tasks.

Rubrics are the common answer, but the design details decide whether the reward teaches the right behavior: how fine-grained the criteria are, whether serious misses count more than cosmetic ones, and whether the policy is penalized for errors the prompt never mentions.

### Mechanism

**Atomic rubric.** A list of criteria, each one auditable statement that can be judged true or false independently (in this collection, a mean of about 11 criteria per task and a maximum of 30).

**Severity.** A label on each criterion for how much a miss matters; here it sets the weight 1.0, 0.6 or 0.3.

**Strict pass versus criterion pass.** Strict pass means every criterion is met; criterion pass is the fraction met. The first is what users feel; the second is the smoother training signal.

**LLM-as-judge reward.** A model reads the response and rubric and says which criteria hold.

**Rubric construction.** For each task, a domain expert writes a workflow-realistic question from work they recently performed, together with an atomic rubric. Each criterion is a single auditable statement with *subjectiveness* and *severity* metadata and one of two types:

- *Primary intent*: what the prompt leads the user to expect.
- *Dodged bullet*: the absence of a likely error the prompt does not mention (for example, not importing associations from a broader table than the one named).

**Reward.** The policy samples 8 responses per prompt. A judge (GPT-5.4 Mini at medium effort) decides each criterion. The reward is the severity-weighted fraction of criteria satisfied: primary-intent criteria weigh 1.0, 0.6 or 0.3 according to severity; dodged-bullet criteria enter as penalties rather than credit; the total is floored at zero.

**Optimization.** Group-relative advantages across the 8 samples, with truncated importance sampling and asymmetric clipping (0.20 and 0.28), training a rank-32 LoRA at full context. Rounds two and three of the paper's recipe use this reward on 865 tasks.

**Evaluation reporting.** Pair mean score with threshold metrics (perfect, at most one or two points missed, at least 90%) and with tokens or turns spent.

### Why it works

Atomic criteria let a partly right answer earn partial credit, which keeps the reward informative on hard tasks; severity weights keep the model from optimizing cosmetic items. Penalty-only dodged-bullet items cover failures the prompt does not mention, which a positive-only rubric would never reward avoiding, such as scope errors or an unchecked formula convention.

The authors did not run an ablation of these choices, so this is a design rationale, not a tested mechanism. My read: the dodged-bullet type is the most transferable idea, because it turns "don't do the thing nobody asked you to avoid" into a trainable signal.

### Evidence

The evidence is indirect, because the paper evaluates the whole recipe, not the reward design alone.

- **Training signal.** The smoothed training reward rose from 0.37 in the first tenth of the run to 0.67 in the last, with a peak of 0.74; it recovered to a new high after each of five falls of more than 0.10 (Figure 1).
- **Holdout.** Strict pass on the 100-task GDP.pdf holdout rose from 11.0% to 24.5% after the second round (mean of four runs, 23% to 27%), with mean tokens generated flat at 9,803 versus 9,733, so the reward did not simply buy longer outputs (Table 3).
- **Behavior.** In paired trajectories, models trained against these rubrics fixed scope errors the base model made: using a serial number to select the correct wiring diagram, reading a map by its legend's fill encoding, defining the applicable case volume before a formula. The share of tasks with an executable check of the written deliverable nearly tripled (Table 6).
- **Out of domain.** On 220 GDPval tool-using tasks mean rubric reward rose from 0.571 to 0.746 after round three, though that rubric format is the benchmark's own.

### Limitations and open questions

The paper offers no ablation of any rubric design choice, so which parts of the reward matter is unknown. The criteria are judged by one LLM (GPT-5.4 Mini), with no reported audit of judge error or of reward hacking. The expert-written rubrics are costly and proprietary to the collection; subjectiveness metadata is recorded but I did not see how it affects the reward. Strict pass varied by four points over four runs of one checkpoint.

My read: with a floor at zero, a response with many penalties and few credits gets the same reward as a response with none, which blunts the signal on the worst outputs.

### How to apply

Reach for this when you have an open-ended task with expert criteria and no programmatic verifier.

1. Write rubrics as single-statement criteria. Add two types: what the user expects, and errors a careful professional would avoid even if unasked.
2. Give each criterion a severity (the paper uses weights 1.0, 0.6, 0.3); make avoid-type items penalties and floor the total at zero.
3. Judge each criterion with a cheap model, and audit it on 50 hand-labeled criteria before training.
4. Track strict pass, at-least-90% rate and tokens, not only mean reward.

**Smallest experiment:** take 100 tasks you already grade by hand, write atomic rubrics with and without the dodged-bullet type, and compare the rank correlation of each reward with your human grade on 500 sampled responses before running any RL.

Assessment: Try in an experiment. The reward is simple and sensible, but no ablation here shows which parts matter, so test it against human grades before training on it.

### Connections

Rubric-based rewards are established (Gunjal et al., 2026; Viswanathan et al., 2025; Mehta et al., 2026; Bhattarai et al., 2026, as cited by the authors). The specific elements documented here are the expert-authored workflow-realistic tasks, the primary versus dodged-bullet typing, the severity-weighted formula with penalty-only items and a zero floor, and the recommendation to report accuracy alongside tokens and turns.

Connections: this reward is the one used in rounds two and three of idea 9, where its effect is measured jointly with the recipe. Idea 2 shows that putting many criteria in one judge prompt can soften individual ones, so judge each criterion separately or measure the effect before trusting the reward. Idea 6 treats the opposite problem: a wrong-answer penalty can starve exploration.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [Post-Training Agents for Long-Document Reasoning](https://arxiv.org/abs/2611.30090) — Used as the reward in rounds two and three of the paper's recipe, it saw training reward rise from 0.37 to 0.67 (peak 0.74) and strict pass on the 100-task GDP.pdf holdout go from 11.0% to 24.5% at flat token spend (9,803 to 9,733), but no ablation isolates the severity weights, the primary versus dodged-bullet typing or the zero floor from the rest of the recipe.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: created from bulletin 2026-11-25 (#11).
