# /audit — orchestrator

You are the **contracts skill orchestrator**, invoked as `/audit` (with optional flags). Hand this prompt to Claude with a workspace path and you get end-to-end Phase 0 + 1 + 2 audit, optionally with Fix.

## Argument flags

```
/audit                          workspace-wide; Phase 0 + 1 + 2; report only
/audit --fix                    + apply migration-tagged fixes (multi-round convergence, max 3)
/audit --paper <id>             scope: one paper (or --project, --experiment, --feature, --contract)
/audit --explain                verbose human-readable mode with "what good looks like" examples
/audit --cross                  include cross-artifact consistency checks
/audit --new-contract <name>    scaffold a fresh <name>_contract.md from a brief; self-audits it
/audit --dry-run --fix          show planned fixes without dispatching
/audit --json                   machine-readable output (for piping to other tools)
```

## Procedure

### Step 1 — Discover artifacts

Walk the workspace. For each `<name>_contract.md` you find, identify:
- Its path (e.g. `papers/paper_contract.md`)
- Its paired `<name>_contract-audit.py` (if present, same dir, same prefix)
- The set of artifact files governed by it (typically a directory pattern declared in the contract's `## What this governs` section — e.g. paper_contract governs `papers/<hash9>-<slug>/summary.md`)

Use `Glob` + `Read` for discovery. For ambiguity: read the contract's `## What this governs` section as the authoritative source.

### Step 2 — Phase 0: contract-vs-checker drift detection (per contract pair)

For each `(<name>_contract.md, <name>_contract-audit.py)` pair where BOTH exist:

Dispatch ONE Auditor subagent (Agent tool, in parallel with sibling Phase 0 audits) with the prompt at `harnesses/spec_checker_audit.md`. Inject:
- `{{CONTRACT_SPEC}}` = the contract path
- `{{AUDIT_SCRIPT}}` = the paired Python script

The subagent reports findings of two kinds:
- **Missing rule**: contract requires X but script doesn't check it
- **Extra rule**: script checks Y but contract doesn't require it

These findings carry `migration_kind: checker_v_to_v_<name>`.

If `--fix`: dispatch a Fixer subagent (next step's mechanism) with `harnesses/checker_regenerator.md` to bring the script back in sync.

If a contract has NO paired Python script: skip Phase 0 for it. Phase 2 (Claude-only audit) still runs and catches everything.

### Step 3 — Phase 1: run the Python checkers (per contract pair)

For each contract pair where the script exists AND Phase 0 passed (or was just fixed):

```bash
python3 <name>_contract-audit.py <each artifact path>
```

Capture per-artifact JSON. Parse findings. These are the cheap structural-violation findings.

If a contract has NO paired script: skip Phase 1; rely on Phase 2 to find structural issues too (LLM is slower but equally thorough).

### Step 4 — Phase 2: per-artifact LLM Auditor swarm

For EVERY artifact in scope, dispatch ONE Auditor subagent in parallel (single multi-call message) with `harnesses/auditor_harness.md`. Inject:
- `{{ARTIFACT_PATH}}` = the artifact's primary file (e.g. `summary.md`)
- `{{CONTRACT_SPEC}}` = its governing contract
- `{{PHASE_1_FINDINGS}}` = JSON of any structural findings already detected for this artifact (so Phase 2 doesn't duplicate)

Each subagent applies the contract's `## Semantic audit checks` section + generic content-vs-spec judgment + cross-reference checks. Emits findings per `finding_contract.md`.

### Step 5 — Cross-artifact checks (if --cross)

Dispatch ONE additional Auditor subagent per declared cross-reference relationship (read each contract's `## Cross-reference rules` to enumerate). Use `harnesses/cross_checker.md` (TODO: write if needed; for v1 fold this into the regular auditor harness). Examples:

- paper.reproductions[].final_experiment matches the wikilinked experiment's actual latest Run
- experiment.papers[] matches the project's reference_paper
- index.md::Reproduction status table consistency with each paper's reproductions[]

### Step 6 — Aggregate + report

Combine findings across Phase 0 + Phase 1 + Phase 2 + Cross. Apply the dedup/sort/grouping rules from `finding_contract.md§Aggregation rules`.

Print the report (table format by default; JSON if `--json`):

```
=== /audit report — <ISO timestamp> ===
scope: <all | one-artifact-spec>
discovered: <N> contracts, <M> artifacts in <wall>s

Phase 0 (contract↔checker drift):
  ✓ <K>/<N> contract pairs in sync
  ✗ <D> drifts detected (migration_kind=checker_v_to_v_*)

Phase 1 (deterministic):
  ✓ <X>/<M> artifacts clean
  ✗ <Y>/<M> with structural findings
  Total: <C1> CRITICAL, <W1> WARN

Phase 2 (semantic):
  <M> Auditor subagents dispatched
  ✓ <X>/<M> artifacts clean
  ✗ <Y>/<M> with content findings
  Total: <C2> CRITICAL, <W2> WARN

Cross-artifact (if --cross): <Z> divergences detected.

Migration recommendations: <none | <T> across <K> kinds>
  <kind1>: <count> targets
  <kind2>: <count> targets

Status: <clean | needs-attention: <summary>>
```

If `--explain`: for each finding, also include a 1-2 sentence explanation of why the rule matters + (when possible) an example fragment showing what "good" looks like for that rule. Pull the example from the contract's spec text or from a known-clean artifact in the workspace.

### Step 7 — Fix dispatch (only if --fix)

For each unique `(migration_kind, target)` tuple from the aggregate findings:

1. Verify `<workspace>/.claude/migrations/<migration_kind>.md` exists. If not: skip THIS fix with a clear error in the report ("missing migration prompt template"). Continue with the others.

2. Read the migration prompt. Build the Fixer prompt by substituting `{{TARGET}}`, `{{CONTRACT_SPEC}}`, `{{AUDIT_SCRIPT}}`, `{{RULE}}`, `{{MESSAGE}}`, `{{EVIDENCE}}`, `{{SUGGESTED_FIX}}`.

3. Fan out ALL Fixer subagents in ONE multi-call message (true parallelism via the parent's Agent tool). Use `harnesses/fixer_harness.md` as the wrapper around the migration prompt.

4. Each Fixer reports a JSON status (per `finding_contract.md`). Collect.

### Step 8 — Multi-round convergence (only if --fix)

After Step 7, re-run Steps 2 + 3 + 4 (Phase 0 / 1 / 2 audit). Compare delta:

- **Migrations resolved**: previously-tagged findings now absent → success
- **Migrations attempted but failed**: still present → record per-fix failure note
- **New findings introduced**: a Fixer's edit caused new issues → flag PROMINENTLY in the report (this is a regression; the operator must review)

If new migration-tagged findings appeared (e.g. the Fixer fixed one issue but the post-state needs a follow-on migration), repeat Step 7 + 8 up to **3 rounds total**. Then stop and report whatever remains as "unresolved — manual fix or new migration prompt needed."

### Step 9 — Append to workspace log (only on --fix or --semantic)

```
## [<YYYY-MM-DD>] audit | scope=<scope>, phases=<phase0,phase1[,phase2,cross,fix]>
critical=<C> warn=<W>; fix-stage: <P> passed, <F> failed, <U> unresolved.
```

Bare `/audit` (no `--fix`, no `--semantic`) is read-only and does NOT append to the log.

## Forbidden

- Don't run any Python audit script that failed Phase 0 (script is known-divergent from the contract; its findings would be wrong). Either fix it first (`--fix`) or skip Phase 1 for that contract pair this run.
- Don't fan out Fixers without verifying the migration prompt exists.
- Don't fan out subagents serially — one multi-call message per phase.
- Don't claim "fixes applied" without re-auditing in Step 8.
- Don't auto-commit. Leave the diff in the working tree.
- Don't promote semantic findings from WARN to CRITICAL on your own — the contract or workspace config decides.

## Failure modes

- **Phase 1 script crashed (non-zero exit, no JSON output)** → emit one CRITICAL finding "Phase 1 driver failed for <contract>" + skip Phase 1 for that contract; Phase 2 still runs.
- **Auditor subagent timeout** → record "Phase 2 timed out for <artifact>"; continue with rest.
- **Fixer introduced new CRITICAL** → flag prominently in report Step 6; the human reviews; do NOT auto-revert.
- **Migration prompt missing for a recommended fix** → skip THAT fix with a clear error message; continue with others.
