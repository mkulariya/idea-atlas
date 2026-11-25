---
id: AG-20261125-05
title: Retries raise cost per completion because easy tasks leave the pool first
tldr: Because easy tasks succeed and leave first, each extra same-task retry draws from a harder task pool, so cost per completion rises above one-shot cost.
section: agents
also_in: []
subtopic: agent-evaluation
type: finding
tags: [agent-evaluation, best-of-n, verifier]
aliases: [retry tax, survivor selection in same-task retries, marginal cost per added completion]
status: emerging
scores: {novelty: 3, leverage: 3, evidence: 4, generality: 4, actionability: 4, surprise: 3}
power: 69
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.29133', title: 'The Retry Tax: Why Static Same-Task Retries Can Cost More per Completion', role: origin}
related: []
superseded_by: null
---

## Summary

Agent services that reset and rerun a failed task pay for every failed attempt, yet pass rate and average cost per attempt hide what the last attempt buys. Under stationary, conditionally independent per-task success probabilities and a constant cost per attempt, easy tasks succeed early and exit, so the pool still failing is enriched in hard tasks; cost per completion therefore exceeds one-shot cost, with retry tax R(K) = 1 - Cov(P, G_K(P))/Q(K) at least 1, with equality only if all tasks share one success probability. The added completions are priced by a marginal cost c/mu_r, where mu_r is the mean success probability among tasks reaching attempt r, and it is nondecreasing wherever completion gains are positive; an attempt is worth allowing only while that marginal cost stays below the value of a completion. Replaying recorded repeated runs at cap four versus one, cost per completion was about 77% higher (median over 12 SWE-bench Verified configurations) and about 24% higher on τ²-bench, while on OpenHands reported in USD completion rose from 59.7% to 70.0% and cost per completion from about $0.70 to $1.32. Replacing recorded costs with task means changed the cap-four ratio by under one percent, so outcomes rather than cost variation drive the effect. Use it to set attempt caps, best-of-N budgets and retry policies for agents with a verifier, starting from logs of at least four repeated runs per task. The main caveat is the stationarity and independence assumption: real retries share state, which would push marginal cost higher still, and the three panels share a benchmark frame and report no intervals.

## Detail

### Problem

A coding-agent service often retries after a failed patch: reset the workspace, rerun, stop at verified success or an attempt cap. More attempts do complete more tasks, but the service pays for every failed attempt too. Looking only at pass rate or at average cost per attempt hides what the last attempt buys.

For a service with a fixed price per attempt, this decides the retry cap. The paper gives the exact accounting and shows on public benchmark records that the price of the extra completions can be far higher than the average suggests.

### Mechanism

**Static same-task retry.** After a failure, restore the original prompt and workspace, give the agent no feedback from the earlier attempt, and stop at verified success or a cap $K$ (a cap of four allows at most four attempts total).

**Heterogeneous tasks.** Each task $i$ has its own success probability $P$ for a fixed agent and verifier; "easy" and "hard" mean high and low $P$.

**Cost per completion.** Total stopped spend divided by the share of tasks completed.

**Survivor selection.** Easy tasks succeed early and exit, so the pool still failing is enriched in hard tasks (the same effect as in survival analysis of mixed populations).

Assume stationary success probabilities, independent attempts given the task, and a constant cost $c$ per attempt. Let $Q(K)$ be the share of tasks completed within $K$ attempts, $B(K)$ the expected spend per incoming task, $E(K)=B(K)/Q(K)$ the cost per completion and $R(K)=E(K)/E(1)$ the **retry tax** relative to one-shot serving. Then, with $G_K(P)$ the expected number of attempts a task of success probability $P$ uses,

$$R(K)=1-\frac{\operatorname{Cov}(P,G_K(P))}{Q(K)}\ \ge 1,$$

with equality for $K\ge2$ only if every task has the same $P$. At two attempts, $R(2)=1+\operatorname{Var}(P)/Q(2)$, so the excess is set by between-task variance.

For the price of the added completions, define marginal cost $M_r=\Delta B_r/\Delta Q_r=c/\mu_r$, where $\mu_r$ is the mean success probability among tasks reaching attempt $r$. Each failure lowers it: $\mu_{r+1}=\mu_r-\operatorname{Var}_r(P)/(1-\mu_r)$, so $M_r$ is nondecreasing wherever completion gains are positive. If each completion is worth $V$, allow attempt $r$ only while $V\ge M_r$.

Worked example: half the tasks at $P=0.9$ and half at $0.1$. Two attempts complete 0.59 of tasks at $1.5c$ spend, so cost per completion is $1.5c/0.59\approx2.5c$, versus $2c$ at one attempt. A homogeneous population with the same one-shot rate ($P=0.5$) completes 0.75 at $1.5c$, which stays at $2c$.

Empirically, the paper replays recorded repeated runs to compute $Q$, $B$ and $M_r$ at every cap.

### Why it works

It is selection, not agent degradation: the agent and price per attempt are fixed. An attempt costs the same for an easy task that will succeed and a hard task that will not, but later attempts are spent on a pool that contains mostly hard tasks. The same-mean homogeneous control shows that adding completions alone does not raise cost per completion; heterogeneity does.

My read: this is a classical result (the authors cite survival-analysis work on heterogeneous populations) applied correctly to agent services, and its value is that the formulas are exact and need only per-task outcome counts.

### Evidence

The authors reanalyze public records; no new model calls were made.

- **Panels at cap four versus one.** Primary panel: 12 fixed configurations, ten attempts on 500 SWE-bench Verified tasks, about 77% higher cost per completion (median over configurations, unit cost per attempt). $\tau^2$-bench: four configurations, 164 airline and telecom tasks, about 24% higher. OpenHands (CodeActAgent v0.62.0 with gpt-5, one configuration, 500 tasks, recorded USD): about 90% higher.
- **OpenHands detail (Figure 2).** Completion rose from 59.7% to 70.0% across caps one to four (66.1% at two, 68.5% at three), while mean spend per incoming task rose from $0.4152 to $0.9266 ($0.6098 at two, $0.7745 at three). My arithmetic from those rounded figures: cost per completion goes from about $0.70 to $1.32, and the fourth attempt costs on the order of $10 per added completion (the third about $7, the second about $3).
- **Realized versus task-mean costs.** Replacing recorded costs with task means changed $R_4$ by less than one percent, so the effect is driven by outcomes, not cost variation.

### Limitations and open questions

Authors' limits: the model assumes stationary success, conditional independence between attempts and a common cost; the empirical estimates describe resampled orders of available runs, not variation across agents or unseen repositories; the three panels share a benchmark frame and are not independent populations; point estimates carry no intervals in the main figures; feedback-conditioned retries and delayed verification are outside scope.

My read: real retries often share state (same prompt, same bug), so independence may overstate completion gains from further attempts; the true marginal cost could be even higher. The result also applies only when a verifier is available at retry time.

### How to apply

Reach for this when you set attempt caps, best-of-N budgets or retry policies for an agent service with a verifier (tests, a grader).

1. Log per-task outcomes and cost for at least four repeated runs per task.
2. Compute $Q(K)$, $B(K)$ and $M_r$ for each cap using the paper's order-averaging replay, or simply by simulation over resampled attempt orders.
3. Pick the largest cap with $M_r\le V$, where $V$ is the value of one completion in the same currency.
4. Consider retrying selectively, since uniform retry wastes spend on hard tasks; the paper notes this needs task-level information and an allocation policy.

**Smallest experiment:** take an existing eval with at least four runs per task and compute $M_2$, $M_3$, $M_4$. If $M_4$ is several times $M_1$, your cap is too high or your retries need feedback.

Assessment: Adopt now. The accounting is exact, needs only logs you probably have, and directly sets a retry cap.

### Connections

Survivor selection in heterogeneous populations is classical (Vaupel and Yashin, 1985; Proschan, 1963, as cited). The paper specializes it to stop-on-success agent retries and gives a covariance identity, a cap-two variance identity, a monotone marginal-cost result and a rule for the best common cap, then computes them from public repeated-run data. It separates average cost per completion from the cost of the completions an extra attempt adds, which are often conflated.

Connections: it is a cost lens on idea 1, whose agents mostly did not retry and fabricated more; retries can be protective, so the economics here must be weighed against reliability. It also relates to idea 3: both reason about what extra samples buy under a sparse terminal reward.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [The Retry Tax: Why Static Same-Task Retries Can Cost More per Completion](https://arxiv.org/abs/2611.29133) — Replaying recorded repeated runs, cost per completion at cap four was about 77% higher than one-shot over 12 SWE-bench Verified configurations (about 24% on τ²-bench, about 90% on OpenHands), with OpenHands completion rising from 59.7% to 70.0%.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: created from bulletin 2026-11-25 (#5).
