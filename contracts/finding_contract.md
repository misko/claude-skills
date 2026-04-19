# finding_contract

The shape of a single audit finding.

Every Phase 0, Phase 1, or Phase 2 check that detects a violation emits ONE finding object. The auditor harness aggregates them into the report (per `report` semantics in `auditor_prompt.md`).

## Required fields (JSON / YAML object)

```yaml
rule: <string>                  # human-meaningful identifier of the violated rule.
                                # Convention: "<contract-name>.md§<section>#<short-name>"
                                # e.g. "paper_contract.md§Semantic audit checks#summary-on-topic"
                                # OR for structural violations: "<contract>.md§<section>"
                                # e.g. "paper_contract.md§Required frontmatter"

severity: <enum>                # one of: CRITICAL | WARN
                                # CRITICAL → blocks dispatch (runner refuses to schedule the
                                #   experiment until resolved)
                                # WARN → logged, visible to user, doesn't block
                                # Phase 2 LLM findings are WARN by default; CRITICAL elevation
                                #   only when the contract explicitly states "this rule is CRITICAL"
                                #   OR the workspace configures it via project-level severity overrides.

path: <string>                  # file path the finding pertains to (absolute or workspace-relative)
                                # for cross-artifact findings, the "primary" file where the
                                #   discrepancy was introduced

message: <string>               # one-sentence description of the violation, in plain prose.
                                # Template: "<noun> <verb> <expected vs actual>"
                                # e.g. "Frontmatter missing required key: feature_dependencies"
                                # e.g. "Reproductions[0].status='attempted' but within_tolerance=true (inconsistent)"

evidence: <string>              # short quote (≤2 sentences) from the artifact showing the
                                #   violation. For purely-structural findings, may be the
                                #   missing-field name or the offending section header.
                                # If multi-line, format as a fenced markdown code block.

suggested_fix: <string>         # actionable, specific. Not "fix this" but "rename X to Y"
                                #   or "add a `## Reproductions` section per
                                #   <contract>.md§Required body sections."

migration_kind: <string>        # OPTIONAL. When set, marks this finding as a candidate for
                                #   automated repair via a registered migration prompt at
                                #   `<workspace>/.claude/migrations/<migration_kind>.md`.
                                # Set when the fix is mechanical/repeatable across artifacts.
                                # Leave empty for one-off content issues.
```

## Two illustrative examples

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

## Aggregation rules

The auditor harness COLLECTS findings from all subagents into the report. Per-finding rules:

- **Dedup**: identical `(path, rule, message)` triples are merged.
- **Sort**: findings sorted first by `severity` (CRITICAL → WARN), then by `path` lexicographic.
- **Migration grouping**: findings with the same `migration_kind` are grouped in the report's "Migration recommendations" section so `/audit --fix` can fan out fixers per-kind.
- **Cross-artifact findings**: if a finding references multiple files (e.g. reciprocity violation between paper A and paper B), set `path` to the file authoritatively responsible (alphabetically lower path is convention).

## Forbidden in findings

- **Vague messages**: "this is wrong" / "needs improvement" / "could be better." Every finding must name the specific rule + the specific violation.
- **Personal-style critiques**: "this is poorly written." If the contract has a `## Semantic audit checks` item like "summary-on-topic", the finding cites that rule; the message is about the rule, not the author.
- **Speculation without evidence**: every finding has an `evidence` field. If you can't quote/cite the violation, it's not a finding.
- **Migration tags without registered prompt**: if you set `migration_kind: <X>`, a prompt at `<workspace>/.claude/migrations/<X>.md` must exist OR the finding itself flags the missing-prompt as a separate finding (`rule: contracts§Change policy#migration-prompt-registered`).
