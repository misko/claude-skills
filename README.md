# claude-skills

Prompt-based skills for Claude Code. Each skill is a self-contained `.md` file — give it to Claude and go.

## Skills

### [Adversarial Review](adversarial-review/)
Multi-agent adversarial review for any artifact (code, plans, specs, designs). Spawns N AI reviewers with distinct personalities who independently review, then debate issues round-robin until consensus or agree-to-disagree. Every panel includes a mandatory Devil's Advocate and Minimalist.

Built-in panels: `code_review` (5 agents), `code_review_extended` (7 agents), `plan_review` (5 agents). Custom panels supported.

### [Engspec](engspec/)
English-first specification layer for any codebase. Three prompts + one shared format spec:

- **engspec_prompt.md** — Code → structured English specifications
- **engspec_tester_prompt.md** — Adversarial Red/Blue/Judge debate on specs
- **engspec_to_code_prompt.md** — Regenerate working code from specs alone
- **engspec_format.md** — Shared `.engspec` format specification (v1.0)

### [Contracts](contracts/)
Audit + fix discipline for any structured artifact (papers, projects, experiments, datasets, ADRs, runbooks, anything with frontmatter + structured prose). You author one `<name>_contract.md` per artifact kind; the skill runs three orthogonal stages:

- **Phase 0** — contract↔checker drift detection (keeps paired Python audit scripts in sync with the contract; auto-regenerates them on `--fix`)
- **Phase 1** — fast deterministic Python audit (when a paired script exists)
- **Phase 2** — per-artifact LLM Auditor swarm reading the contract's `## Semantic audit checks` + applying generic content-vs-spec judgment (sycophancy, generic-vs-specific, pre-registration drift)

Plus `--fix` for parallel Fixer subagents per migration_kind, with multi-round convergence. Includes a worked example for Architectural Decision Records.
