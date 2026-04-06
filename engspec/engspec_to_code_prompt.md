# engspec→code: Regenerate Code from English Specifications

Use this document as instructions to regenerate a full, working codebase from an `.engspec` package. You will produce source code files from specs alone, set up the project, and run tests to verify correctness.

The `.engspec` format — template, metadata, versioning, and checksum rules — is defined in `engspec_format.md`. Read it before proceeding.

---

## How to use this

```
Read /path/to/engspec_to_code_prompt.md for your instructions.

Here is the engspec package: /path/to/<project-name>-engspec.zip

Please regenerate the full codebase and run tests.
```

Or if the zip is already extracted:

```
Read /path/to/engspec_to_code_prompt.md for your instructions.

Here is the engspec package at /path/to/<project-name>-engspec/

Please regenerate the full codebase and run tests.
```

### Partial package (some source files already exist)

```
Read /path/to/engspec_to_code_prompt.md for your instructions.

Here is the engspec package: /path/to/project-engspec/
Here is the existing codebase at: /path/to/existing-repo/

Please regenerate only the spec'd files into the existing codebase.
```

---

## Phase 1: Unpack and Understand

### Step 1: Extract the package

If given a `.zip`, extract it. You should find:

```
<project-name>-engspec/
├── project_context.md
├── call_graph.md
├── test_coverage.md
├── manifest.json
├── specs/
│   └── ... .engspec files mirroring source tree
└── non_code/
    └── ... config, docs, assets, build files copied verbatim
```

### Step 2: Read project context

Read `project_context.md` first. This tells you:
- What the project does
- What language(s) it uses
- Project structure and key directories
- Dependencies and build system
- How to set up the environment
- How to run tests

### Step 3: Read manifest

Read `manifest.json` to get:
- The full list of spec files to regenerate, their languages, and validation status. Only regenerate from `"status": "validated"` specs. For `"status": "failed"` specs that are imported by other files (check the call graph): generate a **stub file** containing only the function signatures and type definitions from the spec's `##` headings. For source file stubs, use `raise NotImplementedError('Spec not validated — stub only')` for bodies. For test file stubs, use the language's skip mechanism instead (e.g., `pytest.skip('Spec not validated')` for Python, `t.Skip()` for Go) so they don't pollute test results with false failures. Report all stubs in the validation report.
- The full list of `non_code_files` — these will be copied verbatim to their original paths in the regenerated project.

Check the `"coverage"` field. If `"partial"`, read `"external_references"` to identify source files that must already exist — they are not regenerated from spec. The user must provide the existing codebase path for these.

Also note the versioning metadata in each `.engspec` file (see `engspec_format.md`): `source_commit` tells you which source version the spec was written against, per-function `checksum` lets you verify spec integrity, and `audited` tells you when the tester last reviewed each function. Report any integrity issues (mismatched checksums, stale source_commit) in the validation report.

### Step 4: Read call graph

Read `call_graph.md` to understand the dependency order. You will regenerate files in **dependency order** — files with no dependencies first, then files that depend on them, etc. This ensures that when you regenerate a function that calls another, you already know what the callee looks like.

---

## Phase 2: Restore Project and Install Dependencies

Set up the project environment BEFORE generating any source code. This ensures third-party APIs are available for introspection during regeneration.

### Step 0: Handle partial packages

If the manifest has `"coverage": "partial"`, the user must provide the existing codebase path. Copy the existing codebase to the output directory first. Verify that all files listed in `"external_references"` exist at their expected paths. Then overwrite only the files being regenerated from specs. External files are treated as already-regenerated when determining dependency order.

### Step 1: Restore non-code files

Copy every file listed in `manifest.json`'s `non_code_files` array from `non_code/` back to its `original_path` in the output project directory. These files are not regenerated — they are exact copies from the original repo:

- Config files (`*.yaml`, `*.toml`, `*.json`, etc.)
- Documentation (`README.md`, `LICENSE`, etc.)
- Build files (`Makefile`, `Dockerfile`, `pyproject.toml`, etc.)
- CI/CD configs (`.github/workflows/*`, etc.)
- Static assets (images, sounds, data files)
- Test fixtures and data files
- Requirements/lockfiles

### Step 2: Create project skeleton

1. Create the directory tree matching the source paths in `manifest.json`
2. Verify build files are in place — `pyproject.toml`, `Cargo.toml`, `package.json`, `Makefile`, etc. should already exist from Step 1
3. If any build/config files are missing (not in the original repo), create them based on `project_context.md`

### Step 3: Install dependencies

Install all project dependencies from the restored build files:

```
# Python
pip install -e ".[dev]"
# or
pip install -r requirements.txt

# Rust
cargo build

# Go
go build ./...

# JS/TS
npm install

# C/C++
mkdir build && cd build && cmake .. && make
```

This is critical — installed dependencies are available for introspection during code regeneration. When the spec says "raise click.BadParameter(...)", you can check `click.BadParameter.__init__` to see its actual constructor signature rather than guessing.

---

## Phase 3: Plan the Regeneration

### Determine file order

From the call graph and manifest, sort files into regeneration order:

**Source files** (by dependency):
1. **Leaf files** — files whose functions don't call functions in other project files (utilities, constants, data models)
2. **Interior files** — files that depend only on already-regenerated files
3. **Root files** — entry points, main scripts, orchestrators

If there are circular dependencies, group the cycle and regenerate those files together.

**Test files** (after ALL source files):
1. **conftest.py** / shared fixtures first — other test files depend on these
2. **Test files** in any order — they depend on source code + fixtures but not on each other

Test files have `"type": "test"` in the manifest. Always regenerate all source files before any test files, since tests import and exercise the source.

### Determine target language

Each `.engspec` file has a `<!-- language: xxx -->` metadata comment. Regenerate in that language. If you're regenerating for a different target language (user-specified), use the user's choice instead.

---

## Phase 4: Regenerate Source Files

For each `.engspec` file, in dependency order:

### Step 1: Read the spec

Read the full `.engspec` file. Note:
- All `## \`function_signature()\`` sections — these become functions
- All `## \`<file-level: ...>\`` sections — these become top-level code
- The ordering of sections in the `.engspec` reflects the logical ordering in the source file
- **Ignore `### Debate Log` sections entirely** — they are audit history. All findings from debate have already been incorporated into the spec sections (Purpose, Preconditions, etc.).

### Step 2: Regenerate file-level code

For each `<file-level: ...>` section, generate the corresponding top-level code:

- `<file-level: initialization>` → imports, setup calls, global initialization
- `<file-level: state>` → global/module-level variable declarations
- `<file-level: main loop>` → `if __name__ == "__main__":` block or equivalent
- Other descriptions → whatever the Purpose and Postconditions describe

Use Preconditions to determine what must already exist (imports, packages).
Use Postconditions to determine what this block must establish.
Use Implementation Notes for language-specific details.

### Step 3: Regenerate functions

For each function section, generate the implementation:

1. **Signature**: Use the exact signature from the `##` header
2. **Body**: Implement to satisfy ALL of:
   - Every Precondition (may assume they hold, or validate if Failure Modes says to)
   - Every Postcondition (must guarantee these)
   - Every Invariant (must preserve these)
   - Every Failure Mode (must raise/return these under the specified conditions)
   - Implementation Notes (follow algorithmic choices specified here)
3. **Context**: Use the Context section to understand how this function fits into the larger system, but don't let it override the spec

### Step 4: Verify third-party API calls

**IMPORTANT:** When the spec references a third-party library call (e.g., "raise click.BadParameter", "calls requests.get"), verify the actual API signature against the installed package before writing the call. Dependencies were installed in Phase 2 — use them:

```
# Python: check a class constructor
python -c "import inspect; import click; print(inspect.signature(click.BadParameter.__init__))"

# Python: check a function signature
python -c "import inspect; import requests; print(inspect.signature(requests.get))"
```

Do NOT guess constructor arguments, keyword parameters, or method signatures. The installed package is the source of truth for API surface. The spec tells you WHAT to call and WHY — the installed library tells you HOW.

### Step 5: Assemble the file

Combine file-level code and functions into a complete source file:

1. File-level initialization / imports at the top
2. Constants and state declarations
3. Functions and classes in logical order
4. Main / entry point blocks at the bottom

Write the file to the path specified in `<!-- source: ... -->`.

### Step 6: Verify imports/compilation

After writing all files, run a basic syntax/import check:

```
# Python
python -c "import <package_name>"

# Rust
cargo check

# Go
go vet ./...

# JS/TS
npx tsc --noEmit
```

If there are import errors or compilation failures, fix them. Common issues:
- Missing imports that the spec implied but didn't list
- Type mismatches between regenerated files
- Circular imports needing restructuring

### Rules for regeneration

- **DO NOT look at any original source code.** You are regenerating from spec alone.
- The regenerated code does NOT need to be character-identical to the original.
- It MUST be functionally equivalent — same inputs produce same outputs, same side effects, same error conditions.
- Follow the language's idiomatic conventions (PEP 8 for Python, rustfmt for Rust, etc.)
- Add necessary imports that the spec implies but doesn't explicitly list (e.g., if Postconditions mention returning a `dict`, import `dict` if needed in the language)
- If the spec is ambiguous about something, make the most natural choice and add a `# NOTE: spec ambiguous — chose X over Y` comment
- **For `[external]` callees:** do NOT regenerate them — they already exist as source. You MAY introspect the existing source file to verify type compatibility (import it, check signatures), similar to how you introspect third-party libraries.
- **For test files:** code blocks in Preconditions, Postconditions, and Implementation Notes are exact — copy them verbatim into the regenerated test. Do NOT recompute expected values, re-derive setup logic, or improvise mock behavior. If the spec has a code block showing how to construct a URL, mock a redirect chain, or assert a byte sequence, reproduce it exactly.

---

## Phase 5: Run Tests

Test files were already regenerated from their `.engspec` specs in Phase 4 (after all source files). Do NOT clone or copy tests from the original repository — they must come from the specs.

### Step 1: Check for test information

Look in `project_context.md` for:
- Test commands (`pytest`, `cargo test`, `go test`, `npm test`, etc.)
- Test configuration files
- Required test fixtures or data

Also check `test_coverage.md` for:
- What functions are tested and how
- Expected test behavior

### Step 2: Generate additional tests if needed

If no test `.engspec` files exist in the package but source `.engspec` files have Test Strategy sections, generate tests from those sections as a fallback:

For each function with a Test Strategy:
1. Create a test file mirroring the source structure
2. Write one test per bullet point in Test Strategy
3. Use the language's standard test framework

This is a fallback only — normally test files have their own `.engspec` specs and are regenerated in Phase 4.

### Step 3: Run tests

Execute the test suite using the commands from project context:

```
# Python
pytest tests/ -v

# Rust
cargo test

# Go
go test ./...

# JS/TS
npm test
```

### Step 4: Report results

Report test results clearly:

```
## Test Results

Total: 15
Passed: 13
Failed: 2
Skipped: 0

### Failures:
- test_collision_detection: expected ball to bounce, got pass-through
  → Spec says "ball reverses y-velocity on paddle contact"
  → Implementation uses >= instead of == for y-coordinate check
  → FIX: changed to exact equality match per spec
```

### Step 5: Fix and re-run

For each failure:
1. Read the failing test to understand what it expects
2. Read the relevant `.engspec` section
3. Determine if the regenerated code violates the spec or if the test is wrong
4. If the code violates the spec: fix the code to match the spec, re-run
5. If the test seems wrong: report as a potential spec gap

**Hard limit: 5 fix-and-rerun iterations.** If tests still fail after 5 attempts, stop and report remaining failures in the validation report. Categorize each as: (a) spec gap — spec didn't contain enough information, (b) regeneration error — code doesn't match the spec, or (c) environment issue — missing system libraries, network dependencies, etc.

---

## Phase 6: Validation Report

After tests pass (or you've exhausted fixes), produce a report:

```markdown
# Regeneration Report: <project-name>

## Summary
- Files regenerated: N
- Total functions: N
- File-level sections: N
- Tests run: N passed / N failed / N skipped

## Files
| Source File | Language | Functions | File-Level Sections | Status |
|-------------|----------|-----------|---------------------|--------|
| src/main.py | python | 4 | 2 | OK |
| src/utils.py | python | 8 | 0 | OK |

## Test Results
- All N tests passed / N failures remaining

## Spec Gaps Found
List any cases where the spec was insufficient to regenerate correct code.
These should be fed back to the original .engspec files for refinement.

## Ambiguities
List any places where the spec was ambiguous and you had to make a choice.
Include the choice you made and why.

## Stub Files (from failed specs)
| Source File | Functions Stubbed | Dependent Files |
|-------------|-------------------|-----------------|

## Partial Coverage (if applicable)
- Coverage: full / partial
- Files regenerated from spec: N
- External files (pre-existing): N
- External files verified: N
- External files missing: (list if any)

## Notes
Any other observations about the regeneration process.
```

---

## Key Principles

1. **Spec is the source of truth.** Do not guess behavior that isn't in the spec. If the spec says the function raises ValueError on negative input, raise ValueError — don't silently clamp to zero.

2. **Dependencies are installed first.** Third-party APIs are verified against the installed package, not guessed from the spec. The spec says WHAT to call; the library tells you HOW.

3. **Dependency order matters.** Regenerate leaves first. When you write a function that calls another, you should already have the callee's code to ensure type compatibility.

4. **File-level code is real code.** The `<file-level: ...>` pseudo-functions describe actual code that must be generated — initialization, state setup, main loops. Don't skip them.

5. **Report gaps, don't paper over them.** If the spec is missing something and you have to invent behavior, mark it clearly. The goal is to find spec gaps, not to hide them.

6. **Tests are the final check.** If tests pass, the regeneration is successful. If they fail, the spec or the regeneration needs fixing — determine which.

---

## Operational Requirements

This workflow requires the most capable available model for all agents and sub-agents. Spec quality degrades significantly with smaller or faster models.

When using Claude Code, pass `model: "opus"` for ALL agent invocations (Explore, Plan, general-purpose, and any other agent type). Do not use Sonnet or Haiku for any agent in this workflow.
