# Idea Atlas — Index

The dashboard for the whole Atlas: totals, recent changes and the top ideas in each section. Generated from the idea files.

<!-- BEGIN:GENERATED -->
_10 ideas (AG 4 · ENV 3 · PT 3) · last updated 2026-11-23_

## Sections

| Section | Ideas | emerging | replicated | established | contested | superseded |
| --- | --- | --- | --- | --- | --- | --- |
| [AI Agents](sections/agents/README.md) | 4 | 4 | 0 | 0 | 0 | 0 |
| [RL Environments](sections/rl-envs/README.md) | 3 | 3 | 0 | 0 | 0 | 0 |
| [Post-Training](sections/post-training/README.md) | 3 | 3 | 0 | 0 | 0 | 0 |

## Recently changed (last 14 days)

- [AG-20261123-02](sections/agents/ideas/AG-20261123-02--which-links-web-agent-clicks-leaks.md) **Which links a web agent clicks leaks its private context past content-level defenses** — A stateful attacker placing one link on a visited page reconstructs a web agent's private query (F1 0.89) and profile (F1 0.80) from which links it clicks; content filters do not help. · emerging · 85 · 2026-11-23
- [AG-20261123-01](sections/agents/ideas/AG-20261123-01--judge-visual-agent-s-candidate-actions.md) **Judge a visual agent's candidate actions by drawing them on the screenshot** — Sample several actions from a frozen visual policy, draw each on its own screenshot copy, score them with a probability-returning judge, and override greedy only on a clear lead; no training needed. · emerging · 81 · 2026-11-23
- [AG-20261123-06](sections/agents/ideas/AG-20261123-06--weights-fine-tuned-under-one-agent.md) **Weights fine-tuned under one agent harness collapse under another** — Weights fine-tuned on trajectories from their own evolved harness score 19.8% under that harness but 2.6% under a reference harness, so harness revisions reopen the training decision. · emerging · 79 · 2026-11-23
- [ENV-20261123-07](sections/rl-envs/ideas/ENV-20261123-07--most-broken-agentic-benchmark-graders-reject.md) **Most broken agentic-benchmark graders reject valid work rather than reward shortcuts** — A 749-task audit across eight benchmarks finds only 29.3% of multi-step tasks have prompt, environment and grader that agree, and 71.7% of defects wrongly reject valid work rather than reward shortcuts. · emerging · 79 · 2026-11-23
- [PT-20261123-04](sections/post-training/ideas/PT-20261123-04--one-token-student-cannot-produce-derail.md) **One token the student cannot produce can derail on-policy distillation** — In sampled-token on-policy distillation, a teacher-preferred token the student almost never samples (like a mismatched EOS) is never learned yet still distorts updates, lengthening and looping outputs while overlap stays high. · emerging · 79 · 2026-11-23
- [PT-20261123-03](sections/post-training/ideas/PT-20261123-03--asking-yes-no-decision-model-per.md) **Asking a yes/no decision model per step gives label-free process rewards for agent RL** — Per step of every rollout, ask a probability-returning decision model whether the step made progress, standardise scores within the task's rollout group, and add them to GRPO advantages; ALFWorld success rises from 66.3% to 90.0%. · emerging · 78 · 2026-11-23
- [PT-20261123-05](sections/post-training/ideas/PT-20261123-05--route-low-coverage-states-forward-kl.md) **Route low-coverage states to forward KL during on-policy distillation** — Applying top-16 forward KL only at states where the student's mass on the teacher's top-p tokens is low, or teacher mass is underestimated, beats sampled-token on-policy distillation and blanket forward KL. · emerging · 76 · 2026-11-23
- [ENV-20261123-08](sections/rl-envs/ideas/ENV-20261123-08--what-users-later-do-agent-s.md) **What users later do with an agent's output is a refreshable, hard-to-game verifier** — Labeling agent output as useful when the person receiving it later acts on it gives a refreshable verifier whose precision rankings match a gold set (rho 0.69) and resist gaming. · emerging · 75 · 2026-11-23
- [AG-20261123-09](sections/agents/ideas/AG-20261123-09--agent-chosen-belief-state-folds-turn.md) **Agent-chosen belief-state folds turn long research runs into rubric-rewarded segments** — Letting a research agent rewrite its context into a four-field belief state at self-chosen checkpoints, then rewarding each resulting segment with an outcome-blind rubric plus validity gate, reaches 0.151 Brier on ForecastBench. · emerging · 72 · 2026-11-23
- [ENV-20261123-10](sections/rl-envs/ideas/ENV-20261123-10--grade-diff-fresh-container-confirmed-bypasses.md) **Grade the diff in a fresh container: confirmed bypasses fall from 14.7% to 2.7%** — Extract only the committed patch or declared outputs, grade in a fresh network-disabled container, blind-solve tasks before freezing tests, and require broken submissions the grader rejects; bypasses fall from 14.7% to 2.7%. · emerging · 66 · 2026-11-23

## Top ideas by power (top 15 per section)

### AI Agents

- [AG-20261123-02](sections/agents/ideas/AG-20261123-02--which-links-web-agent-clicks-leaks.md) **Which links a web agent clicks leaks its private context past content-level defenses** — A stateful attacker placing one link on a visited page reconstructs a web agent's private query (F1 0.89) and profile (F1 0.80) from which links it clicks; content filters do not help. · emerging · 85 · 2026-11-23
- [AG-20261123-01](sections/agents/ideas/AG-20261123-01--judge-visual-agent-s-candidate-actions.md) **Judge a visual agent's candidate actions by drawing them on the screenshot** — Sample several actions from a frozen visual policy, draw each on its own screenshot copy, score them with a probability-returning judge, and override greedy only on a clear lead; no training needed. · emerging · 81 · 2026-11-23
- [AG-20261123-06](sections/agents/ideas/AG-20261123-06--weights-fine-tuned-under-one-agent.md) **Weights fine-tuned under one agent harness collapse under another** — Weights fine-tuned on trajectories from their own evolved harness score 19.8% under that harness but 2.6% under a reference harness, so harness revisions reopen the training decision. · emerging · 79 · 2026-11-23
- [AG-20261123-09](sections/agents/ideas/AG-20261123-09--agent-chosen-belief-state-folds-turn.md) **Agent-chosen belief-state folds turn long research runs into rubric-rewarded segments** — Letting a research agent rewrite its context into a four-field belief state at self-chosen checkpoints, then rewarding each resulting segment with an outcome-blind rubric plus validity gate, reaches 0.151 Brier on ForecastBench. · emerging · 72 · 2026-11-23

### RL Environments

- [ENV-20261123-07](sections/rl-envs/ideas/ENV-20261123-07--most-broken-agentic-benchmark-graders-reject.md) **Most broken agentic-benchmark graders reject valid work rather than reward shortcuts** — A 749-task audit across eight benchmarks finds only 29.3% of multi-step tasks have prompt, environment and grader that agree, and 71.7% of defects wrongly reject valid work rather than reward shortcuts. · emerging · 79 · 2026-11-23
- [ENV-20261123-08](sections/rl-envs/ideas/ENV-20261123-08--what-users-later-do-agent-s.md) **What users later do with an agent's output is a refreshable, hard-to-game verifier** — Labeling agent output as useful when the person receiving it later acts on it gives a refreshable verifier whose precision rankings match a gold set (rho 0.69) and resist gaming. · emerging · 75 · 2026-11-23
- [ENV-20261123-10](sections/rl-envs/ideas/ENV-20261123-10--grade-diff-fresh-container-confirmed-bypasses.md) **Grade the diff in a fresh container: confirmed bypasses fall from 14.7% to 2.7%** — Extract only the committed patch or declared outputs, grade in a fresh network-disabled container, blind-solve tasks before freezing tests, and require broken submissions the grader rejects; bypasses fall from 14.7% to 2.7%. · emerging · 66 · 2026-11-23

### Post-Training

- [PT-20261123-04](sections/post-training/ideas/PT-20261123-04--one-token-student-cannot-produce-derail.md) **One token the student cannot produce can derail on-policy distillation** — In sampled-token on-policy distillation, a teacher-preferred token the student almost never samples (like a mismatched EOS) is never learned yet still distorts updates, lengthening and looping outputs while overlap stays high. · emerging · 79 · 2026-11-23
- [PT-20261123-03](sections/post-training/ideas/PT-20261123-03--asking-yes-no-decision-model-per.md) **Asking a yes/no decision model per step gives label-free process rewards for agent RL** — Per step of every rollout, ask a probability-returning decision model whether the step made progress, standardise scores within the task's rollout group, and add them to GRPO advantages; ALFWorld success rises from 66.3% to 90.0%. · emerging · 78 · 2026-11-23
- [PT-20261123-05](sections/post-training/ideas/PT-20261123-05--route-low-coverage-states-forward-kl.md) **Route low-coverage states to forward KL during on-policy distillation** — Applying top-16 forward KL only at states where the student's mass on the teacher's top-p tokens is low, or teacher mass is underestimated, beats sampled-token on-policy distillation and blanket forward KL. · emerging · 76 · 2026-11-23
<!-- END:GENERATED -->
