---
id: ENV-20261126-01
title: Tools that read hidden simulator state silently solve the search benchmarks claim to test
tldr: 'If a tool can read simulator state the agent never perceived, benchmarks measure the tool author''s shortcut: enforcing evidence closure cuts mid-sized Minecraft agents'' diamond rate from 76%–94% to 32%–40%.'
section: rl-envs
also_in: []
subtopic: env-construction
type: finding
tags: [information-flow, benchmark-quality, harness-coupling, agent-evaluation, privileged-information]
aliases: [evidence closure, EviCraft, grounded vs loaded-state regime, read-site regime audit]
status: emerging
scores: {novelty: 5, leverage: 4, evidence: 4, generality: 3, actionability: 5, surprise: 4}
power: 84
evidence: B
first_seen: '2026-11-26'
last_updated: '2026-11-26'
sources:
- {arxiv: '2611.32765', title: 'EviCraft: Can Minecraft Agents Find What They Cannot See?', role: origin}
related:
- {id: ENV-20261123-10, rel: alternative-to}
- {id: PT-20261126-05, rel: combines-with}
superseded_by: null
---

## Summary

Agent environments often expose high-level tools such as mineBlock(diamond_ore), where a handwritten routine inside the tool does the work; if that routine reads the simulator's full state it can find a buried ore the agent never saw, so the score rewards the tool author rather than the agent's search. The authors formalize evidence closure: every decision of an agent system, including target or route choices inside tools, may depend on the world only through facts for which the agent holds evidence, so two worlds with the same evidence history must yield the same decision distribution. EviCraft enforces this below the agent: a Minecraft runtime owns the world, clock and knowledge state, and the agent's protocol has no method for reading beyond admitted evidence, whatever its prompts or code. The admission rule is a per-read-site regime — perception-grounded (G, line of sight within 64 blocks) or loaded-state (L, standard tool-stack access) — giving a configurable triple over what is pushed to the model, what queries return, and what tools read internally. With six models in one program-writing reference agent on Survival, the share of runs reaching a diamond was 32%–40% under G versus 76%–94% under L for three mid-sized agents, and the strongest agent under G scored below each mid-sized agent under L. Site ablations in two models show removing only the line-of-sight test closes 93%–96% of the score gap and loading state only inside tools recovers 70%–98%, so safeguards on observations and queries leave most of the leak open. Use it when you build or adopt an environment with high-level tools for evaluation or RL: enumerate read sites, run matched worlds under two regimes with a bootstrap interval, then switch one site at a time to locate the privileged information. Caveats: the model comparison uses one reference harness, Mineflayer-based agents cannot run unchanged, G is idealized (all-around sight, exact labels), the isolated-stage ablations use one world and two agents, intervals over 10 worlds are probably wide, and whether RL training under G produces better searchers than training under L is not tested.

## Detail

### Problem

Many agent benchmarks and RL environments give the agent high-level tools such as `mineBlock(diamond_ore)`. The agent writes one call, and a handwritten routine inside the tool does the work. If that routine reads the simulator's full state, the call finds a buried ore the agent has never seen. The score then rewards the tool author's shortcut, not the agent's ability to look for things.

This matters in two ways. Comparisons between models become misleading, because the interface can matter more than the model. And if you train with RL there, the policy learns to rely on a channel that will not exist with real perception or real APIs, so the skill you wanted, deciding where to look next, never gets trained.

### Mechanism

**Harness / tool stack.** The software around a model that exposes actions (here, "abilities" called from short programs the model writes).

**Loaded chunks and line of sight.** Minecraft stores the world in chunks. A tool that queries loaded chunks can see blocks that are underground. A line-of-sight test asks whether a straight ray from the agent's eye reaches a block unobstructed.

**Read sites.** The authors note that world facts can reach an agent system at three places: what the harness pushes into the model's input, what the model asks for through queries, and what tools read while they run. Existing safeguards act on the first (and in one prior system the second).

The authors formalize **evidence closure**: every decision of an agent system, including the choice of target or route inside a tool, may depend on the world only through facts for which the agent holds evidence. Equivalently, two worlds with the same evidence history must give the same distribution over decisions.

EviCraft is a Minecraft runtime that enforces this *below* the agent. The runtime owns the world, the clock and a knowledge state. The agent connects through a protocol that has no method for reading the world beyond admitted evidence, so closure holds for any agent confined to it, whatever its prompts or code.

The rule that admits facts is an experimental variable called a regime:

1. **Perception-grounded regime (G).** A fact is admitted only if it is visible from the agent's eye or returned by the agent's own action. Rays are cast over the full sphere, admit the first occluding surface within 64 blocks, and recognition is exact. While walking, G records 97.9% (surface) and 99.7% (caves) of the blocks that a denser reference sensor sees within 32 blocks.
2. **Loaded-state regime (L).** Reproduces the loaded-chunk access of standard tool stacks.

Each read site can carry its own regime, so a configuration is a triple $(\rho_{push},\rho_{query},\rho_{tool})$. Setting all three to G is closure. $(G,G,L)$ grounds inputs and queries but leaves tools reading loaded state. Changing the regime at all sites with agent, tools and world fixed measures the value of privileged information; changing one site locates where it enters.

Tasks: **Survival** (22 milestones worth 10 to 300 points, 72,000 game ticks) and **Discovery** (distinct item types, 144,000 ticks). Each task uses 10 worlds and 5 repetitions per model, world and regime; comparisons are matched-world differences with a 95% bootstrap interval over worlds.

### Why it works

The authors' explanation is that agents under G stall at occluded resources. Under G the agent's skill library says at which depths ores generate and prescribes branch mining for diamonds, and the agents follow it, digging blindly several times more often and travelling farther than under L. The loaded-state tool bypasses exactly this problem: it removes the need to decide where to look.

The authors also derive four testable implications of this hidden-state account (for example, the loss under G should concentrate at milestones whose target is occluded, and a doubled budget should leave most of it) and report results consistent with them.

My read: the strongest evidence that the effect is about information and not some other difference between regimes is the site-level result, where removing only the line-of-sight test accounts for almost all the gap.

### Evidence

**Setup.** Six models in one reference agent that writes programs calling abilities: DeepSeek v4.1 Flash, Qwen 3.8 Flash, GLM 5.3 Flash, Qwen 3.8 27B, Qwen 3.5 9B and Kimi K3.1 Pro. Minecraft 1.21.1, survival mode, easy difficulty.

The authors report (Survival):

| Quantity | Under G (grounded) | Under L (loaded state) |
|---|---|---|
| Diamond reached, three mid-sized agents | 32%–40% of runs | 76%–94% of runs |

The strongest agent under G scores below each mid-sized agent under L.

Site ablations in two models: removing only the line-of-sight test closes 93%–96% of the score gap; loading state only inside tools recovers 70%–98%. The authors conclude that safeguards on observations and queries leave most of the leak open. In Discovery, the gain from L concentrates in items that depend on buried resources or mobs. Range, retention and budget each leave most of the gap (from the discussion section).

### Limitations and open questions

The authors state that the model comparison uses one program-writing reference harness, without the curriculum and growing skill library of Voyager or the planner-actor-critic split of Odyssey; Mineflayer-based agents cannot run unchanged because their helpers read loaded chunks; other control designs may search better under G; and the isolated-stage experiments use one world and two agents. G is idealized (all-around sight, exact labels, no lighting), though the one idealization they tested, a camera field of view, widened the gap.

My read: with 10 worlds the intervals are probably wide for some comparisons. The finding generalizes in principle, but "mid-sized agents reach the diamond in 32%–40%" is specific to this harness and these models. Whether RL training under G produces better searchers than training under L is not tested.

### How to apply

**When:** you build or adopt an environment with high-level tools, for evaluation or RL.

1. List every read site: what is pushed to the model, what queries return, and what each tool reads internally.
2. Move the world behind a runtime that exposes only an evidence interface (what the agent has perceived or been told by its own actions). Make admission a config flag.
3. Run the same agents under two regimes and report the matched-world difference with a bootstrap interval over worlds.
4. Localize: switch one site at a time to see where the privileged information enters.

**Pitfall:** exact recognition favors G; add a counterfactual test (two worlds, same evidence, same decisions).

**Smallest experiment:** take one tool in your environment that resolves a named target (find, open, mine). Add a visibility check to it and rerun 20 episodes of your current agent. If success drops sharply, your benchmark was measuring the tool.

Assessment: Adopt now. Cheap to audit, large measured effect (32%–40% vs 76%–94%), and the audit transfers to any tool-using environment.

### Connections

Prior Minecraft agent work (Voyager-style skill libraries and semantic tool stacks, and MP5 in the paper's related work) filtered what is pushed into the model's input, and MP5 also filtered queries. The new element is to treat the *tool* as part of the policy and to close that site too, by moving enforcement into the runtime protocol and making the admission rule an experimental variable with matched worlds.

**Connections.** Idea 5 (RLVB) uses simulator privileged information to compute a training reward; this idea shows the same kind of information is harmful when it reaches the agent's tools, so the two together suggest "privileged for the reward, never for the policy". Idea 10 (RecurSciBench) faces the analogous question of whether a measured gain reflects the agent or the setup. Idea 4 (recovery harm) is another case where the tool boundary, not the model, determines outcomes.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-26 · origin · [EviCraft: Can Minecraft Agents Find What They Cannot See?](https://arxiv.org/abs/2611.32765) — With six models in one program-writing Minecraft agent, grounding all observations, queries and tool reads cut diamond-reaching runs from 76%–94% to 32%–40%, and removing only the line-of-sight test closed 93%–96% of the score gap in two ablations.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-26: +relation combines-with PT-20261126-05.
- 2026-11-26: created from bulletin 2026-11-26 (#1).
