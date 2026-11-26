# AI Agents (AG)

> Systems in which a model takes multiple steps of action toward a goal, using tools, memory, environment feedback or other agents.

## State of the field
_Last revised: not yet. The agent rewrites this paragraph (at most 250 words, dated) when a day's ideas change the big picture: the main directions, what is settled, what is contested, what to watch._

## Ideas by subtopic

Generated from the idea files; highest power first.

<!-- BEGIN:GENERATED -->
_13 ideas filed here, 0 cross-listed from other sections._

### architectures-scaffolds
_Agent loops and system designs (planner-executor, reflection, routing, orchestration of a single agent)._

- [AG-20261123-01](ideas/AG-20261123-01--judge-visual-agent-s-candidate-actions.md) **Judge a visual agent's candidate actions by drawing them on the screenshot** — Sample several actions from a frozen visual policy, draw each on its own screenshot copy, score them with a probability-returning judge, and override greedy only on a clear lead; no training needed. · emerging · 81 · 2026-11-23

### tool-use
_Tool and function calling, tool selection, tool creation, tool protocols such as MCP._

- [AG-20261125-01](ideas/AG-20261125-01--agents-fabricate-ingesting-partial-tool-output.md) **Agents fabricate by ingesting partial tool output as valid, not by retrying** — Agents accept plausible but truncated tool results as valid on first contact, fabricating 38% of the time on partial-data faults, while agents that retry visibly failing calls usually report failure honestly. · emerging · 79 · 2026-11-26
- [AG-20261125-10](ideas/AG-20261125-10--check-agent-s-answer-against-tool.md) **Check an agent's answer against its tool outputs in code before any LLM judge** — A monitor extracts the values an answer asserts, vetoes those absent from or not recomputable from tool outputs, and sends a corrective directive rather than a hard block, cutting fabrication from 17.3% to 10.8%. · emerging · 73 · 2026-11-25

### planning-search
_Planning, lookahead, tree search and test-time compute over actions._

_No ideas yet._

### memory-context
_Long-horizon memory, context management, compression, retrieval memory, state tracking._

- [AG-20261125-07](ideas/AG-20261125-07--fold-every-past-attempt-expandable-digest.md) **Fold every past attempt into an expandable digest instead of summarizing history** — Digesting every past attempt so the agent can still expand any of them, plus deleting skills unused for five iterations, beat recency truncation on MLE-Bench and KernelBench across three backbones. · emerging · 73 · 2026-11-26
- [AG-20261123-09](ideas/AG-20261123-09--agent-chosen-belief-state-folds-turn.md) **Agent-chosen belief-state folds turn long research runs into rubric-rewarded segments** — Letting a research agent rewrite its context into a four-field belief state at self-chosen checkpoints, then rewarding each resulting segment with an outcome-blind rubric plus validity gate, reaches 0.151 Brier on ForecastBench. · emerging · 72 · 2026-11-25

### coding-agents
_Software-engineering and coding agents, repository-level work, debugging, code review._

_No ideas yet._

### computer-web-agents
_Web, browser, OS and GUI (computer-use) agents, including grounding on screens._

- [AG-20261126-07](ideas/AG-20261126-07--updating-pending-gui-action-newest-observation.md) **Updating a pending GUI action with the newest observation fixes failures latency cannot explain** — Keeping a GUI agent's pending click revisable until commitment and re-binding its coordinates to the newest observation raised success from 0% to 100% on a moving-target task without faster responses. · emerging · 77 · 2026-11-26

### research-agents
_Deep-research, literature, scientific-discovery and data-analysis agents._

_No ideas yet._

### multi-agent
_Multiple agents that communicate, debate, divide labor or are orchestrated together._

- [AG-20261126-02](ideas/AG-20261126-02--frontier-models-delegate-legwork-small-local.md) **Frontier models that delegate legwork to small local models cut cost by two thirds** — Route subagent calls to a small local model and tell the frontier model that its own turns are expensive; coding cost falls to a third with no fewer tasks resolved. · emerging · 71 · 2026-11-26

### self-improvement
_Agents that improve themselves at run time (skill libraries, self-generated tools, experience reuse)._

- [AG-20261125-04](ideas/AG-20261125-04--self-improving-agents-write-runtime-checks.md) **Self-improving agents write runtime checks that stay silent on most failures they can repair** — Agents that self-improve programs to high task scores write runtime checks that miss 52–89% of failures after an environment change, though the same agents repair most of those failures when called. · emerging · 77 · 2026-11-26
- [AG-20261126-09](ideas/AG-20261126-09--unguided-coding-agents-optimize-other-agents.md) **Unguided coding agents optimize other agents better than published methods; guidance adds little** — Handing an off-the-shelf coding agent a task prompt, train split and dollar budget to edit a target agent beat every published optimizer on five benchmarks, while added guidance skills and subagents did not help. · emerging · 74 · 2026-11-26

### agent-evaluation
_Benchmarks, metrics and methodology for evaluating agents._

- [AG-20261125-05](ideas/AG-20261125-05--retries-raise-cost-per-completion-because.md) **Retries raise cost per completion because easy tasks leave the pool first** — Because easy tasks succeed and leave first, each extra same-task retry draws from a harder task pool, so cost per completion rises above one-shot cost. · emerging · 69 · 2026-11-25

### agent-safety-security
_Prompt injection, sandboxing, monitoring, oversight, reliability and misuse of agents._

- [AG-20261123-02](ideas/AG-20261123-02--which-links-web-agent-clicks-leaks.md) **Which links a web agent clicks leaks its private context past content-level defenses** — A stateful attacker placing one link on a visited page reconstructs a web agent's private query (F1 0.89) and profile (F1 0.80) from which links it clicks; content filters do not help. · emerging · 85 · 2026-11-23
- [AG-20261126-04](ideas/AG-20261126-04--quarter-recoveries-after-partial-execution-harm.md) **A quarter of recoveries after partial execution harm state; repeat-call monitors catch only 41%** — When a code agent's block fails after some writes landed, about 25% of confirmed recoveries leave external state wrong, and a repeat-call monitor catches only 41% of that harm. · emerging · 77 · 2026-11-26

### agent-training
_Training recipes specific to agents (trajectory data, agentic SFT, agent RL setups)._

- [AG-20261123-06](ideas/AG-20261123-06--weights-fine-tuned-under-one-agent.md) **Weights fine-tuned under one agent harness collapse under another** — Weights fine-tuned on trajectories from their own evolved harness score 19.8% under that harness but 2.6% under a reference harness, so harness revisions reopen the training decision. · replicated · 79 · 2026-11-26

### other
_Fits no subtopic above; review weekly._

_No ideas yet._

### cross-listed
_Ideas filed in another section that also matter here (their `also_in` names this section)._

_None yet._
<!-- END:GENERATED -->
