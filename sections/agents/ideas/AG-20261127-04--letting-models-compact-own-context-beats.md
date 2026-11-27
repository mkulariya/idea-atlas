---
id: AG-20261127-04
title: Letting models compact their own context beats harness thresholds at tight budgets
tldr: Giving agents a collapse-and-archive context API instead of harness threshold compaction nearly triples OfficeQA accuracy at 32K for GPT, but helps less for DeepSeek and not for Opus.
section: agents
also_in: []
subtopic: memory-context
type: finding
tags: [context-folding, long-horizon-agents, harness-coupling, cost-reduction, agent-evaluation, training-free]
aliases: [model-driven compaction, collapse API, context API compaction, archive-and-retrieve context, suffix KV reuse, NOOA]
status: emerging
scores: {novelty: 3, leverage: 4, evidence: 4, generality: 3, actionability: 5, surprise: 3}
power: 73
evidence: B
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.34147', title: 'Letting Models Manage Their Own Context: Where We Stand', role: origin}
related:
- {id: AG-20261125-07, rel: variant-of}
- {id: AG-20261123-09, rel: alternative-to}
superseded_by: null
---

## Summary

Long-running agents fill their context with tool outputs, and the default harness answer is compaction: at a fixed fill threshold a model summarizes earlier history, keeping the initial instructions and the latest events, so a heuristic decides when and what to compress and key evidence can be lost. This study compares that baseline in the NOOA harness, which summarizes at 80% of budget and keeps instructions and the 10 most recent events verbatim, against a model-driven regime where the model gets a collapse call that replaces an event range with its own summary and returns a tag, a lookup that retrieves any archived original by tag, a live usage indicator, an instruction to consider compacting only after 85%, and a context-pressure warning. Budgets from 8K to 512K are swept on OfficeQA Pro v2 (90 questions) and MLE-bench Lite (22 competitions) with GPT-6.1 Sol, Kimi K3, DeepSeek V4 Pro and Claude Opus 5.5, with no training. On OfficeQA Pro v2 at a 32K budget GPT-6.1 Sol rises from 21% to 56% accuracy; gains shrink as the budget grows, are smaller for DeepSeek, are limited to the tightest budgets on MLE-bench Lite, and Opus trails the harness baseline until the budget is large. Kimi K3 at 32K uses 41% fewer cumulative input tokens (36M versus 62M) while scoring slightly higher (43% versus 41%), and the models call the API in 42-53% of tasks except Opus at 4%. Removing the 85% timing instruction moves first compaction much earlier, with Kimi dropping from about 84% to 29% of a 128K budget, and those variants score about 8 points below the baseline at budgets of 128K and above. A prototype in SGLang that reuses the KV cache of unchanged text after an edit cut prefill compute by 4-15% per request with no measurable accuracy change. Caveats: model-driven runs receive extra prompt support so the comparison is not purely who decides, the baseline is one fixed rule, the benchmarks have 90 and 22 tasks, and the KV reuse is an approximation tested on one open model.

## Detail

### Problem

Long-running agents fill their context window with tool outputs. Most harnesses handle this by compaction: when usage crosses a threshold, a model summarizes the history, keeping the initial instructions and the latest few events. The heuristic decides when and what to compress, and the model only writes the summary, so key evidence can be lost exactly when a task needs it. Long contexts also degrade model behavior ("context rot"), so a bigger window does not remove the problem.

If models can manage their own context well, an agent can complete longer tasks on smaller windows and fewer tokens. The paper asks whether current models can do this with no extra training, and what it costs to serve.

### Mechanism

**Compaction.** Replacing part of the history with a summary. **Harness-driven** means the harness triggers it; **model-driven** means the model chooses when and what.

**Context budget.** A hard cap on request size in tokens, used here to simulate tasks that are long relative to the window; if a request exceeds the cap a fallback drops the oldest events and retries.

**KV cache and prefix caching.** An inference engine stores the attention keys and values (the KV cache) for a prompt so it need not recompute them. Standard prefix caching reuses the cache only up to the first changed token, so editing the middle of the history forces recomputation of everything after it.

The study uses NOOA, NVIDIA's open-source agent harness, and compares two regimes.

1. **Harness-driven (baseline).** At 80% of the context budget the harness summarizes earlier history; the initial instructions and the 10 most recent events stay verbatim.
2. **Model-driven.** The model gets a context API with a `collapse(start, end, summary_text)` call that replaces a range of events with its own summary and returns a tag, and a lookup that retrieves any archived original by its tag (archived, not deleted). It also sees a live context-usage indicator, an instruction to consider compacting only once usage reaches 85%, and a context-pressure warning near the limit.
3. **Sweep.** Budgets from 8K to 512K tokens on two benchmarks, with four models.
4. **Serving experiment.** A prototype in the SGLang engine reuses the KV cache of the unchanged text after an edit (suffix reuse) rather than recomputing it. This is an approximation, since the later text was originally computed with different earlier context.

The setting changes only who decides when and what to compress; no training is involved.

### Why it works

The authors give no causal proof. Their argument: a model that knows what the task still needs can keep the relevant evidence and compress the rest, whereas a fixed rule treats all history alike; and the gains concentrate at tight budgets, where an agent must manage history repeatedly. Differences between models are attributed to training: they conjecture GPT's stronger results may reflect training for context management, and note that Kimi K3 is trained with RL penalties on long rollouts, which could explain why it compacts early. My read: "context anxiety", compacting while space remains, is the visible cost of that, and suggests that compaction behavior should be trained or at least tuned per model.

### Evidence

**Setup.** OfficeQA Pro v2 (90 questions on US federal receipts and expenditures, via search over a parsed corpus) and MLE-bench Lite (22 Kaggle competitions). Models: GPT-6.1 Sol, Kimi K3, DeepSeek V4 Pro, Claude Opus 5.5.

**Accuracy.** On OfficeQA Pro v2 at a 32K budget, GPT-6.1 Sol rises from 21% to 56% (nearly 3×). Gains shrink as the budget grows, are smaller for DeepSeek, and are limited to the tightest budgets on MLE-bench Lite. Opus trails the harness baseline until the budget is large.

**Tokens.** Kimi K3 at a 32K budget uses 41% fewer cumulative input tokens (36M vs 62M) and scores slightly higher (43% vs 41%); at 64K it uses about 11% fewer and scores 57% vs 52%.

**Behavior.**
- GPT, Kimi and DeepSeek call the API in 42-53% of tasks; Opus edits its context in 4% of tasks.
- Median shrink per compaction: GPT 3.7×, Kimi 2.1×, DeepSeek 1.8×, Opus 1.5×. GPT retrieves archived content again after 19% of compactions, Kimi 7%, DeepSeek 3%, Opus 14% (a lower bound).
- Removing the 85% timing instruction moves the median first compaction for Kimi from about 84% to 29% of the budget at 128K, DeepSeek from 83% to 40% at 256K, and Opus compacts at a median of 16% at 128K. At budgets of 128K and above these variants score about 8 points below the harness baseline for Kimi and DeepSeek.

**Serving.** Suffix KV reuse with Qwen3.6-27B cut prefill compute by 4-15% per request versus prefix caching on MLE-bench Lite, OfficeQA Pro and DeepSWE, with no measurable accuracy change; the largest savings are on OfficeQA Pro, where about 10K tokens (roughly 60% of the prompt) survive an edit.

### Limitations and open questions

The authors state this is where things stand, not a method. Gains are inconsistent: DeepSeek improves less, and Opus rarely uses the API and trails at small budgets. Model-driven runs get extra prompt support (usage indicator, 85% instruction, pressure warning), so the comparison is not purely "who decides". The baseline is one fixed rule (80% threshold, last 10 events). The two benchmarks are document research and ML engineering; both have 90 or 22 tasks, so some differences may be noise. Whether GPT was trained for this is a conjecture. The KV reuse prototype is approximate and tested on one open model.

My read: "tight budget" results are partly a simulation of long tasks; actual long-horizon gains are not shown.

### How to apply

Reach for it if your agent hits its context limit on long tasks, especially with thresholded compaction.

1. Add a `collapse` tool that replaces an event range with a summary and archives the originals under a tag, and a `retrieve` tool.
2. Show the model its context usage and tell it when compaction is allowed. Treat the timing instruction as a per-model setting.
3. Measure per model: share of tasks that use the API, shrink factor, and re-open rate.
4. Compare against your current thresholded compaction at 2-3 budgets below your usual window.

Pitfalls: some models compact too early (cost about 8 points at large budgets in this study) or almost never. **Smallest experiment:** run one benchmark of your own at a 32K cap with and without the API and look at score and cumulative input tokens.

Assessment: Try in an experiment. Cheap to prototype and large gains at tight budgets for some models, but behavior varies a lot between models.

### Connections

Compaction by the harness is the default in coding agents. Related work cited by the authors includes SideQuest, Context Language Models and Cache Engineering for Agents, which explore model-driven context and suffix KV reuse, and Codex's experimental context-management mode. This paper's addition is a broad measurement: four frontier models, two benchmarks, budgets over 8K-512K, plus behavior metrics (usage rate, shrink factor, re-open rate, timing) and a serving check. It is an empirical report, not a new algorithm.

**Connections.** Idea 7 (partial routing) credits its gain to the same mechanism: workers get fresh contexts and return compact reports, so the lead carries less history. Idea 8 and idea 11 show the same pattern in another form, where whether agents use an available capability depends on the harness. Idea 6 (TransHarness) evolves harness behavior; compaction timing is a natural target for such evolution.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [Letting Models Manage Their Own Context: Where We Stand](https://arxiv.org/abs/2611.34147) — Model-driven collapse and archive beat harness compaction at tight budgets for some models (OfficeQA Pro v2 at 32K: GPT-6.1 Sol 21% to 56%; Kimi 41% fewer input tokens, 43% vs 41%) but not for Opus, and suffix KV reuse cut prefill 4-15% per request.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: created from bulletin 2026-11-27 (#4).
