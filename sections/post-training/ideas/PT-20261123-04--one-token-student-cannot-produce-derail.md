---
id: PT-20261123-04
title: One token the student cannot produce can derail on-policy distillation
tldr: In sampled-token on-policy distillation, a teacher-preferred token the student almost never samples (like a mismatched EOS) is never learned yet still distorts updates, lengthening and looping outputs while overlap stays high.
section: post-training
also_in: []
subtopic: distillation
type: failure-mode
tags: [on-policy-distillation, kl-divergence, teacher-student, training-instability, chat-template-mismatch, teacher-student-mismatch]
aliases: [spurious agreement, OPD length collapse, EOS mismatch collapse, sampled-token OPD failure]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 3, actionability: 5, surprise: 4}
power: 79
evidence: B
first_seen: '2026-11-23'
last_updated: '2026-11-23'
sources:
- {arxiv: '2611.27125', title: 'When Students Fail to Learn from Teachers: Learning Dynamics of On-Policy Distillation', role: origin}
related:
- {id: PT-20261123-05, rel: extended-by}
superseded_by: null
---

## Summary

On-policy distillation (OPD) is a standard, cheap post-training stage, but runs can blow up: responses grow, outputs loop and benchmark scores fall even with a strong teacher, while teacher-student overlap — the usual health metric — stays high, a pattern the paper calls spurious agreement. In the sampled-token version only the sampled token is scored, with advantage log pi_T - log pi_theta, so a teacher-preferred token with near-zero student probability moves by roughly eta times advantage times its own probability and barely moves at all, even when the advantage is huge; and since the student rarely samples it, it is rarely updated. Because logits share parameters, the large-advantage mismatched position leaks into other positions: across teacher-student pairs the sign of the advantage agrees with the realized change only about 50% of the time, and in one inspected batch the EOS position (advantage -22.53) carried 31.81% of the total cross-position influence. With Qwen3-1.7B-Base learning from Qwen3-8B, the student's own EOS is pushed down while the teacher's EOS is never learned, so on math mean response length grows from 1.2k to 4.1k tokens and the share of rollouts hitting the 4,096-token cap rises from 14% to 100%. On-policy sampling then locks the failure in: the student visits looping prefixes where the teacher itself starts repeating, so the teacher stops offering useful correction while overlap remains high. Aligning the two termination tokens, or masking the gradient at termination positions, removes the collapse, which establishes causality. Use it before and during any OPD run with base-model students, cross-family teachers or different chat templates: measure top-p coverage (p=0.9) beforehand, and monitor completion rate, response length and n-gram loop rate during training rather than overlap alone. The main caveat is scope: most mechanism evidence comes from Qwen-family students and teachers on math and code, two striking numbers rest on one toy model and one inspected batch, and no seeds or variance are reported.

## Detail

### Problem

On-policy distillation (OPD) is now a standard post-training stage (the paper cites its use in Qwen3, MiMo and DeepSeek-V4): the student generates, and the teacher grades each generated token. It is cheaper and denser than RL. But practitioners report OPD runs that blow up: responses grow, outputs loop, and benchmark scores fall, even with a strong teacher.

The usual health metric hides it. The paper names the pattern **spurious agreement**: the student's probability mass on the teacher's preferred tokens at visited states stays high, so the run looks aligned, while completion rate collapses and generations repeat. If you are distilling across model families or chat templates, a single mismatched special token can be enough to trigger it.

### Mechanism

- **On-policy distillation (OPD):** the student $\pi_\theta$ samples a response $y$; at each prefix $s_t$ the loss pulls the student toward the teacher $\pi_T$, usually via reverse KL $D_{\text{KL}}(\pi_\theta\|\pi_T)$ summed over positions.
- **Sampled-token OPD:** the cheap Monte-Carlo version: only the sampled token is scored, with per-token advantage $A_t=\log\pi_T(y_t\mid s_t)-\log\pi_\theta(y_t\mid s_t)$, and the update is $\sum_t A_t\nabla\log\pi_\theta(y_t\mid s_t)$.
- **Forward vs reverse KL:** forward KL directly raises tokens the teacher likes; reverse KL only reacts to tokens the student samples.
- **Top-$p$ coverage $M_{\text{top-}p}$:** the student's total probability on the smallest set of teacher tokens whose teacher mass reaches $p$ (here $p=0.9$). Low coverage means the student rarely samples what the teacher wants.
- **EOS token:** the token that ends a response; Qwen3 base models use `<|endoftext|>`, the chat teacher uses `<|im_end|>`.

The paper traces the failure in three steps, from a single softmax to full training.

**1. A token with near-zero student probability barely moves.** For one state with softmax logits and one sampled token $u$ (probability $p$), a small SGD step changes the sampled token's probability by

$$\Delta\pi(u\mid s)=\eta A(s,u)\,p\big[(1-p)^2+S_{-u}(s)\big]+O(\eta^2),\qquad |A|p(1-p)^2\le\lim_{\eta\to0}\frac{|\Delta\pi|}{\eta}\le 2|A|p(1-p)^2,$$

where $S_{-u}=\sum_{b\ne u}\pi(b\mid s)^2$. The factor $p$ means a teacher-preferred token with tiny student probability changes very little even when its advantage is huge; and because it is rarely sampled, it is rarely updated at all. Non-sampled tokens move by an amount set by their own probability and their position relative to $\tau(u;s)=\|\pi\|_2^2-\pi(u)$, so they can move *against* the teacher's preference.

**2. Mismatched positions dominate other positions' updates.** In a real LLM, logits share parameters. With $h_i=\nabla_\theta\log\pi_\theta(u_i\mid s_i)$, one step changes position $i$ by $\eta A_i\|h_i\|^2+\sum_{j\ne i}\eta A_j\langle h_i,h_j\rangle$. Empirically, the sign of the advantage agrees with the actual change only about 50% of the time across teacher-student pairs. In one batch (Qwen3-1.7B-Base student), the EOS position had advantage $-22.53$ and carried 31.81% of the total cross-position influence.

**3. On-policy sampling locks in the failure.** In a toy categorical model, repeated sampled-token updates at a mismatched state suppress the student's existing mode without ever raising the teacher's token. With Qwen3-1.7B-Base learning from Qwen3-8B, the student's own EOS is pushed down while the teacher's EOS is never learned, so responses lengthen and completion rate drops. The student then visits looping prefixes where the *teacher itself* starts repeating (teacher loop rate rises when continuing student prefixes near the first repeat), so the teacher stops giving useful correction there, and overlap stays high: spurious agreement.

The authors verify causality: aligning the two termination tokens, or masking the gradient at termination tokens, removes the collapse (Appendix E.4).

### Why it works

The mechanism is a mismatch between where the gradient signal is large and where learning can happen. Reverse-KL sampled-token updates can only reward or punish what the student samples. At a state where the teacher puts almost all its mass on a token outside the student's support, the student samples its own token, receives a large negative advantage, and pushes that token down; the freed mass goes to whatever else the student already likes, not to the teacher's token. Because that position's gradient is large, it also leaks into other positions through shared weights. And because generation is on-policy, the resulting drift changes which states get trained on next, steering into regions (loops) where the teacher is also unreliable.

The authors stress that not every mismatch causes this. It appears when the teacher places almost all of its probability on a token the student cannot produce *and* that token is essential to the response, such as a format or termination token.

My read: this is the distillation analogue of reward hacking through an out-of-distribution action, and it explains why "SFT first, then OPD" works: SFT puts the teacher's special tokens and style inside the student's support before on-policy training starts.

### Evidence

**Setup.** Students Qwen3-1.7B/4B/8B-Base and R1-Distill-Qwen-1.5B; teachers Qwen3-8B, JustRL-1.5B and DeepSeek-Distill-Qwen-7B; math and code.

**Diagnostic evidence:**
- Forward-KL deterioration at fixed held-out prefixes concentrates at low top-$p$ coverage (Fig. 2c).
- With the same teacher (JustRL-1.5B), the R1-Distill-1.5B student keeps high coverage and learns well; the Qwen3-1.7B-Base student has much lower coverage and its forward KL *increases* at high-mismatch, low-entropy states (Fig. 3).
- Fixing the student and changing teachers: higher coverage leads to rising coverage during training and large AMC23 gains; low coverage leads to little improvement (Fig. 4).
- Sign agreement between advantage and realized change ≈ 50% in full training (Fig. 5a); EOS weight 31.81% of cross-position influence in the inspected batch (Fig. 5d).
- Aligning or masking EOS removes length collapse (App. E.4).

**Size of the collapse:** the failure also appears with a top-$K$ ($K=16$) reverse-KL objective (Qwen3-1.7B-Base, Qwen3-8B teacher, Fig. 11): on math, mean response length grows from 1.2k to 4.1k tokens over 200 steps and the share of rollouts hitting the 4,096-token cap rises from 14% to 100%; on code, length grows from 0.8k to 4.0k tokens. Validation loop rate peaks at 81% (math) and 52% (code).

**Consequence for methods (Fig. 7):** sampled-token OPD and EOPD (forward KL triggered by high teacher entropy) both show spurious agreement: top-16 overlap and teacher-support coverage stay high while completion rate collapses. Idea 5's coverage-triggered routing keeps completion stable.

### Limitations and open questions

- Most mechanism evidence comes from Qwen-family students and teachers and math/code tasks; one toy categorical model and one inspected batch underlie two of the striking numbers.
- The first-order analysis ignores optimizer effects (Adam, clipping) beyond the shared-parameter term.
- No seeds or variance are reported for training curves.

My read: the EOS case is a clear, reproducible failure that many teams will hit when distilling chat teachers into base students. The broader claim that low top-$p$ coverage predicts where OPD fails is well motivated but would benefit from tests beyond Qwen-family pairs.

### How to apply

**When:** before and during any OPD run, especially base-model students, cross-family teachers, or teachers with different chat templates.

**Checks before training:**
1. Align special tokens: make sure the student's EOS/turn-end tokens match the teacher's, or remap them; otherwise mask the loss at termination positions.
2. Measure top-$p$ coverage ($p=0.9$) of the student on teacher-generated prefixes for a few hundred prompts; large mass of low-coverage states means do SFT on teacher outputs first.

**Monitor during training:** completion rate within the generation budget, mean response length and an n-gram loop rate, *not just* teacher-student overlap. Rising length with stable overlap is the spurious-agreement signature.

**Smallest experiment:** run 100 OPD steps on a base student with a chat teacher, once with mismatched EOS and once with EOS aligned (or masked). Plot length and completion rate; if the mismatched run's length climbs, you have reproduced the failure.

Assessment: Adopt now. Aligning or masking special tokens and monitoring completion rate rather than overlap are free, and the paper shows the EOS mismatch alone causes collapse.

### Connections

Prior work patched OPD instability: adding forward KL or an SFT loss, selecting tokens, or switching to forward KL by teacher entropy (EOPD) or advantage sign (AOPD).

What is new is the mechanistic account (low-probability teacher tokens cannot be learned by sampled-token updates; their gradients contaminate other positions; on-policy sampling then moves the student to states where the teacher is unreliable) and a concrete, checkable trigger: a format/termination token outside the student's support.

**Connections.** Idea 5 is the fix derived from this diagnosis. Idea 6 shows a similar "the training signal assumes a fixed interface" coupling for agent harnesses.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-23 · origin · [When Students Fail to Learn from Teachers: Learning Dynamics of On-Policy Distillation](https://arxiv.org/abs/2611.27125) — Traces OPD collapse to teacher-preferred tokens outside the student's support: EOS position carried 31.81% of cross-position influence, math length grew 1.2k to 4.1k tokens and cap-hitting rollouts rose 14% to 100%, removed by aligning or masking EOS.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-23: +relation extended-by PT-20261123-05.
- 2026-11-23: created from bulletin 2026-11-23 (#4).
