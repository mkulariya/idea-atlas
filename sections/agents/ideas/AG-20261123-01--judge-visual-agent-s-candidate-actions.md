---
id: AG-20261123-01
title: Judge a visual agent's candidate actions by drawing them on the screenshot
tldr: Sample several actions from a frozen visual policy, draw each on its own screenshot copy, score them with a probability-returning judge, and override greedy only on a clear lead; no training needed.
section: agents
also_in: []
subtopic: architectures-scaffolds
type: method
tags: [gui-agents, test-time-action-selection, best-of-n, verifier, training-free, action-grounding]
aliases: [Bel, mark-ask-act]
status: emerging
scores: {actionability: 5, evidence: 4, generality: 4, leverage: 4, novelty: 4, surprise: 3}
power: 81
evidence: B
first_seen: '2026-11-23'
last_updated: '2026-11-23'
sources:
- {arxiv: '2611.27222', title: 'Bel: Step-Level Policy Improvement of Visual Agents Without Training', role: origin}
related:
- {id: PT-20261123-03, rel: alternative-to}
superseded_by: null
---

## Summary

A visual agent fails a whole task when one step is wrong, and the right action is often already among the policy's own samples while greedy decoding passes it over. Bel samples greedy plus eight temperature-1.0 actions, dedupes them on a 100x100 grid to at most nine distinct candidates, draws each pointed action as a translucent red circle on its own copy of the screenshot, and asks a decision model for the probability that each candidate is correct. It executes the highest-scoring candidate only when greedy's score falls below a threshold and a rival leads by a margin, otherwise it keeps greedy; the two thresholds are tuned on one development set and frozen everywhere else. On six offline GUI step-level sets, GUI-R1-3B rises from 60.7 to 65.4 percent step success and Qwen2.5-VL-3B from 46.7 to 58.8, with paired 95% intervals excluding zero and beating majority vote, region vote, CATTS, a trained GUI-Critic-R1 and prompted Qwen3-VL judges. Grounding also improves (ScreenSpot 81.7 to 88.1 for GUI-R1-3B), and live gains over majority vote hold on AndroidWorld and MiniWoB++ with thresholds tuned for a different policy. Ablations show that giving the judge the action as text alone, or as coordinates on a raw screenshot, recovers significantly less than drawing the mark. Use it when actions have a location in an image and you can sample several per step; the main caveat is that all results use one proprietary decision model, and each step costs eight extra samples plus roughly 4.7 judge calls.

## Detail

### Problem

A visual agent (a model that taps, types and scrolls on a screen, or points at an image to tell a robot where to act) fails a whole task when one step is wrong. The right action is often already among the model's own samples; greedy decoding (the single most likely output) passes it over.

Majority voting mostly repeats greedy's mistake. Text judges and trained critics read a tap as two numbers, such as `click(512, 880)`, which says nothing about whether that point lands on "Post" or on an empty margin.

For GUI agents this is cheap headroom: no training, just a better choice among outputs you can already sample.

### Mechanism

- **Step success:** the share of recorded benchmark steps where the agent's action matches the ground truth (here, a click within 14% of the screen of the true point, or a typed/scrolled argument with token F1 of at least 0.5, following GUI-R1's protocol).
- **Grounding accuracy:** whether a click lands inside the target element (ScreenSpot, ScreenSpot-Pro).
- **Decision model:** a model that, instead of free text, returns a probability for each answer of a closed question. The paper uses Jev (TypeSafe) via an API, so every candidate gets an absolute score on one scale.
- **Paired bootstrap interval:** a 95% interval for the difference between two methods on the same items; "significant" means it excludes zero.

Bel runs three moves at every step ("mark, ask, act"):

1. **Sample.** From the frozen policy take the greedy action $a^{(0)}$ (temperature 0) and $N=8$ samples at temperature 1.0 from the same prompt. Merge same-type candidates in the same cell of a $100\times100$ grid, leaving $K \le 9$ distinct candidates (4.7 per step on average).
2. **Mark.** Draw each candidate with a point on its *own* copy of the screenshot as a translucent red circle with radius about 2.5% of the image's longer side, and also write the action as text. Actions without a point are given as text on the unmarked screenshot.
3. **Ask.** Send (instruction, history, marked image, action) to the decision model and read $p_k = p(\text{correct} \mid x, h_t, \tilde{s}_k, a^{(k)})$, the probability that executing candidate $k$ is the right decision.
4. **Act.** Let $k^\star = \arg\max_k p_k$. Execute

$$\pi'(x,h_t,s_t)=\begin{cases}a^{(k^\star)} & \text{if } p_0<\tau \text{ and } p_{k^\star}-p_0\ge m\\ a^{(0)} & \text{otherwise}\end{cases}$$

where $p_0$ is greedy's score, $\tau$ is the score below which greedy may be replaced and $m$ is the lead another candidate needs.

The only per-policy settings are $\tau$ and $m$, chosen on one development set (AndroidControl-High) and then frozen for every other test set, the live runs and robot pointing. For GUI-R1-3B the live runs use $\tau=0.8$, $m=0$.

### Why it works

The authors' explanation, backed by an ablation, is that the information needed to judge a pointing action is *under the point*. When the decision model sees the action only as text, it regains only a little of the gain; given the screenshot plus raw coordinates it recovers more; only drawing the action where it lands gives the full gain, significantly above both reduced inputs. The judge does not have to perform coordinate-to-pixel grounding itself, which is exactly the skill a small GUI policy lacks.

The override rule protects steps where greedy is already right: Bel changes the action only when greedy looks doubtful and a rival leads, which the authors report keeps the gain of always taking the top score while changing far fewer steps.

My read: a single probability scale also lets a tap, a typed string and a "task complete" stop be compared directly (a common repaired mistake is a premature stop), and Bel's gain keeps rising as samples double up to eight while majority voting gains little.

### Evidence

**Setup.** Policies GUI-R1-3B/7B, Qwen2.5-VL-3B-Instruct and Qwen3-VL-235B-A22B; six offline GUI step-level sets, ScreenSpot(-Pro), Multimodal-Mind2Web, live AndroidWorld (116 tasks) and MiniWoB++ (92 tasks), and seven robot-pointing sets.

**Main results (Table 1, step success mean over six GUI sets / ScreenSpot / ScreenSpot-Pro, %):**

| Policy | Greedy | Best other selector (step) | Bel |
|---|---|---|---|
| GUI-R1-3B | 60.7 / 81.7 / 28.5 | 62.8 (Qwen3-VL-8B judge) | 65.4 / 88.1 / 33.7 |
| GUI-R1-7B | 65.4 / 85.8 / 31.1 | 66.8 (Qwen3-VL-8B judge) | 68.2 / 89.2 / 33.8 |
| Qwen2.5-VL-3B | 46.7 / 64.3 / 13.8 | 54.3 (Qwen3-VL-8B judge) | 58.8 / 79.4 / 17.7 |

Every gain over greedy is significant (paired 95% interval excludes zero), and Bel beats majority vote, region vote, CATTS, the trained GUI-Critic-R1 and two prompted Qwen3-VL judges for every policy. On Multimodal-Mind2Web (Table 2) with thresholds unchanged, GUI-R1-3B goes from 38.0 to 50.8, above the fine-tuned ICM-r2 critic at 45.3.

**Live.** Bel's lead over majority vote has 95% intervals of [+6.6, +13.5] points on AndroidWorld and [+5.4, +15.6] on MiniWoB++ for Qwen2.5-VL-7B, and [+2.9, +11.5] and [+0.7, +10.1] for Qwen3-VL-8B, using thresholds tuned offline for a different policy.

**Ablations.** Text-only and coordinates-only inputs are significantly worse than the drawn mark; every threshold pair on the grid beats greedy for each policy.

### Limitations and open questions

- **One scorer.** All results use a single closed decision model (Jev 1.14). Prompted open Qwen3-VL judges given the same marks are significantly worse than Bel.
- **Cost and latency.** One judge call per distinct candidate (4.7 per step on average) plus 8 extra policy samples; step latency rises.
- **Robotics scope.** Robot results are pointing only (choosing a pixel), not closed-loop control.
- **Live protocol.** The authors disclose that a third live seed was added after the first two were analysed.

My read: the core lesson (show the judge the consequence of an action in the observation space) is likely to transfer beyond this scorer. The size of the gain probably depends on how weak the policy's grounding is; the largest lift is on the general-purpose Qwen2.5-VL-3B (+12.1 points), the smallest on GUI-R1-7B (+2.8).

### How to apply

**When:** any agent whose actions have a location in an image (GUI taps, drag targets, robot points) and where you can sample several actions per step.

**Steps:**
1. Sample greedy plus 8 actions at temperature 1.0; dedupe on a 1% grid.
2. Render each pointed action as a translucent circle (about 2.5% of the long side) on its own screenshot copy, and include the action text.
3. Score each with a judge that returns a probability (a decision model, or a VLM's probability of "Yes").
4. Override greedy only if $p_0<\tau$ and the best candidate leads by $m$; tune $\tau,m$ on one dev set and freeze them.

**Pitfalls:** do not score raw screenshots with coordinates, and do not always take the argmax.

**Smallest experiment:** take 500 recorded steps from your own GUI logs with ground truth, run greedy plus 8 samples, and compare step success for (a) greedy, (b) majority vote, (c) your judge with coordinates as text, (d) the same judge with drawn marks.

Assessment: Try in an experiment. Training-free, simple to replicate, and backed by significant gains across four policies and live runs, but shown with one proprietary scorer.

### Connections

Closest prior work: trained GUI critics that judge an action before execution (GUI-Critic-R1, ICM-r2), GUI-Actor's click verifier, voting and region voting over samples (GUI-RC), and CATTS, which falls back to the policy when the vote is contested.

The precise difference is the input representation, not the scorer: render each candidate into the observation and score it absolutely, with a conservative override of greedy. Nothing is trained, and one setting transfers across screens, live environments and robot pointing.

**Connections.** Same group and same decision model as idea 3, which uses per-step probabilities as training credit rather than test-time selection; the two together suggest one scorer can serve both acting and learning. Idea 2 is a reminder that whatever a selector makes the agent click is observable to the sites it visits.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-23 · origin · [Bel: Step-Level Policy Improvement of Visual Agents Without Training](https://arxiv.org/abs/2611.27222) — Drawing each sampled action on its own screenshot for a probability judge raises step success for every policy tested (GUI-R1-3B 60.7 to 65.4, Qwen2.5-VL-3B 46.7 to 58.8 on six GUI sets), beats voting and trained critics, and transfers to live runs and robot pointing with frozen thresholds.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-23: +relation alternative-to PT-20261123-03.
- 2026-11-23: created from bulletin 2026-11-23 (#1).
