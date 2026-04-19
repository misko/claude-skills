# contract_contract

The meta-contract: what makes a valid `<name>_contract.md`.

Yes, it's recursive — the skill audits its own example contracts against this file. If any rule below isn't checkable by the auditor harness, the skill is broken.

## 1. File naming

`<name>_contract.md` where `<name>` is the artifact kind (singular noun, lowercase, snake_case for multiword): `paper_contract.md`, `experiment_contract.md`, `dataset_card_contract.md`, `runbook_contract.md`.

Optional: paired `<name>_contract-audit.py` for the fast Phase 1 structural checker. NOT required if you only want Claude-driven audits; recommended for workspaces with hundreds of artifacts where speed matters. When present, the skill audits the script against the contract on every run (Phase 0).

## 2. Required frontmatter

```yaml
---
name: <name>                      # the kind name; matches the file's <name>_contract.md
description: <one-line>           # what an artifact of this kind is
contract_version: v1              # bumped on any breaking change
created: <YYYY-MM-DD>
updated: <YYYY-MM-DD>
tags: [contract, <domain-tag>]    # at minimum #contract; add domain tags
---
```

Required keys: `name`, `description`, `contract_version`, `created`, `updated`, `tags`.

`contract_version` is the version under which artifacts of this kind are governed. Bumping it triggers (per §7) a registered migration kind so existing artifacts can be brought up to date.

## 3. Required body sections (in order)

A contract MUST have these top-level sections, in this order:

1. **`# <Name> contract`** — title (the H1)
2. **`## What this governs`** — one paragraph: what kind of artifact this contract applies to, where on disk it lives, how it's named
3. **`## Required frontmatter`** — the YAML block + key list with required vs optional + types
4. **`## Required body sections (in order)`** — the section structure (recursive — contracts have a section saying "these are the sections")
5. **`## Cross-reference rules`** — wikilink/path resolution, reciprocity, foreign-key style relationships across artifacts. May be empty section.
6. **`## Banned content`** — what's forbidden (e.g. copyrighted text in paper.md, modifying immutable layers). May be empty section.
7. **`## Semantic audit checks`** — list of concrete content checks the LLM Auditor evaluates. 5–10 items minimum on a non-trivial contract. Each item: `- **<short-name>** — <one sentence describing what to check>`. THIS is what makes the contract more than a schema; this is where content quality is enforced.
8. **`## Sibling files`** — other files an artifact directory MUST contain (or MAY contain) besides the artifact's own primary file. Optional section if there are no sibling files.
9. **`## Change policy`** — how to evolve the contract: who can change it, what triggers a `contract_version` bump, what migration must accompany a bump
10. **`## CHANGELOG`** — bullet list of versions with one-line summaries

## 4. Cross-reference rules

A contract that requires fields referencing other artifacts MUST specify how the reference is resolved:
- **Wikilink basename** (`[[<id>]]`) — vault-search resolution; works in Obsidian + the auditor
- **Relative path** (`[[../<dir>/<file>]]`) — explicit; verifiable by file existence
- **Pure ID** — when the reference is a string (e.g. `model_id: her-pr-linear`), the contract MUST specify what set of valid IDs exist (e.g. "must match a `models[].id` in this same file") so the auditor can cross-check.

## 5. Banned content

The skill itself bans these in any contract file:
- Domain-specific implementation details (e.g. project-specific paths, project-specific authority figures). Contracts must be re-usable across projects of the same kind.
- "TODO" / "TBD" / "XXX" markers in published versions. (Acceptable in a working draft, but `/audit` flags them.)
- Generic semantic checks ("must be well-written"). Each `## Semantic audit checks` item must be specific enough that two reviewers would flag the same artifacts.

## 6. Semantic audit checks (this contract's checks)

- **section-order-respected** — required sections appear in the order specified; section that's out of order is a finding.
- **frontmatter-keys-typed** — every required frontmatter key has its expected type stated (string / int / list / enum {a,b,c} / wikilink / etc.) — not just key name.
- **semantic-checks-non-empty** — `## Semantic audit checks` section has ≥3 concrete items on any non-trivial contract. Empty or single-item sections are flagged unless the contract is for a strictly mechanical artifact (e.g. a manifest).
- **change-policy-states-version-policy** — `## Change policy` actually describes when to bump `contract_version` and what triggers a migration; "minor changes don't bump" is fine but vague waffle is flagged.
- **CHANGELOG-current** — every `contract_version` value used anywhere in the contract has a corresponding bullet in `## CHANGELOG` with a one-line summary.
- **examples-cited** — the contract refers to at least one concrete example artifact (or example fragment), so an artifact author can see what "right" looks like.
- **cross-references-typed** — if frontmatter has fields like `<other_thing>: [[id]]`, contract specifies what other_thing is, where its id space lives, and how the reference resolves.

## 7. Change policy (for contracts themselves)

Bumping `contract_version` v<N> → v<N+1>:

1. Update the contract file (the new rules)
2. Add CHANGELOG entry (one line: what changed + why)
3. **Author a migration prompt** at `<workspace>/.claude/migrations/<name>_v<N>_to_v<N+1>_<change>.md` per `migration_contract.md`
4. Update the paired `<name>_contract-audit.py` if one exists (or let Phase 0 detect drift + Fixer regenerate)
5. Run `/audit --fix` against the workspace; existing artifacts get migrated by Fixer subagents per the new prompt

Skipping step 3 means existing artifacts will fail Phase 1 indefinitely with no automated path forward — the audit will keep flagging them as `migration_kind: <name>_v<N>_to_v<N+1>_<change>`, which is itself a finding the contract auditor flags ("migration kind referenced but no prompt registered"). The friction of writing the migration prompt IS the discipline.

## 8. Sibling files

If artifacts of this kind have required siblings (e.g. paper.pdf alongside summary.md, manifest.json alongside fetch.py), the contract's `## Sibling files` section lists them with one-line descriptions. The auditor verifies their presence via the structural Phase 1 checker.

## CHANGELOG

- **v1** (initial) — meta-contract for the contracts skill. Defines: required frontmatter, 10 required body sections in order, change-policy step requiring registered migration prompt, semantic checks for the contract itself.
