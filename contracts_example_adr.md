---
name: adr
description: An Architectural Decision Record (ADR) — short document capturing one significant architectural choice, its context, the alternatives considered, and the consequences.
contract_version: v1
created: 2026-04-19
updated: 2026-04-19
tags: [contract, adr, architecture]
---

# ADR contract

> Worked example for the [contracts skill](contracts.md). Demonstrates the `<name>_contract.md` shape the skill audits artifacts against. This contract itself conforms to `contracts.md§Reference: contract_contract` (the meta-contract); running `/audit --contract contracts_example_adr.md` on this file should be clean.

## What this governs

`docs/adr/<NNNN>-<kebab-slug>.md` files. One ADR per file. The number `<NNNN>` is monotonic (zero-padded 4-digit), assigned at creation. The slug is a short human-readable summary of the decision (e.g. `0007-use-postgres-for-events.md`).

ADRs are immutable in spirit: once accepted, an ADR is not edited beyond corrections + status changes. Reversing an ADR means writing a NEW ADR that supersedes it (linked via the `superseded_by` field).

## Required frontmatter

```yaml
---
id: <NNNN>                              # int, zero-padded 4 digits in filename, integer here
title: <one-line decision summary>      # string
status: <enum>                          # one of: proposed | accepted | superseded | deprecated
date: <YYYY-MM-DD>                      # decision date (proposed → accepted)
deciders: [<name1>, <name2>, ...]       # list of strings; can be roles ("platform team")
contract_version: v1
tags: [adr, <domain-tags>]
supersedes: <NNNN | null>               # ADR id this one replaces, if any
superseded_by: <NNNN | null>            # ADR id that replaces this one (set when later ADR supersedes)
related: [<NNNN>, ...]                  # IDs of related ADRs (informational, not supersession)
---
```

Required keys: `id`, `title`, `status`, `date`, `deciders`, `contract_version`, `tags`. Optional: `supersedes`, `superseded_by`, `related`.

`status: proposed` — drafted, not yet decided. May be edited freely.
`status: accepted` — decided. Frozen except for status field changes (→ superseded / deprecated) + minor corrections.
`status: superseded` — replaced by another ADR; `superseded_by` MUST be set.
`status: deprecated` — no longer applicable, no replacement; `note: <reason>` strongly recommended.

## Required body sections (in order)

1. **`# ADR <NNNN>: <title>`** — H1 mirroring the title
2. **`## Context`** — the problem + the conditions that forced the decision; what would happen without one
3. **`## Decision`** — the choice made, in active voice ("we will use Postgres for the event log")
4. **`## Alternatives considered`** — at least 2 alternatives + why each was rejected. Each as a `### <name>` subsection
5. **`## Consequences`** — what this enables, what it constrains, follow-on work it requires. Both positive and negative
6. **`## References`** — links to related design docs, prior ADRs, external resources. Use wikilinks for in-repo refs

## Cross-reference rules

- `supersedes: <NNNN>` MUST resolve to an existing ADR file (`docs/adr/<NNNN>-*.md`).
- `supersedes: <NNNN>` requires the target ADR's `superseded_by` field to be set to THIS ADR's id (reciprocity).
- `superseded_by: <NNNN>` MUST resolve to an existing ADR; reciprocity required.
- `related: [<NNNN>, ...]` IDs MUST resolve. No reciprocity required (informational).

## Banned content

- Tense drift in `## Decision` — must be active voice ("we will X"), not "we should X" or "we should consider X."
- Vendor-locked decisions in `proposed` status without an alternatives-considered comparison vs at least one open alternative.
- "TBD" / "TODO" markers in any section once `status: accepted`.
- Editing the `## Decision` or `## Context` sections after `status: accepted` (use a new ADR that supersedes).

## Semantic audit checks

- **decision-is-decision** — `## Decision` actually states a decision (active, falsifiable). Not "we discussed Postgres vs DynamoDB" but "we will use Postgres."
- **alternatives-real** — `## Alternatives considered` lists ≥2 alternatives that are credible (not strawmen). Each has a 1-paragraph rejection rationale that engages with the alternative's strengths, not just its weaknesses.
- **consequences-include-negatives** — `## Consequences` lists at least one negative consequence. ADRs that only list upsides are red flags (motivated reasoning).
- **supersedes-implies-replacement** — if `status: accepted` AND `supersedes: <NNNN>`, the body explains WHAT the new decision changes vs the old (one paragraph minimum). "We're now using Y instead of X because…" not "supersedes ADR-0007."
- **context-not-solution** — `## Context` describes the problem space, not the decision. If `## Context` reads "we need to use Postgres because…" it's leaking the answer; rewrite to describe the forcing function only.
- **deciders-non-empty** — `deciders: []` is forbidden. At least one deciding person/team named (anonymity is fine: "the data team" but not blank).
- **status-frozen-fields-untouched** — for `status: accepted` ADRs, `## Decision` and `## Context` haven't been edited since the date. (Detectable from git log; auditor flags suspected drift.)
- **wikilinks-resolve** — every wikilink in `## References` resolves to an existing repo file. Phase 1 verifies; you verify they make CONTEXTUAL sense.

## Sibling files

ADRs are single-file artifacts; no required siblings.

`docs/adr/README.md` (project-level) is recommended as a navigation index but isn't governed by this contract.

## Change policy

This contract evolves rarely. To bump `contract_version`:

1. Update this file with the new rules + bump `contract_version`
2. Add CHANGELOG entry (one line)
3. Author migration prompt at `<workspace>/.claude/migrations/adr_v<N>_to_v<N+1>_<change>.md`
4. Run `/audit --fix` against `docs/adr/`; existing ADRs migrate via the Fixer

For ADRs that need to change due to a real architectural reversal (not contract evolution): write a new ADR that supersedes the old one. The old one's `## Decision` and `## Context` stay frozen — its `status` flips to `superseded` and `superseded_by` gets set. That's the discipline that makes the ADR log a useful history.

## CHANGELOG

- **v1** (2026-04-19) — initial contract for ADRs. Inspired by Michael Nygard's classic ADR template + lessons from real-world decision-record drift (negative consequences ignored, alternatives strawmanned, context leaking the answer). Adds explicit semantic checks for those failure modes.
