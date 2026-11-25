---
id: ENV-20261125-02
title: LLM judges grow more lenient on a rule inside a larger ruleset
tldr: A judge flagging a rule violation alone often finds none when the same rule sits in a larger ruleset, leniency growing with ruleset size; per-rule forcing, consistency training and ruleset rewrite shrink it.
section: rl-envs
also_in: []
subtopic: verifiers-rewards
type: finding
tags: [verifier, grader-audit, agent-evaluation, reward-shaping, benchmark-quality]
aliases: [ruleset inconsistency rate, judge leniency under bundling, max-over-rules consistency, RuleForge, self-consistent judge training]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 4, actionability: 5, surprise: 4}
power: 82
evidence: B
first_seen: '2026-11-25'
last_updated: '2026-11-25'
sources:
- {arxiv: '2611.29975', title: Aligning Judge Models to Be Logically Consistent Multi-Rule Interpreters, role: origin}
related:
- {id: ENV-20261123-07, rel: alternative-to}
superseded_by: null
---

## Summary

Rubric rewards, constitutions and model specs usually put many rules in one judge prompt and ask for one verdict, and nothing checks whether the judge enforces each rule the way it would alone. The paper defines a judge as inconsistent when its full-ruleset verdict differs from the max over its separate per-rule verdicts under three-valued strong Kleene logic, and measures this on six judge models over 500 scenarios and 57 rules simplified from OpenAI's model spec. Across the six judges, 64.9% to 78.9% of rules showed a significant change in violation rate between same-call and separate-call judgments, almost all toward leniency, and inconsistency rose with ruleset size from roughly 10-30% at two rules to 40-80% with the full set. Forcing a per-rule verdict before the final verdict helped some models but not all; GRPO consistency training on Gemma-4-31B cut held-out inconsistency from 0.709 to 0.375 while per-rule judgments moved on only 3.7% of rules; and a minimax amender-generator ruleset rewrite (RuleForge) turned 17.8% and 21.4% of held-out scenarios from inconsistent to consistent while regressing 4.0% the other way. Use it on any single judge prompt carrying more than a handful of criteria: score critical rules alone and bundled, then either decompose into per-rule calls or add explicit priority and exemption meta-rules. The caveats are that one simplified ruleset family is used, treating per-rule verdicts as the correct ones is a design choice, and whether consistency-trained judges transfer into RL reward pipelines is untested.

## Detail

### Problem

Rubric rewards, constitutions and model specs usually put many rules in one judge prompt and ask for one verdict. If bundling softens individual rules, the reward signal you train against, or the safety check you rely on, is weaker than the rule text suggests, and nobody sees it because the verdict still looks reasonable.

The paper's example: a user asks whether it is safe to give a 10-year-old melatonin for sleep, and the AI answer says it is generally safe and to "try it for a few weeks". Judged against the single rule on regulated advice, the judge says violated. Judged against the full ruleset, the same judge says it sees no major violations, including for that rule.

### Mechanism

**Judge model.** An LLM asked whether a scenario (a user prompt plus assistant response) violates a rule or a set of rules, answering yes, no or unsure (encoded 1, 0 and 0.5).

**Ruleset / constitution / model spec.** A written list of behavioral requirements that a model is trained or evaluated against.

**GRPO.** Group Relative Policy Optimization: sample several outputs per prompt, score them, and push the policy toward those scoring above the group average. The reward here is defined below.

**Strong Kleene logic.** A three-valued logic (true, false, unknown) in which "any true means true" and "no true but some unknown means unknown". The paper uses it to define what a consistent ruleset verdict should be.

**Measuring inconsistency.** For scenario $s$ and ruleset $C=\{c_1,\dots,c_K\}$, let $Y_C(s)$ be the judge's verdict on the whole ruleset and $Y_{c_i}(s)$ its verdict on rule $c_i$ judged alone in a separate call. Violating any single rule should violate the ruleset, so a judge is inconsistent when

$$Y_C(s)\neq\max_{c_i\in C} Y_{c_i}(s).$$

The inconsistency rate is the share of scenarios where this holds. Separate-call prompts include any meta-rules the ruleset already defines.

**Setup.** Six judge models, 500 scenarios (167 from WildChat, 167 from Zhang et al., 166 from WeirdChat), 57 rules simplified from OpenAI's model spec, 5 draws per call at temperature 1.

**Three interventions:**

1. *Prompting*: naive (one verdict), encouraged (reason through each rule first), forced (a required JSON verdict for every rule before the final verdict).
2. *Consistency training with GRPO* on Gemma-4-31B. The reward for a sampled full-ruleset verdict is the probability that it equals the anchor $a_0(s)=\max_i Y^{(0)}_{c_i}(s)$, built from the frozen base model's separate-call judgments (2 samples per rule). Two format rewards are added, and a forward KL penalty replaces the usual KL so the policy cannot cheat by changing its per-rule judgments.
3. *RuleForge*: a minimax game in which an amender agent rewrites the ruleset to reduce ambiguity while a generator agent searches for scenarios that stay ambiguous, with a regularizer that keeps the original intent.

### Why it works

The authors do not claim a single cause. They show that both fixing the judge and fixing the ruleset help, so they argue the problem mixes a reasoning flaw with poorly specified rules.

Their analogy is statutory interpretation: surrounding provisions shape how a provision is read, but in law the interactions are made explicit through hierarchies, whereas here the judge applies an implicit hierarchy inside its reasoning.

Revised rulesets that gained explicit meta-rules (an ordering between rules and exemptions) were more consistent, which supports this. My read: in a long list, a judge likely weighs each rule against the others for relevance and excuses a borderline violation that a competing rule appears to justify. The paper's evidence is consistent with this but does not test it directly.

### Evidence

- **Flips.** Across the six judges, 64.9% to 78.9% of rules showed a statistically significant change in violation rate between same-call (full ruleset) and separate-call judgments, almost all toward leniency (Figure 1). Three of the 20 root-level model-spec rules, including acting within an agreed scope of autonomy and trying to prevent imminent real-world harm, were judged more leniently inside the full set.
- **Scale.** With randomly sampled rulesets of size 2 to 57, inconsistency rose with size for all six models, from roughly 10-30% at two rules to 40-80% with the full set (Figure 3).
- **Prompting.** Forcing a verdict per rule helped some models but not all (Section 3.1).
- **GRPO.** On held-out scenarios, inconsistency fell from 0.709 to 0.375 for Gemma-4-31B (Figure 5). Per-rule judgments differed from the base model on only 3.7% of rules, aggregated per-rule judgments on 7.2%, and full-ruleset verdicts on 36.4%. The model still output all three verdict types, and its ruleset reasoning grew from 1.6k to 2.2k tokens.
- **RuleForge.** Two revised rulesets turned 17.8% and 21.4% of held-out scenarios from inconsistent to consistent, while regressing 4.0% the other way (Figure 6).

### Limitations and open questions

The authors note that inconsistency is not reduced to zero for every judge, scenario or ruleset, and that OpenAI's model spec has no ground truth, so they measure self-consistency rather than correctness. They also say RuleForge outputs must be human-checked.

My read: the evaluation uses one ruleset family (a simplified 57-rule version of one model spec), and the assumption that per-rule verdicts are the "right" ones is a design choice; the authors say training the other direction is equally valid. Leniency in the full-set setting could sometimes be correct interpretation. Trained-judge results are for Gemma-4-31B (with a Qwen3-8B replication in the appendix), and whether consistency training transfers to judges used inside RL pipelines is untested.

### How to apply

Use it whenever one judge prompt carries more than a handful of criteria: rubric rewards, safety classifiers, spec-compliance checks.

1. Pick your 100-500 most representative scenarios and judge each critical rule twice: alone, and inside the full prompt, with several samples each.
2. Compute the inconsistency rate with the max-over-rules definition and the direction of the flips. Watch hard constraints first.
3. If the gap is large, try decomposition: one call per hard rule, then aggregate with "any violation means violation". This costs $K$ calls but needs no training.
4. If calls are too expensive, force a per-rule verdict before the final verdict, or add explicit priority and exemption meta-rules.

**Smallest experiment:** take your five most important rules, run 200 scenarios through both formats with 3 samples, and report how many violations the bundled judge drops. One afternoon on an API judge.

Assessment: Try in an experiment. The diagnostic is cheap and the effect is large on six judges; the training fix is plausible but shown on one judge family.

### Connections

Constitutional AI and later model-spec work evaluate judges with agreement metrics against humans or between models; He et al. (2026a, cited by the authors) showed that interpretive ambiguity is inherent in constitutions. What this paper adds is a metric that compares a judge with *itself* across prompt formats, evidence that the discrepancy is systematically toward leniency and grows with ruleset size, and two fixes that keep the judge's per-rule behavior mostly intact.

Connections: this is the judge-side counterpart of idea 11, where an RL reward is a severity-weighted list of rubric items judged in a single call, so the same softening could apply. The same effect could matter for idea 9, whose reward also relies on a judge scoring many criteria. Mitigation by decomposing into per-rule calls is the "verify each piece" pattern from idea 10.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-25 · origin · [Aligning Judge Models to Be Logically Consistent Multi-Rule Interpreters](https://arxiv.org/abs/2611.29975) — On six judge models over 500 scenarios and 57 model-spec rules, 64.9%-78.9% of rules changed violation rate significantly between separate-call and bundled judgments (almost all more lenient), inconsistency rose from roughly 10-30% at two rules to 40-80% at 57, and GRPO consistency training cut Gemma-4-31B held-out inconsistency from 0.709 to 0.375.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-25: created from bulletin 2026-11-25 (#2).
