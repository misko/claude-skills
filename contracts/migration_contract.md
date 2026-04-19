# migration_contract

The shape of a migration prompt at `<workspace>/.claude/migrations/<migration_kind>.md`.

A migration prompt is the LLM template a Fixer subagent reads when applying an automated repair. Each `migration_kind` registered in audit findings MUST have a matching prompt file here, or `/audit --fix` refuses to act on those findings.

## File location and naming

`<workspace>/.claude/migrations/<migration_kind>.md`

The `<migration_kind>` is a stable string the auditor emits when tagging findings (e.g. `paper_v1_to_v2_models`, `rationale_needs_specifics`, `checker_v_to_v_paper`). It MUST be lowercase, snake_case, and uniquely name the migration. Conventions:

- `<contract>_v<N>_to_v<N+1>_<change>` for contract-version bumps (e.g. `paper_v2_to_v3_reproductions`)
- `<change>_needs_<noun>` for one-off content rules (e.g. `rationale_needs_specifics`)
- `checker_v_to_v_<contract>` for Phase 0 audit-script regeneration (catches script drift)

## Required frontmatter

```yaml
---
migration_kind: <string>           # MUST match the filename (without .md)
description: <one-line>            # what this migration does
contract_version: <when-this-came-from>   # the contract version this migration targets
                                          # e.g. "paper_contract.md v3" — so future contract
                                          # bumps can decide whether this migration is still relevant
created: <YYYY-MM-DD>
applies_to: <enum>                 # one of: artifact | audit_script | meta
                                   # artifact = fix runs against the artifact files (most common)
                                   # audit_script = fix regenerates the paired Python audit script
                                   # meta = fix runs against a contract file itself (rare)
---
```

## Required body sections

1. **`# Migration: <migration_kind>`** — title H1 matching the kind
2. **`## When this fires`** — what audit finding triggers this migration. Ideally cite the rule by id (e.g. "Triggered when `paper_contract.md§Required frontmatter` flags missing `reproductions[]` field on a paper of contract_version: v2 or earlier").
3. **`## Substitution variables`** — the values the auditor injects when dispatching the Fixer. Standard set:
   - `{{TARGET}}` — path to the artifact being fixed
   - `{{CONTRACT_SPEC}}` — path to the relevant `<name>_contract.md`
   - `{{AUDIT_SCRIPT}}` — path to the paired `<name>_contract-audit.py` (if one exists)
   - `{{RULE}}` — the finding's `rule` field
   - `{{MESSAGE}}` — the finding's `message`
   - `{{EVIDENCE}}` — the finding's `evidence`
   - `{{SUGGESTED_FIX}}` — the finding's `suggested_fix`
   - Any migration-specific extras (declare them here)
4. **`## Process`** — step-by-step what the Fixer does. Numbered list. Reference the substitution variables.
5. **`## Validation`** — how the Fixer verifies its work BEFORE reporting success:
   - Re-run the paired audit script (`python3 {{AUDIT_SCRIPT}} {{TARGET}}`) — expected to be clean for THIS migration kind
   - Re-run the workspace-level `/audit --paper {{TARGET}}` (or equivalent) and verify zero CRITICAL findings remain
6. **`## Report back`** — the JSON shape the Fixer returns to the orchestrator. Standard:
   ```json
   {"status": "pass" | "pass-with-warn" | "fail" | "skip",
    "target": "{{TARGET}}",
    "<migration-specific-stats>": <numbers>,
    "residual_warns": <int>,
    "residual_criticals": <int>,
    "note": "<one-sentence summary>"}
   ```
7. **`## Don't`** — anti-patterns. Examples to include if applicable: don't auto-commit, don't fabricate data, don't reset unrelated fields, don't delete content that's not part of the migration scope.

## Idempotence

The migration prompt MUST be idempotent. Running the same migration twice on a fixed artifact MUST be a no-op (no new findings, no new diff). The Fixer's first action should be: detect whether the target already conforms to the post-migration state; if yes, report `status: skip` and exit.

## Rollback

The migration prompt SHOULD describe how to undo the change manually (one paragraph max, in the body). Auto-rollback is NOT a feature the skill provides — if a Fixer mis-applies, the human reviews the diff before commit (the skill never auto-commits).

## Worked example: contract_v_to_v_paper migration

A migration prompt that regenerates the paired Python audit script when the contract has changed:

```markdown
---
migration_kind: checker_v_to_v_paper
description: Regenerate papers/paper_contract-audit.py from the current paper_contract.md when Phase 0 detects drift between contract and script.
contract_version: paper_contract.md (any)
created: 2026-04-19
applies_to: audit_script
---

# Migration: checker_v_to_v_paper

## When this fires
Triggered when `/audit` Phase 0 finding `contracts§Spec checker drift` reports a divergence between paper_contract.md required-rules and paper_contract-audit.py implemented-rules.

## Substitution variables
- {{TARGET}} = papers/paper_contract-audit.py (the Python script to regenerate)
- {{CONTRACT_SPEC}} = papers/paper_contract.md
- {{RULE}} = the Phase 0 rule citation
- {{MESSAGE}} = which rules are missing/extra in the script
- {{EVIDENCE}} = diff summary

## Process
1. Read {{CONTRACT_SPEC}} in full.
2. Read existing {{TARGET}} (if present) for structure reference.
3. Generate a new {{TARGET}} that:
   - Validates every required frontmatter key from §"Required frontmatter"
   - Validates required body sections + order from §"Required body sections (in order)"
   - Validates cross-reference rules from §"Cross-reference rules"
   - Checks banned content patterns from §"Banned content"
   - Tags findings with migration_kind where applicable
4. Match the existing audit-script style (Finding dataclass, JSON output, exit codes 0/1/2).

## Validation
- python3 {{TARGET}} --help executes without error
- python3 {{TARGET}} <a-known-clean-paper> exits 0 with zero CRITICAL
- Re-run /audit Phase 0 to verify drift is resolved

## Report back
{"status": ..., "target": "{{TARGET}}", "rules_added": <N>, "rules_removed": <N>, "residual_drift": <bool>}

## Don't
- Don't break the existing public function signature (audit() returns list[Finding])
- Don't auto-commit
- Don't add checks beyond what the contract states
```

## Migration registry hygiene

When a contract is bumped (per `contract_contract.md§Change policy`), the author MUST register the corresponding migration prompt in this directory. Failure to do so is itself a finding the contracts skill flags on every audit until resolved.
