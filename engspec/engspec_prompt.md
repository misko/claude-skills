# engspec: Code → English Specification

Use this document as instructions to produce a validated `.engspec` specification for any source file. Works with any language.

---

## How to use this

### Option A: You have source files ready

```
Here is my prompt: engspec_prompt.md
Here is my source file: <paste or attach file>
Here is project context: <brief description of what the project does, how to run tests, etc.>

Please produce a validated .engspec for this file.
```

### Option B: You have a local repo

```
Here is my prompt: engspec_prompt.md
Here is my repo at /path/to/repo
Please produce .engspec files for all source files.
```

### Option C: You only have a git URL (no input.md, no local files)

```
Here is my prompt: engspec_prompt.md

Please clone <git-url>, analyze the repo, and produce .engspec files for all source files.
```

### Option D: You want to spec a subset of files

```
Here is my prompt: engspec_prompt.md
Here is my repo at /path/to/repo
Please produce .engspec files for only: src/parser.py, src/lexer.py
```

### Option E: You want to update existing specs (incremental)

```
Here is my prompt: engspec_prompt.md
Here is my repo at /path/to/repo
Here is the existing engspec package at: /path/to/project-engspec/
Please update specs for files that have changed since the last generation.
```

When no `input.md` or project context is provided, **you must auto-generate it**.
See Phase 1 below.

---

## Phase 1: Project Context and Scope

Before analyzing code, establish context and determine scope.

### Determine scope

- **Full** (default): spec all source files in the repo
- **Partial** (Option D): spec only the files/modules/directories the user specified. Tag functions outside the spec boundary as `[external]` in Context sections.
- **Incremental** (Option E): load the existing engspec package, compare each spec's `source_commit` against `git log -1 --format=%H -- <file>` for the current source. Re-analyze only stale files (source changed since spec was written). Also scan the source repo for files that have no corresponding spec in the existing package — generate new specs for these. Preserve unchanged specs and their checksums.

### Establish context

Use whichever source is available, in priority order:

### If the user provided an `input.md`:
Read it and proceed to Phase 2.

### If the user provided a local repo path or git URL (no `input.md`):
Auto-generate context by examining the repo. Read these files (whichever exist):

1. **README.md**, **CONTRIBUTING.md**, **CHANGELOG.md** — project description, conventions
2. **Build files** — `pyproject.toml`, `Cargo.toml`, `package.json`, `go.mod`, `CMakeLists.txt`, `Makefile`, `justfile`
3. **CI configs** — `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`
4. **Test directories** — `tests/`, `test/`, `spec/`, `__tests__/` — scan for test patterns, frameworks, commands
5. **Source structure** — `ls` the tree to understand layout
6. **Git history** — `git log --oneline -20` for recent activity context
7. **Docstrings / comments** — skim main entry points for inline docs

From this, produce a **Project Context** summary (this is your auto-generated `input.md`):

```markdown
## Project Context (auto-generated)

**Project**: <name>
**Language(s)**: <detected languages>
**Description**: <what it does, key abstractions>
**Structure**: <key directories and what they contain>
**Test setup**: <how to install deps and run tests, or "no tests found">
**Entry point(s)**: <main files, CLI commands, etc.>
**Notes**: <anything else relevant — conventions, external deps, known quirks>
```

Share this summary with the user before proceeding, so they can correct anything.

### If the user provided individual files (no repo):
Infer what you can from the code itself. Note gaps ("no test information available — skipping test coverage analysis").

---

## Phase 2: Deep Analysis

For the source file(s) in scope, analyze and produce (share with the user):

### Call Graph
Trace the major call paths as trees:
```
function_a()
  → function_b()
    → function_c() [external]
  → function_d()
```

For partial packages, mark callees outside the spec boundary as `[external]`. Record these in `external_references` in the manifest.

For incremental updates, regenerate the **full** call graph and test coverage (not just for changed files) — read all existing specs in the package plus the newly changed specs to build a complete picture. Call edges and test coverage may have shifted even for unchanged files.

### Test Coverage
For each function in scope, note:
- What tests exist (file, what they check)
- What is NOT tested (gaps)

---

## Phase 3: Generate .engspec

Read `engspec_format.md` for the complete `.engspec` template, section rules, test file conventions, and versioning metadata. Follow it exactly.

Produce one `.engspec` file per source file in scope. For partial packages, only spec the target files. For incremental updates, only re-spec stale files — copy unchanged specs from the existing package. Do not invent your own format, reorder sections, rename headings, or add custom structure.

**Do NOT add a Debate Log section.** That is added later by `engspec_tester`, not during `code→engspec` generation.

### Versioning

For each `.engspec` file:
1. **`source_commit`** (file-level): If the source is in a git repo, record the current commit hash of the source file: `git log -1 --format=%H -- <source-file>`. Set this during Phase 3.
2. **`checksum`** (per-function): Compute after Phase 5 (Regeneration Verification) converges — this is when the spec is finalized. If Phase 4 (Self-Review) or Phase 5 modified the spec, the checksum must reflect the final version. Follow the canonical form defined in `engspec_format.md` and place it after the `##` heading.

---

## Phase 4: Self-Review

Before running regeneration verification, do an adversarial pass to catch issues cheaply. Fixing problems now avoids wasted regeneration cycles.

### Red Team (find problems)
For each function, ask:
- Is there an input that satisfies all preconditions but produces wrong output?
- Is there a postcondition that the implementation doesn't actually guarantee?
- Is there a failure mode not listed?
- Could a caller reasonably violate a precondition that isn't documented?

### Blue Team (defend or fix)
For each finding:
- Is it a real issue or a misreading of the spec?
- If real: update the spec with the missing information
- If not: explain why the spec is correct as-is

### Record findings
If you find real issues during self-review, fix them directly in the spec (update the relevant section). No need to log them — the formal Debate Log is added later by `engspec_tester`.

---

## Phase 5: Regeneration Verification

This is the critical validation step. The spec is not done until **3 consecutive regenerations all pass without any spec changes between them**. Any fixes applied during Phase 4 (Self-Review) are now tested here.

### Overview

```
regeneration_number = 0
consecutive_passes = 0

while consecutive_passes < 3:
    regeneration_number += 1
    if regeneration_number > 20:
        ERROR: spec failed to converge after 20 regenerations

    result = regenerate_and_compare()

    if result == PASS:
        consecutive_passes += 1
    else:
        fix the spec based on what was missing
        consecutive_passes = 0    # reset — need 3 clean in a row
```

**Hard limit: 20 total regenerations.** If you cannot achieve 3 consecutive passes within 20 regenerations, mark the spec as `status: failed` and report what could not be captured.

### Per-regeneration procedure

**Step 1: Launch a Regenerator agent**

Launch a **separate agent** with the `.engspec` content for the entire file being verified and the following instructions: "You are a Regenerator. Using ONLY the `.engspec` provided, implement every function and file-level section as working code. Follow the exact signatures from the `##` headings. Satisfy all preconditions, postconditions, invariants, and failure modes. Follow Implementation Notes for algorithmic choices. For test files, reproduce code blocks from Preconditions, Postconditions, and Implementation Notes verbatim — do not recompute expected values, re-derive assertions, or improvise mock behavior. When the spec references third-party library calls, you may introspect installed packages to verify API signatures (e.g., `python -c 'import inspect; ...'`). Return the complete source file." Do NOT pass original source code or source file paths. The agent has never seen the original source — its isolation is architectural, not behavioral. The agent MAY access installed third-party packages for API introspection, matching the conditions under which `engspec_to_code` operates.

Verification is **per-file**: the agent regenerates all functions in the file together, so file-level interactions (shared state, initialization order, mutual dependencies) are tested. If any function in the file fails comparison, the entire file fails and the consecutive pass counter resets.

Each regeneration must use a fresh agent — do NOT reuse agents from previous attempts. Agent infrastructure failures (API errors, timeouts) do not count toward the 20-attempt budget — retry the agent call.

**Step 2: Compare against original**

Back in the main context (which has the original source), compare the agent's reimplementation against the original:

| Check | Question |
|-------|----------|
| Preconditions | Does the reimplementation assume/check the same input constraints? |
| Postconditions | Does it guarantee the same outputs and effects? |
| Failure modes | Does it handle the same error cases? |
| Algorithmic properties | Are the core algorithmic behaviors preserved? |
| Side effects | Are side effects identical? |

Things that DON'T matter: code style, variable names, comments, import order, performance (unless specified in Implementation Notes).

**Step 3: Rule PASS or FAIL**

- **PASS**: The reimplementation preserves all essential properties. Increment `consecutive_passes`.
- **FAIL**: The reimplementation missed something because the spec was incomplete. Identify exactly what the spec failed to communicate. Update the spec. Reset `consecutive_passes` to 0.

### Convergence

The spec is **validated** when you achieve 3 consecutive PASS results with no spec edits in between. This proves the spec is sufficient for any independent reader to reproduce the behavior.

### Update metadata

After convergence (3/3 consecutive passes):
```markdown
<!-- status: validated -->
<!-- validated: <timestamp> -->
<!-- regeneration_count: <total attempts, e.g. 8> -->
<!-- regeneration_pass_rate: 3/3 -->
```

`regeneration_count` is the TOTAL number of regenerations performed (including failures). `regeneration_pass_rate` is always `3/3` for a validated spec — it means the final 3 consecutive attempts all passed.

If the spec fails to converge after 20 regenerations:
```markdown
<!-- status: failed -->
<!-- regeneration_count: 20 -->
<!-- regeneration_pass_rate: <best consecutive passes>/<required consecutive passes>, e.g. 2/3 -->
<!-- failure_reason: <what could not be captured> -->
```

### Example trace

```
Regeneration  1: FAIL — spec didn't mention clamping behavior → fix spec, reset to 0
Regeneration  2: FAIL — spec didn't specify collision order → fix spec, reset to 0
Regeneration  3: PASS — consecutive: 1
Regeneration  4: PASS — consecutive: 2
Regeneration  5: PASS — consecutive: 3 → VALIDATED (5 total, 3/3 final)
```

---

## Output Checklist

Before delivering the `.engspec` file, verify:

- [ ] Every public function/method in source files **in scope** has a section
- [ ] Every test file **in scope** has a `.engspec` (including conftest.py)
- [ ] File-level pseudo-functions exist for top-level execution, mutable globals, or interrelated constants (if applicable)
- [ ] All metadata comments are present and accurate
- [ ] Preconditions are specific (no vague "valid input")
- [ ] Postconditions cover return values AND side effects
- [ ] Failure modes list concrete exceptions/conditions
- [ ] Test strategy is actionable (someone could write tests from it)
- [ ] Implementation Notes are SHORTER than the source code
- [ ] Regeneration verification passed: 3 consecutive passes within 20 total attempts
- [ ] Status is `validated` with `regeneration_pass_rate: 3/3`
- [ ] The spec reads as self-contained English — no "see code for details"
- [ ] Every function has a `<!-- checksum: ... -->` computed per `engspec_format.md`
- [ ] File-level `<!-- source_commit: ... -->` is set (if in a git repo)

See `engspec_format.md` for complete examples of validated `.engspec` files (function-only, file with top-level code, post-audit with Debate Log).

---

## Phase 6: Package Output

After all `.engspec` files in scope are validated, produce a standalone output package.

### Partial and incremental packages

- **Partial**: the package contains specs only for the target files. Set `"coverage": "partial"` and populate `"scope"` and `"external_references"` in the manifest.
- **Incremental**: merge new/updated specs with unchanged specs from the existing package. Preserve unchanged spec files, their checksums, and their validation status. The call graph and test coverage are regenerated in Phase 2 (see above). Set `"coverage"` to match the existing package's coverage. If a source file was **deleted** since the last generation, remove its spec from the package and its entry from the manifest.
- **Full**: the package contains specs for every source file (the default).

### What to include

Create a folder named `<project-name>-engspec/` containing:

1. **`.engspec` files** for all source files in scope — mirroring the source directory structure. This includes specs for both source files AND test files.
2. **All non-code files from the repo** — copied verbatim, mirroring directory structure. These are files needed to regenerate a working project but that aren't source code:
   - Config files: `*.yaml`, `*.yml`, `*.toml`, `*.ini`, `*.cfg`, `*.conf`, `*.json`, `*.xml`
   - Documentation: `*.md`, `*.rst`, `*.txt` (README, CONTRIBUTING, CHANGELOG, etc.)
   - Build files: `Makefile`, `justfile`, `CMakeLists.txt`, `Dockerfile`, `docker-compose.yml`
   - CI/CD: `.github/workflows/*`, `.gitlab-ci.yml`, `.circleci/*`
   - Project metadata: `pyproject.toml`, `Cargo.toml`, `package.json`, `go.mod`, `go.sum`
   - Requirements/lockfiles: `requirements.txt`, `Cargo.lock`, `package-lock.json`, `poetry.lock`
   - Static assets: images, fonts, sounds, data files used by the project
   - Test fixtures and test data files (non-code: `.json`, `.env`, `.txt` fixtures — NOT test source code)
   - `.gitignore`, `.editorconfig`, `LICENSE`
3. **`project_context.md`** — the project context from Phase 1 (whether user-provided or auto-generated)
4. **`call_graph.md`** — the call graph analysis from Phase 2
5. **`test_coverage.md`** — the test coverage analysis from Phase 2
6. **`manifest.json`** — index of all files with validation status

**What NOT to include:**
- Source code files (`.py`, `.rs`, `.go`, `.c`, `.cpp`, `.js`, `.ts`, `.java`, etc.) — these are represented by `.engspec` files
- Test source code files — these are also represented by `.engspec` files and get regenerated
- Build artifacts (`__pycache__/`, `target/`, `node_modules/`, `build/`, `dist/`)
- `.git/` directory
- IDE files (`.vscode/`, `.idea/`)
- Environment files with secrets (`.env` with real credentials — include `.env.example` if it exists)

### Directory structure

```
<project-name>-engspec/
├── project_context.md          # Phase 1 output
├── call_graph.md               # Phase 2 call graph
├── test_coverage.md            # Phase 2 test coverage analysis
├── manifest.json               # Index of all specs + non-code files
├── specs/                      # .engspec files mirroring full source tree
│   ├── src/
│   │   ├── main.py.engspec
│   │   ├── utils.py.engspec
│   │   └── models/
│   │       └── player.py.engspec
│   └── tests/                  # Test file specs
│       ├── conftest.py.engspec
│       ├── test_main.py.engspec
│       └── test_utils.py.engspec
└── non_code/                   # Non-code files copied verbatim
    ├── README.md
    ├── LICENSE
    ├── pyproject.toml
    ├── requirements.txt
    ├── .gitignore
    ├── configs/
    │   └── settings.yaml
    ├── assets/
    │   └── sprite.png
    ├── tests/
    │   └── fixtures/
    │       └── sample.env      # Test data files (non-code only)
    └── .github/
        └── workflows/
            └── ci.yml
```

### manifest.json format

Follow the Manifest Schema defined in `engspec_format.md`. Populate `coverage`, `scope`, and `external_references` based on the scope determined in Phase 1. Include `source_commit` for every file entry (omit only if not in a git repo).

### Zip it

Compress the folder into `<project-name>-engspec.zip` and report the location.

```
pong-python-engspec.zip
├── project_context.md
├── call_graph.md
├── test_coverage.md
├── manifest.json
├── specs/
│   ├── pong.py.engspec
│   └── tests/
│       └── test_pong.py.engspec
└── non_code/
    ├── README.md
    ├── LICENSE
    ├── requirements.txt
    └── assets/
        └── beep.wav
```

### Why package separately

The `.engspec` package is a **standalone artifact**. It contains everything needed to:
- Understand the codebase without reading source code
- Re-implement any function from spec alone
- Assess test coverage gaps
- Feed into `engspec_tester` for adversarial analysis

It should be usable by someone who has never seen the original repo.

---

## Operational Requirements

This workflow requires the most capable available model for all agents and sub-agents. Spec quality degrades significantly with smaller or faster models — regeneration verification will fail more often, producing weaker specs.

When using Claude Code, pass `model: "opus"` for ALL agent invocations (Explore, Plan, general-purpose, and any other agent type). Do not use Sonnet or Haiku for any agent in this workflow.
