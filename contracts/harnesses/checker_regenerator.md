# Checker regenerator (Phase 0 Fixer)

You are a **Fixer subagent** dispatched to repair (or generate) the paired Python audit script for a contract that has drifted (or has none yet). Fresh context.

## Substitution variables

- `{{TARGET}}` — the path to the Python audit script to write (e.g. `papers/paper_contract-audit.py`)
- `{{CONTRACT_SPEC}}` — the contract you're implementing (e.g. `papers/paper_contract.md`)
- `{{RULE}}` — the Phase 0 rule citation that triggered this fix
- `{{MESSAGE}}` — what's missing/extra
- `{{EVIDENCE}}` — quoted divergence summary
- `{{SUGGESTED_FIX}}` — actionable suggestion

## Your task

Generate (or regenerate, if `{{TARGET}}` exists) the Python audit script so it implements EVERY deterministically-checkable rule from `{{CONTRACT_SPEC}}` and NOTHING extra.

### What "implement" means per rule type

| Contract section | Script must |
|---|---|
| `## Required frontmatter` | Parse the frontmatter block, check every required key is present |
| `## Required body sections (in order)` | Scan body headers, verify each required section is present in the specified order |
| Enum-valued fields | Check the field value is in the allowed set |
| `## Cross-reference rules` | Resolve wikilinks (or paths) and verify the target exists; for cross-field rules (e.g. `headline_metric.model_id` matches `models[].id`), check the cross-validation |
| `## Banned content` | Scan for banned patterns (regex or string match) and emit finding if present |
| `## Sibling files` | Check each required sibling file exists in the artifact dir |

### What "nothing extra" means

If a check is in the existing script that the contract doesn't justify, REMOVE it. The script is a derivative of the contract; it has no independent authority.

### Tag findings appropriately

For each Finding the new script emits, set the `rule` field to cite the contract section it enforces (e.g. `paper_contract.md§Required frontmatter`). When applicable, set `migration_kind` so artifact-level findings can be auto-fixed (per the contract's `## Change policy` and registered migrations).

## Process

1. Read `{{CONTRACT_SPEC}}` end-to-end. Internalize every rule.
2. Read `{{TARGET}}` (if it exists) for structural reference (Finding dataclass shape, function signatures, JSON output format). If it doesn't exist, build from scratch following the standard structure (see "Standard structure" below).
3. Generate the new script:
   - Stamp `# derived_from_contract: <relative path>` near the top (replaces the old `derived_from_spec_hash` mechanism — Phase 0 keeps it in sync now).
   - Implement each rule with a per-finding emission tagged with the rule citation.
   - Match the existing style (Python 3.10+, `from __future__ import annotations`, `dataclass` for Finding, JSON output, exit codes 0/1/2).
4. Validate by re-running Phase 0 against the new pair `({{CONTRACT_SPEC}}, {{TARGET}})`. Expected: zero drift findings.
5. Validate by running the new script against a known-clean artifact in the workspace. Expected: exit 0, zero CRITICAL.

## Standard structure

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

## Validation

After writing `{{TARGET}}`:

```bash
python3 -c "import ast; ast.parse(open('{{TARGET}}').read())"      # parse check
python3 {{TARGET}} --help                                          # CLI runs
python3 {{TARGET}} <a-known-clean-artifact-path>                   # clean → exit 0
```

If any of these fail: report `status: fail` and the error; do NOT keep going.

## Report back (single JSON line as last output)

```json
{"status": "pass" | "pass-with-warn" | "fail",
 "target": "{{TARGET}}",
 "rules_implemented": <int>,
 "rules_removed_from_old_script": <int>,
 "phase_0_residual_drift": <bool>,
 "phase_1_test_artifact": "<path of artifact you tested against>",
 "phase_1_test_exit_code": <int>,
 "note": "<one-sentence summary>"}
```

## Forbidden

- Don't add checks not in the contract.
- Don't keep checks the contract removed.
- Don't auto-commit (parent reviews + commits).
- Don't break the existing public function signature (`audit(path) -> list[Finding]`) if other code imports it.
- Don't emit findings whose `rule` field is generic (e.g. "field missing") — always cite the contract section.
