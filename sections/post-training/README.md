# Post-Training (PT)

> Everything after pretraining that shapes a model's behavior and capability.

## State of the field
_Last revised: not yet. The agent rewrites this paragraph (at most 250 words, dated) when a day's ideas change the big picture: the main directions, what is settled, what is contested, what to watch._

## Ideas by subtopic

Generated from the idea files; highest power first.

<!-- BEGIN:GENERATED -->
_11 ideas filed here, 0 cross-listed from other sections._

### sft-data
_Supervised fine-tuning, instruction and synthetic data, data selection and mixing._

_No ideas yet._

### distillation
_Distilling capabilities or reasoning from stronger models or from the model itself._

- [PT-20261123-04](ideas/PT-20261123-04--one-token-student-cannot-produce-derail.md) **One token the student cannot produce can derail on-policy distillation** — In sampled-token on-policy distillation, a teacher-preferred token the student almost never samples (like a mismatched EOS) is never learned yet still distorts updates, lengthening and looping outputs while overlap stays high. · emerging · 79 · 2026-11-23
- [PT-20261126-03](ideas/PT-20261126-03--harness-rehearsal-makes-cross-harness-task.md) **Harness rehearsal makes cross-harness task demonstrations transfer to a target interface** — Adding rehearsal traces that state the solution in the instruction, so the model only practices the target interface, lifts cross-harness distillation from 8.5% to 58.8% on BFCL multi-turn. · emerging · 77 · 2026-11-26
- [PT-20261123-05](ideas/PT-20261123-05--route-low-coverage-states-forward-kl.md) **Route low-coverage states to forward KL during on-policy distillation** — Applying top-16 forward KL only at states where the student's mass on the teacher's top-p tokens is low, or teacher mass is underestimated, beats sampled-token on-policy distillation and blanket forward KL. · emerging · 76 · 2026-11-23

### preference-optimization
_RLHF and preference objectives (DPO family and successors)._

_No ideas yet._

### rl-algorithms
_Policy-optimization algorithms (PPO and GRPO variants), advantage estimation, credit assignment, entropy, exploration, off-policy and asynchronous RL._

- [PT-20261123-03](ideas/PT-20261123-03--asking-yes-no-decision-model-per.md) **Asking a yes/no decision model per step gives label-free process rewards for agent RL** — Per step of every rollout, ask a probability-returning decision model whether the step made progress, standardise scores within the task's rollout group, and add them to GRPO advantages; ALFWorld success rises from 66.3% to 90.0%. · emerging · 78 · 2026-11-26
- [PT-20261125-03](ideas/PT-20261125-03--evolution-strategies-beat-policy-gradients-only.md) **Evolution strategies beat policy gradients only when effective horizon is long and skills few** — ES and policy gradients estimate nearly the same gradient but pay for different quantities: ES scales with the number of independent skills, PG with reward-relevant tokens, so ES wins only on long, few-skill tasks. · emerging · 78 · 2026-11-25
- [PT-20261126-05](ideas/PT-20261126-05--rewarding-actions-against-simulator-s-exact.md) **Rewarding actions against the simulator's exact posterior teaches agents when to stop searching** — Reward each agent action by the environment's exact posterior-aware teacher's log-probability, comparing turns at the same history, so training stops premature guessing without a critic or judge. · emerging · 75 · 2026-11-26
- [PT-20261125-09](ideas/PT-20261125-09--rubric-rl-after-self-distillation-more.md) **Rubric RL after self-distillation more than doubles long-document strict pass at flat token spend** — Privileged-context self-distillation then two rounds of rubric-reward RL at full context lifts strict pass on long professional documents from 11.0% to 24.5% at flat token spend. · emerging · 69 · 2026-11-25
- [PT-20261125-08](ideas/PT-20261125-08--contrasting-sae-features-correct-incorrect-rollouts.md) **Contrasting SAE features of correct and incorrect rollouts gives GRPO token-level credit** — Score each token by whether its SAE features match correct rather than incorrect rollouts of the same problem, then add that to the outcome advantage in GRPO. · emerging · 67 · 2026-11-26

### reasoning-training
_Training long chain-of-thought, controlling thinking length, training for test-time scaling._

- [PT-20261126-08](ideas/PT-20261126-08--training-summarizers-preserve-frozen-solver-s.md) **Training summarizers to preserve a frozen solver's short-context predictions extends reasoning horizons** — Reward a summarizer by how well a frozen solver's next-token predictions under its summary match those under the full reasoning prefix, extending effective horizon without answers or a verifier. · emerging · 76 · 2026-11-26
- [PT-20261125-06](ideas/PT-20261125-06--train-binary-reward-then-abstain-using.md) **Train with binary reward, then abstain using a conformal cutoff on answer-tail surprise** — Freeze a reasoning model trained with plain correctness reward under a forced token cap, then submit an answer only when its answer-span tail token surprise falls below a conformal cutoff calibrated on held-out data. · emerging · 66 · 2026-11-25

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

- [PT-20261126-06](ideas/PT-20261126-06--replaying-model-s-own-answers-general.md) **Replaying a model's own answers to general instructions largely prevents forgetting during factual fine-tuning** — Adding a forward-KL penalty to a frozen starting model on that model's own saved answers to general instructions cuts fine-tuning forgetting from 28.07 to 2.44 points at equal fact acquisition. · emerging · 73 · 2026-11-26

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
