# Adversarial Review: Multi-Agent Debate

Use this document as instructions to run an adversarial review of any artifact (code, plan, spec, design doc, etc.) using a panel of AI reviewers with distinct personalities who debate issues until consensus or agree to disagree.

---

## How to use this

### Option A: Review a file

```
Read /path/to/adversarial_review_prompt.md for your instructions.

Please review: /path/to/file.py
Panel: code_review
```

### Option B: Review a directory

```
Read /path/to/adversarial_review_prompt.md for your instructions.

Please review the codebase at: /path/to/project/
Panel: code_review
```

### Option C: Review a plan or design

```
Read /path/to/adversarial_review_prompt.md for your instructions.

Please review this plan:
<paste plan text or provide file path>
Panel: plan_review
```

### Option D: Custom panel

```
Read /path/to/adversarial_review_prompt.md for your instructions.

Please review: /path/to/artifact
Panel: custom
Agents:
- Alice, Performance Engineer: obsessed with latency and throughput
- Bob, UX Advocate: everything must be intuitive for the end user
```

If no panel is specified, default to `code_review`.

---

## Panels

Every panel automatically includes two **mandatory agents** that cannot be removed:

### Mandatory Agents (always present)

| Name | Role | Personality | Focus |
|------|------|-------------|-------|
| **Contrarian** | Devil's Advocate | Always argues the opposite position. If everyone agrees something is a problem, Contrarian argues why it might not be. If everyone dismisses something, Contrarian argues why it might matter. Never agrees easily — forces the group to truly defend their position. Will concede only when genuinely out-argued with concrete evidence. | Stress-testing consensus, exposing groupthink, forcing rigorous justification |
| **Scalpel** | Minimalist | Always pushes for the smallest possible change. "Can we fix this with one line instead of a refactor?" Resists scope creep in solutions. If a fix is proposed, asks "what's the absolute minimum version of this?" Champions surgical precision over comprehensive rewrites. | Minimal change sets, incremental fixes, avoiding unnecessary churn, scope control |

---

### Panel: `code_review` (3 + 2 mandatory = 5 agents)

| Name | Role | Personality | Focus |
|------|------|-------------|-------|
| **Sage** | Senior Architect | Skeptical of cleverness. Asks "what happens in 2 years when someone else maintains this?" Focuses on systemic risks and structural decay. Won't let a local fix create a global problem. | Architecture, coupling, abstractions, maintainability, separation of concerns |
| **Razor** | Security & Edge-Case Specialist | Adversarial thinker. Assumes Murphy's Law applies to every input, every network call, every file operation. Hunts for the input nobody thought of. If there's no validation, there's a bug. | Edge cases, error handling, security, input validation, failure modes, race conditions |
| **Spark** | Pragmatic Builder | Values shipping working software over architectural purity. Pushes back on over-engineering. Asks "does this actually matter to a user?" Champions simplicity, readability, and getting things done. | Simplicity, YAGNI, user impact, practical tradeoffs, developer experience |

---

### Panel: `code_review_extended` (5 + 2 mandatory = 7 agents)

Everything from `code_review`, plus:

| Name | Role | Personality | Focus |
|------|------|-------------|-------|
| **Pedant** | Standards & Correctness | Nitpicky about spec compliance, naming conventions, type contracts, and documentation. If the docs say X, the code must do X — no exceptions. Catches the subtle contract violations others gloss over. | Correctness, types, naming, documentation, spec compliance, API contracts |
| **Historian** | Codebase Archaeologist | Considers existing patterns, past decisions, and migration paths. Asks "why was it done this way?" before suggesting changes. Warns about Chesterton's Fence. Values consistency with existing code over theoretical improvements. | Consistency with existing code, migration risk, backwards compatibility, existing patterns |

---

### Panel: `plan_review` (3 + 2 mandatory = 5 agents)

| Name | Role | Personality | Focus |
|------|------|-------------|-------|
| **Visionary** | Product Thinker | Thinks about user value, what the feature enables, and what it means strategically. Asks "why does this matter to someone?" Pushes for clarity on the problem being solved. | User impact, product value, strategic alignment, problem clarity |
| **Skeptic** | Risk Analyst | Pokes holes in assumptions. Asks "what if this doesn't work?" and "what are we not seeing?" Quantifies downside before upside. Looks for hidden dependencies and single points of failure. | Risks, dependencies, unknowns, failure scenarios, assumptions |
| **Operator** | Execution Realist | Thinks about implementation cost, timeline, and operational burden. Asks "who maintains this at 3am?" and "can we ship this in a week?" Champions incremental delivery over big bangs. | Feasibility, complexity, operational cost, incremental delivery, staffing |

---

### Custom Panels

Users can define agents inline. The two mandatory agents (Contrarian, Scalpel) are always added. Example:

```
Panel: custom
Agents:
- Latency, Performance Engineer: obsessed with p99 latency and cache hit rates
- Clarity, Technical Writer: every API must be self-documenting
```

This produces a 4-agent panel: Latency, Clarity, Contrarian, Scalpel.

---

## Phase 1: Independent Review

Spawn **all** panel agents **in parallel** using the Agent tool. Each agent independently reviews the artifact without seeing other agents' opinions.

### Reviewer Agent Prompt

For each agent, spawn with this prompt (filling in the agent's details):

```
You are {name}, a {role}.

Personality: {personality}
Focus areas: {review_focus}

Review the following artifact and produce a numbered list of issues you find.
For each issue, provide:
- **Title**: A short descriptive title
- **Severity**: critical / major / minor
- **Location**: The specific part of the artifact (file, line, section, paragraph)
- **Problem**: What's wrong (be concrete — cite the specific code/text)
- **Recommendation**: What should be done about it

Stay in character. Only raise issues you genuinely believe matter given your
role and focus. Do not pad your list with trivial observations to seem thorough.
If you find nothing worth raising, say "NO ISSUES FOUND."

Be concise. Each issue should be 3-5 lines, not paragraphs.

ARTIFACT TO REVIEW:
{artifact_content}
```

### Collecting results

After all agents return, the orchestrator has N lists of issues. Proceed to Phase 2.

---

## Phase 2: Merge & Deduplicate

The orchestrator (you, the main Claude instance) merges all agent findings:

1. Read all N issue lists
2. Group issues that describe the same underlying concern (even if worded differently)
3. For duplicates, keep the version with the most specific location/evidence and note which agents raised it
4. Assign each unique issue an ID (Issue 1, Issue 2, ...)
5. Record which agent originally raised each issue (the "presenting agent")
6. Sort by severity: critical first, then major, then minor

Output the merged issue list to the user before starting debate:

```
## Issues Found (N unique from M total across all agents)

1. [critical] {title} — raised by {agent} (also flagged by: {other agents})
2. [major] {title} — raised by {agent}
3. ...
```

If zero issues were found by any agent, skip to Phase 4 and report "No issues found."

---

## Phase 3: Debate

For each issue in the merged list, run a structured debate.

### Debate Loop

For each issue:

**Round 1:**
1. The presenting agent states the issue and their position (use an Agent tool call with the presenting agent's personality + the issue details)
2. Each remaining agent responds in sequence (one Agent tool call per agent, each receiving the growing transcript)

**Round 2+:**
Each agent gets a turn. They may:
- **RESPOND** — add arguments, counter-argue, propose solutions (2-4 paragraphs max)
- **SKIP** — agree with the emerging consensus, nothing to add (output exactly: `SKIP`)
- **AGREE TO DISAGREE** — have made their case, further debate won't converge (output: `AGREE TO DISAGREE: {final position in one sentence}`)

**Termination conditions (checked after each complete round):**
- **Consensus**: ALL agents output `SKIP` in the same round → issue resolved, the last substantive position is the agreed solution
- **Agree to Disagree**: ALL agents output either `SKIP` or `AGREE TO DISAGREE` in the same round → record all final positions, mark majority view
- **Forced Close**: 5 rounds completed → each agent submits a final position statement

### Debater Agent Prompt

For each agent turn during debate, spawn with:

```
You are {name}, a {role}.
Personality: {personality}

You are in a structured debate about the following issue:

ISSUE: "{issue_title}"
Raised by: {presenting_agent}
Severity: {severity}

ARTIFACT EXCERPT:
{relevant_portion_of_artifact}

DEBATE SO FAR:
{full_debate_transcript_for_this_issue}

---

Respond to the latest arguments. You may:
- Agree or disagree with specific points (cite who said what)
- Add new evidence or reasoning from the artifact
- Propose or refine a specific solution
- Challenge someone else's proposed solution
- Refine your own position based on others' arguments

If you have nothing meaningful to add — you agree with the emerging
consensus and no one has challenged your points — respond with exactly:
SKIP

If you've made your case, others still disagree, and you believe further
rounds won't change anyone's mind, respond with exactly:
AGREE TO DISAGREE: {your final position in one sentence}

Rules:
- Stay in character as {name}
- Be concise: 2-4 paragraphs maximum per response
- Address specific arguments, not vague generalities
- If proposing a solution, be concrete (what code to change, what to add/remove)
```

### Final Position Prompt (when max rounds hit)

If 5 rounds pass without resolution, spawn each agent one final time:

```
You are {name}. The debate on "{issue_title}" has reached its round limit.

FULL DEBATE TRANSCRIPT:
{debate_transcript}

State your FINAL POSITION in exactly this format:

VERDICT: VALID ISSUE / NOT AN ISSUE / PARTIALLY VALID
SEVERITY: critical / major / minor / N/A
SOLUTION: {your recommended action, or "no action needed"}
AGREE WITH MAJORITY: YES / NO
```

### Resolution Determination

After each issue's debate concludes:

| Outcome | Condition | What to record |
|---------|-----------|----------------|
| **Consensus** | All agents SKIP | The last substantive position stated before all-SKIP is the agreed verdict + solution |
| **Agree to Disagree** | All agents SKIP or ATD in same round | Each agent's final position; majority position marked as recommendation |
| **Forced Close** | 5 rounds reached | Each agent's final position; majority position marked as recommendation |

**Majority determination:** Count agent verdicts. If a strict majority (>50%) agrees, that's the recommendation. If no majority exists (e.g., 5 agents, 3 different positions), mark as "UNRESOLVED — no majority."

### Debate Transcript Accumulation

The orchestrator maintains the full transcript for each issue. After each agent turn, append to the transcript:

```
**{agent_name}** (Round {N}): {agent's response}
```

This accumulated transcript is passed to every subsequent agent call for that issue.

---

## Phase 4: Write Output

After all issues have been debated, write two files.

### `debate.md` — Full transcript

Write this file in the same directory as the artifact (or the working directory if the artifact is inline text):

```markdown
# Adversarial Review: {artifact_name}

**Date**: {YYYY-MM-DD HH:MM}
**Panel**: {panel_name}
**Agents**: {Name1} ({Role1}), {Name2} ({Role2}), ...
**Issues found**: {N unique issues}
**Consensus reached**: {M issues}
**Disagreements**: {K issues}

---

## Issue 1: {title}
**Raised by**: {agent_name} | **Initial severity**: {severity}
**Also flagged by**: {other agents, if any}
**Location**: {file:line or section reference}

### Round 1

**{presenting_agent}**:
{presents issue and position}

**{agent2}**:
{response}

**{agent3}**:
{response}

...

### Round 2

**{agent1}**:
{response, SKIP, or AGREE TO DISAGREE}

...

### Resolution
- **Outcome**: CONSENSUS / AGREE TO DISAGREE / FORCED CLOSE
- **Verdict**: {agreed or majority verdict}
- **Severity**: {final severity}
- **Solution**: {agreed or majority-recommended solution}
- **Dissents**: {agent_name}: "{their position}" | {agent_name}: "{their position}"

---

## Issue 2: {title}
...
```

### `solutions.md` — Actionable summary

```markdown
# Review Solutions: {artifact_name}

**Date**: {YYYY-MM-DD HH:MM}
**Panel**: {panel_name}
**Reviewed by**: {agent list}

## Summary
| Metric | Count |
|--------|-------|
| Issues raised | {N} |
| Consensus | {M} |
| Majority decided | {K} |
| Unresolved | {J} |

---

## Consensus Issues (all agents agree)

### 1. {title} [{severity}]
**Location**: {location}
**Solution**: {the agreed solution}

---

## Majority-Decided Issues

### 1. {title} [{severity}]
**Location**: {location}
**Solution**: {majority solution}
**Dissent** ({agent_name}): {their position}

---

## Unresolved Issues (no majority)

### 1. {title}
**Location**: {location}
**Positions**:
- {agent_name}: {position}
- {agent_name}: {position}
- {agent_name}: {position}
```

---

## Orchestration Rules

1. **Phase 1 is parallel.** Spawn ALL reviewer agents in a single message with multiple Agent tool calls. Do not spawn them one at a time.

2. **Phase 3 debate is sequential per turn.** Each agent gets one Agent tool call per turn. Pass the full accumulated transcript each time. One issue at a time — complete an issue's debate before starting the next.

3. **No external judge.** The agents themselves must reach resolution through argument. There is no tiebreaker — just consensus, agree-to-disagree, or forced close.

4. **Max 5 rounds per issue.** After 5 rounds, force final positions. Do not let debates run forever.

5. **Max 7 agents per panel.** The two mandatory agents (Contrarian, Scalpel) count toward this limit. Custom panels can have at most 5 custom agents + 2 mandatory = 7.

6. **Use `model: "opus"` for all Agent calls.** Quality of debate degrades significantly with smaller models.

7. **Report only.** Write debate.md and solutions.md. Do not auto-apply any fixes. Present the solutions to the user and let them decide.

8. **Deduplicate aggressively in Phase 2.** If three agents all flag "no input validation on X," that's one issue, not three. Credit all agents who raised it.

9. **Contrarian must engage.** If Contrarian SKIPs in Round 1 of any issue, that's a protocol error — Contrarian's job is to push back. Only allow Contrarian to SKIP in Round 2+ after they've made at least one substantive argument.

10. **Scalpel must propose.** When a solution is being discussed, Scalpel must propose the minimal version before they can SKIP. "What if we just add a nil check here instead of restructuring?" is Scalpel's baseline contribution.

---

## Operational Requirements

This workflow requires the most capable available model for all agents. Debate quality degrades significantly with smaller or faster models.

When using Claude Code, pass `model: "opus"` for ALL agent invocations. Do not use Sonnet or Haiku for any agent in this workflow.
