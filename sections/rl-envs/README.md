# RL Environments (ENV)

> The worlds, tasks and reward signals used to train or evaluate models with reinforcement learning.

## State of the field
_Last revised: not yet. The agent rewrites this paragraph (at most 250 words, dated) when a day's ideas change the big picture: the main directions, what is settled, what is contested, what to watch._

## Ideas by subtopic

Generated from the idea files; highest power first.

<!-- BEGIN:GENERATED -->
_9 ideas filed here, 1 cross-listed from other sections._

### env-construction
_Designing and building environments; procedural and synthetic task generation; scaling the number and diversity of environments._

- [ENV-20261126-01](ideas/ENV-20261126-01--tools-read-hidden-simulator-state-silently.md) **Tools that read hidden simulator state silently solve the search benchmarks claim to test** — If a tool can read simulator state the agent never perceived, benchmarks measure the tool author's shortcut: enforcing evidence closure cuts mid-sized Minecraft agents' diamond rate from 76%–94% to 32%–40%. · emerging · 84 · 2026-11-26
- [ENV-20261127-02](ideas/ENV-20261127-02--planted-proof-depth-overstates-what-logic.md) **Planted proof depth overstates what logic problems require; certify the mandatory steps** — A checker can compute how many proof steps every solution must take; on FLD×2, problems labelled depth 5 need about two mandatory steps, so planted-depth labels overstate difficulty. · emerging · 82 · 2026-11-27

### verifiers-rewards
_Verifiers and reward functions (unit tests, programmatic checkers, rubrics, LLM-judge rewards inside environments)._

- [ENV-20261125-02](ideas/ENV-20261125-02--llm-judges-grow-more-lenient-rule.md) **LLM judges grow more lenient on a rule inside a larger ruleset** — A judge flagging a rule violation alone often finds none when the same rule sits in a larger ruleset, leniency growing with ruleset size; per-rule forcing, consistency training and ruleset rewrite shrink it. · emerging · 82 · 2026-11-25
- [ENV-20261123-07](ideas/ENV-20261123-07--most-broken-agentic-benchmark-graders-reject.md) **Most broken agentic-benchmark graders reject valid work rather than reward shortcuts** — A 749-task audit across eight benchmarks finds only 29.3% of multi-step tasks have prompt, environment and grader that agree, and 71.7% of defects wrongly reject valid work rather than reward shortcuts. · emerging · 79 · 2026-11-27
- [ENV-20261123-08](ideas/ENV-20261123-08--what-users-later-do-agent-s.md) **What users later do with an agent's output is a refreshable, hard-to-game verifier** — Labeling agent output as useful when the person receiving it later acts on it gives a refreshable verifier whose precision rankings match a gold set (rho 0.69) and resist gaming. · emerging · 75 · 2026-11-23
- [ENV-20261123-10](ideas/ENV-20261123-10--grade-diff-fresh-container-confirmed-bypasses.md) **Grade the diff in a fresh container: confirmed bypasses fall from 14.7% to 2.7%** — Extract only the committed patch or declared outputs, grade in a fresh network-disabled container, blind-solve tasks before freezing tests, and require broken submissions the grader rejects; bypasses fall from 14.7% to 2.7%. · emerging · 66 · 2026-11-26
- [ENV-20261125-11](ideas/ENV-20261125-11--severity-weighted-atomic-rubrics-penalty-only.md) **Severity-weighted atomic rubrics with penalty-only dodged-bullet items make a usable RL reward** — Typing atomic expert-rubric criteria as primary-intent or dodged-bullet and rewarding the severity-weighted share satisfied, with dodged-bullet items penalty-only, gives a trainable reward for open-ended professional deliverables. · emerging · 61 · 2026-11-25

### reward-hacking
_Specification gaming, exploits of environments or verifiers, and defenses._

_No ideas yet._

### curricula-open-endedness
_Curricula, difficulty control, auto-curricula and open-ended task generation._

_No ideas yet._

### simulated-users-tools
_Simulated users, simulated tools and APIs, and world models used as environments._

_No ideas yet._

### task-suites-benchmarks
_Task suites and benchmarks that double as training environments._

- [ENV-20261127-01](ideas/ENV-20261127-01--skillsbench-skill-lift-partly-grader-quirks.md) **SkillsBench skill lift is partly grader quirks, and its per-task losses are within noise** — Permutation tests with FDR control find no per-task skill harms in SkillsBench, grader quirks inflate part of the lift, and label-shuffled replay shows log-fitted skill gates gain 4-7 pp from noise alone. · emerging · 82 · 2026-11-27
- [ENV-20261126-10](ideas/ENV-20261126-10--self-improving-research-agents-transfer-unevenly.md) **Self-improving research agents transfer unevenly, and learned rules can backfire on new tasks** — Freezing four self-improving research agents after evolution and testing on independent tasks yields gains from −3.34% to +2.62%, task-level regressions everywhere, and stored anti-overfitting rules that discard inputs new tasks need. · emerging · 73 · 2026-11-27

### env-infrastructure
_Sandboxes, rollout servers, asynchronous and large-scale rollout systems._

_No ideas yet._

### multi-agent-self-play
_Multi-agent, competitive and self-play environments._

_No ideas yet._

### other
_Fits no subtopic above; review weekly._

_No ideas yet._

### cross-listed
_Ideas filed in another section that also matter here (their `also_in` names this section)._

- [AG-20261125-01](../agents/ideas/AG-20261125-01--agents-fabricate-ingesting-partial-tool-output.md) **Agents fabricate by ingesting partial tool output as valid, not by retrying** — Agents accept plausible but truncated tool results as valid on first contact, fabricating 38% of the time on partial-data faults, while agents that retry visibly failing calls usually report failure honestly. · emerging · 79 · 2026-11-26 _(from agents › tool-use)_
<!-- END:GENERATED -->
