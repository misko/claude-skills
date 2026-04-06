# engspec tester: Adversarial Debate for Spec Validation

Use this document as instructions to run adversarial analysis on `.engspec` files. You will find spec gaps, ambiguities, internal contradictions, and missing edge cases through structured Red/Blue/Judge debate, then update the specs with your findings.

The `.engspec` format — template, metadata, versioning, checksum rules, round output format, and report template — is defined in `engspec_format.md`. Read it before proceeding.

**Two modes of operation:**
- **Spec + Source** (recommended): provide both the engspec package and the source repo. Red and Blue see source code. The Blind Judge still never sees source.
- **Spec only**: provide just the engspec package. All roles work from specs alone.

---

## How to use this

### Spec + Source (recommended)

```
Read /path/to/engspec_tester_prompt.md for your instructions.

Here is the engspec package at: /path/to/<project-name>-engspec/
Here is the source repo at: /path/to/repo

Please run adversarial analysis.
```

### Spec only

```
Read /path/to/engspec_tester_prompt.md for your instructions.

Here is the engspec package at: /path/to/<project-name>-engspec/

Please run adversarial analysis.
```

---

## Phase 1: Load and Understand

### Step 1: Read project context

If given a zip, extract it. Read `project_context.md`, `call_graph.md`, `test_coverage.md`, and `manifest.json`. Check `"coverage"` — if `"partial"`, load `"external_references"` to understand which callees are outside the spec boundary. If source repo is provided, verify that the `.engspec` files correspond to the source files.

### Step 2: Build the function map

For every `.engspec` file, build a map of: function name → full spec, source code (if available), callers/callees (from Context), and which spec file it lives in. Mark functions tagged `[external]` in Context sections — these have no spec and are outside the analysis boundary.

### Step 3: Identify analysis targets

Analyze **both source specs and test specs** (`"type": "source"` and `"type": "test"` in manifest). Test specs are where most regeneration failures occur — do not skip them.

**Prioritize:** (1) weak test coverage, (2) complex call chains, (3) error handling, (4) file-level pseudo-functions, (5) negative boundaries. For test specs: (1) complex setup/mocks, (2) computed expected values, (3) error path tests, (4) parametrized tests, (5) integration tests. Skip trivial functions where the spec is clearly complete.

---

## Phase 2: Batch into Subgraphs

Group functions into subgraphs of **at most 10** each. Source subgraphs are grouped by call graph (walk from roots, depth 3, split at module boundaries). Test subgraphs are grouped by the source module they cover, including the relevant source specs.

Debate source subgraphs first, then test subgraphs. Subgraphs can be processed in parallel within their category.

---

## Phase 3: Debate

For each subgraph, run the debate protocol. You play Red and Blue, then launch separate agents for the Judges. **This is the core of the analysis.**

### The Four Roles

| Role | Sees Source? | Runs As | Goal |
|------|-------------|---------|------|
| **Red** (Attacker) | Yes (if available) | Main context | Find gaps, ambiguities, composition failures |
| **Blue** (Defender) | Yes (if available) | Main context | Argue correctness, propose fixes |
| **Blind Judge** | **NEVER** | Separate agent | Rule using spec only — tests self-containment |
| **Sighted Judge** | Yes (if available) | Separate agent | Rule with source context — tests accuracy |

**Disagreements are the highest-value signal:**

| Blind | Sighted | Meaning | Action |
|-------|---------|---------|--------|
| SpecGap | NoIssue | Ambiguous spec, correct code | Clarify spec → tag `clarification` |
| NoIssue | SpecGap | Spec confidently wrong | Fix spec → tag `silent_divergence`, severity `critical` |

In spec-only mode, only the Blind Judge runs.

### Red Attack Strategies

**Source-dependent** (skip in spec-only mode):

| # | Strategy | Question |
|---|----------|----------|
| 1 | Spec-source divergence | Does source do something spec doesn't describe, or vice versa? |
| 2 | Undocumented behavior | Does source handle edge cases not in the spec? |
| 3 | Wrong negative boundaries | Does source implement behaviors Purpose says it excludes? |
| 4 | Missing error paths | Does source catch/handle errors not in Failure Modes? |
| 5 | Third-party API details | Does source call APIs with kwargs not in Implementation Notes? |

**Always applicable:**

| # | Strategy | Question |
|---|----------|----------|
| 6 | Usage-grounded analysis | Trace callers → find real value ranges → verify hardcoded limits. Apply the Hardcoded Limit Checklist in `engspec_format.md`. Zero-margin = minimum `major`. |
| 7 | Reimplementation test | Could two reasonable implementations from this spec differ? |
| 8 | Pre/postcondition consistency | Could a valid input produce output violating a postcondition? |
| 9 | Composition verification | Does each callee's postconditions satisfy its caller's preconditions? Skip for `[external]` callees (no spec). If source available, do best-effort check against source. |
| 10 | Failure mode completeness + defense chain | For each precondition: what happens when violated? Cite guard `file:line` (or "unguarded"). Require tests BEYOND limits, not just AT. Auto-escalate if unguarded AND untested beyond boundary. |
| 11 | Cross-function state | Do functions sharing mutable state have consistent invariants? |
| 12 | Test strategy gaps | Does Test Strategy cover all postconditions, failure modes, edge cases? |
| 13 | External boundary analysis | For each `[external]` callee: does the caller document what it expects from it? Does it handle failure modes from the external function? Flag missing boundary documentation as `composition_gap`. |

### Round Protocol

For each round, process every function in the subgraph:

**Step 1 — Red:** Produce 0-5 challenges. Each has: **category** (one of: `spec_source_divergence`, `spec_ambiguity`, `composition_gap`, `missing_failure_mode`, `postcondition_gap`, `negative_boundary_leak`, `invariant_conflict`, `test_coverage_gap`, `unguarded_constraint`), **concrete scenario**, **the gap**, and **severity**. Do not repeat prior findings.

**Severity** has two dimensions: *impact* (`critical` = incorrect regeneration or silent corruption; `major` = real edge case or zero-margin boundary; `minor` = unlikely or cosmetic) and *silent failure escalator* (auto-escalate one level if failure is silent, blocked only by citing the exact guard). Format: `[category/impact→escalated]` or `[category/impact]`. Track escalations in two buckets for the report: `silent-failure` (silent failure escalator applied) and `unguarded-untested` (strategy #10 defense chain escalator applied).

**Step 2 — Blue:** For each challenge: verdict (`agree`/`disagree`/`partial`), reasoning citing spec sections, and proposed fix if agreeing. **Guard citation required**: In spec+source mode, Blue must cite exact `file:line`. In spec-only mode, Blue must cite the exact spec section and bullet (e.g., "Failure Modes bullet 3 documents the ValueError guard"). Vague claims like "there's probably a check somewhere" are invalid in either mode — Judges treat uncited defenses as no defense offered.

**Step 3 — Sanitize challenges for Blind Judge:** Before launching the Blind Judge, produce a **spec-only version** of each Red challenge. Strip all source-specific references: file paths, line numbers, source code snippets, and descriptions of source behavior. Reframe each challenge as a spec concern only. Example:
- Full (for Sighted Judge): "Source at line 47 catches `ConnectionResetError` but spec only lists `IOError` in Failure Modes"
- Sanitized (for Blind Judge): "Failure Modes lists `IOError` but doesn't specify behavior for `ConnectionResetError` — a reasonable implementer might handle it differently"

Challenges that are already spec-only (e.g., `spec_ambiguity`, `composition_gap`) can be passed unchanged.

For `spec_source_divergence` challenges: reframe as a question about what the spec covers, without revealing what the source does. The Blind Judge's job is to determine if the spec *should* address the scenario, not whether the source is right. Example:
- Full: "Source silently discards malformed headers instead of raising ValueError"
- Sanitized: "Spec doesn't specify behavior when headers are malformed — a reasonable implementer could raise, discard, or log"

**Step 4 — Launch Judges as separate agents:**
- **Blind Judge:** Launch a separate agent. Pass it ONLY: function spec, `project_context.md`, **sanitized** challenges, Blue's responses, and the following instructions: "You are a Judge evaluating spec quality. For each challenge, rule: **SpecGap** (spec is incomplete or ambiguous — specify which section needs what), **NoIssue** (spec already covers this — cite the specific bullet), or **SpecConflict** (two parts of the spec contradict — identify both). Optionally recommend a TestAddition. You have no access to source code — rule based on the spec alone." No source code, no source paths, no unsanitized challenges.
- **Sighted Judge** (spec+source mode only): Launch a second separate agent with the same instructions as the Blind Judge, plus: **full** (unsanitized) challenges, and the relevant source code files. Add to the instructions: "You also have access to the source code. Use it to verify whether the spec accurately describes the implementation."

**Step 5 — Compare rulings:** Per the disagreement table above; prefer the stricter ruling on disagreements.

Follow the round output format defined in `engspec_format.md`.

---

## Phase 4: Apply Findings

After each round, apply findings immediately:

1. For each SpecGap/SpecConflict: update the `.engspec` file and add a Debate Log entry (see `engspec_format.md` for table schema)
2. For NoIssue with TestAddition: add to Test Strategy section
3. **Recompute checksums** for modified functions per `engspec_format.md`. Update per-function `audited` timestamps. Update `source_commit` if re-verified against source. Do NOT update the file-level `validated` timestamp — it refers to the original regeneration validation by `engspec_prompt`. If spec content changed significantly, note in the report that re-validation via `engspec_prompt` is recommended.

---

## Phase 5: Convergence

Deduplicate findings across rounds (same concern worded differently = duplicate).

**Convergence rules:**
- If a subgraph has **never had any findings** (0 findings in the first round), it is converged immediately — do not run empty rounds.
- If a subgraph **has had findings** (at least one finding in any round), it requires **3 consecutive clean rounds** (zero novel findings after dedup) to converge. This proves the fixes stabilized.
- **Hard cap**: 20 rounds per subgraph regardless.

Confidence score per subgraph:

Per-subgraph confidence:

```
confidence = 0.3 * round_factor + 0.4 * resolution_rate + 0.3 * clean_factor

round_factor    = min(total_rounds / 10, 1.0)
resolution_rate = resolved_findings / total_findings  (1.0 if no findings)
clean_factor    = min(consecutive_clean_rounds / 3, 1.0)

Exception: if a subgraph converged immediately (0 findings in first round),
set clean_factor = 1.0 (a single clean round is conclusive when there are
no fixes to verify).
```

Overall confidence (for the report): weighted average of per-subgraph scores, weighted by function count in each subgraph.

---

## Phase 6: Report

Produce a report following the Analysis Report Format in `engspec_format.md`. For partial packages, include an **External Boundaries** section listing each `[external]` callee, whether the boundary is documented in the caller's spec, and the assessed risk.

**Output artifacts:**
1. Updated `.engspec` files with Debate Log entries and fixes
2. Report: `<project-name>-analysis-report.md`
3. Updated package: `<project-name>-engspec-tested.zip`

---

## Key Principles

1. **Two Judges, two questions.** Blind tests self-containment, Sighted tests accuracy. Disagreements are highest-value findings.
2. **Concrete over abstract.** Every challenge needs a specific scenario, not "this might be ambiguous."
3. **Fix specs immediately.** Don't batch — fix after each round so subsequent rounds debate the improved spec.
4. **Don't repeat yourself.** Review all prior findings before each Red round.
5. **Composition is where gaps hide.** Cross-function precondition/postcondition mismatches are invisible in isolation.
6. **Reimplementation is the primary attack.** If two reasonable implementations from the spec would differ, it's a SpecGap.
7. **Source access amplifies Red.** Undocumented behavior and missing error paths are the findings that most improve spec quality.
8. **Severity matters.** One critical finding outweighs five minor cosmetic issues.
9. **"Currently works" is not a defense.** Zero-margin boundaries warrant minimum `major`. "It works in production" is irrelevant — specs must handle all valid inputs.

---

## Operational Requirements

This workflow requires the most capable available model for all agents and sub-agents. Spec quality degrades significantly with smaller or faster models.

When using Claude Code, pass `model: "opus"` for ALL agent invocations. Do not use Sonnet or Haiku for any agent in this workflow.
