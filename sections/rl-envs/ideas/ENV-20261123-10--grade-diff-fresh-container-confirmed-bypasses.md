---
id: ENV-20261123-10
title: 'Grade the diff in a fresh container: confirmed bypasses fall from 14.7% to 2.7%'
tldr: Extract only the committed patch or declared outputs, grade in a fresh network-disabled container, blind-solve tasks before freezing tests, and require broken submissions the grader rejects; bypasses fall from 14.7% to 2.7%.
section: rl-envs
also_in: []
subtopic: verifiers-rewards
type: recipe
tags: [verifier, grader-audit, harness-coupling, reward-shaping, benchmark-quality]
aliases: [fresh-container diff grading, git-diff grading, blind solve before freezing, grader negative tests, Benchmarks' Last Exam checks]
status: emerging
scores: {novelty: 2, leverage: 4, evidence: 3, generality: 4, actionability: 5, surprise: 2}
power: 66
evidence: C
first_seen: '2026-11-23'
last_updated: '2026-11-23'
sources:
- {arxiv: '2611.27622', title: Benchmarks' Last Exam, role: origin}
related:
- {id: ENV-20261123-07, rel: builds-on}
superseded_by: null
---

## Summary

Agentic task graders fail in both directions: mostly by rejecting valid work, but in shared execution harnesses also by letting an agent emit a pass without doing the task, which is the classic reward-hacking hole for RL environments. The paper pairs each of its five grader defect classes with a pre-release check: a blind solve of the task from the public prompt alone before rubric or tests are frozen, two or three plausible broken submissions per task that the grader must reject in CI, and grading only the extracted git diff or declared output files in a fresh network-disabled container. Isolation is the measured piece: confirmed bypass or leakage in shared-container terminal harnesses is 14.7% (20/136) versus 2.7% (3/113) in fresh network-disabled git-diff containers, a 5.5x reduction at P = 8.4x10^-4, while read-only or static suites show 0 of 400. Wrong references and unstated requirements together are 67.9% of confirmed defects, which is the share blind solving targets, and defect repair coverage rises from 46.2% Accepted to 76.6% after those repairs and 91.1% after the rest. Use it when you build RL environments or eval suites where agents execute code or change state; hidden tests also need a reserved symbol prefix and documented paths, and a clean reference run in the grading container. The main caveats are that the isolation comparison is across benchmarks rather than a controlled experiment, and the blind-solve and negative-test checks are recommendations mapped to defect classes, not tested interventions.

## Detail

### Problem

Idea 7 shows that agentic task graders fail in both directions: mostly by rejecting valid work, but also, in shared execution harnesses, by letting agents emit a pass without doing the work. For RL environments the second kind is the classic reward-hacking hole, and the first kind silently corrupts the reward.

This idea is the operational half: a small set of design rules and checks, each tied to a measured defect class, that environment builders can add to their task-authoring pipeline. The container-isolation result is the strongest piece of evidence in the paper: a 5.5× difference in confirmed bypasses between grading designs.

### Mechanism

- **Shared-container grading:** the agent's processes run in the same container lineage that stages test fixtures and reports pass tokens; the grader then runs there too (e.g. Terminal-Bench 4.0 and Terminal-Bench-Science).
- **Fresh-container (git-diff) grading:** only the agent's committed patch or declared output files are extracted and applied in a new grading container with no network, then the tests run there (DeepSWE v1.1 and Harvey LAB already do this).
- **Defect classes** (from idea 7): C1 wrong reference, C2 unstated requirement, C3 missed critical error, C4 bypass/leakage, C5 infeasible constraint or grading collision.
- **Negative (mutation) test:** a deliberately wrong submission that the grader must reject.

The paper's Table 2 pairs each defect class with a pre-release check. As a pipeline:

**1. Blind solve before freezing (prevents C1, C2).** Before freezing any rubric or test script, a second contributor or a frontier agent attempts the task using only the public prompt and input attachments. Every check the blind solution fails is reviewed against primary sources: either the solution is wrong, or the prompt must state the requirement, or the reference/rubric is wrong. The paper lists the kinds of issues this surfaces: miscalculated basis-point targets, unforwarded deliverables, missing attachment details, unstated sign or symmetry conventions.

**2. Paired adversarial negative tests (prevents C3).** Every task pull request includes two or three plausible broken submissions, e.g. a sign-reversed accounting ledger, a live-state snapshot instead of the computed result, or rescaled medical-imaging headers. CI fails the task if the grader accepts any of them. This turns "are my assertions complete?" into an automated check.

**3. Separate solve and grade containers (prevents C4 and most C5).**
- Extract only declared output files or committed patches into a fresh grading container; this removes shared result pipes and readable test generators.
- Reserve an explicit namespace prefix for hidden test symbols so agent-written code cannot collide with them.
- Preserve documented filesystem paths, and execute a clean reference run in the grading container.

**4. Version everything.** Pin commit identifiers and versioned task releases so repairs are traceable.

### Why it works

Each check removes the structural cause of one defect class rather than detecting instances after the fact.

- **Isolation** closes the channels C4 exploits: if grading happens in a container the agent never touched, inherited pipes, staged fixtures and readable test generators are simply not available to the agent's code. The only path to a pass is the artefact being graded.
- **Blind solving** exposes the unstated choices of the original author, which idea 7 identifies as the main source of false negatives; a second solver makes different choices and the grader's hidden expectations show up as failures.
- **Negative tests** probe assertion completeness from the other side: if a sign-reversed ledger passes, the assertions do not check what matters (exactly the AutomationBench case, where reversing every entry passes all eight assertions).

For RL environments, the same logic gives a reward that is harder to hack (isolation, negative tests) and less noisy (blind-solve alignment).

### Evidence

**Isolation (Fig. 5b, exact 95% intervals):** confirmed C4 bypass/leakage in shared-container terminal harnesses 14.7% (20/136) vs fresh network-disabled git-diff containers (DeepSWE v1.1) 2.7% (3/113), a 5.5× reduction, P = 8.4×10⁻⁴. AutomationBench 9.0%. Read-only or static suites 0/400.

**Coverage of checks:** C1 and C2 together are 67.9% of confirmed defects (the share blind solving targets). Primary-defect repair coverage: 46.2% Accepted → 76.6% after C1/C2 alignment → 91.1% after C3-C5 repairs.

**Concrete exploits closed by isolation:** Terminal-Bench 4.0 `vf2-speedup-networkx` (pass signal over an inherited pipe); Terminal-Bench-Science `dna-storage-codec` (deterministic test-message generation rules readable in local files).

**Caveat:** the isolation comparison is between benchmarks, not a controlled experiment; domain and authoring practice differ. The checks other than isolation are recommendations mapped to defect classes, not tested interventions.

### Limitations and open questions

- The isolation result is a cross-benchmark comparison with confounds the authors acknowledge (different domains, task styles and authors).
- Blind-solve and negative-test checks are proposed, not evaluated: there is no before/after measurement of defect rates when they are applied.
- Same undocumented review process as idea 7 (evidence grade C).
- Isolation does not address C1/C2, the bulk of defects.

My read: the design rules are sound engineering regardless of the paper's exact numbers; adopt isolation and negative tests, and treat the specific rates as indicative.

### How to apply

**When:** building RL environments or eval suites where agents execute code or change state.

**Checklist for each environment:**
1. Agent runs in container A; at the end, extract only `git diff` or declared output files.
2. Apply them in fresh container B (no network, clean fixtures), run hidden tests with reserved name prefixes, and do a clean reference run there.
3. Require 2-3 broken submissions per task in the task repo; CI asserts the grader rejects each.
4. Before releasing a task, run one blind solve (person or a different strong agent) and review every failing check.

**Smallest experiment:** for 20 tasks in your environment, write one sign-flipped or snapshot-style broken submission each, run the grader, and count how many pass. Then try one task in a shared vs fresh grading container with a deliberately cheating agent script.

Assessment: Adopt now. Fresh-container grading and negative tests are low-cost, standard-engineering safeguards, and the paper ties them to a measured 5.5x drop in bypasses.

### Connections

Container isolation and hidden tests are established practice (SWE-bench-style grading, DeepSWE, Harvey LAB), and mutation testing is standard in software engineering. What is new is the evidence linking a measured defect distribution to specific checks across benchmark families, and the quantified isolation gap between shared and fresh-container grading designs.

**Connections.** Idea 7 is the measurement this recipe acts on. Idea 8 offers a complementary verifier (downstream outcomes) for settings where hidden tests do not exist. Idea 3's rule-credit result shows the same lesson inside training: a check that can be satisfied without doing the task will be.

## Evidence log
<!-- Append-only. Newest first. One line per source. -->
- 2026-11-23 · origin · [Benchmarks' Last Exam](https://arxiv.org/abs/2611.27622) — Confirmed grader bypass or leakage was 14.7% (20/136) in shared-container terminal harnesses versus 2.7% (3/113) in fresh network-disabled git-diff containers (P = 8.4e-4), and C1/C2 defects made up 67.9% of confirmed defects, but the blind-solve and negative-test checks were proposed, not evaluated.

## Revision history
<!-- Append-only. Newest first. -->
- 2026-11-23: created from bulletin 2026-11-23 (#10).
