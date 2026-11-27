---
id: ENV-20261127-02
title: Planted proof depth overstates what logic problems require; certify the mandatory steps
tldr: A checker can compute how many proof steps every solution must take; on FLD×2, problems labelled depth 5 need about two mandatory steps, so planted-depth labels overstate difficulty.
section: rl-envs
also_in: [post-training]
subtopic: env-construction
type: method
tags: [benchmark-quality, verifier, reasoning-rl, certified-depth, difficulty-labels, synthetic-reasoning-data]
aliases: [certified depth, ProofPlant, semforge, mandatory proof steps, shortcut-free proof depth]
status: emerging
scores: {novelty: 4, leverage: 4, evidence: 4, generality: 4, actionability: 5, surprise: 4}
power: 82
evidence: B
first_seen: '2026-11-27'
last_updated: '2026-11-27'
sources:
- {arxiv: '2611.33303', title: 'Certify What You Plant: Shortcut-Free Proof Depth in Logical Reasoning Data', role: origin}
related:
- {id: ENV-20261127-01, rel: combines-with}
superseded_by: null
---

## Summary

Synthetic reasoning data sets difficulty by planting a proof of depth d and labelling the problem depth d, but a planted proof is only one witness: shorter proofs may exist, or the premises may be routed around. The paper computes, relative to a stated rule set, the shortest proof, the mandatory steps that appear in every proof (certified depth), the number of minimal proofs and the indispensable premises, and exports them as machine-checkable certificates verified by an independent checker. Auditing eight public datasets, FLD×2 problems labelled depth 5 require about two mandatory steps (95% bootstrap interval 1.94-2.21), while ProofWriter's depth claim is largely forced (93% of its smallest proof is mandatory) and Mystery Blocksworld is confirmed structurally identical to Blocksworld. A generator (ProofPlant) builds problems backward to a requested certified structure and prunes candidates with a catalog of shortcuts proved once per rule set, certifying every output in full. The training results are weaker: certified depth alone changed little, a first comparison's gain came from a negation shortcut, and 14 seeds of Qwen3-0.6B showed no two-way difference once negation was balanced. Training on deep examples only made the model answer neutral on about half of FLD×2's derivable problems, while every mixture containing shallow examples avoided it, and mixtures of nine certified problem types beat an FLD×2-style baseline at equal size for Qwen3 0.6B to 8B. Use it before trusting a depth label in a curriculum or a verifiable-reward task, and store a certificate with each task. Certified depth is defined only relative to a stated calculus, the training runs use small models with no RL, and the mixture search selected on development sets.

## Detail

### Problem

Synthetic reasoning data is a main source of verifiable tasks for training and evaluating models: you sample a proof of depth $d$, state its premises as the problem, and call the result "depth $d$". If other, shorter proofs exist, or the premises can be routed around the planted steps, the label overstates what solving the problem requires. Curricula keyed to depth, difficulty-controlled RL environments and "accuracy versus depth" plots then rest on a number that does not measure what it says.

This matters for anyone building reward-verified task generators: you cannot tell whether a model fails because a problem is deep or because the generator's depth label is wrong.

### Mechanism

**Planted proof.** A generator samples a derivation of a chosen length, then writes its premises and conclusion as a problem. The label is the planted length.

**Mandatory steps.** Rule applications that appear in every proof of the conclusion from the given premises, under a stated set of inference rules. The paper calls their count the *certified depth*. The shortest proof can be longer than the mandatory set, but never shorter.

**Certificate.** A machine-checkable record (the proof, the mandatory steps, and countermodels for "neutral" labels) that an independent checker verifies. Labels are entailment (E), contradiction (C), or neutral (N, undecidable from the premises).

**FLD×2.** A synthetic logic dataset built by forward-sampling proofs; its depth-5 problems are the main case.

1. **Measure realized structure.** For each problem, relative to a stated calculus (rule set), compute the shortest proof, the mandatory steps, the number of minimal proofs, and the indispensable premises. Export these as certificates and verify them with a separate checker.
2. **Audit datasets.** Run the measures over eight public datasets, translated into the paper's logics (FLD×2 labels agree with the z3 solver on all 455 sampled problems).
3. **Construct problems with guaranteed structure (ProofPlant).** Given a request such as "every proof needs five steps", build the intended derivation backward, bind it to fresh constants, then certify the problem. A catalog of shortcuts, proved once per rule set, guides construction and prunes candidates; a pruned search still certifies every output in full, so pruning only loses valid problems.
4. **Train on certified mixtures.** Fine-tune Qwen3 models (0.6B to 8B) on 3,000 examples for label prediction, matching training sets on label, premise count and how neutrals are built, and score two-way accuracy (E vs C) so answering "neutral" more often cannot help.

The audit thus replaces "trust the generator's label" with a verifiable property of each problem.

### Why it works

A planted proof is only one witness. Whether it is the only route depends on the rules and premises, which the generator never checks. Computing the mandatory steps exposes the gap and, by construction, removes it: constructed problems lie on the diagonal where planted depth equals certified depth. The authors trace part of the FLD×2 errors to universal facts worded "some" or "something", which models read as particular. My read: certification mostly makes the controls possible (matching data on certified structure while holding labels and premises fixed), and the paper's best finding is how often a "deeper" training set turned out to differ in a hidden shortcut.

### Evidence

**Setup.** Eight datasets audited; DeepSeek-V4-Flash and gpt-oss-120b (low reasoning effort) label 763 derivable FLD×2 test problems; Qwen3 models fine-tuned on 3,000-example sets, 14 seeds in controlled contrasts.

**Audit (Table 1, Figure 2).**
- ProofWriter's depth claim is forced: 93% of the smallest proof is mandatory, though 85-90% of theory facts are never used. Blocksworld is similar: 84% of an optimal plan's actions are landmarks.
- FLD×2 problems labelled depth 5 require about two mandatory steps (95% bootstrap interval 1.94-2.21).
- Mystery Blocksworld renames Blocksworld; certification confirms the two are structurally identical, so the drop that prior work reports (62.6% to under 5% for pre-o1 models) reflects wording alone.

**Empirical audit (Section 4.2).** On FLD×2, universal facts predict errors beyond every depth measure, costing gpt-oss 20 to 40 points at fixed claimed length. Rewording the quantifier ("Something that..." to "Everything that...") on 161 problems raised gpt-oss accuracy from .59 to .66 (interval over problems .00-.13) and DeepSeek from .58 to .63 (-.01 to .11). On constructed problems with 1-20 mandatory steps, gpt-oss-120b goes from near-perfect on short proofs to near floor on long ones; at fewer than 15 premises, accuracy falls from .58 (up to 7 steps) to .17 (13 or more).

**Training (Section 7).**
- Certified depth alone changes little. A first comparison seemed to favor certified-deep examples (two-way accuracy +.11 on QMLTP), but most of that came from a negation shortcut (negation separated contradictions from entailments by .34 in the deep set against .13 in the shallow one). With negation balanced, 14 seeds of Qwen3-0.6B show no two-way difference on any test (largest QMLTP +.05, interval -.00 to .10).
- Training on deep examples only makes the model answer neutral on about half of FLD×2's derivable problems; every mixture that includes shallow examples avoids this.
- At equal size, mixtures of nine certified problem types beat an FLD×2-style baseline for Qwen3 0.6B to 8B (Figure 4; exact deltas are in Table 8, which I did not check).

### Limitations and open questions

Certified depth is relative to a stated rule set; a model may use other inference patterns. Training experiments predict a label with small models (mainly Qwen3-0.6B) and run no RL or long chain-of-thought training. The first, flawed training comparison shows how easily shortcuts creep in, and the mixture search selected on development sets. ZebraLogic proof enumeration hits its cap on 46% of cells, so its shortest proof is only an upper bound.

My read: the audit is the strongest claim; the mixture gains may be specific to these logics and baselines.

### How to apply

Use it if you generate verifiable reasoning tasks or tune a depth curriculum.

1. Before trusting a depth label, compute the shortest proof and the mandatory steps for a sample of 200-500 problems (the repository `sileod/semforge` is the authors' code). If the median certified depth is well below the label, relabel or regenerate.
2. When comparing two training sets, match them on label, premise count and negation rate, and report two-way accuracy next to the neutral rate.
3. Include short proofs in any depth mixture; deep-only data made models declare problems underivable.
4. For RL environments, store a certificate with every task and re-check it with code that shares nothing with the generator.

Cost: for propositional or monadic logics, certification is a solver call per problem. **Smallest experiment:** audit your current easy and hard splits, and test whether accuracy tracks certified depth better than the generator's label.

Assessment: Try in an experiment. A cheap audit that often exposes mislabeled difficulty, with solid controls; the training gains are small and tested only on small models.

### Connections

Prior generators (ProofWriter, FLD×2, ProverQA-style pipelines) set difficulty by the planted proof. ProofWriter itself defines depth as the shallowest proof, so its label is correct by definition. The new element is checking, at the problem level and with an independent verifier, what is mandatory, then building data to a requested certified structure and showing which structural variables matter for training.

**Connections.** Idea 1 audits a benchmark's grader in the same spirit (what does the checker really require?). Idea 3 relies on an environment-checked evidence flag as its reward; here the certificate plays that role for data. For RL environments, this suggests attaching certificates to verifiable-reward tasks so that difficulty claims can be tested.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-27 · origin · [Certify What You Plant: Shortcut-Free Proof Depth in Logical Reasoning Data](https://arxiv.org/abs/2611.33303) — Auditing eight datasets, FLD×2 problems labelled depth 5 require only about two mandatory steps (95% interval 1.94-2.21); deep-only training made Qwen3 models answer neutral on about half of derivable FLD×2 problems, while certified mixtures including shallow examples beat an FLD×2-style baseline at equal size.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-27: created from bulletin 2026-11-27 (#2).
