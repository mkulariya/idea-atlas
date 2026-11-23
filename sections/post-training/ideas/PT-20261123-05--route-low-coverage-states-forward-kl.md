---
id: PT-20261123-05
title: Route low-coverage states to forward KL during on-policy distillation
tldr: Applying top-16 forward KL only at states where the student's mass on the teacher's top-p tokens is low, or teacher mass is underestimated, beats sampled-token on-policy distillation and blanket forward KL.
section: post-training
also_in: []
subtopic: distillation
type: recipe
tags: [on-policy-distillation, kl-divergence, teacher-student, teacher-student-mismatch, training-instability]
aliases: [coverage-gated forward KL, routed forward KL, top-K forward KL routing, OPD loss routing]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 3, actionability: 4, surprise: 3}
power: 76
evidence: B
first_seen: '2026-11-23'
last_updated: '2026-11-23'
sources:
- {arxiv: '2611.27125', title: 'When Students Fail to Learn from Teachers: Learning Dynamics of On-Policy Distillation', role: origin}
related:
- {id: PT-20261123-04, rel: builds-on}
superseded_by: null
---

## Summary

Sampled-token on-policy distillation scores only the token the student samples, so a teacher-preferred token outside the student's support is never learned, and the large misdirected gradient at low-coverage states can destabilise training. This paper keeps that cheap objective but routes each state: if the student's mass on the teacher's top-p set (p=0.9) falls below 0.2, or the teacher's top-p mass sitting on tokens the student underestimates by more than 6x exceeds 0.3, the state gets an unnormalised top-16 forward-KL loss instead. Everything else in the OPD pipeline is unchanged; the only added data is the teacher's top-16 log-probabilities, which raise teacher scoring from 2.6-3.1 s to 4.5-5.9 s per step and total step time by 2.5-6.8%. Against sampled-token OPD, mean math Avg@8 rises by 1.7-2.5 points from Base initialisation and 1.4-2.1 from SFT initialisation for Qwen3-1.7B/4B/8B students, and code Avg@8 by 2.9-3.4 points, with LiveCodeBench gains of 6.0-10.9 points. The routing beats entropy-based (EOPD) and advantage-sign-based (AOPD) routing and applying forward KL everywhere, and it keeps completion rate stable where OPD and EOPD collapse. Use it when an OPD run shows length growth, loops or low teacher-support coverage, preferably after an SFT warm-up on teacher outputs that raises initial coverage. The caveats: all students and the main teacher are Qwen3, benchmarks are small with no seeds reported, and the five thresholds are hand-set.

## Detail

### Problem

Idea 4 explains *why* sampled-token on-policy distillation (OPD) fails: at states where the teacher's preferred tokens sit outside the student's support, the sampled-token gradient cannot teach them, and it corrupts the rest of the update. The obvious repair, a forward-KL term that directly raises teacher tokens, costs more (it needs the teacher's top-$K$ log-probabilities) and, applied everywhere, did not do better in this paper.

Existing adaptive schemes pick where to apply forward KL by proxies that do not measure the problem: EOPD by teacher entropy, AOPD by the sign of the sampled token's advantage. This paper's router measures the student's coverage of the teacher's support directly. For anyone running OPD as a post-training stage, it is a drop-in change to the loss that targets the known failure without paying for full-vocabulary objectives everywhere.

### Mechanism

- **Sampled-token OPD:** student samples a response; each sampled token $y_t$ gets advantage $A_t=\log\pi_T(y_t\mid s_t)-\log\pi_\theta(y_t\mid s_t)$ (reverse-KL estimate), used as in policy gradient.
- **Forward KL (FKL):** $\sum_v\pi_T(v\mid s)\log\frac{\pi_T(v\mid s)}{\pi_\theta(v\mid s)}$; mass-covering, pulls the student toward *every* token the teacher likes, sampled or not.
- **Top-$K$ FKL:** the same sum restricted to the teacher's top $K$ tokens, unnormalised (no renormalisation over the $K$ tokens).
- **Teacher top-$p$ set $\mathcal{T}_p(s)$:** the smallest set of tokens whose teacher probability sums to at least $p$.
- **Avg@8 / Best@8:** mean accuracy over 8 samples per problem / whether any of 8 is correct.

**Two measurements per state** (computed from the teacher's top-$K$ log-probabilities and the student's probabilities, at every generated position):

1. **Coverage** $M_{\text{top-}p}(s)=\sum_{v\in\mathcal{T}_p(s)}\pi_\theta(v\mid s)$: how much student mass lands on the teacher's preferred set ($p=0.9$).
2. **Underestimated mass** $V(\theta;s)$: the teacher's top-$p$ mass on tokens the student underestimates by a large factor, i.e. those with $\pi_T(v\mid s)/\pi_\theta(v\mid s)>\varepsilon_1$, with $\varepsilon_1=6$.

**Routing rule** (eq. 18):

$$r(s_t)=\mathbf{1}\big[V(\theta;s_t)>\varepsilon_2\ \lor\ M_{\text{top-}p}(s_t)<\varepsilon_3\big],\qquad \varepsilon_2=0.3,\ \varepsilon_3=0.2.$$

**Loss.** Routed states ($r=1$) use unnormalised top-$K$ forward KL against the teacher ($K=16$); all other states keep the sampled-token OPD update. Everything else in the OPD pipeline (rollouts, teacher scoring, optimiser) is unchanged; the only extra data is the teacher's top-16 log-probabilities.

**Recommended pairing.** The paper also shows that SFT on teacher outputs before OPD raises initial coverage (fewer states need routing) and reports results for both Base and SFT initialisation. Practically: SFT first if you can, then OPD with routing as a safety net for the states that remain mismatched.

**Settings.** Students Qwen3-1.7B/4B/8B-Base; teacher Qwen3-8B (non-thinking mode from Base init; thinking mode from an SFT init trained on 50K OpenThoughts3 prompts × 8 responses). Math prompts from DAPO; code from CodeR1-12k. verl.

### Why it works

The router turns on forward KL exactly where idea 4's analysis says the sampled-token update is ineffective or harmful: when the student puts little mass on what the teacher wants (low coverage) or underweights a large share of the teacher's mass (high $V$). There, forward KL supplies a gradient for teacher tokens the student would almost never sample, so they can actually be learned, and it replaces the large, misdirected negative advantage that would otherwise suppress the student's own token and spill into other positions.

Everywhere else, the cheaper mode-seeking sampled-token loss keeps its usual advantages (on-policy, focused on what the student does). Applying forward KL everywhere (All-FKL) was not better: it spends teacher targets and gradient on states that were already fine, which plausibly dilutes the signal at the states that matter.

The training-dynamics comparison supports this reading: with routing, completion rate stays stable and coverage of the teacher's top-$p$ support is highest, while using forward KL at the fewest positions among the adaptive methods.

### Evidence

**Baselines.** Sampled-token OPD; EOPD (FKL where teacher entropy is high); AOPD (FKL where the sampled-token advantage is non-positive); All-FKL in the dynamics comparison. Benchmarks: AMC23, AIME24-26 (math); HumanEval+, MBPP+, LiveCodeBench v6 (code). Avg@8 and Best@8.

**Gains over sampled-token OPD (mean over benchmarks, percentage points):**

| Setting | Metric | 1.7B | 4B | 8B |
|---|---|---|---|---|
| Math, Base init | Avg@8 | +2.36 | +2.50 | +1.72 |
| Math, Base init | Best@8 | +2.60 | +4.26 | +6.24 |
| Math, SFT init | Avg@8 | +2.08 | +1.38 | +1.59 |
| Math, SFT init | Best@8 | +3.33 | +2.30 | +1.01 |
| Code | Avg@8 | +2.92 | +3.19 | +3.44 |
| Code, LiveCodeBench | Avg@8 | +6.00 | +10.93 | +8.14 |

From Base initialisation the method has the highest mean Avg@8 and Best@8 at all three scales. Code Best@8 gains are 1.27-2.88 points. The LiveCodeBench gains drive most of the code improvement.

**Dynamics (Fig. 7).** OPD and EOPD show spurious agreement (overlap high, completion collapsing); routing keeps completion stable.

**Cost.** Teacher scoring rises from 2.6-3.1 s to 4.5-5.9 s per step at $K=16$, about 6-10% of step time; total step time +2.5-6.8%; same peak memory.

### Limitations and open questions

- **One model family:** all students and the main teacher are Qwen3; the effect with cross-family teachers, where mismatch is larger, is untested here.
- **Small benchmarks, no seeds:** AIME sets have 30 problems each and no run-to-run variance is reported, so 1-2 point Avg@8 differences are noisy.
- **Hand-set thresholds:** $\varepsilon_1=6$, $\varepsilon_2=0.3$, $\varepsilon_3=0.2$, $p=0.9$, $K=16$; a sensitivity appendix exists (I did not read it in depth).
- **Needs teacher top-$K$ log-probs**, which closed-API teachers may not expose.

My read: the stability benefit (no collapse) is the strongest result; every reported mean gain is positive, but most are small apart from LiveCodeBench.

### How to apply

**When:** you run on-policy distillation and see length growth, loops or low coverage, or you distil across templates/families.

**Steps:**
1. Have the teacher return top-16 log-probs at every student-generated position (most inference servers support this).
2. Per position compute $M_{\text{top-}p}$ with $p=0.9$ and $V$ with $\varepsilon_1=6$.
3. Set $r=1$ if $V>0.3$ or $M<0.2$; apply unnormalised top-16 FKL there, sampled-token OPD elsewhere.
4. Log the routed fraction; it should fall as training proceeds. If it stays high, do an SFT warm-up on teacher outputs.

**Smallest experiment:** take an existing OPD config, add the router, run both for the same steps on 3 seeds; compare completion rate, response length and Avg@8 on your eval set.

Assessment: Try in an experiment. A cheap, principled change to the OPD loss that prevents collapse in the paper's runs; accuracy gains are modest and shown only within the Qwen3 family.

### Connections

Closest prior work: adding forward KL to OPD globally; EOPD (route by teacher entropy); AOPD (route by negative advantage); adding an SFT loss during OPD; trust-region style constraints around the teacher.

New here: the routing *signal* — student coverage of the teacher's top-$p$ set plus the underestimated-mass test — derived from a learning-dynamics analysis rather than a heuristic, and evidence that this beats both entropy-based and advantage-based routing and blanket forward KL at modest cost.

**Connections.** Idea 4 is the diagnosis this method treats; read them together. Idea 3 is the RL analogue: a dense per-step correction added only where the base signal is uninformative. Idea 6 is a reminder that post-training gains measured under one setup (here a Qwen3 teacher family) may not carry over to another.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-23 · origin · [When Students Fail to Learn from Teachers: Learning Dynamics of On-Policy Distillation](https://arxiv.org/abs/2611.27125) — Routes unnormalised top-16 forward KL only where student coverage of the teacher's top-p set is low or underestimated teacher mass is high; mean math Avg@8 rises 1.4-2.5 points and code Avg@8 2.9-3.4 points over sampled-token OPD across Qwen3-1.7B/4B/8B at +2.5-6.8% step time, with completion stable where OPD and EOPD collapse.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-23: created from bulletin 2026-11-23 (#5).
