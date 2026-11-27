---
id: AG-20261127-05
title: Evolving what a guardrail says against a message-reading attacker cuts adaptive attack success
tldr: Guardrail rejection messages that help benign users recover also inform adaptive attackers; evolving decision and disclosure skills against a message-reading attacker cuts adaptive attack success from 78.4% to 14.1% at over 85% utility.
section: agents
also_in: []
subtopic: agent-safety-security
type: method
tags: [agent-safety, prompt-injection, information-flow, skills, agent-evaluation]
aliases: [FORK, FORK-World, safe disclosure, co-evolving safe disclosure and decision skills]
status: emerging
scores: {actionability: 4, evidence: 4, generality: 4, leverage: 4, novelty: 4, surprise: 4}
power: 80
evidence: B
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.33879', title: 'FORK: Co-Evolving Safe Disclosure and Decision Skills Against Attackers Who Read Every Refusal', role: origin}
related: []
superseded_by: null
---

## Summary

Agent guardrails gate every tool call, and their rejection messages enter the trajectory where an attacker who writes to the agent's inputs can read them: silent or fixed messages leak little but strand legitimate users, while explanations let an agent repair its call and also tell an adaptive attacker which boundary was hit. FORK learns two kinds of natural-language skills for a guardrail LLM — decision skills (may this call proceed?) and disclosure skills (what the public message says after a rejection) — without gradient updates. Each training episode pairs a harmful objective with a benign twin in the same world so one episode scores both breach and recovery; the attacker makes sequential attempts, reads every public message and evolves its own skill library, a programmatic verifier over environment state decides what took effect, and maximin opponent selection over past attacker checkpoints keeps new patches from reopening older holes. Averaged over three agents on FORK-World, adaptive attack success rate falls from 78.4% to 14.1% with utility above 85%, and on unseen MT-AgentRisk environments and tasks from 70.6% to 9.1% with 93.1% benign utility. A non-learned explanatory message makes things worse: swapping the silent seed for an explanatory one raises ASR from 30.3% to 39.3% on FORK-World, while full FORK lowers ASR by 8.2 points versus the explanatory seed with 8.3 points more utility. Static attacks understate the risk — two frontier agents show 0.1-0.2% ASR on AgentDojo without a guardrail but 66.3-73.9% on FORK-World. At deployment FORK costs one extra LLM call per proposed action and the learned library is plain text that runs on a different guardrail LLM without retraining; attackers are LLM agents with limited strategies, malicious tool metadata and compromised tools are out of scope, the library comes from 65 training pairs, and FORK-World is the authors' own testbed.

## Detail

### Problem

Agent guardrails sit at the action boundary and gate each tool call. When they reject a call, the message enters the trajectory, and any attacker who can write to the agent's inputs can read it. A silent or fixed message leaks little but leaves a legitimate user stuck. An explanation lets the agent repair its call, but it can also reveal which boundary was hit, so the attacker reroutes instead of guessing.

Most guardrail evaluation uses fixed attacks that never see the rejection, so this leak is invisible. Teams that add helpful rejection messages may be making adaptive attacks easier.

### Mechanism

**Prompt injection.** Text in a tool output or user turn that tries to redirect the agent. **ASR** is attack success rate; **utility** (UA for under attack, BU for benign) is the fraction of authorized tasks completed. **Safe disclosure** is the paper's term for explaining rejections so authorized tasks recover without raising adaptive ASR.

**Skill library.** Here, natural-language instructions the guardrail LLM follows, revised from evidence and not by gradient updates. **Maximin selection** picks the opponent (earlier attacker checkpoint) against which the current guardrail does worst.

FORK learns two kinds of skills for a guardrail: decision skills (may this call proceed?) and disclosure skills (what should the public message say after a rejection?).

1. **Paired episodes.** Each training episode pairs a harmful objective with a benign twin in the same world, so one episode scores both branches (breach vs recovery) and surface features cannot separate them.
2. **Attacker reads everything.** The attacker makes sequential attempts, reads every public message, and evolves its own skill library.
3. **Programmatic verification.** A verifier over environment state decides what actually took effect.
4. **Update with asymmetric views.** A skill manager revises only skill families the evidence supports. The attacker's updates see only what a real attacker could observe; guardrail updates see the full record, including whether a message was later exploited.
5. **Maximin opponent selection** over past checkpoints, so patches for the newest attacker do not reopen older holes.

At deployment FORK makes one LLM call per proposed tool call or reply, and the learned library is plain text that runs on a different guardrail LLM without retraining. The example in Table 1 contrasts three messages for one rejected call: silent ("can't be completed"), rule-level (names a forgeable condition), and one grounded in the authenticated request.

### Why it works

The authors' observation is that the public message that creates the risk also supplies the learning signal: an attack that succeeds after a rejection shows an over-permissive decision or over-revealing message; a benign task that completes after a rejection marks a useful message. Pairing and the adaptive attacker make both observable. A message anchored in what the authenticated user asked gives the agent a repair but no condition the attacker can meet by merely making a claim. My read: the mechanism is a form of adversarial training in text space, and the maximin step addresses forgetting.

### Evidence

**Setup.** Four benchmarks: AgentDojo and MT-AgentRisk (fixed attacks), STAC (hybrid), and FORK-World, a fully adaptive testbed ported from 677 published scenarios. One library distilled from 65 training pairs. Co-evolution agent Qwen3-Next-80B, guardrail Qwen3-32B; evaluation also on other agents and guardrail LLMs (Sonnet 4.5 among them).

**Main results.** Averaged over three agents on FORK-World, ASR under adaptive attack falls from 78.4% to 14.1% with utility above 85%. On MT-AgentRisk, whose environments and tasks are unseen in training, ASR falls from 70.6% to 9.1% with 93.1% benign utility.

**Static attacks understate risk.** Two frontier agents show 0.1-0.2% ASR on AgentDojo without a guardrail but 66.3-73.9% on FORK-World. TS-Guard lowers Qwen3-Next-80B's ASR to 5.3% on AgentDojo but only to 54.6% on FORK-World.

**Ablations (Table 3).** Without learned skills, explanation barely helps utility but helps the attacker: Silent seed to Explanatory seed raises ASR from 30.3% to 39.3% on FORK-World (1.2% to 2.7% on AgentDojo). Full FORK lowers ASR by 8.2 points on FORK-World versus the Explanatory seed while raising UA by 8.3 points; relative to the Silent seed it stays within 0.8 ASR points and improves UA by 9.8 and BU by 7.0.

**ASR hides utility costs.** Qwen3-32B as guardrail gives the lowest ASR on some benchmarks but the lowest benign utility (87.8% vs 92.7-93.1%) on MT-AgentRisk.

### Limitations and open questions

Attackers write to the user turn and tool observations; malicious tool metadata and compromised tools are out of scope. The verified-user setting assumes the platform protects the authenticated request path and that operator profiles are accurate; without these, rogue-user results show lower ASR but may reject more. The library comes from 65 training pairs, and the attacker is itself an LLM with limited strategies. FORK-World is the authors' own testbed. The authors note that some baselines look good on ASR by being strict, which hides utility costs.

My read: transfer to MT-AgentRisk is encouraging but real adaptive attackers (humans, white-box optimization) are untested.

### How to apply

Use it if your agent runs with a guardrail whose rejection text reaches the model. 

1. Audit what the rejection message reveals. Replace rule-level explanations ("needs two records") with messages grounded in the authenticated request.
2. Build paired episodes: an attack and a benign twin in the same environment, with a state-based verifier.
3. Run an adaptive attacker that reads the messages; do not rely only on fixed injection suites.
4. Evolve decision and disclosure instructions from exploited messages and refused benign tasks; keep older attacker checkpoints for regression.

Cost: one extra LLM call per proposed action; co-evolution compute was not checked. **Smallest experiment:** take your current guardrail, run a small adaptive attacker that sees rejection messages, and compare ASR with a silent message versus your current explanatory one.

Assessment: Try in an experiment. Treating rejection messages as an attack surface is a useful frame and the numbers are large, but the testbed is partly the authors' own.

### Connections

Silent guardrails (fixed refusals) and explanatory guardrails (rejection-specific feedback so agents can revise) are both cited by the authors as prior approaches, and they treat feedback as if only the protected agent reads it. FORK's novelty is treating the rejection message itself as an attack surface and learning the message and the decision together against an adaptive attacker, using evolving text skills instead of weights. The paper also contributes FORK-World as an adaptive testbed.

**Connections.** Idea 3 (PROVE) is the training counterpart for agent-to-agent verification; both target agents that exploit feedback. Idea 6 (TransHarness) and idea 8 (EBU-Evolve) also evolve text artifacts from success-failure comparisons, though FORK adds an adversary. Idea 4 shows that harness choices change behavior; here the message design is that choice.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [FORK: Co-Evolving Safe Disclosure and Decision Skills Against Attackers Who Read Every Refusal](https://arxiv.org/abs/2611.33879) — Co-evolving guardrail decision and disclosure skills cut adaptive ASR from 78.4% to 14.1% at above 85% utility over three agents on FORK-World and from 70.6% to 9.1% on MT-AgentRisk, while a non-learned explanatory rejection message raised ASR from 30.3% to 39.3%.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: created from bulletin 2026-11-27 (#5).
