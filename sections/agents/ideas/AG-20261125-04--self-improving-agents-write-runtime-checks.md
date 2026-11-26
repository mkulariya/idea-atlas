---
id: AG-20261125-04
title: Self-improving agents write runtime checks that stay silent on most failures they can repair
tldr: Agents that self-improve programs to high task scores write runtime checks that miss 52–89% of failures after an environment change, though the same agents repair most of those failures when called.
section: agents
also_in: []
subtopic: self-improvement
type: finding
tags: [agent-evaluation, verifier, distribution-shift, fault-injection, honesty]
aliases: [detection gap, detection-versus-score, runtime check coverage, Blind to What They Can Fix]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 3, actionability: 4, surprise: 4}
power: 77
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.29807', title: 'Blind to What They Can Fix: A Detection Gap in Self-Improving Agents', role: origin}
related:
- {id: AG-20261125-01, rel: alternative-to}
- {id: AG-20261125-10, rel: combines-with}
- {id: ENV-20261126-10, rel: combines-with}
superseded_by: null
---

## Summary

Programmatic agents can write and improve a task program and then let it run without a model call at each step, but only if something notices when the program stops working; the improvement loop selects versions by task score alone and never tests whether the program reports its own failure. The authors measure three things on the same pairs of frozen program and controlled environment change: task performance, whether any detector alarms, and whether the authoring agent repairs the program when called without being told what changed. Across five domains (Seaquest, Kangaroo, Craftax, service autoscaling, AppWorld), Codex's own checks missed 52% of failures in Seaquest, 55% in Kangaroo, 64% in autoscaling and 89% in AppWorld, though they caught all eight in Craftax. Outside detectors did better: an outcome rule that alarms when success falls below half the reference mean missed 7-9% of failures with false alarms on at most 5% of unchanged episodes, and a general monitor learned from 20 normal episodes missed 0-18% but alarmed on 52% to 69% of units where the program kept working. Repair was possible: agents repaired most failures their checks missed, and adding 32 recorded-relation checks removed most misses, while telling Codex its checks were the only call for repair cut misses from 69% to 6% in Seaquest and from 95% to 0% in AppWorld. Use it when shipping agent-written programs that run unattended: add an outcome rule or a general monitor, ask for checks on recorded relations rather than mechanisms, and report detection and false-alarm rates next to score. The main caveat is that the tasks are simulated games, a simulated service and one API suite, with small change catalogues and few independent programs with checks outside Seaquest.

## Detail

### Problem

A common design is for an agent to write a program for a task, improve it from execution feedback, and then let the program run without the agent. Nothing watches each step, so the system only improves further if something notices a failure and calls the agent back.

The improvement loop selects versions by task score alone. It never tests whether the program would report its own failure. So a program can reach a high score during development and then, after an API changes or a workload shifts, keep acting with every action valid while its checks keep passing. The paper shows this is the normal outcome, and that a failure that raises no alarm stays unrepaired, which caps what self-improvement is worth in unattended operation.

### Mechanism

**Programmatic agent.** An agent that writes a program once and lets it act, so the task runs without a model call at every step and the logic stays readable.

**Recursive self-improvement (RSI).** The agent runs the program, reads failures, revises the code and tests the revision; the model stays fixed while the program improves. A version is kept only if its task performance is better.

**Runtime check, monitor, alarm.** A check is code that inspects observations or records and returns confirmed, contradicted or none. A monitor turns such signals into an alarm that calls the agent back. In the paper, a check counts as broken once it has been contradicted at least three times and more often than confirmed.

**Task failure.** A frozen program under an environment change keeps less than half of its original performance ($J^-_u<\tfrac12 J^0_u$).

The authors built a framework that measures three things on the same pairs of frozen program and environment change: task performance, failure detection and repair.

1. Agents (Codex as the main author, Claude Code and five open models for parts of the study) write and improve programs in five domains: Seaquest and Kangaroo (arcade games), Craftax (survival game), service autoscaling, and AppWorld (a daily API report).
2. Programs are frozen and deployed under controlled changes to a regularity they might rely on: what an action does, a resource condition, an observation delay or the meaning of an API response. Most changes were written by sessions that saw neither programs nor results ("blind changes"). A change is kept only if the task is still doable.
3. Each failure is classified twice: did any detector alarm (in at least half of the unit's episodes), and does the authoring agent repair the program when called without being told what changed? The detection gap is the share of failures that raised no alarm yet were repaired.
4. Detectors compared: the program's own checks, an outcome rule (alarm if success falls below half the reference mean), a general monitor (learned usual ranges of generic statistics from 20 normal episodes) and a monitor written on request.
5. Cause tests: add checks on recorded relations to a fixed program (32 added relations, even drawn at random), and tell the agent its checks are the only thing that calls it back.

### Why it works

The failures are visible in execution records that the checks never inspect. The checks agents write under the standard instructions test mechanisms (what an action costs, how fast things move), not statistical relations among recorded variables, so many changes slip past them.

Purpose also matters: instructions outside AppWorld do not say the program will run unattended or that an alarm calls the author back. Once the strongest agent was told its checks are the only call for repair, it wrote checks that missed few failures.

My read: this is an incentive and specification gap, not a capability gap, because the loop optimizes score and nothing in it rewards detection. The finding that open models still missed most failures even when informed suggests capability does matter for writing good checks.

### Evidence

- **Own checks miss most failures.** Codex's checks missed 52% of failures in Seaquest, 55% in Kangaroo, 64% in service autoscaling and 89% in AppWorld; in Craftax they caught all eight (Table 1). Sixteen new Codex programs under blind Seaquest changes missed 69%, and 16 new AppWorld programs missed 95% in a registered replication. Over three generations of autoscaling programs the miss rate went 95%, 79%, 64%. No threshold on how often checks were contradicted made them a usable alarm.
- **Outside detectors do better.** The outcome rule missed 7%, 9% and 8% of failures in Seaquest, Kangaroo and autoscaling, with false alarms on at most 5% of unchanged episodes. The general monitor missed 5%, 18% and 0%, but alarmed on 52% to 69% of units where the program kept working.
- **Repair is possible.** Agents repaired most failures their checks missed, when called without being told the change.
- **Cause.** Adding 32 recorded-relation checks (even random ones) removed most misses. An informed Codex cut misses from 69% to 6% in Seaquest and from 95% to 0% in AppWorld; three open models still missed 84% to 100%.
- **Cost.** In replayed deployments a timely trigger recovered most lost value (0.82 for both a success-halving rule and the general monitor), while a perfect repairer behind the program's own checks recovered almost none.

### Limitations and open questions

The authors list five limits. The program's own checks exist only in formats that ask for them, and versions are selected by task performance only; they did not test improvement loops that select versions by detection quality. Outside Seaquest, few independent programs have checks and the change catalogues are small, with joint intervals on pooled misses reaching down to 40% in Seaquest, 44% in autoscaling and 8% in Kangaroo. The AppWorld failure criterion was fixed after seeing results, then registered for a replication.

My read: tasks are simulated games, a simulated service and one API suite, so generalization to messy production agents is open. Repair was one session with a fixed tool-call budget.

### How to apply

Reach for this if you ship agent-written or agent-maintained programs that run unattended.

1. Do not trust "the checks pass" as evidence of health. Add an outcome rule: alarm when the success rate over a window drops below half its reference mean (the paper's rule missed 7-9% of failures where outcomes are observable).
2. Where outcomes are not observable, add a general monitor on recorded variables, learned from about 20 normal episodes, and accept its false alarms if repair sessions are cheap.
3. When asking an agent to write checks, say the program will run unattended and that an alarm is the only thing that calls it back, and ask for checks on recorded relations, not just mechanisms.
4. Report detection rate and false-alarm rate next to score in your self-improvement evals.

**Smallest experiment:** take one deployed task program, apply three environment changes you can script (a format change, a delay, a changed API field), and log whether its own checks, an outcome rule and a 20-episode monitor alarm.

Assessment: Try in an experiment. The detection-versus-score distinction is cheap to measure in your own system, though the evidence is simulated tasks with small catalogues.

### Connections

Runtime verification, invariant detection and drift detection (Ernst et al., 2007; Gama et al., 2014; Breck et al., 2019, as cited) check executions with external tools, and self-improvement work scores programs by task performance. This paper measures whether the improved program reports its *own* failures, with preregistered predictions (of 120, 69 held, 36 failed and 14 were undetermined).

Connections: it is the self-improving-agent version of idea 1: both show agents failing silently when a signal looks fine. Idea 10 is a concrete detector in the same spirit (check outputs against ground-truth records in code). Idea 5 prices what a retry costs, which matters for how often such an alarm should trigger a repair session.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [Blind to What They Can Fix: A Detection Gap in Self-Improving Agents](https://arxiv.org/abs/2611.29807) — Under controlled environment changes, Codex-written program checks missed 52-89% of failures in four of five domains (including 69% and 95% in blind replications), while an outcome rule missed 7-9% with at most 5% false alarms, and agents repaired most missed failures when called.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: +relation combines-with ENV-20261126-10.
- 2026-11-25: +relation combines-with AG-20261125-10.
- 2026-11-25: created from bulletin 2026-11-25 (#4).
