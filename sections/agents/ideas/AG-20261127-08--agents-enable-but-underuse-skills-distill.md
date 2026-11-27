---
id: AG-20261127-08
title: Agents enable but underuse skills; distill paired-run differences into the skill text
tldr: Agents given a relevant skill often do not open or apply it; revising the skill text from same-task natural-versus-supported run differences closes much of that underuse gap.
section: agents
also_in: []
subtopic: architectures-scaffolds
type: method
tags: [skills, agent-evaluation, harness-coupling, instruction-following, scaffold-search]
aliases: [EBU-Evolve, EBU-Bench, EBU gap, MedSkill-E2, enable-but-underuse gap]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 4, actionability: 4, surprise: 4}
power: 80
evidence: B
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.34250', title: 'MedSkill-E$^{2}$: Evaluating and Evolving Skills for Medical Agents', role: origin}
related:
- {id: AG-20261127-06, rel: variant-of}
- {id: AG-20261126-09, rel: alternative-to}
- {id: ENV-20261127-01, rel: combines-with}
superseded_by: null
---

## Summary

A skill is a folder of instructions and resources the harness exposes by name and description, and the agent opens it only if it decides to. Across nine models and two harnesses on 154 medical tasks, mean skill-use accuracy is 38.38% when the skill is merely available versus 82.32% when the run is instructed to read and execute it, an enable-but-underuse gap of 43.94 pp that appears in every model-harness pair, and general encouragement does not close it. EBU-Evolve improves a skill from differences between two executions of the same task: run each training task naturally and with operational support, keep pairs whose scores differ, diagnose whether the agent failed to open the skill or failed to apply a rule it read, distill the difference into body-only, description-only and joint revisions, validate under natural use, then freeze the winner. On 87 unseen test tasks of four skill groups, frozen skills lift accuracy by 55.17, 33.33 and 40.23 pp in three configurations and take the common GLM-5.2 setting from 41.38% to 96.55%, against 2.30 pp for the best GEPA configuration and 18.39 pp for SkillOpt. Frozen skills transfer across models (16.09 to 50.57 pp) and harnesses (22.99 to 40.23 pp from Claude Code to Codex), and description-only editing keeps 85.42% of the joint net gain by raising skill opening from 62.07% to 100%. Use it when you maintain skills or tool documentation your agent often ignores, logging whether the skill was opened and pairing 10-30 tasks with and without an instruction to apply it. The caveat is that the tasks are biomedical only, evolution was tested on 87 tasks of four skill groups, reflection used a strong Opus 4.8 model the baselines may not match, and a description rewritten until it is always opened may overfit the benchmark's phrasing.

## Detail

### Problem

A skill is a folder of instructions and resources the agent can open on demand. The harness shows only the skill's name and description; the body and reference files are read if the agent decides to open them. If the agent skips the skill, or opens it and misapplies a constraint, the skill adds nothing. In medical tasks a single overlooked criterion gives a plausible but wrong result.

The paper names this gap EBU (Enable But Underuse) and measures how large it is. It matters to any team that ships skills or tool documentation and assumes availability is enough.

### Mechanism

**Natural Skill Use.** The skill is available through the harness's standard mechanism and the agent decides whether and how to use it. **Prompted Skill Use** adds general encouragement. **Operationally Supported Skill Use** requires reading the skill and executing its steps, without giving the task's answer.

**EBU gap.** $\Delta_{\mathrm{EBU}}(m,h)=P_{\text{supported}}(m,h)-P_{\text{natural}}(m,h)$, the difference in skill-use accuracy for model $m$ in harness $h$.

**Harness.** The agent runtime; the paper uses Claude Code and Codex, which differ in resource discovery and tool interaction.

**Benchmark (EBU-Bench).** 331 tasks and associated skills in 12 task groups: public-benchmark tasks and tasks authored from research questions. Each task has a fixed scoring program (numeric tolerance or rule match). Most experiments use EBU-Bench Lite (154 tasks).

**EBU-Evolve.** It improves a skill from differences between two executions of the same task.

1. Run each training task under Natural and Operationally Supported Skill Use.
2. **Select cases** where scores differ, and **diagnose utilization**: did the agent fail to open the skill, or fail to apply a rule it read?
3. **Distill** the difference into a revision, producing three candidates from one edit: body-only, description-only, and joint.
4. **Validate** under Natural Skill Use on validation tasks (three runs each) and keep the best; if nothing improves, the original skill is retained.
5. **Freeze** the selected skill and test it on unseen tasks of the same skill.

The reflection, editing and review were done by Claude Opus 4.8; evolved skills were frozen and then tested on other models and harnesses.

### Why it works

The authors identify two underuse mechanisms in traces: failing to acquire the rule (the agent never opens or reads the relevant part) and failing to apply it correctly after reading. Operational support names a skill and demands execution, which is guidance that is not available at deployment. Paired runs provide a controlled contrast on the same task, so the revision targets exactly what the support supplied. Description edits fix access; body and reference edits support task completion. My read: this is a form of "learn the system prompt from a teacher run" and the paired contrast is the useful part.

### Evidence

**Setup.** Nine models (GPT-5.5, Claude Opus 4.8, Gemini 3.8 Flash, DeepSeek V4 Pro, GLM-5.2, Atria-Dawn-Preview, Kimi K2.7 Code, Qwen3.8 Max, Qwen3.8 27B FP8), two harnesses, 154 tasks and 11,088 retained run records.

**The gap.** Mean skill-use accuracy is 38.38% under Natural Skill Use and 82.32% with operational support. All model-harness combinations show EBU, with a mean gap of 43.94 pp. For the same model, natural accuracy differs by up to 38.96 pp across harnesses, and model rankings can change. General encouragement (Prompted Skill Use) was not enough. EBU can persist even after a skill is opened.

**EBU-Evolve on 87 unseen test tasks of four skill groups (Natural Skill Use).**
- Gains of 55.17 (GLM-5.2), 33.33 (GPT-5.5) and 40.23 pp (Atria).
- In the common GLM-5.2 configuration, accuracy goes from 41.38% to 96.55%, against gains of 2.30 pp (best GEPA configuration) and 18.39 pp (best SkillOpt). Giving GEPA and SkillOpt the same paired feedback adds 2.30 and 10.34 pp to their standard settings.
- Frozen skills transfer: gains of 16.09 to 50.57 pp across models, and 22.99 to 40.23 pp when the same model moves from Claude Code to Codex; all 12 transfer configurations gain.
- Description-only editing keeps 85.42% of the joint net gain. Fixing the evolved description but removing body and reference access drops accuracy from 91.95% to 50.57%; the description edit alone raises skill opening from 62.07% to 100%.

### Limitations and open questions

Biomedical tasks only. Evolution was tested on four skill groups built from literature questions, with 87 test tasks, so confidence intervals are wide. Reflection and editing use a strong model (Opus 4.8), which the baselines may not match in access. Operational support differs in specificity between public and authored tasks, so the authors report results by source. The paper's AI-use statement says generative AI was used to generate synthetic data and assist implementation. No check by clinical experts is reported; the authors list expert validation in real workflows as future work.

My read: a skill whose description is rewritten until it is opened 100% of the time may overfit to the benchmark's phrasing; testing on a new task family would show it.

### How to apply

Use it if you maintain skills or tool docs and your agent often ignores them.

1. Log whether the skill was opened, per task.
2. For 10-30 tasks, run once with no help and once with "read the skill fully and apply its steps".
3. For pairs with different outcomes, write down which rule or constraint the supported run used.
4. Edit the skill description (so it is opened) and the body (so the rule is stated where it is needed); keep a copy of the original.
5. Validate on held-out tasks of the same skill, repeated three times, and on a second harness.

Cost: two runs per task plus an LLM reflection pass. Pitfall: operational prompts that leak the answer. **Smallest experiment:** measure your own EBU gap on one skill with 20 tasks.

Assessment: Try in an experiment. A clear way to measure and shrink the gap between available and used skills, but tested only in biomedical tasks with a small evolution set.

### Connections

Prior methods named by the authors: GEPA (prompt optimization from reflection) and SkillOpt (skill optimization), plus the observation from skills benchmarks that skills help on average. EBU-Evolve differs in using a paired contrast between unaided and supported runs of the same task to decide what to write, and in separating description (access) from body (execution) edits. The paper also contributes the EBU gap as a metric.

**Connections.** Idea 1 questions how much of skill lift on SkillsBench is real and how much grader quirks; this paper finds a large gap between available and used skills. Both say that "skill present" is not "skill used", and idea 1's permutation test is how to check an evolved skill's gain. Idea 6 (TransHarness) also evolves guidance from same-task success-failure pairs and adds replay checks that EBU-Evolve lacks. Idea 4 shows harness differences in a second domain.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [MedSkill-E$^{2}$: Evaluating and Evolving Skills for Medical Agents](https://arxiv.org/abs/2611.34250) — Across nine models and two harnesses on 154 medical tasks, mean skill-use accuracy is 38.38% with the skill available versus 82.32% with operational support (EBU gap 43.94 pp), and EBU-Evolve lifts unseen-task accuracy from 41.38% to 96.55% for GLM-5.2 versus +2.30 pp (GEPA) and +18.39 pp (SkillOpt).

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: created from bulletin 2026-11-27 (#8).
