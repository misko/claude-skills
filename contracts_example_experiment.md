---
name: experiment
description: A self-contained, reproducible experiment — its own folder with a summary describing background, task, input features (with provenance), approach, sanity check, output schema, results, and citations.
contract_version: v1
created: 2026-04-19
updated: 2026-04-19
tags: [contract, experiment, reproducibility]
---

# Experiment contract

> Worked example for the [contracts skill](contracts.md). Demonstrates the `<name>_contract.md` shape the skill audits artifacts against. This contract itself conforms to `contracts.md§Reference: contract_contract`; running `/audit --contract contracts_example_experiment.md` on this file should be clean.

## What this governs

`experiments/exp<NNN>-<kebab-slug>/` directories. One experiment per directory. The number `<NNN>` is monotonic (zero-padded 3-digit), assigned at creation. The slug is a short human-readable summary (e.g. `experiments/exp007-pt-her-rediscovery/`).

The whole point of an experiment is **standalone reproducibility**: a reader who only has access to the experiment directory (and the symlinked / copied input features) must be able to (a) understand what was tried and why, (b) verify the code runs end-to-end via the documented sanity check, (c) re-run the experiment and compare against the documented output schema. Experiments that cannot be reproduced from their own folder violate the spirit of this contract.

Experiments are append-only in spirit: once results are recorded, an experiment is not rewritten. To revise an approach, write a new experiment (`exp<NNN+k>-<...>`) that supersedes or builds on the prior one.

## Required frontmatter

```yaml
---
id: exp<NNN>                            # string, must match folder prefix; e.g. "exp007"
title: <one-line experiment summary>    # string
status: <enum>                          # one of: planned | running | complete | abandoned
created: <YYYY-MM-DD>                   # date the experiment folder was created
completed: <YYYY-MM-DD | null>          # date status flipped to complete (or null)
authors: [<name1>, <name2>, ...]        # list of strings; can be roles ("the data team")
contract_version: v1
tags: [experiment, <domain-tags>]
parent: <exp<NNN> | null>               # the prior experiment this one builds on / supersedes, if any
children: [<exp<NNN>>, ...]             # experiments that build on this one (set when later experiments declare parent)
related: [<exp<NNN>>, ...]              # informational only, not parent/child
seed: <int | null>                      # the random seed used; null only if the experiment is fully deterministic
runtime_env: <string>                   # e.g. "python==3.11, torch==2.4.0, cuda==12.4"; pinning required
---
```

Required keys: `id`, `title`, `status`, `created`, `authors`, `contract_version`, `tags`, `seed`, `runtime_env`. Optional: `completed`, `parent`, `children`, `related`.

`status: planned` — folder + summary skeleton exist, code not yet written. May be edited freely.
`status: running` — code exists; results not yet recorded. `## Results` may be empty/partial.
`status: complete` — `## Results` populated; `completed` date set; experiment frozen except for typos and `children` field updates.
`status: abandoned` — experiment was started but not finished; `## Results` should explain why (failed setup, infeasible, superseded mid-run).

## Required body sections (in order)

The summary file is `experiments/exp<NNN>-<slug>/summary.md`. It MUST have these sections, in order:

1. **`# Experiment exp<NNN>: <title>`** — H1 mirroring the title and id
2. **`## Background`** — why this experiment exists; what prior work / observation motivated it; what's the gap it addresses. Cite related literature with wikilinks ([[exp<NNN>]] for prior experiments, [[paper-id]] for papers, or external URLs/DOIs)
3. **`## Task`** — the falsifiable question the experiment answers. State as: "Does X improve Y when Z?" or "What is the effect of X on Y?" — concrete, measurable, with a clear success/failure criterion
4. **`## Input features`** — every feature used, one per `### <feature-name>` subsection. Each subsection MUST include:
   - **Provenance** — source: file path, generation script + commit, dataset citation (wikilink or DOI), or "human-curated by <author> on <date>"
   - **Description** — what the feature represents (units, shape, semantics)
   - **Path in `inputs/`** — the relative path inside the experiment's `inputs/` subdir where this feature lives (or is symlinked from)
5. **`## Approach`** — the method. Specific enough to reproduce: model architecture (with hyperparameters), training procedure (optimizer, schedule, batch size, epochs), evaluation protocol. Reference the specific code file(s) in the experiment dir or repo. NOT "we used a transformer."
6. **`## Sanity check`** — an executable command (or short script) a reviewer can run from the experiment directory to verify the code works end-to-end on a tiny subset / smoke-test inputs. MUST be a fenced bash block with the actual command and expected output (or "expected: exits 0"). NOT narrative prose.
7. **`## Output format`** — schema for the experiment's results: field names, types, units, where files land. Specify enough that a downstream consumer (or `/audit`) could parse results without reading code. Typical: a fenced JSON / YAML block showing the metrics file shape.
8. **`## Results`** — populated when `status: complete`. Reports the experiment's actual outcome against the falsifiable Task. MUST include concrete numbers and engage with the Task's success/failure criterion. Negative results are valid; they MUST be reported as such, not papered over.
9. **`## References`** — citations to all related literature. Use wikilinks for in-repo papers/experiments ([[paper-id]], [[exp<NNN>]]); URLs or DOIs for external work. Every citation referenced inline in the body MUST appear here.

## Cross-reference rules

- `parent: exp<NNN>` MUST resolve to an existing experiment directory (`experiments/exp<NNN>-*/`).
- `parent: exp<NNN>` requires the parent's `children` list to include THIS experiment's id (reciprocity).
- `children: [exp<NNN>, ...]` IDs MUST resolve. Reciprocity required (each child's `parent` field MUST point back).
- `related: [exp<NNN>, ...]` IDs MUST resolve. No reciprocity required (informational).
- Every feature listed under `## Input features` MUST have a corresponding file or symlink at the path declared by its **Path in `inputs/`** subsection. Conversely: every file or symlink in `inputs/` MUST be described by exactly one `### <feature-name>` subsection in `## Input features`. (Strict 1:1 mapping; no orphan files, no undocumented features.)
- Wikilinks `[[paper-id]]` MUST resolve to an existing paper in the workspace's papers/ directory.
- Wikilinks `[[exp<NNN>]]` MUST resolve to an existing experiment.

## Banned content

- **Hardcoded absolute paths** in `summary.md`, `inputs/`, or any script in the experiment directory. Use relative paths (`./inputs/feature.parquet`) or environment variables (`$EXPERIMENT_DIR`). Absolute paths break reproducibility on other machines.
- **Unspecified random seeds** — if the experiment uses any randomness (data shuffling, weight init, sampling), `seed` in frontmatter MUST be set to a concrete integer. `seed: null` is allowed ONLY if the experiment is fully deterministic.
- **`TODO` / `TBD` / `XXX`** markers anywhere in `summary.md` once `status: complete`. Planning-stage TODOs are fine while `status: planned`.
- **Generic feature descriptions** — "the input data," "training set," "embeddings." Each feature must name what it specifically is, where it came from, and what shape/units.
- **Editing `## Background`, `## Task`, or `## Approach` after `status: complete`** — these are the pre-registered claim. Use a new experiment that supersedes if the approach needs revision.
- **Inline result numbers in `## Background` or `## Task`** — those sections describe the question, not the answer. Mixing them is pre-registration drift.
- **Empty `inputs/` directory** — every experiment has at least one input feature. If the experiment is a pure simulation with no inputs, document the simulation's parameter set as a feature with provenance "synthesized in-experiment per <approach section>."
- **Sanity check that just says "see code"** — `## Sanity check` MUST be an actual runnable command + expected outcome. If it can't be run standalone, the experiment isn't standalone-reproducible.

## Semantic audit checks

- **task-is-falsifiable** — `## Task` states a concrete, measurable question with a clear success/failure criterion. Not "explore X" but "does X reduce Y by ≥Z%?" An auditor must be able to look at `## Results` and decide whether the answer is yes or no.
- **background-motivates-not-describes** — `## Background` explains WHY this experiment exists (gap, prior failure, hypothesis source) — not a re-statement of the task. If `## Background` reads "we will train a model on Y," it's leaking the approach; rewrite to describe the motivating gap.
- **features-have-real-provenance** — every `### <feature-name>` has a concrete provenance: file path + commit, generation script + commit, dataset citation, or named human curation. NOT "from the data team" or "from prior work." Provenance is what makes the experiment re-buildable from scratch.
- **features-match-inputs-dir-1to1** — the set of `### <feature-name>` subsections in `## Input features` matches the set of files/symlinks in `inputs/` exactly. Auditor verifies both directions.
- **approach-is-specifically-reproducible** — `## Approach` names the architecture, hyperparameters, optimizer, schedule, batch size, epochs (or equivalent specifics for non-ML experiments). A second engineer reading only this section MUST be able to reproduce the run. Vagueness ("we tuned the model") is a finding.
- **sanity-check-is-executable** — `## Sanity check` is a fenced bash block with an actual command, runnable from the experiment directory. Includes expected exit code or expected output snippet. Narrative substitutes ("run the training script") fail this check.
- **output-format-is-schema-not-prose** — `## Output format` is a fenced JSON / YAML block (or schema-equivalent) showing field names + types + units + file paths. Prose like "we save metrics to a JSON file" fails — the auditor can't verify result shape from prose.
- **results-engage-with-task** — `## Results` references the falsifiable claim from `## Task` with concrete numbers and a yes/no/partial verdict. If `## Task` asked "does X reduce Y by ≥Z%?", `## Results` MUST say "Y was reduced by Q%, so the criterion was met / not met / partially met."
- **negative-results-not-papered-over** — when results don't match the prediction, `## Results` says so directly. Watch for: hedging ("the results are interesting and warrant further study"), goal-shifting ("while X didn't reduce Y, we found Z, which is also useful"), or omission of the original criterion. Each is a finding.
- **citations-engage-with-content** — for every `[[paper-id]]` cited in the body, the surrounding sentence references a specific claim, method, or finding from that paper — not "see [[paper-id]] for context." Auditor checks the wikilink resolves AND the citation is contextually meaningful.
- **inputs-dir-no-orphans** — every file or symlink in `inputs/` is described by a corresponding `### <feature-name>` subsection. Symlinks must point to files that actually exist (broken symlinks are a finding).
- **runtime-env-pinned** — `runtime_env` frontmatter is concrete (`python==3.11, torch==2.4.0`), not loose (`python 3.x, recent torch`). Reproducibility requires version pinning.

## Sibling files

Each experiment directory MUST contain:

- **`summary.md`** — the file governed by this contract (per `## Required body sections (in order)` above).
- **`inputs/`** — directory containing input features (files or symlinks). MUST be non-empty. Every file maps 1:1 to a `### <feature-name>` subsection in `summary.md§Input features`.

Each experiment directory SHOULD contain (recommended but not required):

- **`results/`** — directory for output artifacts (metrics, plots, checkpoints) matching the `## Output format` schema. Created when `status: running` or `complete`.
- **A run script** (`run.sh`, `train.py`, `Makefile`, or equivalent) — the executable that produces the results. Referenced from `## Approach` and invoked by `## Sanity check`.

Optional:

- **`logs/`** — captured stdout/stderr from runs. Useful for debugging; not contract-enforced.
- **`notes.md`** — informal scratch notes. Not contract-enforced; lives alongside `summary.md` for human-only context.

## Change policy

This contract evolves rarely. To bump `contract_version`:

1. Update this file with the new rules + bump `contract_version`
2. Add CHANGELOG entry (one line)
3. Author migration prompt at `<workspace>/.claude/migrations/experiment_v<N>_to_v<N+1>_<change>.md`
4. Run `/audit --fix --apply` against `experiments/`; existing experiments migrate via the Fixer

For experiments that need to change due to a real methodological revision (not contract evolution): write a new experiment that supersedes the old one (set `parent: exp<NNN>` on the new; the old's `children` field gets the reciprocity update; the old's `## Background`, `## Task`, `## Approach`, and `## Results` stay frozen). That's the discipline that makes the experiment log a useful history of what was tried, not just what currently works.

Editing `## Results` to make a failed experiment "look better" is the worst form of drift this contract guards against. If results need re-interpretation, write a new experiment.

## CHANGELOG

- **v1** (2026-04-19) — initial contract for experiments. Inspired by reproducibility lessons from real-world ML and scientific computing: pre-registration drift (results editing the task), feature provenance gaps (datasets that can't be re-built), missing sanity checks (code that mysteriously stops working months later), unpinned environments (re-runs that don't match), generic-feature descriptions ("the embeddings"). Each banned-content item and semantic check addresses a specific failure mode observed in practice.
