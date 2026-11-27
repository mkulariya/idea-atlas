---
id: AG-20261127-10
title: No AI review effort level rejects most wrong patches; medium suffices only for Opus
tldr: Sweeping reviewer reasoning effort on 90 agent-written patches, higher effort caught more wrong patches for all three models, but even the best setting accepted at least 45%.
section: agents
also_in: []
subtopic: coding-agents
type: finding
tags: [coding-agents, verifier, agent-evaluation, cost-reduction]
aliases: [plausible-but-wrong patches, review effort plateau, reviewer false success rate]
status: emerging
scores: {actionability: 4, evidence: 4, generality: 3, leverage: 3, novelty: 3, surprise: 3}
power: 66
evidence: B
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.33639', title: 'How Much Effort Does an AI Code Reviewer Need? Evidence from Claude 5.5 Models on Plausible-but-Wrong Patches', role: origin}
related: []
superseded_by: null
---

## Summary

Coding agents write patches faster than people can review them, so teams insert AI reviewers and must choose a reasoning-effort setting that trades cost for diligence. The authors collected 90 agent-written Django patches from SWE-bench Verified; strengthened SWE-ABS tests label 55 as plausible-but-wrong and 35 as correct, and three Claude 5.5 models reviewed each patch at low, medium and high effort with tools and tests available. False success fell with effort for all three models: Opus .67 to .45, Sonnet .65 to .49, Haiku .78 to .51 from low to high. For Opus, preregistered hypotheses held on accuracy but not on cost: medium and high disagreed on only 3 of 55 wrong patches (difference .018, 95% CI [-.065, .108], inside the 15-point margin), while medium cost only 20% less than high, short of the required 30%. Sonnet and Haiku were cheaper at medium (39% and 51% less than high) but were not non-inferior: they disagreed with high on 10 and 15 wrong patches, mostly accepted by medium. No setting is a usable last gate; under the released 16.7% wrong-patch rate, accepted patches were 91.2% correct at best, and if half the patches were wrong, only 67.5%. Exploratory coding found 32 of the 55 wrong patches tested behavior the issue did not require, and without them no accuracy hypothesis holds, so part of "false success" is label noise the effort dial cannot fix.

## Detail

### Problem

Coding agents produce patches faster than people can read them, so teams add AI reviewers. Reasoning models expose an "effort" setting that trades cost for diligence. The question is how much effort a reviewer needs to catch patches that pass the benchmark's own tests but are wrong by stronger tests (plausible-but-wrong patches). If medium effort matches high at lower cost, that is a free saving; if not, cheap settings leave a hole in the gate.

### Mechanism

**Plausible-but-wrong patch.** A patch that passes the tests shipped with a benchmark such as SWE-bench Verified but fails stronger tests (here SWE-ABS tests). **False success** is the share of wrong patches the reviewer accepts; **false alarm** the share of correct patches it rejects. **Balanced agreement** is the average of the reviewer's accuracy on wrong and correct patches (Youden-style).

**Preregistration.** The Opus hypotheses were filed before the data were collected; Sonnet and Haiku analyses are exploratory. **Non-inferiority.** Medium counts as matching high if its false-success rate is within a stated margin (15 percentage points) of high.

1. Take 90 patches written by agents for Django issues in SWE-bench Verified. By strengthened SWE-ABS tests, 55 are wrong and 35 correct.
2. Each of three models (Claude Opus 5.5, Sonnet 5.5, Haiku 5.5) reviews every patch at low, medium and high effort, giving 270 reviews per model. The reviewer reads the issue and patch, may think, read code, run commands and tests, and then returns a verdict.
3. Test three preregistered hypotheses for Opus: H1 high effort has lower false success than low; H2 medium is non-inferior to high within 15 points; H3 medium costs at least 30% less than high (cost ratio below 0.70). Guardrails limit false alarms.
4. Exploratory coding of reviews explains why patches were accepted or rejected.

### Why it works

The authors do not give a mechanism beyond coding of rejections: most rejections named the tested defect, and more effort added such rejections. They also find that for 32 of the 55 wrong patches the issue may not require what the failing test checks; reviewers accept many of those because, read against the issue, the patch looks fine. My read: this means a large part of "false success" is label noise or underspecified issues, and the effort setting cannot fix that.

### Evidence

**False success (low vs high effort).** Opus .67 vs .45; Sonnet .65 vs .49; Haiku .78 vs .51. The difference is supported for Opus and Haiku (Sonnet's one-sided exact test gives p = .0176, not significant after Holm correction; it is significant with a mid-p variant).

**Opus, medium vs high (preregistered).** Medium and high disagreed on only 3 of 55 wrong patches; the false-success difference is .018 (95% CI [-.065, .108]), inside the 15-point margin, so H2 is supported. Medium cost 20% less (ratio .798), short of the preregistered 30%, so H3 is not supported.

**Sonnet and Haiku (exploratory).** Medium cost 39% (ratio .614) and 51% (.487) less than high, but medium and high disagreed on 10 and 15 wrong patches, mostly accepted by medium; non-inferiority was not shown (intervals reach .234 and .269).

**Balanced agreement (low, medium, high).** Opus .635, .735, .744; Sonnet .644, .657, .712; Haiku .581, .582, .688. For Opus, low to medium cost $0.051 more per review and caught 11 more wrong patches; medium to high cost $0.026 more and caught 1 more. Haiku at high cost about 8% as much per review as Opus at medium, with the accuracy difference unresolved.

**Gate quality.** If 16.7% of incoming patches are wrong (the released SWE-ABS rate for Django), 83.3% of patches are correct with no reviewer. Among accepted patches, Opus gives 87.5%, 90.9% and 91.2% correct at low, medium and high; if half the patches are wrong, 58.4%, 66.6% and 67.5%.

**Issue requirement (Table 6, exploratory).** Of 55 wrong patches, 32 were coded as testing behavior the issue does not require; at Opus low effort, 24 of those 32 (.75) were accepted. Without the 32, neither accuracy hypothesis holds for any model.

### Limitations and open questions

Only 90 patches from Django, produced by agents, with 55 wrong and 35 correct; three models from one family; single run per setting. Labels come from strengthened tests that may themselves be wrong or over-strict (the 32 patches). Post hoc coding of reasons used language-model coders. Sonnet and Haiku analyses were exploratory and some used mid-p corrections. The medium-effort result for Opus has wide intervals (up to about .108 over high).

My read: the strongest result is that no setting is a strong gate (each accepted at least 45% of wrong patches), and the cheap-effort conclusion for Opus cannot be assumed for other models.

### How to apply

Use it when you set review effort for an AI reviewer.

1. Build a small labeled set from your own repository: 30-50 patches that pass your CI but were later reverted or fixed, plus correct ones.
2. Run each model at each effort level; record false success, false alarm, cost and wall time.
3. Choose the cheapest setting within a margin you accept (15 points in the paper), and check the cost per extra catch.
4. Do not treat the reviewer as the last gate; the best setting still accepted at least 45% of wrong patches.

Pitfall: labels you generate with stronger tests can penalize behavior the issue never asked for. **Smallest experiment:** 40 patches, three effort levels, one model; compare false success and cost between low and medium.

Assessment: Watch. A careful study with a clear caution that effort settings differ by model, but small and limited to one repository and model family.

### Connections

This is an empirical study, not a method. It contributes a preregistered design, a cost-aware framing (cost per extra catch) and the observation that the effort-accuracy curve differs by model. The setup builds on SWE-ABS strengthened tests to define plausible-but-wrong patches; the paper compares effort levels within one model family.

**Connections.** Idea 1 and this idea share the problem that a test or grader can demand more than the issue states; here 32 of 55 "wrong" patches are in that group. Idea 6 and idea 7 propose adding checks or routing to cheaper models; this paper gives a cost per extra catch, which could be used to set routing thresholds. Idea 3 considers verifiers that approve work without evidence, which is the failure that low effort shows.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [How Much Effort Does an AI Code Reviewer Need? Evidence from Claude 5.5 Models on Plausible-but-Wrong Patches](https://arxiv.org/abs/2611.33639) — On 90 agent-written Django patches (55 wrong under SWE-ABS), raising reviewer effort cut false success (Opus .67 to .45, Sonnet .65 to .49, Haiku .78 to .51) but the best setting still accepted at least 45% of wrong patches, and medium matched high only for Opus, at 20% lower cost rather than the preregistered 30%.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: created from bulletin 2026-11-27 (#10).
