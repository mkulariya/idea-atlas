# Post-Training (PT)

> Everything after pretraining that shapes a model's behavior and capability.

## State of the field
_Last revised: not yet. The agent rewrites this paragraph (at most 250 words, dated) when a day's ideas change the big picture: the main directions, what is settled, what is contested, what to watch._

## Ideas by subtopic

Generated from the idea files; highest power first.

<!-- BEGIN:GENERATED -->
_3 ideas filed here, 0 cross-listed from other sections._

### sft-data
_Supervised fine-tuning, instruction and synthetic data, data selection and mixing._

_No ideas yet._

### distillation
_Distilling capabilities or reasoning from stronger models or from the model itself._

- [PT-20261123-04](ideas/PT-20261123-04--one-token-student-cannot-produce-derail.md) **One token the student cannot produce can derail on-policy distillation** — In sampled-token on-policy distillation, a teacher-preferred token the student almost never samples (like a mismatched EOS) is never learned yet still distorts updates, lengthening and looping outputs while overlap stays high. · emerging · 79 · 2026-11-23
- [PT-20261123-05](ideas/PT-20261123-05--route-low-coverage-states-forward-kl.md) **Route low-coverage states to forward KL during on-policy distillation** — Applying top-16 forward KL only at states where the student's mass on the teacher's top-p tokens is low, or teacher mass is underestimated, beats sampled-token on-policy distillation and blanket forward KL. · emerging · 76 · 2026-11-23

### preference-optimization
_RLHF and preference objectives (DPO family and successors)._

_No ideas yet._

### rl-algorithms
_Policy-optimization algorithms (PPO and GRPO variants), advantage estimation, credit assignment, entropy, exploration, off-policy and asynchronous RL._

- [PT-20261123-03](ideas/PT-20261123-03--asking-yes-no-decision-model-per.md) **Asking a yes/no decision model per step gives label-free process rewards for agent RL** — Per step of every rollout, ask a probability-returning decision model whether the step made progress, standardise scores within the task's rollout group, and add them to GRPO advantages; ALFWorld success rises from 66.3% to 90.0%. · emerging · 78 · 2026-11-23

### reasoning-training
_Training long chain-of-thought, controlling thinking length, training for test-time scaling._

_No ideas yet._

### reward-models
_Outcome, process and generative reward models; training LLM judges._

_No ideas yet._

### alignment-safety
_Refusals, honesty, specifications and constitutions, character, safety training._

_No ideas yet._

### efficiency-systems
_Parameter-efficient methods, rollout efficiency and systems for post-training._

_No ideas yet._

### merging-continual
_Model merging, continual learning and catastrophic forgetting._

_No ideas yet._

### self-improvement
_Self-play, self-rewarding and self-training loops for models._

_No ideas yet._

### scaling-analysis
_Scaling laws and analyses of what post-training changes._

_No ideas yet._

### other
_Fits no subtopic above; review weekly._

_No ideas yet._

### cross-listed
_Ideas filed in another section that also matter here (their `also_in` names this section)._

_None yet._
<!-- END:GENERATED -->
