---
name: work
description: Execute a Linear Work Unit issue with disciplined TDD workflow
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, mcp__linear-server__get_issue, mcp__linear-server__update_issue, mcp__linear-server__create_comment
---

# Work Unit Execution

Execute a Linear Work Unit following the disciplined workflow: validate → test → implement → verify → update.

## Coding Standards

**Before writing any code, read and internalize `~/.claude/CLAUDE.md`.**

This file contains the user's engineering principles and coding standards. Key principles to follow:

- **Deep Modules**: Simple public interfaces, complex internals
- **Data Contracts**: Structured returns (dataclasses/interfaces), never raw tuples
- **Semantic Failures**: Domain-specific errors, not technical crashes
- **Diagnostic Discipline**: Validate root cause before fixing

Language-specific rules are also defined:
- 🐍 Python/Django: State hygiene, database sovereignty, bulk operations
- ⚛️ React Native: Functional components, hooks for logic, discriminated unions

**All code written during implementation MUST follow these standards.**

## Usage
```
/work <issue-id>
/work <issue-id> --worktree <path>
```

Examples:
- `/work BHU-140` — Run in current directory
- `/work BHU-140 --worktree .worktrees/wu-BHU-140` — Run in isolated worktree

## Worktree Mode

When `--worktree` is provided:
1. **All file operations** must happen in the worktree path
2. **All git commands** must use `-C <worktree-path>` flag
3. **Tests** must run from within the worktree
4. **Commits** go to the worktree's branch (not main working directory)

```bash
# Example: running tests in worktree
cd .worktrees/wu-BHU-140 && pytest tests/ -v

# Example: checking diff in worktree
git -C .worktrees/wu-BHU-140 diff --stat

# Example: committing in worktree
git -C .worktrees/wu-BHU-140 add . && git -C .worktrees/wu-BHU-140 commit -m "message"
```

**Critical:** Never modify files in the main working directory when in worktree mode.

## Commit Message Format

**Every commit MUST start with the Linear ticket ID in brackets.**

Format:
```
[<ISSUE-ID>] <type>: <description>
```

Examples:
```bash
# Standard commit
git commit -m "[BHU-140] feat: add land value calculation"

# With body
git commit -m "[BHU-140] feat: add land value calculation

Implements the core valuation algorithm based on
comparable sales and market adjustments."

# In worktree mode
git -C .worktrees/wu-BHU-140 commit -m "[BHU-140] feat: add land value calculation"
```

**Commit types:**
- `feat` — New feature
- `fix` — Bug fix
- `refactor` — Code restructuring
- `test` — Adding/updating tests
- `docs` — Documentation only
- `chore` — Maintenance tasks

**Why this matters:**
- Ticket ID immediately visible in `git log --oneline`
- Links commits to Linear issues
- Enables `git log --grep="BHU-140"` to find all related commits

## Important: Using Linear MCP

**You MUST use the Linear MCP server tools directly as function calls, NOT via bash commands.**

Correct:
```
Use the mcp__linear-server__get_issue tool with parameter id: "BHU-140"
```

Wrong:
```bash
claude mcp run linear-server get_issue  # DO NOT DO THIS
```

The MCP tools are available as regular Claude Code tools. Call them like any other tool.

## Workflow

### Step 1: Fetch and Validate Issue

1. Fetch issue by calling the `mcp__linear-server__get_issue` tool with `id: "$ARGUMENTS"` (the issue ID passed to /work)
2. Parse the description and validate required sections:
   - **Want** (required): 1-2 sentences describing outcome
   - **Done When** (required): Gherkin scenarios in ```gherkin block
   - **Approach** (required): Can be "TBD" if agent should propose
   - **Decisions** (optional): Updated during implementation

3. **If validation fails**: STOP and report exactly what's missing. Do not proceed.

4. Extract and display Gherkin scenarios:
   ```
   Found N scenarios:
   1. [Scenario name]
   2. [Scenario name]
   ...
   ```

### Step 2: Check Budget

1. Check issue labels for `budget:small`, `budget:medium`, or `budget:large`
2. Budget limits:
   - `small`: < 200 LOC
   - `medium`: 200-400 LOC (requires justification in commit)
   - `large`: > 400 LOC (requires human approval before starting)

3. **If `budget:large`**: STOP and ask human for approval before proceeding.

### Step 3: Propose Approach (if TBD)

If Approach section says "TBD":
1. Explore codebase to understand context
2. Propose approach in 1 paragraph
3. Wait for human approval before proceeding
4. Once approved, update issue Approach section via MCP

### Step 4: Generate Tests (RED)

Based on proof type label:

**proof:tdd** (Python/Django):
1. Create pytest-bdd feature file from Gherkin
2. Create test file with step definitions
3. Run `pytest <test_file> --collect-only` to validate syntax
4. Run `pytest <test_file>` — tests MUST FAIL (RED)
5. If tests pass before implementation → STOP, scenarios may be wrong

**proof:rn-component** (React Native):
1. Create Jest test file with describe/it blocks from scenarios
2. Run `npm test -- <test_file>` — tests MUST FAIL

**proof:dbt-test** (dbt):
1. Create/update schema.yml with tests matching scenarios
2. Run `dbt compile --select <model>` to validate
3. Run `dbt test --select <model>` — tests MUST FAIL

**proof:infra-runbook**:
1. Document expected state from Gherkin as checklist
2. No automated RED phase — proceed to implementation

### Step 5: Implement (GREEN)

1. Write minimal code to make tests pass
2. Follow YAGNI — no features beyond what scenarios specify
3. After implementation, run tests again
4. Tests MUST PASS (GREEN)
5. Report actual test output (exit code + summary)

### Step 6: Verify Budget

1. Run `git diff --stat` to count lines changed
2. Compare against budget label threshold
3. **If over budget without justification**: STOP and either:
   - Split into smaller commits
   - Add justification and get human approval

### Step 7: Update Issue

1. If any decisions were made during implementation, update Decisions section:
   ```
   - **[Decision topic]**: [What was decided]. Rationale: [Why].
   ```

2. Call the `mcp__linear-server__update_issue` tool with the issue `id` and updated `description`

### Step 8: Compile Execution Evidence

**CRITICAL: This step ensures verifiability. Do not skip or summarize.**

Before proceeding to review, compile evidence that each scenario was actually tested:

```markdown
## Execution Evidence

### Scenario 1: [Scenario name from Gherkin]
**Verified by:**
```bash
[exact command that was run]
```
**Output:**
```
[verbatim output - copy/paste, not paraphrased]
```
**Exit code:** [actual exit code]
**Result:** PASS / FAIL

### Scenario 2: [Scenario name]
**Verified by:**
```bash
[exact command]
```
**Output:**
```
[verbatim output]
```
**Exit code:** [code]
**Result:** PASS / FAIL
```

**Rules:**
- Every Gherkin scenario MUST have a corresponding evidence entry
- Output MUST be verbatim (copy-paste from terminal), never paraphrased or summarized
- If a scenario was NOT actually tested, mark as `**Result:** NOT EXECUTED` — this blocks completion
- Include the actual exit code from the command
- Human can re-run any command to verify the claim

**If any scenario lacks evidence or shows NOT EXECUTED:** STOP. Do not proceed to reviewer.

### Step 9: Run Reviewer

1. Launch the `reviewer` agent with the issue ID
2. Wait for review verdict:
   - **APPROVE** → proceed to summary
   - **NEEDS_CHANGES** → fix issues, re-run reviewer
   - **BLOCKED_FOR_HUMAN** → stop, report to human, wait for decision

3. Max 3 review iterations. If still failing, escalate to human.

### Step 10: Post Completion Comment

Post a completion comment to the Linear issue using `mcp__linear-server__create_comment`.

This comment serves as the permanent record of execution evidence and learnings.

```markdown
## Completion Summary

**Commit:** [hash] [message]
**Files changed:** [count]
**Lines:** +[added] -[removed]

### Execution Evidence

| Scenario | Command | Exit | Result |
|----------|---------|------|--------|
| [Scenario 1 name] | `[cmd]` | 0 | ✓ |
| [Scenario 2 name] | `[cmd]` | 0 | ✓ |

<details>
<summary>Full verification output</summary>

```
[verbatim output from test run]
```

</details>

### Learnings

- [Discovery 1 - things learned during implementation]
- [Discovery 2 - gotchas for future reference]
- [Discovery 3 - context that helps understand the work]

### Verify

To re-run verification:
```bash
[main test command]
```
```

**What goes in Learnings:**
- Unexpected behaviors discovered ("Datastream prefixes tables with `public_`")
- Environment/setup gotchas ("Need ADC credentials, not service account")
- Context for future work ("Only X is currently configured")
- Performance observations ("Query takes ~2s for 10k rows")

**What does NOT go in Learnings (goes in Decisions section instead):**
- Architectural choices that changed the approach
- Trade-offs that were explicitly decided

### Step 11: Summary to User

Present the final summary to the user:

```markdown
## Work Unit Complete

**Issue:** [ID] - [Title]
**Scenarios:** N/N passing
**Lines changed:** X (budget: Y)
**Decisions logged:** N
**Learnings posted:** Y
**Review:** APPROVED

Completion comment posted to Linear.
Ready for merge.
```

**The completion comment MUST be posted before marking done. This is the audit trail.**

## Validation Rules

| Section | Required | Validation |
|---------|----------|------------|
| Want | Yes | Non-empty, < 50 words |
| Done When | Yes | Contains valid Gherkin (Given/When/Then) |
| Approach | Yes | Non-empty (can be "TBD") |
| Proof label | Yes | Exactly one of: proof:tdd, proof:dbt-test, proof:rn-component, proof:infra-runbook |
| Budget label | Yes | Exactly one of: budget:small, budget:medium, budget:large |

## Error Handling

- **Missing section**: Report which section is missing, do not proceed
- **Invalid Gherkin**: Report parsing error, do not proceed
- **Tests pass before implementation**: Report as error, scenarios may not be testing new behavior
- **Budget exceeded**: Report actual vs allowed, require split or justification
- **MCP failure**: Report error, do not claim success
