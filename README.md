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
