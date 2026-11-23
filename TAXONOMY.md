# TAXONOMY — controlled vocabulary of the Idea Atlas

Every `section`, `subtopic`, `type`, `status`, relation and source role used anywhere in the Atlas must come from the YAML block below. Tools read this block, so keep it valid YAML.

## How to choose a section (primary-section rule)

Ask: **"What would someone copy to reuse this idea?"**

- A system, loop or interaction design → `agents` (AG)
- A task, environment, verifier or reward signal → `rl-envs` (ENV)
- A training objective, algorithm or data recipe → `post-training` (PT)

Put other relevant sections in `also_in`. The primary section is permanent because it is part of the ID.

## How to change this taxonomy

- Add a subtopic only when **three or more** ideas fit no existing subtopic (they will be sitting in `other`).
- Never rename or delete a slug that ideas use. To split a subtopic, add the new slugs, move ideas by editing their `subtopic` field, and keep the old slug until no idea uses it.
- Record every change in `CHANGELOG.md` under that day's entry.
- Subtopic order below is the order used in section READMEs.

```yaml
sections:
  agents:
    code: AG
    title: AI Agents
    scope: Systems in which a model takes multiple steps of action toward a goal, using tools, memory, environment feedback or other agents.
    subtopics:
      architectures-scaffolds: Agent loops and system designs (planner-executor, reflection, routing, orchestration of a single agent).
      tool-use: Tool and function calling, tool selection, tool creation, tool protocols such as MCP.
      planning-search: Planning, lookahead, tree search and test-time compute over actions.
      memory-context: Long-horizon memory, context management, compression, retrieval memory, state tracking.
      coding-agents: Software-engineering and coding agents, repository-level work, debugging, code review.
      computer-web-agents: Web, browser, OS and GUI (computer-use) agents, including grounding on screens.
      research-agents: Deep-research, literature, scientific-discovery and data-analysis agents.
      multi-agent: Multiple agents that communicate, debate, divide labor or are orchestrated together.
      self-improvement: Agents that improve themselves at run time (skill libraries, self-generated tools, experience reuse).
      agent-evaluation: Benchmarks, metrics and methodology for evaluating agents.
      agent-safety-security: Prompt injection, sandboxing, monitoring, oversight, reliability and misuse of agents.
      agent-training: Training recipes specific to agents (trajectory data, agentic SFT, agent RL setups).
      other: Fits no subtopic above; review weekly.
  rl-envs:
    code: ENV
    title: RL Environments
    scope: The worlds, tasks and reward signals used to train or evaluate models with reinforcement learning.
    subtopics:
      env-construction: Designing and building environments; procedural and synthetic task generation; scaling the number and diversity of environments.
      verifiers-rewards: Verifiers and reward functions (unit tests, programmatic checkers, rubrics, LLM-judge rewards inside environments).
      reward-hacking: Specification gaming, exploits of environments or verifiers, and defenses.
      curricula-open-endedness: Curricula, difficulty control, auto-curricula and open-ended task generation.
      simulated-users-tools: Simulated users, simulated tools and APIs, and world models used as environments.
      task-suites-benchmarks: Task suites and benchmarks that double as training environments.
      env-infrastructure: Sandboxes, rollout servers, asynchronous and large-scale rollout systems.
      multi-agent-self-play: Multi-agent, competitive and self-play environments.
      other: Fits no subtopic above; review weekly.
  post-training:
    code: PT
    title: Post-Training
    scope: Everything after pretraining that shapes a model's behavior and capability.
    subtopics:
      sft-data: Supervised fine-tuning, instruction and synthetic data, data selection and mixing.
      distillation: Distilling capabilities or reasoning from stronger models or from the model itself.
      preference-optimization: RLHF and preference objectives (DPO family and successors).
      rl-algorithms: Policy-optimization algorithms (PPO and GRPO variants), advantage estimation, credit assignment, entropy, exploration, off-policy and asynchronous RL.
      reasoning-training: Training long chain-of-thought, controlling thinking length, training for test-time scaling.
      reward-models: Outcome, process and generative reward models; training LLM judges.
      alignment-safety: Refusals, honesty, specifications and constitutions, character, safety training.
      efficiency-systems: Parameter-efficient methods, rollout efficiency and systems for post-training.
      merging-continual: Model merging, continual learning and catastrophic forgetting.
      self-improvement: Self-play, self-rewarding and self-training loops for models.
      scaling-analysis: Scaling laws and analyses of what post-training changes.
      other: Fits no subtopic above; review weekly.

types:
  method: A new algorithm, objective, architecture or loop.
  recipe: A practical trick or configuration that reliably helps.
  finding: An empirical insight about how things behave.
  framing: A new abstraction that clarifies a problem.
  env-design: A way to build tasks, environments, verifiers or rewards.
  failure-mode: A documented way things break, with its fix if known.

statuses:
  emerging: One source, or one research group.
  replicated: Supported by two or more independent groups.
  established: Four or more supporting sources, or used in widely adopted open recipes.
  contested: Credible counter-evidence (grade B or better) not yet resolved.
  superseded: A newer idea does the same job better; superseded_by points to it.

# relation -> its inverse (write both directions)
relations:
  builds-on: extended-by
  extended-by: builds-on
  variant-of: has-variant
  has-variant: variant-of
  supersedes: superseded-by
  superseded-by: supersedes
  alternative-to: alternative-to
  contradicts: contradicts
  combines-with: combines-with

source_roles:
  origin: First paper to propose the idea (as far as the Atlas knows).
  extends: Improves or generalizes the idea.
  replicates: Independently confirms the idea's main claim.
  contradicts: Provides evidence against the idea's claim.
  applies: Uses the idea in a new setting and reports results.
  analyzes: Studies why or when the idea works.

evidence_grades:
  A: Several model families or scales, strong baselines, ablations, variance reported, code released.
  B: Solid results with ablations in one main setting; reasonable baselines.
  C: Narrow experiments, weak baselines, small gains without variance, or heavy reliance on LLM judges.
  D: Conceptual, position or anecdotal; no real experiments.
```
