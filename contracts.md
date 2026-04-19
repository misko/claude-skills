# Contracts skill

Audit + fix discipline for any kind of structured artifact — papers, projects, experiments, datasets, ADRs, runbooks, design docs, anything you'd write down and want to enforce structure + content quality on over time.

You write a `<name>_contract.md` describing what an artifact must contain. The skill audits artifacts against it (structure + semantics in one Claude pass), can apply migrations when the contract changes, and verifies its own paired auditor scripts stay in sync with the contract. Self-healing.

Single-file drop-in: hand this `.md` to Claude with a workspace path and you get the full audit + fix loop.

---

## What's a contract

A contract is a markdown file `<name>_contract.md` describing what an artifact of that kind must contain:

- Structural rules — frontmatter keys, body sections, allowed enum values
- Cross-reference rules — wikilinks resolve, fields cross-validate
- Content quality rules — prose is on-topic, predictions are reasoned, etc.
- Banned content
- Change-management policy — versioning, CHANGELOG, migration registration

"Format" would be too weak a word — these are agreements that bind structure AND behavior AND change semantics. You audit a contract; you don't audit a format.

## Three-stage audit (one Claude command)

```
/audit
  │
  ├─► Phase 0  Contract-vs-checker drift detection
  │            (Claude reads <name>_contract.md + paired Python audit script;
  │             does the script implement every rule the contract states?
  │             does it check anything the contract doesn't require?)
  │            → if drift: tag finding migration_kind: checker_v_to_v_<name>
  │              → /audit --fix dispatches a Fixer → regenerates the Python
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

Phase 0 keeps the script in sync with the contract. Phase 1 keeps it fast. Phase 2 keeps it honest.

If a contract has no paired Python script, Phase 0 + 1 are skipped; Phase 2 alone catches everything (slower but equally thorough).

---

## Quick start

1. **Author one or more `<name>_contract.md` files** in your workspace. Model on the worked example for ADRs at the bottom of this file. Each contract describes one kind of artifact.

2. **Optional: ship paired `<name>_contract-audit.py` scripts** for cheap fast structural checks. Don't write them by hand — let `/audit --fix` regenerate them from the contract on first run via the Phase 0 Fixer.

3. **Run `/audit`** (or hand this file directly to Claude with a workspace path):

   ```
   /audit                          workspace-wide; Phase 0 + 1 + 2; report only
   /audit --fix                    + apply migration-tagged fixes (multi-round, max 3)
   /audit --paper <id>             scope: one paper (or --project, --experiment, --feature, --contract)
   /audit --explain                verbose; for each finding, show what "good" looks like
   /audit --cross                  include cross-artifact consistency checks
   /audit --new-contract <name>    scaffold a fresh <name>_contract.md from a brief
   /audit --dry-run --fix          plan only; don't dispatch
   /audit --json                   machine-readable
   ```

4. **Author migration prompts** at `<workspace>/.claude/migrations/<kind>.md` whenever you bump a contract version. Each migration_kind the audit emits MUST have a matching prompt file. The prompt template (per `## Reference: migration_contract` below) tells the Fixer subagent how to apply the change.

## Why this exists

Long-running workspaces accumulate hundreds of structured artifacts. Without contract enforcement they drift — sections rename, key fields go missing, prose becomes generic, cross-references break. With this skill, drift becomes a finding instead of a surprise. With `--fix`, drift becomes a self-healing event.

The skill itself is meant to be invariant: contract authors write contracts, the skill audits + heals. No per-project Python audit code to maintain — Phase 0 keeps the (optional) Python in sync with the contract automatically.

## Architecture invariants

- **Contracts are the only source of truth.** Python audit scripts, when present, are derived. Phase 0 detects + repairs drift.
- **Contracts self-describe.** Every `<name>_contract.md` follows `## Reference: contract_contract` (the meta-contract below). The skill audits its own examples against itself; if its own meta-spec is wrong, you'll see it.
- **One audit pass = one Claude read of (contract + artifact).** Structural and semantic checks aren't bifurcated; the contract has both, the auditor applies both.
- **Fixes are migration-prompt-driven.** No clever inference; the operator authors a migration prompt at `<workspace>/.claude/migrations/<kind>.md` and the Fixer applies it. New migrations are explicit; the audit never silently changes content.
- **Multi-round convergence.** Fixer applies, re-audits, re-fixes if new findings surfaced. Converges or escalates to `<workspace>/gates/<kind>-stuck.md` for human review.
- **Subagent dispatch via the parent's `Agent` tool.** The only cross-context-reliable path. `claude -p` from inside another Claude session inherits the parent harness sandbox and can't write reliably.

---

## Orchestrator

When you (Claude) are invoked as the contracts orchestrator (`/audit` or by being given this file with a workspace path), follow this procedure.

### Step 1 — Discover artifacts

Walk the workspace. For each `<name>_contract.md` you find, identify:
- Its path (e.g. `papers/paper_contract.md`)
- Its paired `<name>_contract-audit.py` (if present, same dir, same prefix)
- The set of artifact files governed by it — read the contract's `## What this governs` section as the authoritative source for the path/glob pattern

Use `Glob` + `Read` for discovery.

### Step 2 — Phase 0: contract-vs-checker drift detection (per contract pair)

For each `(<name>_contract.md, <name>_contract-audit.py)` pair where BOTH exist, dispatch ONE subagent in parallel using **`## Harness: contract↔checker drift (Phase 0)`** below verbatim as the prompt. Substitute `{{CONTRACT_SPEC}}` and `{{AUDIT_SCRIPT}}`.

Findings carry `migration_kind: checker_v_to_v_<name>`.

If `--fix`: when Phase 0 detects drift, also dispatch a Fixer using **`## Harness: checker regenerator (Phase 0 Fix)`** to bring the script back in sync. The Phase 0 Fixer must converge BEFORE Phase 1 runs (Phase 1 with a stale script is meaningless).

If a contract has NO paired Python script: skip Phase 0 for it. Phase 2 alone still runs.

### Step 3 — Phase 1: run the (verified) Python checkers

For each contract pair where the script exists AND Phase 0 passed (or was just fixed):

```bash
python3 <name>_contract-audit.py <each artifact path>
```

Capture per-artifact JSON. Parse findings.

If a contract has NO paired script: skip Phase 1 for it; Phase 2 catches structural issues too (slower).

### Step 4 — Phase 2: per-artifact LLM Auditor swarm

For EVERY artifact in scope, dispatch ONE Auditor subagent in parallel (single multi-call message) using **`## Harness: per-artifact Auditor (Phase 2)`** below verbatim. Substitute:
- `{{ARTIFACT_PATH}}` = the artifact's primary file (e.g. `summary.md`)
- `{{CONTRACT_SPEC}}` = its governing contract path
- `{{PHASE_1_FINDINGS}}` = JSON of structural findings already detected for this artifact (or `[]` if no Phase 1 ran)

Each subagent emits findings per the **`## Reference: finding_contract`** shape below.

### Step 5 — Cross-artifact checks (if --cross)

For each contract's `## Cross-reference rules` section that declares a multi-artifact relationship (e.g. paper.reproductions[].final_experiment matches the wikilinked experiment's actual latest Run), dispatch one Auditor subagent with the cross-check rule injected. Use the same Phase 2 harness; just expand the `{{ARTIFACT_PATH}}` to the related-set.

### Step 6 — Aggregate + report

Combine findings across Phase 0 + Phase 1 + Phase 2 + Cross. Apply dedup/sort/grouping rules from `## Reference: finding_contract`.

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

If `--explain`: for each finding, also include a 1-2 sentence explanation of why the rule matters + (when possible) an example fragment showing what "good" looks like for that rule. Pull the example from the contract's spec text or a known-clean artifact in the workspace.

### Step 7 — Fix dispatch (only if --fix)

For each unique `(migration_kind, target)` tuple from the aggregate findings:

1. Verify `<workspace>/.claude/migrations/<migration_kind>.md` exists. If not: skip THIS fix with a clear error in the report ("missing migration prompt template"). Continue with the others.

2. Read the migration prompt. Build the Fixer subagent prompt by combining **`## Harness: artifact-level migration fixer`** (the wrapper) below + the migration prompt's `## Process` (the per-migration logic). Substitute `{{TARGET}}`, `{{CONTRACT_SPEC}}`, `{{AUDIT_SCRIPT}}`, `{{RULE}}`, `{{MESSAGE}}`, `{{EVIDENCE}}`, `{{SUGGESTED_FIX}}`, `{{MIGRATION_PROMPT}}`.

3. Fan out ALL Fixer subagents in ONE multi-call message (true parallelism via the parent's Agent tool).

4. Each Fixer reports a JSON status (per `## Reference: finding_contract`). Collect.

### Step 8 — Multi-round convergence (only if --fix)

After Step 7, re-run Steps 2 + 3 + 4 (Phase 0 / 1 / 2 audit). Compare delta:

- **Migrations resolved**: previously-tagged findings now absent → success
- **Migrations attempted but failed**: still present → record per-fix failure note
- **New findings introduced**: a Fixer's edit caused new issues → flag PROMINENTLY in the report (this is a regression; the operator must review)

If new migration-tagged findings appeared (e.g. the Fixer fixed one issue but the post-state needs a follow-on migration), repeat Step 7 + 8 up to **3 rounds total**. Then stop and report whatever remains as "unresolved — manual fix or new migration prompt needed."

If still unresolved after 3 rounds: write `<workspace>/gates/<migration_kind>-stuck.md` describing which artifact is stuck, what migration was attempted, and what findings remain. The operator handles manually.

### Step 9 — Append to workspace log (only on --fix or --semantic)

```
## [<YYYY-MM-DD>] audit | scope=<scope>, phases=<phase0,phase1[,phase2,cross,fix]>
critical=<C> warn=<W>; fix-stage: <P> passed, <F> failed, <U> unresolved.
```

Bare `/audit` (no `--fix`, no `--semantic`) is read-only and does NOT append to the log.

### Forbidden (orchestrator)

- Don't run any Python audit script that failed Phase 0 (script is known-divergent from the contract; its findings would be wrong). Either fix it first (`--fix`) or skip Phase 1 for that contract pair this run.
- Don't fan out Fixers without verifying the migration prompt exists.
- Don't fan out subagents serially — one multi-call message per phase.
- Don't claim "fixes applied" without re-auditing in Step 8.
- Don't auto-commit. Leave the diff in the working tree.
- Don't promote semantic findings from WARN to CRITICAL on your own — the contract or workspace config decides.

### Failure modes (orchestrator)

- **Phase 1 script crashed** (non-zero exit, no JSON) → emit one CRITICAL finding "Phase 1 driver failed for <contract>" + skip Phase 1 for that contract; Phase 2 still runs.
- **Auditor subagent timeout** → record "Phase 2 timed out for <artifact>"; continue with rest.
- **Fixer introduced new CRITICAL** → flag prominently in report Step 6; the human reviews; do NOT auto-revert.
- **Migration prompt missing** for a recommended fix → skip THAT fix with a clear error message; continue with others.

---

## Harness: per-artifact Auditor (Phase 2)

> Copy this section verbatim as the subagent prompt when dispatching Phase 2 Auditors. Substitute `{{ARTIFACT_PATH}}`, `{{CONTRACT_SPEC}}`, `{{PHASE_1_FINDINGS}}`.

You are an **Auditor subagent** dispatched by the contracts skill orchestrator. You audit ONE artifact against ONE contract. Per Principle #1 (fresh context), you have no memory — read everything from disk.

### Substitution variables

- `{{ARTIFACT_PATH}}` — the file you're auditing (e.g. `papers/d82afb989-…/summary.md`)
- `{{CONTRACT_SPEC}}` — its governing contract (e.g. `papers/paper_contract.md`)
- `{{PHASE_1_FINDINGS}}` — JSON list of structural findings already emitted by the paired Python script for this artifact (may be empty / `[]`). Use to avoid duplicating findings.

### Your task

Read `{{CONTRACT_SPEC}}` end-to-end. Read `{{ARTIFACT_PATH}}` end-to-end. Apply ALL of the following checks:

#### A. Structural checks (skip any already in {{PHASE_1_FINDINGS}})

For each rule in the contract that's deterministic — required frontmatter keys, required body sections in order, allowed enum values, cross-reference resolution, banned content patterns, sibling files — verify the artifact complies. If `{{PHASE_1_FINDINGS}}` already covers a rule, skip it (don't duplicate). If `{{PHASE_1_FINDINGS}}` is empty (no paired Python script), apply ALL structural checks yourself.

#### B. Semantic checks (always run)

Read the contract's `## Semantic audit checks` section. For each item, evaluate the artifact's content against it. Be specific:

- Cite evidence (short quote or section reference from the artifact)
- Don't be vague ("could be better") — every finding names what's wrong + suggests a fix
- Don't rationalize a violation — your job is anti-sycophancy. The contract is the contract.

#### C. Generic content-vs-spec judgment (always run)

Independent of the explicit checklist, apply judgment:

- **Section-existence vs section-content** — Phase 1 verifies a section header exists. You verify the BODY of the section actually says what the contract describes — not just placeholder text or off-topic content.
- **Sycophancy / motivated reasoning** — particularly in any prose section that follows a falsifiable claim (predictions, hypotheses, Run::Observations, Results discussion). Flag rationalizations of failure.
- **Generic-vs-specific** — spec asks for the architecture; the artifact says "neural network." Flag.
- **Pre-registration drift** — if the artifact has BOTH pre-registered fields (predictions, parent_gap, etc.) AND post-result fields (Run entry, Results discussion), check that the post-result content engages honestly with the pre-registered claim — neither editing the prediction to match the result NOR ignoring the prediction.
- **Wikilink semantic sense** — Phase 1 verifies links resolve. You verify they make contextual sense.

### Migration tagging

When a finding's fix is mechanical/repeatable across artifacts (e.g. "rewrite this generic prediction_rationale to reference the parent run with specifics"), set `migration_kind: <some-stable-name>` on the finding. The orchestrator's `--fix` mode will look for `<workspace>/.claude/migrations/<migration_kind>.md` and dispatch a Fixer.

For one-off content issues with no general fix recipe, leave `migration_kind` unset.

### Output

Emit ONE JSON object (single line) as your last response:

```json
{
  "audit_id": "phase2-<artifact-basename>-<UTC-timestamp>",
  "phase": 2,
  "artifact": "{{ARTIFACT_PATH}}",
  "contract": "{{CONTRACT_SPEC}}",
  "findings": [<one per finding shape; see ## Reference: finding_contract>],
  "summary": {
    "critical_count": <int>,
    "warn_count": <int>,
    "migration_recommendations": <int>
  }
}
```

Then stop. Don't commit, don't edit anything beyond optionally writing the JSON to `audits/<YYYY-MM-DD>/semantic/<artifact-id>.json` if the orchestrator told you to.

### Forbidden (Auditor)

- Don't modify the artifact (you're an Auditor, not a Fixer).
- Don't promote findings from WARN to CRITICAL on your own.
- Don't duplicate findings already in `{{PHASE_1_FINDINGS}}`.
- Don't hand-wave on content. If you can't quote evidence, it's not a finding.

---

## Harness: contract↔checker drift (Phase 0)

> Copy this section verbatim as the subagent prompt when dispatching Phase 0 drift detectors. Substitute `{{CONTRACT_SPEC}}` and `{{AUDIT_SCRIPT}}`.

You are an Auditor subagent for **Phase 0**: detecting drift between a contract spec and its paired Python audit script. Fresh context.

### Substitution variables

- `{{CONTRACT_SPEC}}` — the contract markdown
- `{{AUDIT_SCRIPT}}` — the paired Python script

### Your task

Read both files. For every deterministically-checkable rule the contract states, verify the script implements it. For every check the script implements, verify the contract justifies it. Report drift in either direction.

### What to check

#### Rules the contract states (script MUST implement)

From `{{CONTRACT_SPEC}}`, enumerate:

1. **Required frontmatter keys** — every key in `## Required frontmatter` should be in the script's `REQUIRED_FRONTMATTER_KEYS` set (or equivalent).
2. **Required body sections + order** — every section in `## Required body sections (in order)` should be in the script's required-sections list, in the same order.
3. **Enum-valued fields** — every `<field>: <enum>` constraint should map to a check.
4. **Wikilink resolution** — every "wikilinks resolve" requirement in `## Cross-reference rules` should have a script check.
5. **Cross-field constraints** — e.g. "headline_metric.model_id must match a models[].id" — script must enforce.
6. **Banned content patterns** — any specific banned strings/patterns from `## Banned content` should be flagged by the script.
7. **Sibling files** — files listed as required should be checked for existence.

For each rule the contract states that the script doesn't implement: emit a finding with `severity: CRITICAL` (the script is the load-bearing fast path; if it doesn't enforce a rule, artifacts will silently violate it).

#### Checks the script implements (contract MUST justify)

From `{{AUDIT_SCRIPT}}`, enumerate every check. For each, search the contract for a matching rule:

- Script checks for `key X` but contract doesn't list it as required: emit `severity: WARN` (extra check; either the contract should require it or the check should be removed).
- Script enforces enum `{a, b}` but contract specifies `{a, b, c}`: emit `severity: CRITICAL` (script more restrictive than contract — artifacts complying with contract will be wrongly flagged).
- Script applies a regex / format constraint not stated in contract: emit `severity: WARN`.

#### Style / API consistency

- Script's `Finding` dataclass / output JSON shape matches `## Reference: finding_contract`. If not: WARN.
- Script's exit codes match convention (0 = clean, 1 = CRITICAL, 2 = script error). If not: WARN.

### Migration tagging

If you find ANY drift, tag the findings with `migration_kind: checker_v_to_v_<contract-name-without-_contract>`. E.g. for `paper_contract.md ↔ paper_contract-audit.py` drift, the kind is `checker_v_to_v_paper`.

### Output

```json
{
  "audit_id": "phase0-<contract-basename>-<UTC-timestamp>",
  "phase": 0,
  "contract": "{{CONTRACT_SPEC}}",
  "audit_script": "{{AUDIT_SCRIPT}}",
  "findings": [
    {
      "rule": "contract-vs-checker-drift",
      "severity": "CRITICAL" | "WARN",
      "path": "{{AUDIT_SCRIPT}}",
      "message": "<concrete: what's missing or extra>",
      "evidence": "<quote from contract or script showing the divergence>",
      "suggested_fix": "<actionable>",
      "migration_kind": "checker_v_to_v_<contract>"
    }
  ],
  "summary": {
    "critical_count": <int>,
    "warn_count": <int>,
    "drift_detected": <bool>
  }
}
```

If no drift detected: emit empty `findings`, `summary.drift_detected: false`.

### Forbidden (Phase 0)

- Don't modify the script (Fixer's job).
- Don't tolerate "small" drifts. Either it implements the rule or it doesn't.
- Don't speculate beyond evidence. If a check could be in a helper function, read the helper. If you genuinely can't determine, emit a WARN with `message: "Could not determine whether script enforces <rule X>; manual review needed."`

---

## Harness: checker regenerator (Phase 0 Fix)

> Copy this section verbatim as the subagent prompt when dispatching Phase 0 Fixers (regenerating a Python audit script from its contract). Substitute `{{TARGET}}`, `{{CONTRACT_SPEC}}`, `{{RULE}}`, `{{MESSAGE}}`, `{{EVIDENCE}}`, `{{SUGGESTED_FIX}}`.

You are a **Fixer subagent** dispatched to repair (or generate) the paired Python audit script for a contract that has drifted (or has none yet). Fresh context.

### Substitution variables

- `{{TARGET}}` — the path to the Python audit script to write (e.g. `papers/paper_contract-audit.py`)
- `{{CONTRACT_SPEC}}` — the contract you're implementing
- `{{RULE}}`, `{{MESSAGE}}`, `{{EVIDENCE}}`, `{{SUGGESTED_FIX}}` — Phase 0 finding details

### Your task

Generate (or regenerate, if `{{TARGET}}` exists) the Python audit script so it implements EVERY deterministically-checkable rule from `{{CONTRACT_SPEC}}` and NOTHING extra.

#### What "implement" means per rule type

| Contract section | Script must |
|---|---|
| `## Required frontmatter` | Parse the frontmatter block, check every required key is present |
| `## Required body sections (in order)` | Scan body headers, verify each required section is present in the specified order |
| Enum-valued fields | Check the field value is in the allowed set |
| `## Cross-reference rules` | Resolve wikilinks (or paths) and verify the target exists; for cross-field rules, check the cross-validation |
| `## Banned content` | Scan for banned patterns (regex or string match) and emit finding if present |
| `## Sibling files` | Check each required sibling file exists in the artifact dir |

#### What "nothing extra" means

If a check is in the existing script that the contract doesn't justify, REMOVE it. The script is a derivative of the contract; it has no independent authority.

### Process

1. Read `{{CONTRACT_SPEC}}` end-to-end. Internalize every rule.
2. Read `{{TARGET}}` (if it exists) for structural reference. If it doesn't exist, build from scratch following the standard structure below.
3. Generate the new script:
   - Stamp `# derived_from_contract: <relative path>` near the top.
   - Implement each rule with a per-finding emission tagged with the rule citation.
   - Match the existing style (Python 3.10+, `from __future__ import annotations`, `dataclass` for Finding, JSON output, exit codes 0/1/2).
4. Validate by re-running Phase 0 against the new pair `({{CONTRACT_SPEC}}, {{TARGET}})`. Expected: zero drift findings.
5. Validate by running the new script against a known-clean artifact. Expected: exit 0, zero CRITICAL.

### Standard structure

```python
#!/usr/bin/env python3
# derived_from_contract: <relative/path/to/<name>_contract.md>
"""Deterministic audit (Phase 1) for <kind> per <name>_contract.md."""
from __future__ import annotations

import json
import re
import sys
from dataclasses import dataclass, asdict
from pathlib import Path

CRITICAL = "CRITICAL"
WARN = "WARN"

REQUIRED_FRONTMATTER_KEYS = {...}
REQUIRED_BODY_SECTIONS = [...]
# Other constants derived from contract: enums, regex patterns, etc.

@dataclass
class Finding:
    rule: str
    severity: str
    path: str
    message: str
    suggested_fix: str = ""
    migration_kind: str = ""

def audit(artifact_path: Path) -> list[Finding]:
    findings = []
    # ... per-rule checks, each emitting a Finding with rule citation ...
    return findings

def main(argv: list[str]) -> int:
    if len(argv) < 2 or argv[1] in ("-h", "--help"):
        print(__doc__)
        return 2
    artifact_path = Path(argv[1])
    findings = audit(artifact_path)
    critical = sum(1 for f in findings if f.severity == CRITICAL)
    warn = sum(1 for f in findings if f.severity == WARN)
    print(json.dumps({
        "audit": "<contract-name>",
        "target": str(artifact_path),
        "critical_count": critical,
        "warn_count": warn,
        "findings": [asdict(f) for f in findings],
    }, indent=2))
    return 1 if critical > 0 else 0

if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

### Validation

```bash
python3 -c "import ast; ast.parse(open('{{TARGET}}').read())"      # parse check
python3 {{TARGET}} --help                                          # CLI runs
python3 {{TARGET}} <a-known-clean-artifact-path>                   # clean → exit 0
```

If any fail: report `status: fail` and the error; don't keep going.

### Report back

```json
{"status": "pass" | "pass-with-warn" | "fail",
 "target": "{{TARGET}}",
 "rules_implemented": <int>,
 "rules_removed_from_old_script": <int>,
 "phase_0_residual_drift": <bool>,
 "phase_1_test_artifact": "<path>",
 "phase_1_test_exit_code": <int>,
 "note": "<one-sentence summary>"}
```

### Forbidden (Phase 0 Fixer)

- Don't add checks not in the contract.
- Don't keep checks the contract removed.
- Don't auto-commit.
- Don't break the existing public function signature (`audit(path) -> list[Finding]`).
- Don't emit findings whose `rule` field is generic.

---

## Harness: artifact-level migration fixer

> Copy this section verbatim as the wrapper subagent prompt when dispatching artifact-level Fixers. Combine with the `## Process` section of `<workspace>/.claude/migrations/<migration_kind>.md`. Substitute `{{TARGET}}`, `{{CONTRACT_SPEC}}`, `{{AUDIT_SCRIPT}}`, `{{RULE}}`, `{{MESSAGE}}`, `{{EVIDENCE}}`, `{{SUGGESTED_FIX}}`, `{{MIGRATION_PROMPT}}`.

You are a **Fixer subagent** dispatched to apply ONE migration to ONE artifact. Fresh context.

### Substitution variables

- `{{TARGET}}` — the artifact file to fix
- `{{CONTRACT_SPEC}}` — the contract the artifact must conform to after the fix
- `{{AUDIT_SCRIPT}}` — paired Python audit (use to validate post-fix); may be `null`
- `{{RULE}}` — the audit rule that triggered this fix
- `{{MESSAGE}}` — the violation summary
- `{{EVIDENCE}}` — short quote of the violating content
- `{{SUGGESTED_FIX}}` — actionable suggestion from the auditor
- `{{MIGRATION_PROMPT}}` — the per-migration prompt body from `<workspace>/.claude/migrations/<migration_kind>.md`

### Your task

You are a wrapper. The bulk of "what to do" lives in `{{MIGRATION_PROMPT}}`. Your job:

1. **Idempotence check** — read `{{TARGET}}`. If it already conforms to the post-migration state described in `{{MIGRATION_PROMPT}}::Process`: report `status: skip` and exit. Don't touch the file.

2. **Apply the migration** per `{{MIGRATION_PROMPT}}::Process`. Follow it; don't improvise.

3. **Self-validate** per `{{MIGRATION_PROMPT}}::Validation`. Typical:
   - Re-run `{{AUDIT_SCRIPT}}` against `{{TARGET}}`; expected to be clean for the rule that triggered this fix
   - The fix didn't introduce new CRITICAL findings on OTHER rules
   - The post-state matches what the migration prompt says it should produce

4. **Report back** per `{{MIGRATION_PROMPT}}::Report back` shape, plus the standard fields in `## Reference: finding_contract`.

### Convergence loop

If your fix introduces NEW migration-tagged findings that weren't there before — meaning a downstream migration is needed — that's reported in your `note` field. The orchestrator detects this in its multi-round convergence loop and may dispatch another Fixer.

You don't loop yourself. You apply ONE migration once. Convergence is the orchestrator's responsibility.

### Forbidden (artifact Fixer)

- Don't modify files OTHER than `{{TARGET}}` (and possibly its sibling files if the migration explicitly requires it). Cross-artifact updates are not your scope.
- Don't auto-commit. The parent reviews + commits.
- Don't fabricate data. If the migration requires content you don't have evidence for, report `status: fail` with a clear note.
- Don't apply your judgment about whether the migration is "really needed." The auditor + orchestrator already decided. You execute.
- Don't overwrite human-curated content unnecessarily. Migrations are enrichment + targeted-corrections.

---

## Reference: contract_contract (the meta-contract)

What makes a valid `<name>_contract.md`. Recursive — the skill audits its own example contracts against this.

### Required frontmatter

```yaml
---
name: <name>                      # the kind name; matches the file's <name>_contract.md
description: <one-line>           # what an artifact of this kind is
contract_version: v1              # bumped on any breaking change
created: <YYYY-MM-DD>
updated: <YYYY-MM-DD>
tags: [contract, <domain-tag>]
---
```

`contract_version` is the version under which artifacts of this kind are governed. Bumping it triggers (per Change policy below) a registered migration kind so existing artifacts can be brought up to date.

### Required body sections (in order)

A contract MUST have these top-level sections, in this order:

1. **`# <Name> contract`** — title (the H1)
2. **`## What this governs`** — one paragraph: what kind of artifact, where on disk, naming
3. **`## Required frontmatter`** — the YAML block + key list (required vs optional + types)
4. **`## Required body sections (in order)`** — the section structure (recursive — contracts have a section saying "these are the sections")
5. **`## Cross-reference rules`** — wikilink/path resolution, reciprocity, cross-artifact relationships. May be empty.
6. **`## Banned content`** — what's forbidden. May be empty.
7. **`## Semantic audit checks`** — list of concrete content checks the LLM Auditor evaluates. 5–10 items minimum. Each: `- **<short-name>** — <one sentence>`. **THIS** is what makes the contract more than a schema.
8. **`## Sibling files`** — other files an artifact directory MUST contain. Optional section if no required siblings.
9. **`## Change policy`** — how to evolve the contract: who can change it, what triggers a `contract_version` bump, what migration must accompany.
10. **`## CHANGELOG`** — bullet list of versions with one-line summaries.

### Cross-reference rules

A contract that requires fields referencing other artifacts MUST specify how the reference resolves:
- **Wikilink basename** (`[[<id>]]`) — vault-search resolution
- **Relative path** (`[[../<dir>/<file>]]`) — explicit; verifiable by file existence
- **Pure ID** — when reference is a string (e.g. `model_id: her-pr-linear`), the contract MUST specify the valid ID space (e.g. "must match a `models[].id` in this same file")

### Banned content (in any contract file)

- Domain-specific implementation details (project-specific paths, project-specific authority figures). Contracts must be re-usable across projects of the same kind.
- "TODO" / "TBD" / "XXX" markers in published versions.
- Generic semantic checks ("must be well-written"). Each `## Semantic audit checks` item must be specific enough that two reviewers would flag the same artifacts.

### Semantic audit checks (for the meta-contract itself)

- **section-order-respected** — required sections appear in the order specified.
- **frontmatter-keys-typed** — every required key has its expected type stated.
- **semantic-checks-non-empty** — `## Semantic audit checks` has ≥3 concrete items on any non-trivial contract.
- **change-policy-states-version-policy** — `## Change policy` describes when to bump `contract_version` and what triggers a migration.
- **CHANGELOG-current** — every `contract_version` value used has a corresponding bullet in `## CHANGELOG`.
- **examples-cited** — the contract refers to at least one concrete example artifact (or example fragment).
- **cross-references-typed** — fields referencing other artifacts specify what they reference and how the reference resolves.

### Change policy (for contracts themselves)

Bumping `contract_version` v<N> → v<N+1>:

1. Update the contract file (the new rules)
2. Add CHANGELOG entry (one line: what changed + why)
3. **Author a migration prompt** at `<workspace>/.claude/migrations/<name>_v<N>_to_v<N+1>_<change>.md` per the migration_contract reference below
4. Update the paired `<name>_contract-audit.py` (or let Phase 0 detect drift + Fixer regenerate)
5. Run `/audit --fix` against the workspace; existing artifacts get migrated

Skipping step 3 means existing artifacts will fail Phase 1 indefinitely with no automated path forward — the audit will keep flagging them as `migration_kind: <name>_v<N>_to_v<N+1>_<change>`, which the contract auditor itself flags ("migration kind referenced but no prompt registered"). The friction of writing the migration prompt IS the discipline.

### Sibling files

If artifacts of this kind have required siblings (e.g. paper.pdf alongside summary.md, manifest.json alongside fetch.py), the contract's `## Sibling files` section lists them with one-line descriptions.

---

## Reference: finding_contract

The shape of a single audit finding.

### Required fields

```yaml
rule: <string>                  # human-meaningful identifier of the violated rule.
                                # Convention: "<contract-name>.md§<section>#<short-name>"
                                # e.g. "paper_contract.md§Semantic audit checks#summary-on-topic"
                                # OR for structural: "<contract>.md§<section>"
                                # e.g. "paper_contract.md§Required frontmatter"

severity: <enum>                # CRITICAL | WARN
                                # CRITICAL → blocks dispatch (runner refuses to schedule
                                #   the experiment until resolved)
                                # WARN → logged, visible to user, doesn't block.
                                # Phase 2 LLM findings are WARN by default; CRITICAL elevation
                                #   only when the contract explicitly states "this rule is CRITICAL"
                                #   OR the workspace configures it via project-level overrides.

path: <string>                  # file path the finding pertains to (absolute or workspace-relative)

message: <string>               # one-sentence description of the violation, in plain prose.
                                # e.g. "Frontmatter missing required key: feature_dependencies"
                                # e.g. "Reproductions[0].status='attempted' but within_tolerance=true (inconsistent)"

evidence: <string>              # short quote (≤2 sentences) from the artifact showing the violation.
                                # For purely-structural findings, may be the missing-field name or
                                #   the offending section header.
                                # If multi-line: format as a fenced markdown code block.

suggested_fix: <string>         # actionable, specific. Not "fix this" but "rename X to Y"
                                #   or "add a `## Reproductions` section per
                                #   <contract>.md§Required body sections."

migration_kind: <string>        # OPTIONAL. When set, marks this finding as a candidate for
                                #   automated repair via a registered migration prompt at
                                #   `<workspace>/.claude/migrations/<migration_kind>.md`.
                                # Set when the fix is mechanical/repeatable across artifacts.
                                # Leave empty for one-off content issues.
```

### Two illustrative examples

**Phase 1 structural finding (auto-detectable):**

```yaml
rule: "paper_contract.md§Required frontmatter"
severity: CRITICAL
path: "papers/d82afb989-open-catalyst-experiments-2024-ocx24/summary.md"
message: "Frontmatter missing required key: contract_version"
evidence: "Frontmatter contains: name, description, tags, created, updated. Missing: contract_version."
suggested_fix: "Add `contract_version: v1` to frontmatter per paper_contract.md§Required frontmatter."
migration_kind: "paper_v0_to_v1_contract_version"
```

**Phase 2 semantic finding (LLM-judged):**

```yaml
rule: "experiment_contract.md§Semantic audit checks#prediction-rationale-non-generic"
severity: WARN
path: "projects/improve-co2rr/experiments/abc123def-foo/summary.md"
message: "prediction_rationale doesn't reference the parent run's diagnosis or any specific code change."
evidence: |
  ```
  prediction_rationale: |
    Based on intuition this should improve the result.
  ```
suggested_fix: "Rewrite the prediction_rationale to reference: (a) the parent run's specific diagnosis from its Run::Observations section, (b) the concrete file/function being changed, (c) a back-of-envelope estimate of the predicted gap reduction."
migration_kind: "rationale_needs_specifics"
```

### Aggregation rules

- **Dedup**: identical `(path, rule, message)` triples are merged.
- **Sort**: findings sorted first by `severity` (CRITICAL → WARN), then by `path` lexicographic.
- **Migration grouping**: findings with the same `migration_kind` are grouped in the report's "Migration recommendations" section so `/audit --fix` can fan out fixers per-kind.
- **Cross-artifact findings**: if a finding references multiple files, set `path` to the file authoritatively responsible (alphabetically lower path is convention).

### Forbidden in findings

- **Vague messages**: "this is wrong" / "needs improvement." Every finding names the specific rule + the specific violation.
- **Personal-style critiques**: "this is poorly written." Findings cite the rule.
- **Speculation without evidence**: every finding has an `evidence` field. If you can't quote/cite, it's not a finding.
- **Migration tags without registered prompt**: if you set `migration_kind: <X>`, a prompt at `<workspace>/.claude/migrations/<X>.md` must exist OR the finding itself flags the missing-prompt as a separate finding.

---

## Reference: migration_contract

The shape of a migration prompt at `<workspace>/.claude/migrations/<migration_kind>.md`.

A migration prompt is the LLM template a Fixer subagent reads when applying an automated repair. Each `migration_kind` registered in audit findings MUST have a matching prompt file, or `/audit --fix` refuses to act on those findings.

### File location and naming

`<workspace>/.claude/migrations/<migration_kind>.md`

The `<migration_kind>` is a stable string the auditor emits. Lowercase, snake_case. Conventions:

- `<contract>_v<N>_to_v<N+1>_<change>` for contract-version bumps (e.g. `paper_v2_to_v3_reproductions`)
- `<change>_needs_<noun>` for one-off content rules (e.g. `rationale_needs_specifics`)
- `checker_v_to_v_<contract>` for Phase 0 audit-script regeneration

### Required frontmatter

```yaml
---
migration_kind: <string>           # MUST match the filename (without .md)
description: <one-line>            # what this migration does
contract_version: <when-this-came-from>   # the contract version this migration targets
created: <YYYY-MM-DD>
applies_to: <enum>                 # artifact | audit_script | meta
                                   # artifact = fix runs against artifact files (most common)
                                   # audit_script = fix regenerates the paired Python audit script
                                   # meta = fix runs against a contract file itself (rare)
---
```

### Required body sections

1. **`# Migration: <migration_kind>`** — title H1 matching the kind
2. **`## When this fires`** — what audit finding triggers this migration. Cite the rule by id.
3. **`## Substitution variables`** — the values the auditor injects when dispatching the Fixer (standard set: `{{TARGET}}`, `{{CONTRACT_SPEC}}`, `{{AUDIT_SCRIPT}}`, `{{RULE}}`, `{{MESSAGE}}`, `{{EVIDENCE}}`, `{{SUGGESTED_FIX}}`; declare migration-specific extras here)
4. **`## Process`** — step-by-step what the Fixer does. Numbered list. Reference the substitution variables.
5. **`## Validation`** — how the Fixer verifies its work BEFORE reporting success. Typical: re-run `{{AUDIT_SCRIPT}}` and verify clean for THIS migration kind; re-run `/audit --paper {{TARGET}}` (or equivalent) and verify zero CRITICAL.
6. **`## Report back`** — the JSON shape the Fixer returns. Standard:
   ```json
   {"status": "pass" | "pass-with-warn" | "fail" | "skip",
    "target": "{{TARGET}}",
    "<migration-specific-stats>": <numbers>,
    "residual_warns": <int>,
    "residual_criticals": <int>,
    "note": "<one-sentence summary>"}
   ```
7. **`## Don't`** — anti-patterns. Examples: don't auto-commit, don't fabricate data, don't reset unrelated fields, don't delete content not part of the migration scope.

### Idempotence

The migration prompt MUST be idempotent. Running the same migration twice on a fixed artifact MUST be a no-op. The Fixer's first action: detect whether the target already conforms to the post-migration state; if yes, report `status: skip` and exit.

### Rollback

Auto-rollback is NOT a feature — if a Fixer mis-applies, the human reviews the diff before commit. The prompt SHOULD describe how to undo the change manually (one paragraph).

### Worked example: checker_v_to_v_paper

A migration that regenerates the paired Python audit script when the contract changed:

```markdown
---
migration_kind: checker_v_to_v_paper
description: Regenerate papers/paper_contract-audit.py from paper_contract.md when Phase 0 detects drift.
contract_version: paper_contract.md (any)
created: 2026-04-19
applies_to: audit_script
---

# Migration: checker_v_to_v_paper

## When this fires
Triggered when /audit Phase 0 finding `contract-vs-checker-drift` reports a divergence between paper_contract.md required-rules and paper_contract-audit.py implemented-rules.

## Substitution variables
- {{TARGET}} = papers/paper_contract-audit.py
- {{CONTRACT_SPEC}} = papers/paper_contract.md
- {{RULE}} = the Phase 0 rule citation
- {{MESSAGE}} = which rules are missing/extra in the script
- {{EVIDENCE}} = diff summary

## Process
1. Read {{CONTRACT_SPEC}} in full.
2. Read existing {{TARGET}} (if present).
3. Generate a new {{TARGET}} that implements every rule from the contract (see `## Harness: checker regenerator` standard structure).
4. Match the existing audit-script style.

## Validation
- python3 {{TARGET}} --help executes without error
- python3 {{TARGET}} <a-known-clean-paper> exits 0 with zero CRITICAL
- Re-run /audit Phase 0 to verify drift resolved

## Report back
{"status": ..., "target": "{{TARGET}}", "rules_added": <N>, "rules_removed": <N>, "residual_drift": <bool>}

## Don't
- Don't break the existing public function signature
- Don't auto-commit
- Don't add checks beyond what the contract states
```

### Migration registry hygiene

When a contract is bumped (per `## Reference: contract_contract::Change policy`), the author MUST register the corresponding migration prompt in this directory. Failure is itself a finding the contracts skill flags on every audit until resolved.

---

## Worked example

See `contracts_example_adr.md` (sibling file) for a full ADR contract demonstrating the `<name>_contract.md` shape this skill audits against. That example contract is itself audited against the meta-contract above; if `contracts_example_adr.md` ever drifts from `## Reference: contract_contract`, you'll see it in any `/audit --contract contracts_example_adr.md` run.
