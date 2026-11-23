---
id: AG-20261123-02
title: Which links a web agent clicks leaks its private context past content-level defenses
tldr: A stateful attacker placing one link on a visited page reconstructs a web agent's private query (F1 0.89) and profile (F1 0.80) from which links it clicks; content filters do not help.
section: agents
also_in: []
subtopic: agent-safety-security
type: finding
tags: [agent-safety, privacy-leakage, covert-channel, prompt-injection, web-agents, information-flow, browsing-agents]
aliases: [Snoop, Snoop Tree, Selection-Obfuscated Retrieval, SOR, behavioral covert channel]
status: emerging
scores: {novelty: 5, leverage: 4, evidence: 4, generality: 4, actionability: 4, surprise: 4}
power: 85
evidence: B
first_seen: '2026-11-23'
last_updated: '2026-11-23'
sources:
- {arxiv: '2611.27046', title: 'Actions Speak Louder than Words: Stateful Privacy Sniffing of LLM Agents via Behavioral Covert Channels in Web Interactions', role: origin}
related: []
superseded_by: null
---

## Summary

Web agents read the user's query, history and memory before deciding what to search and which links to open, so each navigation choice is a noisy function of private context. Current agent-security work covers passive traffic analysis (coarse topics only) and prompt-injection exfiltration (which content filters increasingly catch), but not the semantics of benign choices. Snoop exploits this: one ordinary-looking reference link brings the agent to an attacker site that serves pages of semantically distinct candidate links, and the agent's click selects the next hypothesis in a coarse-to-fine tree covering domain, entity, attribute and task expansion, with state kept across interrupted visits. The attack needs no privileged observer and no instruction-following, and it reconstructs both the query and, across turns, a user profile. Averaged over four agents times three backbones, induction rate is 0.94 (Stack) and 0.83 (iKAT), query concept F1 is 0.89 and 0.79, and profile F1 is 0.80 with about six probes per query. Spotlighting, tool firewalls, Task Shield and Fides each leave F1 at roughly 0.77-0.88, while Selection-Obfuscated Retrieval (prefetch every link, choose among local copies, batch new fetches with decoys) drops F1 to 0.39 at 4.3-4.5x page fetches. Use it to threat-model link following when a browsing agent holds memory, documents or conversation history. The main caveats are that the attacker must get one link onto a page the agent reads and that SOR still leaves F1 near 0.39 at a large fetching cost.

## Detail

### Problem

Web and deep-research agents read the user's prompt, history, memory and connected data, then decide what to search and which links to open. Current agent-security work focuses on two threats: passive traffic analysis by a network observer (which yields only coarse topics) and prompt injection that tricks the agent into writing secrets into an outgoing request (which content-level defenses increasingly catch; the paper's comparison finds explicit exfiltration succeeding 0.38% of the time).

This paper shows a third channel that existing defenses are not designed for: the agent's *benign choices*. If a page offers four plausible references and the agent picks the one about "immune response" rather than "clinical outcomes", the request itself reveals something about the private task. Nothing malicious is written, no instruction is obeyed, and every request is task-relevant, so spotlighting, tool firewalls, task-alignment checks and information-flow control all let it through. Any agent with browsing plus personal memory has this exposure today.

### Mechanism

- **Covert channel:** a way to transmit information through something not meant for communication (here, which URL is requested, in which order, how often, and when).
- **Indirect prompt injection:** instructions hidden in content the agent reads (a web page, an email) that try to make it act for the attacker. Snoop does not need it.
- **Hypothesis tree:** a tree of guesses about the private context, from broad (domain: "health") to specific (attribute: "work-related stress"), refined one level per probe.
- **Concept F1:** the overlap between concepts the attacker recovers and the ground-truth concepts of the user's question (entities linked to Wikidata), combining precision and recall.
- **Induction rate:** the share of tasks where the agent follows the injected link to the attacker's site at all.

**Threat model.** The attacker can host a website and log requests (endpoint, count, order, timing), and can place a generic reference link or pop-up on some page the agent will read during normal work. It knows nothing about the query, cannot modify the agent, and sees no other traffic.

**Channels.** The paper characterizes six behavioral channels: presence (1 bit), identity of the chosen link ($\log_2 k$ bits for $k$ candidates), frequency, order ($\log_2 m!$ for $m$ ordered resources), timing and size. Choice-type channels are the most reliable. One-shot probing degrades fast: accuracy falls from 93.8% to 50% as candidate width grows, and at depth four progressive probing keeps 49% accuracy against 14% for flat probing.

**Snoop loop:**
1. An injected link ("For additional references, see …") brings the agent to the probing site.
2. The *Snoop Tree* starts from a fixed taxonomy of top-level domains and probes them in batches until a click confirms one.
3. The *Adaptive Probe Constructor* picks an operator (Domain, Entity, Attribute-Descriptive, Attribute-Quantitative, Task Expansion) and retrieves semantically distinct candidates from knowledge bases (MeSH, FIBO, OpenAlex and others), then renders a page whose links map each candidate to its own endpoint.
4. The agent's next GET reveals its choice; that node becomes the current hypothesis and the next probe refines it.
5. The *Snoop Agenda* keeps state across interrupted visits and retries failed probes.
6. Confirmed paths are aggregated into a reconstructed query and, across turns, a user profile.

**Defense (Selection-Obfuscated Retrieval, SOR):** (i) fetch *all* resources linked from an external page regardless of the private context and cache them; (ii) let the agent choose among local copies, so its selection produces no external request; (iii) when it needs something outside the cache, fetch it in randomized batches mixed with independently sampled decoys.

### Why it works

The authors' explanation: agents run an observe-reason-act loop in which retrieved content and private context *jointly* decide the next web action, so each action is a noisy function of the private context. Offering candidates that differ in exactly one semantic dimension turns each click into a near-clean answer to one question about the user. Stateful, coarse-to-fine probing keeps each question small (few candidates), which is where choice channels are reliable.

Content defenses fail because they inspect *what* is sent and whether the action fits the task. Here every request is a legitimate, task-relevant click; the leak is in *which* legitimate click was made.

Their controls show the reconstruction really comes from observed behavior: replacing each observed choice with a random candidate drops F1 to 0.10/0.09 (Stack/iKAT), and reconstructing from domain priors alone gives 0.13/0.12, against 0.81/0.80 for full Snoop in that ablation setting. SOR works by breaking the link between choice and observable request: with all links prefetched, the attacker sees many requests that do not reflect the agent's needs, so precision collapses while recall stays high.

### Evidence

**Setup.** Four agents (Codex, OpenHands, IterResearch, Open Deep Research) times three backbones (GPT-5.5, Claude Sonnet 4.6, Gemini 3.6 Flash) = 12 configurations; 500 StackExchange questions (27 topics) and 378 TREC iKAT conversational turns from 26 personas with profile ground truth; a local snapshot of 42K HTML pages from 3.8K domains with the same generic link injected on 2.6K pages.

**Attack (Table IV, averaged):** induction rate 0.94 (Stack) and 0.83 (iKAT); query concept F1 0.89 and 0.79; profile F1 0.80. About 6.48 probes per Stack query and 5.91 per iKAT turn; four probing levels capture 99% and 96% of five-level recovery.

**Existing defenses (Table VII):**

| Defense | Stack F1 | iKAT query F1 | iKAT profile F1 |
|---|---|---|---|
| None | 0.89 | 0.79 | 0.80 |
| Spotlighting | 0.86 | 0.78 | 0.77 |
| Firewalls | 0.88 | 0.78 | 0.77 |
| Task Shield | 0.88 | 0.76 | 0.77 |
| Fides | 0.87 | 0.77 | 0.77 |

**SOR (Tables VIII-IX):** Stack F1 0.89 → 0.39 (precision 0.87 → 0.29), iKAT profile F1 0.80 → 0.39; page fetches ×4.3-4.5, LLM tokens ×1.1-1.3, time ×1.3.

### Limitations and open questions

- The attacker must get one link onto a page the agent reads and the agent must follow it (induction 0.83-0.94 here); agents with a strict policy of not following unknown references would blunt the attack.
- Evaluation runs on a frozen snapshot, not the live web, and the same injected sentence is used on all pages.
- SOR still leaves F1 around 0.39 and costs 4.3-4.5 times more page fetches.

My read: the important takeaway is architectural rather than about Snoop specifically. Any agent that both holds private context and makes observable choices on untrusted sites leaks through those choices; content filters cannot fix that.

### How to apply

**When:** you run browsing agents with access to user memory, documents or conversation history, especially in enterprise or health/finance settings.

**Steps:**
1. Threat-model link following: treat "which outbound URL" as data that leaves your system.
2. Restrict following links from untrusted pages to third-party domains not in the original search results; log and rate-limit deep navigation chains on a single unknown site.
3. Where privacy matters, implement SOR-style retrieval: prefetch all links from a page into a local cache, let the agent choose locally, and fetch new resources in shuffled batches with decoys.

**Smallest experiment:** stand up a test site with a four-way link page whose options differ in one attribute of a known synthetic user profile, inject a reference link into a page your agent visits, and run 50 tasks with seeded private profiles. Predicting the hidden attribute well above 25% means you have the channel.

Assessment: Adopt now. Treat agent navigation choices as a privacy channel now; the attack is cheap, evades current defenses in a 12-configuration study, and the mitigation is implementable at the retrieval layer.

### Connections

Closest prior work: website-fingerprinting and traffic-analysis attacks on agents (coarse topic inference from a privileged network vantage point) and indirect-prompt-injection exfiltration (the agent is tricked into sending secrets).

What is new: an *active, stateful* attack that needs no privileged observer and no instruction-following, using only the semantics of benign navigation choices, plus a retrieval-layer defense that decouples selection from observable fetches.

**Connections.** Idea 1 makes agents better at picking the right element; this idea is the flip side: any selection policy that tracks the user's intent is, by design, informative to whoever controls the options. Idea 9's agents browse heavily for forecasting and would be exposed the same way.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-23 · origin · [Actions Speak Louder than Words: Stateful Privacy Sniffing of LLM Agents via Behavioral Covert Channels in Web Interactions](https://arxiv.org/abs/2611.27046) — Snoop reconstructs private queries (concept F1 0.89 Stack, 0.79 iKAT) and profiles (F1 0.80) from which links 12 agent-backbone configs click, with spotlighting, firewalls, Task Shield and Fides all staying near 0.86-0.88/0.76-0.78, while prefetch-based SOR cuts F1 to 0.39 at 4.3-4.5x page fetches.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-23: created from bulletin 2026-11-23 (#2).
