---
id: PT-20261126-03
title: Harness rehearsal makes cross-harness task demonstrations transfer to a target interface
tldr: Adding rehearsal traces that state the solution in the instruction, so the model only practices the target interface, lifts cross-harness distillation from 8.5% to 58.8% on BFCL multi-turn.
section: post-training
also_in: []
subtopic: distillation
type: recipe
tags: [harness-coupling, agent-fine-tuning, distribution-shift, teacher-student, agent-rl]
aliases: [harness rehearsal, 3H multi-harness distillation, interface competence rehearsal, what vs how decomposition]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 3, actionability: 4, surprise: 4}
power: 77
evidence: B
first_seen: '2026-11-26'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.31427', title: 'Learning What vs. Learning How: Decomposing Task and Interface Competence in Agentic Training', role: origin}
related:
- {id: AG-20261123-06, rel: builds-on}
superseded_by: null
---

## Summary

Standard agent distillation trains on successful teacher trajectories from one harness, but each trace teaches both how to solve the task and how to operate that interface, and a single-harness recipe can raise accuracy on its own harness while wrecking others: fc-only distillation drove BFCL-MT json from 33.5% to 0.1%. The authors split agentic competence into task competence and interface competence and test the split with a 2x2 design crossing cross-harness task supervision with target-harness rehearsal. Rehearsal writes the ground-truth solution into the instruction (the operations to call and values to submit) and runs the 3H checkpoint through the target harness, keeping only verifier-passing traces, so the model practices the interface without having to learn task content. Mixing both lifts BFCL V3 multi-turn fc from 8.5% with cross-harness SFT alone and 32.5% with rehearsal alone to 58.8%, against a base of 44.6%, with the same ordering on MedAgentBench and Reasoning Gym. Use it when you distill an agent that will run behind several tool formats, or will move to a new harness after training, and check performance on a held-out harness rather than only the training ones. The caveats are that the student is a 4B model, most conditions are a single training run, rehearsal needs a programmatic way to write the solution into the instruction, and the rehearsal demonstrator is the 3H checkpoint, so the data depends on a prior distillation step.

## Detail

### Problem

A model that knows how to solve a task can still fail because it cannot express the solution through a given harness (the system prompt, action format and feedback loop it acts through). Standard agent distillation trains on successful teacher trajectories from one harness. Each trace carries two lessons at once, so single-harness training can raise accuracy on that harness while wrecking others: the authors report one case where json-format performance falls from 33.5% to 0.1%.

If you deploy the same policy behind more than one interface, or change harnesses after training, you cannot tell whether the model lost knowledge or lost interface skill. The fix differs: more task data in one case, a little interface practice in the other.

### Mechanism

**Harness.** The interface and loop through which an agent acts: system prompts, action syntax (native function calls, JSON, XML, Python calls), feedback and budgets. Examples here: Terminus-2, mini-swe-agent, Goose, Aider.

**Distillation (SFT on traces).** Training a student on a stronger teacher's successful trajectories with next-token loss over all tokens.

**GRPO (group relative policy optimization).** An RL method that samples $G$ rollouts per task, scores each with a reward, and uses reward minus the group mean as the advantage, with no learned critic.

**Held-out harness.** A harness never used in training, to test transfer.

**Interference.** A drop in performance on one harness caused by training on another.

The authors decompose agentic competence into **task competence** (knowledge and reasoning to solve the task) and **interface competence** (acting on it through a given harness).

**Setup.** Teacher Qwen3-32B, student Qwen3-4B. For each of four benchmarks there are three training harnesses and one held-out harness. A policy $\pi$ acts on task $x$ through harness $h$, producing trajectory $\tau$, scored by a verifier $s(\tau)\in[0,1]$; success is $r(\tau)=\mathbf{1}[s(\tau)\ge 0.99]$.

**Multi-harness distillation (3H).** Keep only successful teacher traces from each training harness and train on their union with $\mathcal{L}(\pi)=-\mathbb{E}_{\tau}\sum_t\log\pi(x_t\mid x_{<t})$ over every token of the serialized trajectory.

**Harness rehearsal.** For each training task $x$ with ground-truth solution $y$, programmatically write an instruction $\rho(x)$ that spells out the operations needed to obtain and submit $y$ (including values to extract from intermediate results). A demonstrator, the 3H checkpoint, runs on $(\rho(x),h)$ through the *target* harness; trajectories that pass the original verifier are kept. An example: "Call ls(a=True) and read its result. Let V be the first entry that does not start with a dot. Call touch(file_name='rehearsal_<V>.txt')". The environment and verifier are unchanged.

**2x2 design.** Cross-harness task supervision (traces from other harnesses) crossed with target rehearsal.

**RL stage.** GRPO with $\hat A_{i}=r(\tau_i)-\tfrac1G\sum_j r(\tau_j)$, $G=6$, no standard-deviation normalization, no KL penalty, applied on tasks where the initial checkpoint succeeds on some but not all attempts. Single-harness and multi-harness GRPO are matched on rollout count.

### Why it works

The authors' explanation: weak transfer after distillation reflects interference in interface competence, not loss of task knowledge. Cross-harness demonstrations become actionable once the model has (re)learned the target interface. They support this with three observations: rehearsal alone reproduces the interference (rehearsing only on fc collapses json from 33.5 to 0.0), the interaction survives when rehearsal traces contain no task knowledge (on Reasoning Gym rehearsal is just transcribing a given answer), and gains on MedAgentBench are comparable on task templates unseen in rehearsal.

They also report that on-policy RL does not show the interference pattern seen with single-harness SFT.

My read: a plausible mechanism for the RL result is that on-policy rollouts only reinforce behavior the model already produces, so they move other interfaces less; the paper reports the pattern but I did not see a test of that explanation.

### Evidence

**Setup.** MedAgentBench, Reasoning Gym, AppWorld and BFCL V3, with disjoint train and test tasks, run in a common sandbox (Harbor; SkyRL for RL). Mean success over four attempts with 95% task-level bootstrap intervals; main distillation and rehearsal comparisons repeated over four training runs; GRPO rows use two seeds.

Table 3 (target harness, success %):

| Benchmark | Cross-harness task SFT | Rehearsal only | Both |
|---|---|---|---|
| BFCL V3 multi-turn (fc; base 44.6) | 8.5 | 32.5 | 58.8 |
| MedAgentBench | 2.0 | 9.5 | 34.4 |
| Reasoning Gym | 19.8 | 20.0 | 37.0 |

Other numbers: on MedAgentBench, replacing target-harness trajectories in 3H with an equal number of rehearsal trajectories lifts target performance from 16.9 to 34.4. fc-only distillation drives BFCL-MT json from 33.5 to 0.1; 3H reaches 50.2 versus 47.7 for json-only. The worst single-harness specialist falls below half of base on the held-out harness in four of six settings, while 3H beats the worst specialist by 10 to 66 points but beats base only on MedAgentBench and AppWorld, stays near base on BFCL and declines on Reasoning Gym. For a frozen base policy, MedAgentBench success ranges from about 4% to 36% across harnesses.

GRPO from the 3H checkpoint: MedAgentBench fc rises from 56.4 to 61.4 with parity-harness rollouts and 66.9 with Terminus-2 rollouts; single-harness GRPO from the base model leaves other harnesses at or above base.

### Limitations and open questions

The authors state that GRPO was run only on MedAgentBench and Reasoning Gym and only on tasks the initial checkpoint solves sometimes; the RL experiments ask how RL reshapes existing competence, not whether it can solve unsolved tasks.

My read: the student is a 4B model, most conditions are a single training run (the headline comparisons have four), and the rehearsal demonstrator is the 3H checkpoint, so rehearsal data quality depends on a prior distillation step. Rehearsal requires a programmatic way to write the solution into the instruction, which is easy for BFCL-style calls and harder for open-ended tasks. 3H does not consistently beat the base on held-out harnesses.

### How to apply

**When:** you distill or fine-tune an agent that will run behind several tool formats, or you will swap harnesses after training.

1. Collect successful teacher traces from several harnesses (task supervision).
2. For tasks with known solutions, auto-write instructions that state the operations, run a demonstrator on the target harness, and keep verifier-passing traces (interface supervision).
3. Mix both; check held-out-harness performance, not only the training ones.
4. If you follow with RL, expect single-harness GRPO to preserve other interfaces better than single-harness SFT did.

**Pitfall:** rehearsing on only one harness reproduces interference on the others.

**Smallest experiment:** take your current SFT data from one harness, add a few hundred rehearsal traces for a second harness, and compare success on that second harness against SFT on the first harness alone.

Assessment: Try in an experiment. A cheap, well-controlled recipe with large effects in the authors' 4B experiments, but tested on small models and mostly single training runs.

### Connections

Prior work treats harness choice as an evaluation variable (Harness-Bench) or optimizes the harness itself (Meta-Harness, HarnessOpt-Bench). The new element is a controlled separation of what a trace teaches, plus rehearsal traces that supply interface supervision without task knowledge, and a 2x2 test showing the two interact.

**Connections.** Idea 6 (InstructReplay) addresses a related problem, keeping behavior intact while fine-tuning, with replay of the model's own outputs; rehearsal is its interface-specific cousin. Idea 5 uses a similar group-relative learner. Idea 9 (coding agents optimizing skills) edits the harness side instead of the policy side. ShellBench (also in today's batch, but not selected as an idea) found that harness choice moved off-the-shelf models little, which is compatible: this idea concerns models *trained* through a harness.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [Learning What vs. Learning How: Decomposing Task and Interface Competence in Agentic Training](https://arxiv.org/abs/2611.31427) — In a 2x2 harness decomposition with a Qwen3-4B student, adding target-harness rehearsal traces to cross-harness SFT raised BFCL V3 multi-turn fc from 8.5% to 58.8% (base 44.6%) and MedAgentBench from 2.0% to 34.4%, while fc-only distillation dropped BFCL-MT json from 33.5% to 0.1%.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: created from bulletin 2026-11-26 (#3).
