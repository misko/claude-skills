# contracts

Prompt-based discipline for any kind of structured artifact — papers, projects, experiments, datasets, ADRs, runbooks, design docs, anything you'd write down and want to enforce structure + content quality on over time.

You write a `<name>_contract.md` describing what an artifact must contain. The skill audits artifacts against it (structure + semantics in one Claude pass), can apply migrations when the contract changes, and verifies its own paired auditor scripts stay in sync with the contract. Self-healing.

## What's a contract

A contract is a markdown file (`<name>_contract.md`) describing what an artifact of that kind must contain:
- Structural rules (frontmatter keys, body sections, allowed enum values)
- Cross-reference rules (e.g. wikilinks resolve, fields cross-validate)
- Content quality rules (prose is on-topic, predictions are reasoned, etc.)
- Banned content
- Change-management policy (versioning, CHANGELOG, migration registration)

`format` would be too weak a word — these are agreements that bind the artifact's structure AND behavior AND change semantics. You audit a contract; you don't audit a format.

## Three-stage audit (one Claude command)

```
/audit
  │
  ├─► Phase 0  Contract-vs-checker drift detection
  │            (Claude reads <name>_contract.md + paired Python audit script;
  │             does the script implement every rule the contract states?
  │             does it check anything the contract doesn't require?)
  │            → if drift: tag finding migration_kind: checker_v_to_v_<name>
  │              → /audit --fix dispatches Fixer → regenerates the Python
  │
  ├─► Phase 1  Run the (verified) Python checker  (fast, deterministic, free)
  │            → structural findings per artifact
  │
  └─► Phase 2  Per-artifact LLM Auditor swarm  (parallel, one subagent / artifact)
               → reads contract + artifact + applies the
                 ## Semantic audit checks section
               → emits content-quality findings (sycophancy, on-topic,
                 cross-artifact consistency, etc.)
```

Phase 0 keeps the script in sync with the contract.
Phase 1 keeps it fast.
Phase 2 keeps it honest.

## Files in this skill

| File | What it is |
|---|---|
| [`contract_contract.md`](contract_contract.md) | The meta-contract — what makes a valid `<name>_contract.md`. The skill audits its own examples against this. |
| [`finding_contract.md`](finding_contract.md) | Shape of one audit finding (severity, rule, path, message, evidence, suggested_fix, optional migration_kind). |
| [`migration_contract.md`](migration_contract.md) | Shape of a migration prompt at `<workspace>/.claude/migrations/<kind>.md`. |
| [`auditor_prompt.md`](auditor_prompt.md) | Main entry point — the orchestrator. Hand this to Claude with a workspace path and it runs Phase 0 + 1 + 2 (and Fix on `--fix`). |
| [`harnesses/auditor_harness.md`](harnesses/auditor_harness.md) | Prompt template for one Phase 2 Auditor subagent. |
| [`harnesses/spec_checker_audit.md`](harnesses/spec_checker_audit.md) | Phase 0 prompt — does the Python script implement the contract? |
| [`harnesses/checker_regenerator.md`](harnesses/checker_regenerator.md) | Phase 0 fixer — regenerate the Python script from the contract. |
| [`harnesses/fixer_harness.md`](harnesses/fixer_harness.md) | Artifact-level migration fixer (consumes `<workspace>/.claude/migrations/<kind>.md`). |
| [`examples/adr_contract.md`](examples/adr_contract.md) | Worked example: contract for Architectural Decision Records. |

## Quick start

You're an LLM-native consumer. Drop the skill folder next to your project's `.claude/`, then:

1. **Author one or more `<name>_contract.md` files** in your workspace (model on `examples/adr_contract.md`). Each contract describes one kind of artifact.

2. **Optional but recommended: ship paired `<name>_contract-audit.py` scripts** for the cheap-and-fast structural checks. Don't write them by hand — let `/audit --fix` regenerate them from the contract on first run via the Phase 0 Fixer.

3. **Run `/audit`** (or hand `auditor_prompt.md` directly to Claude with a workspace path):
   ```
   /audit                           workspace-wide; Phase 0 + 1 + 2; report only
   /audit --fix                     + apply migration-tagged fixes (multi-round convergence)
   /audit --paper <id>              narrow scope
   /audit --explain                 verbose report with "what good looks like" examples
   /audit --cross                   include cross-artifact consistency checks
   /audit --new-contract <name>     scaffold a fresh contract from a brief; self-audits it
   ```

4. **Author migration prompts** at `<workspace>/.claude/migrations/<kind>.md` whenever you bump a contract version. The kind is whatever you want — a string the audit emits as `migration_kind` when artifacts need updating. The prompt template (per `migration_contract.md`) tells the Fixer subagent how to apply the change.

## Why this exists

Long-running workspaces accumulate hundreds of structured artifacts. Without contract enforcement they drift — sections rename, key fields go missing, prose becomes generic, cross-references break. With this skill, drift becomes a finding instead of a surprise. With `--fix`, drift becomes a self-healing event.

The skill itself is meant to be invariant: contract authors write contracts, the skill audits + heals. No per-project Python audit code to maintain.

## Architecture invariants

- **Contracts are the only source of truth.** Python audit scripts, when present, are derived. Phase 0 detects + repairs drift.
- **Contracts self-describe.** Every `<name>_contract.md` follows `contract_contract.md`. The skill audits its own examples against itself; if its own meta-spec is wrong, you'll see it.
- **One audit pass = one Claude read of (contract + artifact).** Structural and semantic checks aren't bifurcated; the contract has both, the auditor applies both.
- **Fixes are migration-prompt-driven.** No clever inference; the operator authors a migration prompt at `<workspace>/.claude/migrations/<kind>.md` and the Fixer applies it. New migrations are explicit; the audit never silently changes content.
- **Multi-round convergence.** Fixer applies, re-audits, re-fixes if new findings surfaced. Converges or escalates to `gates/<kind>-stuck.md` for human review.
