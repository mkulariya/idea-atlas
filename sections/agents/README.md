# AI Agents (AG)

> Systems in which a model takes multiple steps of action toward a goal, using tools, memory, environment feedback or other agents.

## State of the field
_Last revised: not yet. The agent rewrites this paragraph (at most 250 words, dated) when a day's ideas change the big picture: the main directions, what is settled, what is contested, what to watch._

## Ideas by subtopic

Generated from the idea files; highest power first.

<!-- BEGIN:GENERATED -->
_4 ideas filed here, 0 cross-listed from other sections._

### architectures-scaffolds
_Agent loops and system designs (planner-executor, reflection, routing, orchestration of a single agent)._

- [AG-20261123-01](ideas/AG-20261123-01--judge-visual-agent-s-candidate-actions.md) **Judge a visual agent's candidate actions by drawing them on the screenshot** — Sample several actions from a frozen visual policy, draw each on its own screenshot copy, score them with a probability-returning judge, and override greedy only on a clear lead; no training needed. · emerging · 81 · 2026-11-23

### tool-use
_Tool and function calling, tool selection, tool creation, tool protocols such as MCP._

_No ideas yet._

### planning-search
_Planning, lookahead, tree search and test-time compute over actions._

_No ideas yet._

### memory-context
_Long-horizon memory, context management, compression, retrieval memory, state tracking._

- [AG-20261123-09](ideas/AG-20261123-09--agent-chosen-belief-state-folds-turn.md) **Agent-chosen belief-state folds turn long research runs into rubric-rewarded segments** — Letting a research agent rewrite its context into a four-field belief state at self-chosen checkpoints, then rewarding each resulting segment with an outcome-blind rubric plus validity gate, reaches 0.151 Brier on ForecastBench. · emerging · 72 · 2026-11-23

### coding-agents
_Software-engineering and coding agents, repository-level work, debugging, code review._

_No ideas yet._

### computer-web-agents
_Web, browser, OS and GUI (computer-use) agents, including grounding on screens._

_No ideas yet._

### research-agents
_Deep-research, literature, scientific-discovery and data-analysis agents._

_No ideas yet._

### multi-agent
_Multiple agents that communicate, debate, divide labor or are orchestrated together._

_No ideas yet._

### self-improvement
_Agents that improve themselves at run time (skill libraries, self-generated tools, experience reuse)._

_No ideas yet._

### agent-evaluation
_Benchmarks, metrics and methodology for evaluating agents._

_No ideas yet._

### agent-safety-security
_Prompt injection, sandboxing, monitoring, oversight, reliability and misuse of agents._

- [AG-20261123-02](ideas/AG-20261123-02--which-links-web-agent-clicks-leaks.md) **Which links a web agent clicks leaks its private context past content-level defenses** — A stateful attacker placing one link on a visited page reconstructs a web agent's private query (F1 0.89) and profile (F1 0.80) from which links it clicks; content filters do not help. · emerging · 85 · 2026-11-23

### agent-training
_Training recipes specific to agents (trajectory data, agentic SFT, agent RL setups)._

- [AG-20261123-06](ideas/AG-20261123-06--weights-fine-tuned-under-one-agent.md) **Weights fine-tuned under one agent harness collapse under another** — Weights fine-tuned on trajectories from their own evolved harness score 19.8% under that harness but 2.6% under a reference harness, so harness revisions reopen the training decision. · emerging · 79 · 2026-11-23

### other
_Fits no subtopic above; review weekly._

_No ideas yet._

### cross-listed
_Ideas filed in another section that also matter here (their `also_in` names this section)._

_None yet._
<!-- END:GENERATED -->
