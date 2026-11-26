---
id: AG-20261126-09
title: Unguided coding agents optimize other agents better than published methods; guidance adds little
tldr: Handing an off-the-shelf coding agent a task prompt, train split and dollar budget to edit a target agent beat every published optimizer on five benchmarks, while added guidance skills and subagents did not help.
section: agents
also_in: []
subtopic: self-improvement
type: method
tags: [scaffold-search, budget-forcing, agent-evaluation, cost-reduction, coding-agents]
aliases: [AGL-SKILL, agentic skill optimization]
status: emerging
scores: {novelty: 3, leverage: 4, evidence: 4, generality: 3, actionability: 5, surprise: 4}
power: 74
evidence: B
first_seen: '2026-11-26'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.31794', title: 'AGL-SKILL: Coding Harnesses as Agentic Optimizers', role: origin}
related:
- {id: ENV-20261126-10, rel: alternative-to}
superseded_by: null
---

## Summary

Agent skills — folders of instructions, scripts and reference material loaded on demand — change agent performance, but handcrafting them is limited by the author and score-driven search that keeps only edits raising the measured score is inefficient. AGL-SKILL gives an off-the-shelf coding agent (Claude Code, Codex, GitHub Copilot CLI) a benchmark's train split, a task prompt and a fixed dollar budget of $5, $10 or $25, reaching the model API through a proxy that meters tokens and blocks web access. Within budget the optimizer may inspect training examples, edit the target's prompts, code, tools and skills, run the target on any subset of the split, change its reasoning effort and decide when to stop; the edited target is then frozen and scored on the held-out test split by a separate evaluator. Against the strongest published method as reported by SkillOpt, optimized rows were better on every benchmark at every harness and budget combination, from 3% on SearchQA to 98% on LiveMath, an average of +0.15 accuracy or 35%. Guidance did not pay: the Vanilla condition (task prompt only) scored higher than the two-phase, purpose-written skill, subagent and worktree variants more often than lower, with differences attributed to per-run variance, while $25 beat $5 in two thirds of benchmark-harness pairs by 0.03 on average and a quarter of $25 optimizers stopped with over a fifth unspent. Use it when you own a target agent with editable prompts, skills or code and a train/test split: start with the plain version at $10, run 2-3 repetitions and compare means, and log the frozen agent's inference cost, which fell to almost nothing on ALFWorld but rose on three of five benchmarks despite a prompt asking for lower cost. Caveats: at most three repetitions per condition, published-method numbers come from another paper's setup, train splits can be small (35 LiveMath examples), and the optimizer models are stronger than the gpt-5.4-mini target.

## Detail

### Problem

Agent skills (folders of instructions, scripts and reference material loaded on demand) change how well an agent performs, but handcrafting them is limited by the author, and score-driven search, where a model tweaks the skill and keeps only edits that raise the score, is inefficient. If a general coding agent can already do the optimizing, teams can drop specialized optimization pipelines and spend effort on the training split and the evaluation instead.

It also tells you where not to spend: the authors find that scaffolding the optimizer with a purpose-written skill, parallel subagents or Git worktrees does not reliably help.

### Mechanism

**Harness.** The software loop around a model (here Claude Code, Codex, GitHub Copilot CLI) with tools, context management and subagents.

**Skill.** Reusable workflow, code or prompts that guide an agent on a class of tasks.

**Target agent / optimizer agent.** The target is the agent being improved; the optimizer is the coding agent that edits it.

**Held-out split.** Data the optimizer never sees; the frozen target is scored there.

**Score-driven search.** Hill climbing where edits are kept only if the measured score improves (as in SkillOpt and GEPA-style methods).

**AGL-SKILL** setup, per run:

1. The optimizer gets the train split of a benchmark, a task prompt and a fixed dollar budget ($5, $10 or $25). It reaches the model API through a proxy that monitors token use and blocks web access.
2. Within budget it may inspect training examples; edit the target's prompts, code, tools and skills; run the target on any subset of the split; and decide when to stop. It may also change the target's reasoning effort.
3. The edited target is frozen and run on the test split by a separate evaluator.

Five optimizer conditions: **Vanilla** (task prompt only), **Two-phase** (maximize accuracy first, then cut inference cost), **Agent Lightning skill** (a published skill covering budget tracking, the action space of edits, reading failure evidence), **Subagents** (the skill plus two subagents exploring in parallel) and **Git worktrees** (the skill plus candidate solutions in separate worktrees compared on training cases). The last two run at $10 only. Up to three repetitions per condition.

Five benchmarks: SpreadsheetBench (120 train / 280 test), ALFWorld (3,553 / 134), OfficeQA (50 / 172), SearchQA (400 / 1,400) and LiveMath (35 / 125). The target uses gpt-5.4-mini; the optimizers are claude-opus-5 in Claude Code and gpt-5.6-sol in Codex and Copilot.

### Why it works

The authors' explanation comes from reading optimizer traces: with or without guidance, optimizers arrive at much the same solutions, and these are seldom far-fetched; making the target more robust is often enough to score highly. Agents have a wider but directed search space than a hill climber and already have the tools and environment to inspect failures.

For guidance adding little, they note that guided optimizers find the same solutions slightly less often; for example on ALFWorld nearly every Vanilla run removes the model from the final agent, versus about two thirds of two-phase runs and half of Subagents runs.

My read: when the optimizer is a stronger model than the target, much of the gain may be the optimizer writing the task-specific code the target lacked, which would not carry over if both were the same size.

### Evidence

**Setup.** Held-out accuracy, target gpt-5.4-mini; baselines are the unoptimized target and published methods as reported by SkillOpt (Yang et al.). Optimized rows average harness-budget means over all three harnesses and budgets.

The authors report:

| Quantity | Value |
|---|---|
| Versus the strongest published method | better on every benchmark at every harness and budget combination |
| Improvement over best previous result | 3% on SearchQA to 98% on LiveMath; average +0.15 accuracy, 35% |
| $25 budget versus $5 budget | $25 beats $5 in two thirds of benchmark-harness pairs, by 0.03 on average |
| Unspent budget | at $25, a quarter of optimizers stop with over a fifth unspent |
| Where the spend goes | about two thirds on the optimizer's own calls |

Harnesses differ mostly within run-to-run variance; Codex leads on SpreadsheetBench and LiveMath, Claude Code on OfficeQA, and the three are roughly even on ALFWorld and SearchQA. Skills, two-phase, subagents and worktrees score lower than Vanilla more often than higher, with differences attributed to per-run variance. Test-time cost of the optimized target falls to virtually nothing on ALFWorld and by about a third on LiveMath, but on the other three benchmarks the optimized agent is more expensive than baseline despite a prompt asking to keep cost down. The HTML tables were scrambled in conversion, so I quote only numbers stated in the text.

### Limitations and open questions

The authors report that the final score varies from run to run without tracking the optimization cost, that differences between harnesses and between guidance variants are mostly attributable to per-run variance, and that runs are not always limited by their budget.

My read: at most three repetitions per condition, the published-method numbers come from another paper's setup, the optimizer models are stronger than the target, and train splits can be small (35 LiveMath examples). A held-out split reduces but does not remove selection effects across many runs. I did not read the appendix tables.

### How to apply

**When:** you own a target agent with a measurable benchmark and a train/test split, and it uses editable prompts, skills or code.

1. Start with the plain version: a coding agent, a clear task prompt, the train split and a $10 budget; freeze and evaluate on a held-out split.
2. Run 2 to 3 repetitions and compare means, since scores vary run to run.
3. Log the final agent's inference cost and check it did not rise, since the prompt asking for lower cost did not prevent increases on three benchmarks.
4. Only then try extras such as a purpose-written skill, parallel subagents or worktrees, and expect small or no gains.

**Cost:** $5 to $25 per run in the paper.

**Smallest experiment:** one benchmark of yours, Vanilla at $5 versus $25, three runs each, compare held-out accuracy and test-time cost.

Assessment: Try in an experiment. A strong simple baseline that beat published optimizers, though based on few repetitions and with an optimizer stronger than the target.

### Connections

Closest work named in the paper: SkillOpt and GEPA-style score-driven skill optimizers; VeRO (a harness of versioned snapshots and budgeted evaluation for a coding agent), HarnessOpt-Bench, CASD, SkillOpt-Lite and OEO; plus self-editing agents such as Reflexion, STOP, SICA and the Darwin Godel Machine. The difference is the question asked: an off-the-shelf coding agent given only the task prompt as optimizer, with controlled tests of guidance variants and budget.

**Connections.** Idea 10 (RecurSciBench) asks whether agent self-improvement transfers and finds uneven results; this idea optimizes in-distribution on a train split and tests on a held-out split of the same benchmark. Idea 3 changes the policy through training, while this idea changes the harness and skills. Idea 2 (delegation) is an architectural choice an optimizer could discover.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [AGL-SKILL: Coding Harnesses as Agentic Optimizers](https://arxiv.org/abs/2611.31794) — A vanilla off-the-shelf coding agent optimizing a gpt-5.4-mini target beat the strongest published skill-optimization method on all five benchmarks at every harness and budget combination, averaging +0.15 accuracy (35%) from 3% to 98%, while purpose-written skill, subagent and worktree guidance scored lower than Vanilla more often than higher.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: +relation alternative-to ENV-20261126-10.
- 2026-11-26: created from bulletin 2026-11-26 (#9).
