---
id: ENV-20261123-08
title: What users later do with an agent's output is a refreshable, hard-to-game verifier
tldr: Labeling agent output as useful when the person receiving it later acts on it gives a refreshable verifier whose precision rankings match a gold set (rho 0.69) and resist gaming.
section: rl-envs
also_in: []
subtopic: verifiers-rewards
type: method
tags: [verifier, agent-evaluation, benchmark-quality, reward-shaping, action-grounding]
aliases: [CodeReviewBench, revealed-preference verifier, developer-action label, implicit-feedback verifier]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 3, actionability: 3, surprise: 4}
power: 75
evidence: B
first_seen: '2026-11-23'
last_updated: '2026-11-23'
sources:
- {arxiv: '2611.26942', title: 'CodeReviewBench: An Automated Benchmark for Evaluating the Software Factory', role: origin}
related:
- {id: ENV-20261123-07, rel: alternative-to}
superseded_by: null
---

## Summary

Open-ended agent outputs such as code reviews, research notes or support replies have no unit test, and current verifiers are small expert-labelled gold sets or LLM judges that are expensive, static, leak into training data and share the blind spots of the models they grade. CodeReviewBench instead labels each AI code-review comment as useful if the pull-request author later acts on it, mining this daily from public GitHub activity. A daily pipeline discovers merged PRs where a known AI review tool commented (4.1M discovered, about 900K fully enriched across about 190K repositories), an LLM judge extracts the concerns the author addressed and matches bot suggestions to those actions, and six cumulative filters drop PRs where silence is uninformative before macro-averaged precision, recall and F-beta are scored per tool. Across the 18 tools with enough volume, the resulting precision ranking agrees with an independent expert-labelled offline benchmark on 11 overlapping tools (Spearman rho = 0.69, bootstrap 95% CI [+0.12, +0.94]) and tracks human expert accuracy (rho = 0.688), while F1 rankings do not transfer (rho = -0.05) because online recall is measured only against the issues developers chose to fix. The signal also detects real product change: eight significant level shifts across six tools, five aligned with public vendor changelogs, some within a day. Use it when you deploy an agent whose output a person or system later accepts, edits, merges or ignores and you need an eval or reward without gold labels, and report precision-like metrics only. The main caveats are that the authors include vendors of evaluated tools, the validation rests on 11 tools with a wide interval, more than half of bot-reviewed merged PRs show zero human activity so heavy filtering (41% retained) shapes what is measured, and as a training reward it invites feedback loops if the agent can influence whether users act.

## Detail

### Problem

Open-ended agent outputs such as code reviews, research notes or support replies have no unit test. Current verifiers are small expert-labelled gold sets or LLM judges. Gold sets are expensive, static (they leak into training data and saturate), incomplete, and in code review, every vendor tends to win its own. LLM judges share the blind spots of the models they grade.

This paper uses a different source of truth: **revealed preference**, meaning what the person receiving the output did next. That signal is produced continuously by real usage, cannot be memorised in advance, and gaming it requires changing real developer behaviour at scale. For anyone building reward signals or evals for deployed agents with logged downstream actions (merges, edits, follow-up commits, accepted suggestions), this is a validated template, with clear evidence about which metrics survive the switch from gold labels to behaviour.

### Mechanism

- **Revealed preference:** inferring value from what people do (act on a suggestion or not) rather than what they say or what an annotator marks.
- **Precision / recall for review bots:** precision = share of a bot's suggestions that the developer acted on; recall = share of developer-addressed issues that the bot had flagged. F1 combines them.
- **Gold set:** a fixed list of expert-labelled issues per PR (here an extended 50-PR, 136-issue offline set).
- **Spearman ρ:** rank correlation between two orderings of tools.
- **Fleiss / Cohen κ:** agreement between several / two raters beyond chance.
- **PELT:** a standard changepoint detector for time series.
- **DML (double machine learning):** a method to estimate effects while adjusting for observed confounders.

**Pipeline (daily):**
1. **Discover** merged PRs on public GitHub where a known AI review tool commented (4.1M discovered; about 900K fully enriched across about 190K repositories).
2. **Extract developer actions** after the review: commits and replies that address a concern. An LLM judge reads the post-review activity and lists the concerns the author acted on.
3. **Match suggestions to actions:** for each bot suggestion, the judge decides whether it raises substantially the same concern as an extracted action (partial overlap counts). Matched = true positive; unmatched suggestions = false positives; unmatched actions = false negatives.
4. **Filter for signal quality:** six cumulative filters remove PRs where silence is uninformative (e.g. the developer likely never looked); the four-filter default keeps 41% of data.
5. **Score** per-PR precision, recall and $F_\beta$, macro-averaged across PRs, for the 18 tools with enough volume.

**Validation design:**
- **Judge robustness:** three judge model families compared.
- **Human check:** experts repeat the matching decision.
- **External validity:** compare tool rankings with an independent offline gold-set benchmark (11 overlapping tools) and with human expert accuracy ratings.
- **Sensitivity:** PELT changepoints in each tool's daily score, checked against vendor changelogs.
- **Confounding:** DML adjusting for 9 observed confounders (repository and PR properties).

### Why it works

**Why precision transfers but recall does not.** When a bot flags something and the author fixes it, that is strong evidence the comment was useful. So precision based on actions measures the same thing as precision based on expert labels. But authors fix selectively: they do not act on every real issue, so "developer-addressed issues" is a subset of true issues, and online recall is measured against a smaller denominator. The paper sees this as a shifted precision/recall balance (online P/R ratio 1.53 vs 0.78 offline) and an F1 ranking that does not agree with the offline one (ρ = −0.05).

**Why it resists gaming.** The label is produced by people who are not part of the evaluation, on fresh PRs every day, so there is no fixed test set to overfit and no single judge prompt that decides the label; a tool can only score higher by making suggestions that developers actually act on.

**Why it detects real change.** A live signal shows product updates: the staggered changepoints (not synchronised across tools, so not a pipeline artefact) line up with announced releases.

My read: as an RL reward this gives a precision-like signal (penalise unhelpful output) that is well grounded, while coverage-type objectives still need a separate verifier.

### Evidence

**Judge reliability:** Fleiss κ = 0.86 across three judge families; experts agree with the automated matching on 85.6% of decisions (Cohen κ = 0.71), conditional on the extracted action list.

**Agreement with an independent benchmark (11 tools):** precision rankings ρ = 0.69 (p = 0.019; bootstrap 95% CI [+0.12, +0.94]). Human expert accuracy vs automated precision ρ = 0.688. F1 rankings do not transfer: ρ = −0.05.

**Sensitivity to real changes:** eight significant level shifts across six tools; five of eight align with public vendor changelogs, some within a day; for one tool an internal version marker reproduces the pre/post means exactly.

**Confounding:** DML adjustment for 9 confounders keeps the ranking (ρ = 0.86 with the unadjusted one).

**Scale and context:** about 500K judged PRs, 160K+ scorable; of 518K bot-reviewed merged PRs, more than half show zero human activity after the bot's review. F1 drops about 28 points from small to extra-large PRs across tools (recall collapse on large diffs).

**Gold-set weaknesses motivating the approach:** 6 golden issues caught by none of 20 tools; 41 issue clusters flagged by at least 3 tools absent from the gold set.

### Limitations and open questions

- **Conflict of interest:** authors include vendors of evaluated tools (Martian, Greptile, Augment).
- **Small validation n:** rank agreement rests on 11 tools; the CI is wide ([+0.12, +0.94]).
- **Ignored but useful suggestions count as false positives**, and public repos only; teams self-select tools.
- **LLM judge in the loop** with context limits similar to the tools on large diffs.
- **More than half of PRs get no human activity**, so heavy filtering (41% retained) shapes what is measured.

My read: strong for precision-style comparisons and drift detection; not a full measure of usefulness. As a training reward, beware feedback loops if the agent can influence whether users act (e.g. by being persuasive rather than right).

### How to apply

**When:** you deploy an agent whose output a person or system later accepts, edits, merges or ignores, and you need an eval or reward without gold labels.

**Steps:**
1. Log agent outputs with stable IDs, and log downstream actions (edits, commits, accepted suggestions, follow-up questions).
2. Use an LLM to extract the actions and match them to outputs; validate on 100-200 cases against people.
3. Filter out cases where silence is uninformative (no human activity at all).
4. Report precision-like metrics; do not trust recall/F1 from this signal.
5. Validate once against a small gold set before using it for decisions or as a reward.

**Smallest experiment:** for one feature (e.g. suggested edits), compute weekly "acted-on rate" per model version for a month and check it ranks versions the same way as your offline eval.

Assessment: Try in an experiment. A validated, refreshable precision signal for agents with logged downstream actions, though validated on 11 tools by authors who include evaluated vendors.

### Connections

Closest prior work: static code-review gold sets (50-137 issues), SWE-rebench-style refreshed benchmarks, LLM-as-judge evaluation, and implicit-feedback signals in recommender systems and search (clicks, dwell time).

New: applying revealed developer action as the primary label for agent output at this scale, with an explicit validation against an independent gold set, and the finding of *which* metric survives (precision yes, recall/F1 no).

**Connections.** Ideas 7 and 10 show static graders are often wrong; this is an alternative source of truth for tasks without tests. Idea 2 is a reminder that agent behaviour logs are also sensitive data. Idea 3 and idea 9 use model judges as rewards; here a model judge only *extracts* and matches human behaviour, which is a narrower and more checkable role.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-23 · origin · [CodeReviewBench: An Automated Benchmark for Evaluating the Software Factory](https://arxiv.org/abs/2611.26942) — Across about 500K judged PRs (160K+ scorable), action-based precision rankings agreed with an independent gold-set benchmark on 11 tools at rho = 0.69 (95% CI [+0.12, +0.94]) and with human expert accuracy at rho = 0.688, while F1 rankings did not transfer (rho = -0.05); eight changepoints across six tools, five matching vendor changelogs.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-23: created from bulletin 2026-11-23 (#8).
