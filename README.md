# Idea Atlas

A living directory of the most powerful ideas in **AI agents**, **RL environments** and **post-training**, updated every weekday from new arXiv papers.

## How to browse

1. Start at [INDEX.md](INDEX.md): totals, everything that changed in the last 14 days, and the top ideas in each section.
2. Open a section:
   - [AI Agents](sections/agents/README.md) (AG)
   - [RL Environments](sections/rl-envs/README.md) (ENV)
   - [Post-Training](sections/post-training/README.md) (PT)

   Each section opens with a dated "State of the field" paragraph, then lists every idea by subtopic, highest power score first.
3. Open an idea. Every idea file has the same layers:
   - **TL;DR**: the `tldr` line in the header
   - **Summary**: 5-8 sentences
   - **Detail**: problem, mechanism, why it works, evidence, limits, how to apply, connections
   - **Evidence log**: every paper that supported, extended or challenged it, newest first

## How to search

- `grep -ril "process reward" sections/` finds every idea that mentions a term.
- `atlas.jsonl` has one line per idea (title, TL;DR, tags, aliases, status, score), ready for scripts or embeddings.
- In Obsidian, open this folder as a vault; frontmatter becomes searchable properties.

## Reading the labels

- **Status:** `emerging` (one source) → `replicated` (two or more independent groups) → `established` (widely used); or `contested` / `superseded`.
- **Power (0-100):** six criteria scored 1-5 and weighted: novelty 0.25, leverage 0.25, evidence 0.20, generality 0.15, actionability 0.10, surprise 0.05.
- **Evidence (A-D):** the strongest supporting paper's experimental quality. See [TAXONOMY.md](TAXONOMY.md).

## How it changes

Ideas are never deleted; a replaced idea is marked `superseded` and points to its successor. IDs are permanent. Every change is logged in the idea's Evidence log and Revision history, and each day's changes in [CHANGELOG.md](CHANGELOG.md).
