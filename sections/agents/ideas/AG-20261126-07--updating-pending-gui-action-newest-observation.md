---
id: AG-20261126-07
title: Updating a pending GUI action with the newest observation fixes failures latency cannot explain
tldr: Keeping a GUI agent's pending click revisable until commitment and re-binding its coordinates to the newest observation raised success from 0% to 100% on a moving-target task without faster responses.
section: agents
also_in: []
subtopic: computer-web-agents
type: method
tags: [gui-agents, action-grounding, agent-evaluation, replay]
aliases: [Chronotope, EODA, ChronotopeGym, late binding, late-bound GUI action]
status: emerging
scores: {actionability: 4, evidence: 4, generality: 3, leverage: 4, novelty: 4, surprise: 4}
power: 77
evidence: B
first_seen: '2026-11-26'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.31522', title: 'One Step, Four Clocks: How GUI Agents Fall Out of Step', role: origin}
related:
- {id: AG-20261126-04, rel: combines-with}
superseded_by: null
---

## Summary

GUI agents are usually evaluated as if the interface waits for them, but real screens keep changing: pages finish loading, targets move, authenticator codes expire, so an action can be right for the screenshot the agent saw and wrong for the screen it lands on. The authors split each response into four events (environment change, observation availability, decision commitment, action receipt) and derive four failure modes: missed, stale, invalidated and late. ChronotopeGym tests five dynamic tasks under Paused, Replay and Live conditions, where Replay gives the same observations at the same times as Live but keeps the response valid until it is acted on. With the same evidence at the same times, a planner-grounder agent scored 96% in Replay versus 29% in Live. A Chronotope controller keeps observing during deliberation and, before commitment, keeps the proposed action if still valid, updates only outdated arguments such as coordinates or typed values from the freshest observation, or restarts inference if the operation itself is no longer appropriate; on Moving Balloon at 4, 8 and 16 pixels/s with Qwen3.8-27B this moved success from 0% to 100%, and on three tasks from 0% to 99-100%, with UI-TARS-1.5-7B reaching 69%. Use it when your agent acts on self-changing screens or runs a large slow model, logging the four timestamps per step rather than total latency. The caveat is that the tasks are synthetic and few, and a late-bound argument cannot repair a change that happens after commitment.

## Detail

### Problem

GUI agents are usually evaluated as if the interface waits for them: observe, think, act, judge the action. Real interfaces keep changing. A page finishes loading, a notification disappears, a target moves, an authenticator code expires. A click can be right for the screenshot the agent saw and wrong for the screen it lands on.

The obvious explanation, "the agent is too slow", does not say what to fix. The authors show the same model latency can hide different failures that need different remedies. An agent that waits for a faster model when its real problem is an outdated click target will not improve.

### Mechanism

**GUI / computer-use agent.** A model that acts on a screen by reading screenshots and issuing clicks and keystrokes.

**Latency.** Time from observation to action. The authors argue a single latency number cannot say where correctness was lost.

**Grounding.** Mapping a decision like "click the balloon" to pixel coordinates.

**Agent S3, UI-TARS-1.5-7B, Qwen3.8-27B.** The comparison controller and models used in the paper.

**Late binding.** Leaving a state-dependent argument (a coordinate, a value) unspecified until just before commitment, then filling it from the freshest observation.

**The four clocks (EODA).** Each response is split into four events: environment change $E$, observation availability $O$, decision commitment $D$ (when the action can no longer be changed), and action receipt $A$ by the application. This yields four failure modes:

1. **Missed:** relevant evidence disappears before it is observed.
2. **Stale:** the agent acts on an outdated observation when newer evidence existed.
3. **Invalidated:** an action correct at $D$ becomes wrong before $A$.
4. **Late:** the response reaches $A$ after its opportunity closed.

**ChronotopeGym** tests five dynamic tasks (Disappearing Notification, Moving Balloon, Expiring Authenticator Code, Displaced Button, Reversing Trading Signal) under three conditions. **Paused**: the interface waits while the agent deliberates. **Replay**: the agent sees the same changing observations at the same times as in Live, but the response stays valid until it acts. **Live**: both observation and task state keep advancing. Comparing Replay with Live shows what the task changing under the agent costs; further interventions vary which observation supports the final action while holding decision start, commitment and receipt fixed.

**Chronotope controller.** It (1) keeps an interaction history of EODA event streams with an Event Dependency Matrix recording which events depend on which; (2) keeps observing while the model deliberates; (3) runs decision meta-control before commitment: keep the proposed action if still valid, update only outdated arguments (late binding), or restart inference if the operation itself is no longer appropriate; (4) runs history maintenance and meta-control on the accelerator to avoid host-device round trips.

### Why it works

The authors' reasoning: the gap is not the model choosing a wrong action but the action being right for an earlier state. When only an argument (a click position or a typed code) is outdated, restarting inference wastes tokens and time; replacing just that argument with a value from the newest observation preserves the intended operation. Remedies differ by failure mode: continued observation fixes missed evidence, revising a pending decision fixes stale decisions, and neither can repair a change after commitment, which needs a faster path to $A$.

My read: the clean part is the controlled intervention, which holds timing fixed and changes only which observation supplies the click, so the improvement cannot be a speed effect.

### Evidence

**Setup.** Five tasks, 100 random seeds each, separate from development seeds; one NVIDIA GB10 GPU with local models served by vLLM and TensorRT. Chronotope mainly uses Qwen3.8-27B, Gemma-4-31B and UI-TARS-1.5-7B; Agent S3 mainly EvoCUA-8B and UI-TARS-1.5-7B. Each configuration enters a timing study only after passing a Paused qualification check.

The authors report:

| Result | Value |
|---|---|
| Planner-grounder agent, same evidence same times | 96% (Replay) versus 29% (Live) |
| Moving Balloon at 4, 8, 16 pixels/s, updating pending click, Qwen3.8-27B | 0% to 100% |
| Update on three tasks, Qwen3.8-27B | 0% to 99–100% |
| Update with UI-TARS-1.5-7B | 0% to 69% |
| Chronotope versus Agent S3, 27B model | 85.5% lower median end-to-end latency |
| Token use versus Agent S3 | up to 87.5% lower |

In the Disappearing Notification figure, an Agent S3 model scores 96% when paused and falls as the time limit shortens in Live. I read the setup and the Paused/Replay/Live analysis in full; results for the later claims are from the paper's introduction and abstract.

### Limitations and open questions

Tasks are synthetic and few (five), each with 100 seeds. The comparison with Agent S3 uses different models for the two controllers (EvoCUA-8B versus Qwen3.8-27B), so latency and token savings mix controller and model effects. I read through the Paused/Replay/Live study design and not the later ablations, so the exact breakdown of the 85.5% latency saving is not known to me.

My read: this is a convincing existence proof for a failure class, but there is no test on real applications, and a revisable pending action can also make bad moves if the newest observation is noisy.

### How to apply

**When:** your GUI or web agent acts on screens that change on their own (loading content, timers, moving UI) or runs with a large, slow model.

1. Log the four timestamps per step (change, observation, commit, receipt), not just total latency.
2. Run Paused, Replay and Live variants of a few dynamic tasks to see which gap you have.
3. Keep the proposed action editable until commit; bind coordinates and values from the freshest screenshot, and re-decide only if the operation is invalid.
4. Skip duplicate screenshots and keep control state close to the model on the accelerator if overhead matters.

**Pitfall:** a late-bound argument cannot fix a change after commitment.

**Smallest experiment:** build a Moving-Balloon-style page, run your agent at three speeds with and without re-reading the screen just before the click, and compare success.

Assessment: Try in an experiment. A clear diagnosis and a cheap mitigation with large effects on synthetic tasks; real-application validation is missing.

### Connections

Existing timing work includes benchmarks with timing-specific tasks, newer-evidence methods (AOI), future-state prediction (MobileDreamer, TraceR1) and computation preparation or reuse (UFO2, AAPT). The paper argues these do not update where the agent clicks or types when the screen changes. Its contributions are the EODA decomposition, the Paused/Replay/Live design and a controller that revises a pending action before commitment.

**Connections.** Idea 4 (recovery harm) is the API-write counterpart: state changes between decision and effect. Idea 2 (delegation) adds latency (38 versus 25 minutes median on DeepSWE), which would worsen live-interface failures. Idea 1 (evidence closure) is about what information the agent holds; this idea is about how stale it is.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [One Step, Four Clocks: How GUI Agents Fall Out of Step](https://arxiv.org/abs/2611.31522) — Splitting GUI responses into change/observation/commitment/receipt events and holding timing fixed, Replay scored 96% versus 29% Live for a planner-grounder agent, and re-binding the pending click to the newest observation took Moving Balloon from 0% to 100% (Qwen3.8-27B), 0% to 99-100% on three tasks, and 0% to 69% with UI-TARS-1.5-7B.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: created from bulletin 2026-11-26 (#7).
