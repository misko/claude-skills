# engspec Format Specification v1.0

Reference document defining the `.engspec` file format. Referenced by:
- `engspec_prompt.md` — produces `.engspec` files
- `engspec_tester_prompt.md` — reads, debates, and updates `.engspec` files
- `engspec_to_code_prompt.md` — consumes `.engspec` files for code regeneration

---

## File-Level Metadata

Every `.engspec` file starts with metadata as HTML comments:

```markdown
<!-- engspec v1 -->
<!-- source: {relative/path/to/source.ext} -->
<!-- language: {python|rust|go|cpp|c|javascript|typescript|java} -->
<!-- model: {model used to generate this spec, e.g. claude-opus-4-6} -->
<!-- status: {validated|failed} -->
<!-- source_commit: {git SHA of source file at spec generation time, omit if not in a git repo} -->
```

### After validation (set by `engspec_prompt`)
```markdown
<!-- validated: {ISO 8601 timestamp} -->
<!-- regeneration_count: {total attempts} -->
<!-- regeneration_pass_rate: {3/3 for validated} -->
```

### If validation failed
```markdown
<!-- status: failed -->
<!-- failure_reason: {what could not be captured} -->
```

---

## Function Section Template

Every function section MUST follow this structure. Copy it, fill in the `{placeholders}`, remove sections that don't apply, but never rename or reorder anything.

````markdown
## `{function_name(param1: Type, param2: Type) -> ReturnType}`
<!-- checksum: {MD5 hex digest of canonical spec content} -->
<!-- audited: {ISO 8601 timestamp, omit if not yet audited} -->

### Purpose
{1-3 sentences. What and why, not how.}
{If implementing one standard among common alternatives, state which}
{it follows AND which it does NOT. Negative boundaries matter.}

### Context
- Called by: {caller1(), caller2()}
- Calls: {callee1(), callee2()}
- Test coverage: {what's tested, what's not}
{For partial packages: tag functions outside the spec boundary as [external].}
{Example: "Calls: validate() [external], transform()"}
{This tells the tester to skip composition checks and the regenerator to expect existing source.}

### Preconditions
{- Each precondition on its own bullet}
{- Be specific: "n >= 1" not "valid input"}
{- Include constants this function depends on: "TICK_RATE controls frame cap, currently 60"}

### Postconditions
{- Each guarantee on its own bullet}
{- Include return value, side effects, and what is NOT modified}

### Invariants
{- Properties preserved before and after the call}

### Implementation Notes
{- Only what affects correctness. MUST be shorter than the source code.}
{- When correctness depends on an exact literal value (regex, format string,}
{  query template, protocol sequence), embed it in a fenced code block.}
{  Do not paraphrase — the code block IS the spec for that value.}
{- For third-party API calls where kwargs affect behavior, specify the exact}
{  call signature: "calls idna.encode(host) (without uts46=True)"}

### Failure Modes
{- Concrete: "Raises ValueError if n < 0", not "may raise errors"}
{- Always specify the EXACT error type — not a parent type.}
{  "catches BaseException" and "catches OSError" have different behavior.}
{- For each error, state: **recovered** (caught, fallback, continues) or **propagated** (re-raised to caller)}
{- Example: "If X raises FileNotFoundError: **recovered** — sets fallback_value, continues"}
{- Example: "If Y raises PermissionError: **propagated** — closes handle, re-raises to caller"}

### Test Strategy
{- Actionable bullets. Someone should be able to write tests from this alone.}

---
````

---

## Template Rules

**Every section uses exactly these heading names, in this order.** If a section doesn't apply to a particular function (e.g., a pure function has no Failure Modes), omit the section entirely — don't include it with "N/A" or empty content.

**The `##` heading MUST be the exact signature in backticks.** The signature is a contract — it defines the API that callers depend on. Include all parameter defaults exactly as they appear in the source. If a parameter has no default, it must not have one in the heading. Do not invent defaults.

For regular functions:
```
## `function_name(param1: Type, param2: Type) -> ReturnType`
```

For types/classes, the heading includes the full type declaration — base types, interfaces, traits, and behavioral modifiers:
```
## `class Atom(metaclass=ABCMeta)`
## `struct Point: Hash + Display`
## `class Config extends BaseConfig implements Serializable`
```
Methods within the type use the standard function heading. Abstract/virtual/interface methods that must be implemented by subtypes should state this in Purpose: "Abstract — must be implemented by subtypes."

**For file-level code** (top-level execution, mutable globals, interrelated constants), use the same template with a pseudo-function signature:
```
## `<file-level: description>`
```

A file can have multiple `<file-level>` sections:
- `## \`<file-level: initialization>\`` — imports, setup calls, global init
- `## \`<file-level: state>\`` — mutable globals, interrelated constants
- `## \`<file-level: main loop>\`` — entry point / main block

**When to use `<file-level>`:**

| Situation | Action |
|-----------|--------|
| File has top-level execution (runs on import/load/execute) | Add a `<file-level>` section |
| File has mutable globals or interrelated constants | Add a `<file-level: state>` section |
| File has only simple constants (`PI = 3.14`) | No section — mention in function Preconditions |
| File has nothing but functions | No section |

For languages where everything is a function (Rust, Go, Java), `<file-level>` is rarely needed. `fn main()` and `func init()` are real functions and get regular sections.

**What to skip entirely:**
- Trivial getters/setters (unless they have validation logic)
- Auto-generated code — identified by: (1) a generated-by comment/header in the file (e.g., `# Generated by protoc`, `// Code generated by ... DO NOT EDIT`), (2) files in conventionally generated paths (`*_pb2.py`, `generated/`, `*.gen.go`), or (3) files listed as generated in build configuration. When in doubt, spec it — an unnecessary spec is cheaper than a missing one.
- Vendored dependencies (`vendor/`, `third_party/`) — these are third-party code and should be restored from their package manager, not spec'd

**Do NOT skip** methods that affect equality, hashing, ordering, comparison, or string representation — even if they look trivial. These determine how objects behave in collections, comparisons, and debugging output. Omitting them changes observable behavior.

If a function's behavior is fully captured by its signature, Purpose alone is enough — omit other sections.

---

## Section Guidance

| Section | Key rule |
|---------|----------|
| Purpose | 1-3 sentences. What and why, not how. When the function implements one standard/specification out of multiple common alternatives, state which it follows AND which it does not. "Follows RFC 3986" is incomplete without "does NOT implement WHATWG special-scheme behaviors" when both are common choices. Negative boundaries prevent the regenerator from adding behaviors the original deliberately excludes. |
| Context | Callers, callees, test coverage status. For partial packages, tag functions outside the spec boundary as `[external]`. |
| Preconditions | Every input constraint. If a function silently accepts invalid input, note what happens. |
| Postconditions | Every output guarantee. Include mutation guarantees ("input list is not modified"). |
| Invariants | What doesn't change. Important for stateful objects. |
| Implementation Notes | Only what affects correctness. Algorithm choice, not code style. SHORTER than source. When correctness depends on an exact literal value (regex pattern, format string, query template, protocol sequence, magic constant), embed it in a fenced code block — do not paraphrase. When calling third-party APIs where keyword arguments affect behavior, specify the exact call signature — e.g., "`idna.encode(host)` (without `uts46=True`)" — since different kwargs produce different results. |
| Failure Modes | Concrete exceptions and conditions. Not "may raise errors". Always specify the **exact error type** — not a parent type ("catches BaseException" and "catches OSError" have different behavior). For each error, specify whether it is **recovered** (caught, fallback applied, execution continues) or **propagated** (re-raised to caller). |
| Test Strategy | Actionable. Someone writes tests from this section alone. |

---

## Test File Specifications

Test files get `.engspec` specs just like source files. They use the same template with these adjustments:

**Use the same template exactly.** The `##` heading is the test function signature. The sections mean:

| Section | Meaning for test functions |
|---------|---------------------------|
| Purpose | What property or behavior this test verifies. |
| Context | What source functions/modules this test covers. Link to their `.engspec` files. "Tests: dotenv.main.set_key()", "Tests: dotenv.parser.parse_binding()". |
| Preconditions | **All test setup that affects correctness must be in code blocks.** List all required fixtures, files, and environment, but when the setup method matters, embed it as code. Don't write "test with an uppercase URL" — write: ```url = httpbin("get"); scheme = url.split("://")[0].upper(); url = scheme + url[len(scheme):]``` (uppercases only the scheme, not the path). Don't write "requires TLS cert files" — write: ```# Requires files created by conftest cert_fixture: client_cert = str(tmp_path / "client.pem")``` List ALL required fixture files, data files, and external resources explicitly — if a test needs a cert file, name it. |
| Postconditions | **EVERY assertion must be written as a code block.** Do not describe assertions in English — embed the actual assertion expression. The code block IS the spec. This is mandatory because English paraphrasing of binary data, encoded values, and computed results introduces errors. Example: don't write "asserts the base64 output matches expected" — write: ```assert header == b"Basic dXNlcm5hbWU6cGFzc3dvcmQ="``` The comparison method matters too — `obj == expected` vs `convert(obj) == expected` produce different results. For parametrized tests, include the full parameter table as a code block. |
| Invariants | What is cleaned up / restored after the test (tmp files deleted, env vars reset, etc.). |
| Implementation Notes | **Use code blocks for any setup logic where the method matters.** Testing patterns: parametrize values, fixture chains, framework-specific runner usage. **For test doubles (mocks, stubs, patches, fakes):** specify the exact target being replaced (full qualified path), what it's replaced with, and any framework-specific arguments that affect behavior. When a mock must simulate multi-step behavior (e.g., redirect chains, retry sequences), embed the mock setup as a code block — don't describe it in English. A mock that returns a blank object vs one that returns the actual sent request will break different assertions. **For fixture data files** (JSON, CSV, YAML test data): specify the exact field/key used to access test input — e.g., "reads `test_case["href"]` (pre-resolved URL), NOT `test_case["input"]`." **For object lifecycle:** note whether objects are used with a context manager, async context manager, or plain construction — e.g., "creates client via `client = Client()` (no context manager)" vs "creates client via `async with AsyncClient() as client:`." |
| Failure Modes | Omit — tests don't have failure modes (they either pass or fail). |
| Test Strategy | Omit — tests don't need a test strategy for themselves. |

**conftest.py files** get specs too. Fixtures are functions — each fixture gets a `##` section:

```
## `my_fixture(tmp_path) -> Path`
<!-- checksum: 8a3b1c... -->

### Purpose
Creates a temporary .env file with KEY=value for testing.

### Preconditions
- tmp_path fixture is available (pytest built-in)

### Postconditions
- Returns Path to a file containing "KEY=value\n"
- File exists on disk and is readable

### Implementation Notes
- Session-scoped — shared across all tests in the session
```

**`<file-level>` sections** for test files capture:
- Module-level marks (`pytestmark = pytest.mark.usefixtures(...)`)
- Shared test constants / test data
- Parametrize data defined at module level

**Regeneration verification** applies to test files too. A regenerated test must reproduce every assertion code block from the Postconditions section **exactly** — same values, same comparison methods, same expected outputs. Do not recompute expected values; copy them from the spec. If the spec says `assert result == b"\xc5\xa9sername"`, the regenerated test must use that exact byte sequence. A regeneration that changes any assertion value is a FAIL.

Structural differences are acceptable: assertion ordering, helper extraction, parametrize-vs-individual-tests, and variable naming may differ. What must match exactly: (1) every assertion code block from Postconditions appears verbatim in the regenerated test, (2) every setup code block from Preconditions/Implementation Notes is reproduced, (3) no assertions are added or omitted. The code blocks are the contract; the surrounding scaffolding is not.

---

## Debate Log Section

The Debate Log is appended to a function section **only by `engspec_tester`**, never by `engspec_prompt`. It records adversarial findings and rulings.

```markdown
### Debate Log
| Round | Finding | Blind | Sighted | Tag | Severity | Action |
|-------|---------|-------|---------|-----|----------|--------|
| 1 | Normalization level not specified | SpecGap | SpecGap | — | major | Added RFC 3986 §6.2.2 to Implementation Notes |
```

The Debate Log is **excluded from the checksum** (see Versioning below). Adding Debate Log entries does not invalidate the function's checksum. Modifying spec content (Purpose, Preconditions, etc.) does.

---

## Versioning Metadata

### Per-Function Checksum

Each function section carries an MD5 checksum of its canonical spec content, placed immediately after the `##` heading:

```markdown
## `function_name(param: Type) -> ReturnType`
<!-- checksum: 9f86d081884c7d659a2feaa0c55ad015 -->
<!-- audited: 2026-03-15T14:00:00Z -->
```

#### What is checksummed

**Included:**
- The `##` heading line (function signature)
- All content under: Purpose, Context, Preconditions, Postconditions, Invariants, Implementation Notes, Failure Modes, Test Strategy
- Fenced code blocks within those sections (exact bytes)

**Excluded:**
- The `<!-- checksum: ... -->` and `<!-- audited: ... -->` metadata lines themselves
- The `### Debate Log` subsection and everything under it
- The `---` separator at the end of the function section (file-level metadata above the first `##` is never captured by Step 1)
- Trailing whitespace on each line (stripped before hashing). Leading whitespace is **preserved** — it is significant in code blocks and indented content.
- Runs of blank lines (collapsed to a single blank line before hashing)

#### Canonical form for hashing

**Preamble:** Throughout these steps, `---` separators and `##` headings inside fenced code blocks (between matching `` ``` `` delimiters) are **content, not structure** — ignore them when identifying section boundaries.

1. Take all lines from the `##` heading through the last non-blank line before the next `---` separator or `##` heading. If neither exists (last function in file), take through the last non-blank line in the file.
2. Remove `<!-- checksum: ... -->` and `<!-- audited: ... -->` lines
3. Remove the `### Debate Log` subsection and everything under it (up to the next `##` or `---`)
4. Normalize line endings to `\n` (LF) — no `\r\n` or `\r`
5. Strip trailing whitespace from each line (preserve leading whitespace)
6. Collapse runs of blank lines to a single blank line
7. Encode as UTF-8
8. Compute MD5 hex digest

#### Who sets the checksum

- **`engspec_prompt`**: Computes checksum after generating and validating each function spec
- **`engspec_tester`**: Recomputes checksum for any function whose spec content was modified during debate (SpecGap/SpecConflict rulings). Functions with only NoIssue rulings retain their existing checksum.

### File-Level `source_commit`

The git SHA of the source file when the spec was generated or last verified against source.

```markdown
<!-- source_commit: 3642c64a... -->
```

- **Set by `engspec_prompt`** during initial generation (from `git log -1 --format=%H -- <source-file>`)
- **Updated by `engspec_tester`** when verifying spec against source in "spec + source" mode
- **File-level** (not per-function) — a source file has one commit hash
- **Omitted** if the source is not in a git repository

### Per-Function `audited` Timestamp

ISO 8601 timestamp of the last `engspec_tester` adversarial audit for this specific function.

```markdown
<!-- audited: 2026-03-15T14:00:00Z -->
```

- **Set by `engspec_tester`** after completing adversarial debate for each function
- **Distinct from** file-level `<!-- validated: ... -->` which records initial regeneration validation by `engspec_prompt`
- **Per-function** because different functions in the same file may be audited in different rounds or sessions

### Invalidation Rules

**Checksum is stale when:**
1. Any spec content included in the hash is modified (Purpose, Preconditions, Postconditions, etc.)
2. The function signature in the `##` heading changes

**Checksum is NOT invalidated when:**
1. A Debate Log entry is added (excluded from hash by design)
2. File-level metadata changes (`source_commit`, `status`, etc.)
3. Other functions in the same file change
4. The `<!-- audited: ... -->` timestamp is updated

**`source_commit` is stale when:**
1. The source file has commits after the recorded SHA (detectable via `git log`)
2. The source file no longer exists

**`audited` timestamp is stale when:**
1. The function's checksum has changed since the `audited` timestamp (spec was modified after last audit)
2. The function was not yet audited (no `audited` field present)

### How versioning affects each prompt

| Prompt | Reads | Writes |
|--------|-------|--------|
| `engspec_prompt` | In incremental mode: reads existing `checksum` and `source_commit` to detect stale specs | Sets `checksum` per function; sets `source_commit`, `validated`, `regeneration_count`, `regeneration_pass_rate` per file |
| `engspec_tester` | Reads `checksum` to detect if spec changed since last audit; reads `source_commit` to check if source is current | Recomputes `checksum` for modified functions; updates `audited` per function; updates `source_commit` if re-verified |
| `engspec_to_code` | Reads `checksum` to verify spec integrity; reads `source_commit` and `audited` for informational reporting | Does not modify versioning metadata |

---

## Examples

### Example: Function-only file

For a simple function like:
```python
def fibonacci(n: int) -> int:
    if n < 0:
        raise ValueError("n must be non-negative")
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b
```

The `.engspec` would be:

```markdown
<!-- engspec v1 -->
<!-- source: math_utils.py -->
<!-- language: python -->
<!-- model: claude-opus-4-6 -->
<!-- status: validated -->
<!-- source_commit: a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4 -->
<!-- validated: 2026-03-14T12:00:00Z -->
<!-- regeneration_count: 5 -->
<!-- regeneration_pass_rate: 3/3 -->

## `fibonacci(n: int) -> int`
<!-- checksum: 9f86d081884c7d659a2feaa0c55ad015 -->

### Purpose
Computes the n-th Fibonacci number (0-indexed: fib(0)=0, fib(1)=1, fib(2)=1, ...).

### Context
- Called by: (varies by project)
- Calls: none
- Test coverage: (fill based on actual tests)

### Preconditions
- n is an integer >= 0

### Postconditions
- Returns the n-th Fibonacci number
- Return value >= 0
- fib(0) = 0, fib(1) = 1, fib(n) = fib(n-1) + fib(n-2) for n >= 2

### Invariants
- Pure function, no side effects
- Deterministic

### Implementation Notes
- Iterative (not recursive), O(n) time, O(1) space
- For very large n (e.g., n > 10^6), big integer arithmetic dominates runtime

### Failure Modes
- Raises ValueError if n < 0: **propagated** to caller

### Test Strategy
- Property: fib(n) = fib(n-1) + fib(n-2) for n in [2, 20]
- Edge: fib(0) = 0, fib(1) = 1
- Edge: n = -1 raises ValueError
- Large: fib(100) = 354224848179261915075

---
```

No `<file-level>` section needed — the file is just a pure function with no top-level execution, mutable globals, or interrelated constants.

### Example: File with top-level code and state

For a game script like:
```python
import pygame

SCREEN_W, SCREEN_H = 800, 600
TICK_RATE = 60

pygame.init()
screen = pygame.display.set_mode((SCREEN_W, SCREEN_H))
clock = pygame.time.Clock()

score = 0
ball_x, ball_y = SCREEN_W // 2, SCREEN_H // 2

def reset_ball():
    global ball_x, ball_y
    ball_x, ball_y = SCREEN_W // 2, SCREEN_H // 2

if __name__ == "__main__":
    running = True
    while running:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
        reset_ball()
        clock.tick(TICK_RATE)
    pygame.quit()
```

The `.engspec` would be:

```markdown
<!-- engspec v1 -->
<!-- source: game.py -->
<!-- language: python -->
<!-- model: claude-opus-4-6 -->
<!-- status: validated -->
<!-- source_commit: f7e8d9c0b1a2 -->
<!-- validated: 2026-03-14T14:00:00Z -->
<!-- regeneration_count: 7 -->
<!-- regeneration_pass_rate: 3/3 -->

## `<file-level: initialization>`
<!-- checksum: 4a8b2c1d3e5f... -->

### Purpose
Initializes pygame subsystems and creates the display surface and clock.

### Preconditions
- pygame package is installed
- Display server is available

### Postconditions
- pygame is initialized
- `screen` is a pygame Surface of size (800, 600)
- `clock` is a pygame.time.Clock instance
- SCREEN_W = 800, SCREEN_H = 600, TICK_RATE = 60

### Failure Modes
- pygame.error if no display server is available

---

## `<file-level: game state>`
<!-- checksum: 7d6e5f4a3b2c... -->

### Purpose
Defines shared mutable state for the game.

### Postconditions
- `score` is initialized to 0
- `ball_x` is initialized to SCREEN_W // 2 (400)
- `ball_y` is initialized to SCREEN_H // 2 (300)

### Invariants
- `score` is always >= 0

### Implementation Notes
- `ball_x`, `ball_y` are module-level globals, modified by reset_ball()
- `score` is a module-level global (not yet modified by any function in this file)

---

## `reset_ball() -> None`
<!-- checksum: 1a2b3c4d5e6f... -->

### Purpose
Resets ball position to center of screen.

### Preconditions
- SCREEN_W and SCREEN_H are defined (set during initialization)

### Postconditions
- `ball_x` = SCREEN_W // 2
- `ball_y` = SCREEN_H // 2
- `score` is unchanged

### Invariants
- Pure positional reset — no score or other state changes

---

## `<file-level: main loop>`
<!-- checksum: 9e8d7c6b5a4f... -->

### Purpose
Runs the game loop until the user closes the window.

### Preconditions
- Initialization block has run (screen, clock exist)
- Game state is initialized (ball_x, ball_y, score exist)

### Postconditions
- Game exits cleanly, pygame.quit() is called
- No resources leaked

### Implementation Notes
- Runs only when file is executed directly (`if __name__ == "__main__"`)
- Loop: poll events → call reset_ball() → tick clock at TICK_RATE
- Exits on pygame.QUIT event

### Failure Modes
- If initialization failed (no display), this block is never reached

---
```

Note: the spec is **shorter** than the code but captures everything needed to reimplement it correctly, including the file-level behavior.

### Example: Function after adversarial audit

After `engspec_tester` has run, a function section may look like:

```markdown
## `normalize_url(url: str) -> str`
<!-- checksum: b5c6d7e8f9a0... -->
<!-- audited: 2026-03-15T14:00:00Z -->

### Purpose
Normalizes a URL following RFC 3986 only. Does NOT implement WHATWG special-scheme behaviors.

### Preconditions
- url is a valid RFC 3986 URI string
- url length <= 2048

### Postconditions
- Returns a normalized URL string
- Applies RFC 3986 Section 6.2.2 syntax-based normalization (case, percent-encoding, path-segment removal)

### Failure Modes
- Raises ValueError if len(url) > 2048: **propagated** to caller

### Test Strategy
- Property: normalize_url(normalize_url(x)) == normalize_url(x) for random valid URLs
- Edge: mixed-case scheme "HTTP://Example.COM" → "http://example.com"
- Edge: dot segments "http://example.com/a/../b" → "http://example.com/b"
- Boundary: url of exactly 2048 characters succeeds
- Beyond: url of 2049 characters raises ValueError

### Debate Log
| Round | Finding | Blind | Sighted | Tag | Severity | Action |
|-------|---------|-------|---------|-----|----------|--------|
| 1 | Normalization level not specified | SpecGap | SpecGap | — | major | Added RFC 3986 §6.2.2 to Postconditions |
| 2 | MAX_URL_LENGTH unguarded, silent truncation | SpecGap | SpecGap | — | major→critical | Added Precondition + Failure Mode for url length |

---
```

---

## Manifest Schema (`manifest.json`)

The manifest is the index of all files in an engspec package. It is produced by `engspec_prompt`, read by `engspec_tester` and `engspec_to_code`.

```json
{
  "engspec_version": "1.0",
  "project": "<project-name>",
  "generated": "<ISO 8601 timestamp>",
  "source_repo": "<git URL or local path>",
  "coverage": "full",
  "scope": {
    "type": "all",
    "targets": [],
    "description": "Full codebase"
  },
  "languages": ["python"],
  "files": [
    {
      "spec": "specs/src/main.py.engspec",
      "source": "src/main.py",
      "type": "source",
      "language": "python",
      "status": "validated",
      "regeneration_count": 7,
      "regeneration_pass_rate": "3/3",
      "functions": 4,
      "file_level_sections": 2,
      "source_commit": "a1b2c3d4..."
    },
    {
      "spec": "specs/tests/test_main.py.engspec",
      "source": "tests/test_main.py",
      "type": "test",
      "language": "python",
      "status": "validated",
      "regeneration_count": 6,
      "regeneration_pass_rate": "3/3",
      "functions": 12,
      "file_level_sections": 1,
      "source_commit": "a1b2c3d4..."
    }
  ],
  "non_code_files": [
    {
      "path": "non_code/README.md",
      "original_path": "README.md",
      "type": "documentation"
    }
  ],
  "external_references": [],
  "summary": {
    "total_specs": 5,
    "validated": 5,
    "failed": 0,
    "total_functions": 23,
    "total_file_level_sections": 4,
    "total_regenerations": 31,
    "total_non_code_files": 6,
    "external_references": 0
  }
}
```

### Manifest fields

| Field | Required | Description |
|-------|----------|-------------|
| `engspec_version` | Yes | Format version (currently `"1.0"`) |
| `coverage` | Yes | `"full"` (all source files spec'd) or `"partial"` (subset) |
| `scope` | Yes | What was targeted: `type` (`all`, `files`, `modules`, `directories`), `targets` (list), `description` |
| `files[].type` | Yes | `"source"` (production code) or `"test"` (test files, conftest). Determines regeneration order — all source files before any test files. |
| `files[].source_commit` | If git | Git SHA of source file at spec generation time |
| `files[].status` | Yes | `validated` or `failed` |
| `external_references` | If partial | Source files referenced by specs but not themselves spec'd. Each entry: `source` path, `functions_referenced`, `referenced_by` |

**Invariant:** `summary.total_specs` must equal `len(files)`. `summary.validated + summary.failed` must equal `summary.total_specs`.

For a partial-package example:

```json
{
  "coverage": "partial",
  "scope": {
    "type": "files",
    "targets": ["src/parser.py", "src/lexer.py"],
    "description": "Parser and lexer modules"
  },
  "external_references": [
    {
      "source": "src/utils.py",
      "functions_referenced": ["parse_header", "format_output"],
      "referenced_by": ["specs/src/parser.py.engspec"]
    }
  ]
}
```

---

## Hardcoded Limit Checklist

When Red finds a constant that implicitly bounds inputs (buffer sizes, loop bounds, block sizes, max retries, timeouts, array dimensions), check these 6 properties:

1. *What is the limit?* — the exact value and where it appears
2. *What inputs reach it?* — trace callers (source mode) or Context sections (spec-only mode) to find the real range
3. *How close do real values get?* — measure the margin (e.g., "max observed 248, limit 256, margin 8")
4. *What happens at the limit?* — does it clamp, wrap, error, or silently corrupt?
5. *What happens one step beyond?* — test limit+1 (or limit-1 for lower bounds)
6. *Is the limit guarded?* — is there a runtime check, or does the code silently assume?

If real values reach within 1 of the limit (zero margin), flag as at minimum `major` severity regardless of other factors.

---

## Round Output Format

Each round's findings are structured per function:

```markdown
### Round {N}: `{function_name}`

#### Red Challenges
1. **[category/severity]** Description of the gap.
   - Scenario: specific input or call sequence
   - (For unguarded_constraint: Limit, margin, guard status, escalation reason)

#### Blue Responses
1. **agree/disagree/partial** — Reasoning with spec section citations.

#### Blind Judge Rulings
1. **SpecGap/NoIssue/SpecConflict** — Reasoning and action.

#### Sighted Judge Rulings
1. **SpecGap/NoIssue/SpecConflict** — Reasoning with source context.

#### Ruling Comparison
1. **Agree/Disagree** — Final action and confidence level.
```

---

## Analysis Report Format

After all subgraphs converge (or hit the cap), produce a report:

```markdown
# Adversarial Analysis Report: <project-name>

## Summary
- Mode: spec + source / spec only
- Functions analyzed: N
- Subgraphs: N
- Total rounds: N
- Findings: N (SpecGap: N, SpecConflict: N, NoIssue: N)
- Judge agreement rate: N/N (X%)
- Silent divergences found: N (Blind=NoIssue, Sighted=SpecGap)
- Clarifications needed: N (Blind=SpecGap, Sighted=NoIssue)
- Severity escalations: N (N silent-failure, N unguarded-untested)
- Spec updates applied: N
- Tests added to spec: N
- Overall confidence: X.XX

## Findings by Severity

### Critical
- `function_name` [SpecGap]: description (Round N) — FIXED in spec

### Major
- `function_name` [SpecGap]: description (Round N) — FIXED in spec

### Minor
- `function_name` [SpecGap]: description (Round N) — FIXED in spec

## Spec Conflicts Found
| Function | Contradiction | Resolution | Round |
|----------|--------------|------------|-------|

## Spec Updates Applied
| Function | Section Updated | What was added | Round |
|----------|----------------|----------------|-------|

## Tests Added to Spec
| Function | Test Description | From Round |
|----------|-----------------|------------|

## Confidence Scores
| Subgraph | Functions | Rounds | Findings | Resolved | Confidence |
|----------|-----------|--------|----------|----------|------------|

## Convergence Details
| Subgraph | Converged? | Clean Rounds | Total Rounds |
|----------|-----------|--------------|--------------|

## External Boundaries (partial packages only)
| Spec'd Function | External Callee | Boundary Documented? | Risk |
|-----------------|-----------------|---------------------|------|
```
