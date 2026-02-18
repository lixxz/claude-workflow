---
name: work
description: Execute a Linear Work Unit issue with disciplined TDD workflow
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, mcp__linear-server__get_issue, mcp__linear-server__update_issue, mcp__linear-server__create_comment, mcp__linear-server__list_comments
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

## File Creation Rules

**NEVER create documentation or markdown files in the code repository.**

This includes:
- Verification runbooks
- README files
- Design documents
- Checklists
- Any `.md` files

**Why:** Linear is the source of truth. Documentation in repos drifts from Linear and clutters the codebase.

**Where state goes instead:**

| Information | Location |
|-------------|----------|
| Verification steps | Linear issue "Done When" (Gherkin) |
| Progress tracking | Linear comments (create/update as you work) |
| Execution evidence | Completion comment (Step 10) |
| Decisions | Issue description "Decisions" section |
| Learnings | Completion comment |

**Only create files that are actual code artifacts** (source code, tests, migrations, config files).

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

1. Fetch issue by calling the `mcp__linear-server__get_issue` tool with `id: "$ARGUMENTS"` and `includeRelations: true` (the issue ID passed to /work)
2. Parse the description and validate required sections:
   - **Want** (required): 1-2 sentences describing outcome
   - **Repo** (required): Which repository this work belongs to
   - **Done When** (required): Gherkin scenarios in ```gherkin block
   - **Approach** (required): Can be "TBD" if agent should propose
   - **Decisions** (optional): Updated during implementation

3. **If validation fails**: STOP and report exactly what's missing. Do not proceed.

4. **Validate current repository matches the Repo field:**
   ```bash
   # Get current repo name
   basename $(git rev-parse --show-toplevel)
   # Or for full path comparison
   git rev-parse --show-toplevel
   ```

   - If repo matches: proceed
   - If repo does NOT match: STOP and report:
     ```
     ⚠️ Repository mismatch

     This Work Unit is for: [repo from issue]
     Current directory is: [current repo]

     Please run `/work [issue-id]` from the correct repository.
     ```

   **Do not proceed if in the wrong repository.**

5. Extract and display Gherkin scenarios:
   ```
   Found N scenarios:
   1. [Scenario name]
   2. [Scenario name]
   ...
   ```

6. **Mark issue as In Progress:**
   ```
   mcp__linear-server__update_issue:
     id: [issue ID]
     state: "In Progress"
   ```

7. **Update parent issue if needed:**

   If this Work Unit has a parent issue (the Outcome):
   a. Fetch the parent issue:
      ```
      mcp__linear-server__get_issue:
        id: [parent ID from step 1]
      ```
   b. If parent status is "Backlog" or "Todo", mark it as "In Progress":
      ```
      mcp__linear-server__update_issue:
        id: [parent ID]
        state: "In Progress"
      ```
   c. Report:
      ```
      Parent issue [parent ID] marked as In Progress.
      ```

### Step 2: Check Budget

1. Check issue labels for `budget:small`, `budget:medium`, or `budget:large`
2. Budget limits:
   - `small`: < 200 LOC
   - `medium`: 200-400 LOC (requires justification in commit)
   - `large`: > 400 LOC (requires human approval before starting)

3. **If `budget:large`**: **HARD STOP**: End your message here after reporting the budget. Do NOT call any more tools or proceed. Wait for the user to explicitly approve before continuing.

### Step 3: Validate and Confirm Approach

The approach may have been written during brainstorm before dependencies were completed. Validate it's still correct given what actually happened.

#### 3.1: Gather Dependency Context

1. Fetch the issue with relations:
   ```
   mcp__linear-server__get_issue:
     id: [issue ID]
     includeRelations: true
   ```

2. For each **blockedBy** issue (if any):
   a. Fetch the blocker issue details:
      ```
      mcp__linear-server__get_issue:
        id: [blocker ID]
      ```
   b. Check if blocker is **Done** — if not Done, STOP:
      ```
      ⚠️ Dependency not complete

      This Work Unit is blocked by: [blocker ID] - [title]
      Status: [blocker status]

      Complete the blocking issue first, or remove the dependency if no longer needed.
      ```
   c. Extract from blocker's description:
      - **Decisions** section (architectural choices made)
      - **Approach** section (what they planned to do)
   d. Fetch blocker's completion comment:
      ```
      mcp__linear-server__list_comments:
        issueId: [blocker ID]
      ```
      Extract **Learnings** from the completion comment.

3. Compile dependency context:
   ```
   ## Dependency Context

   ### [Blocker ID]: [Title]
   **Status:** Done
   **Approach taken:** [from their Approach section]
   **Decisions:**
   - [decision 1]
   - [decision 2]
   **Learnings:**
   - [learning 1]
   - [learning 2]

   ### [Blocker ID 2]: [Title]
   ...
   ```

#### 3.2: Explore Current Codebase

1. Identify what the blockers likely produced:
   - Search for commits: `git log --oneline --grep="[blocker-id]"`
   - Check files changed: `git show --stat [commit-hash]`
   - Read relevant code to understand actual implementation

2. Note any differences from what was anticipated:
   - New abstractions or patterns introduced
   - Different file locations than expected
   - API contracts that differ from assumptions

#### 3.3: Validate or Propose Approach

**If Approach is "TBD":**
1. Using dependency context + codebase exploration, propose approach in 1 paragraph
2. **HARD STOP**: End your message here. Do NOT call any more tools, do NOT proceed to Step 4, do NOT update the issue. Wait for the user to explicitly approve or revise the approach.
3. Once user approves, update issue Approach section via MCP, then proceed to Step 4.

**If Approach exists (not TBD):**

Check for invalidation signals:

| Signal | Example | Action |
|--------|---------|--------|
| **Assumption violated** | Approach assumes table X exists, but blocker created table Y instead | Revise |
| **Decision conflict** | Approach uses pattern A, but blocker decided on pattern B | Revise |
| **Learning reveals blocker** | Blocker learned "X doesn't work because Y" and approach relies on X | Revise |
| **Code structure changed** | Approach references files/functions that don't exist or moved | Revise |
| **New abstraction available** | Blocker created a utility that this WU should use | Revise to use it |

**If any invalidation signal detected:**
1. Report what invalidated the approach:
   ```
   ## Approach Invalidated

   **Original approach:**
   [existing approach text]

   **Invalidation reason:**
   [Blocker ID] decided/learned: [specific conflict]

   **Current codebase state:**
   [what actually exists now]

   **Proposed revised approach:**
   [new approach in 1 paragraph]
   ```
2. **HARD STOP**: End your message here. Do NOT call any more tools, do NOT proceed to Step 4, do NOT update the issue. Wait for the user to explicitly approve or revise the approach.
3. Once user approves, update issue Approach section via MCP, then proceed to Step 4.

**If no invalidation signals:**
1. Confirm approach is still valid:
   ```
   ✓ Approach validated against dependency chain

   Dependencies completed:
   - [Blocker 1]: [key decision/learning relevant to this WU]
   - [Blocker 2]: [key decision/learning]

   Approach remains valid. Proceeding with:
   [existing approach text]
   ```
2. Proceed to Step 4 (no human approval needed)

### Step 4: Generate Tests (RED)

#### Testing Principle: Test Real Behavior, Mock Only at System Boundaries

Tests must verify that the **system actually works**, not that mocked internals return expected values. A mock-heavy test can pass while the real system is broken — that's worse than no test.

**Default to integration tests** that exercise the real interface (API endpoint, component render, SQL query). Only mock things you don't control:

| Mock this (system boundaries) | Do NOT mock this (internals) |
|-------------------------------|------------------------------|
| External APIs (payment gateways, third-party services) | ORM / database queries / model methods |
| Async task dispatch (Celery `.delay()`, job queues) | View logic / serializers / middleware |
| Network I/O (HTTP calls, email, push notifications) | Component state / hooks / rendering |
| Native modules (camera, GPS, filesystem) | Business logic / access control |
| Time (`now()`) when testing time-dependent logic | Internal function calls within the same app |

**The test:** if you can delete the `@patch` decorator and the test still makes sense (just slower or needs a real service), it should probably not be mocked. If removing the mock would call a payment gateway or send a push notification, keep the mock.

Based on proof type label:

**proof:tdd** (Python/Django):
1. Write integration tests that call real API endpoints via the test client (`self.client.get/post`)
2. Create real DB records in `setUp` — users, models, related objects
3. Use real authentication (`force_authenticate` or `AccessToken.for_user()`)
4. Assert on: HTTP status codes, response field values, **DB state changes** (`obj.refresh_from_db()`)
5. Mock only: payment gateways, Celery task dispatch, external API calls
6. Run tests — they MUST FAIL (RED)
7. If tests pass before implementation → STOP, scenarios may be wrong

**proof:rn-component** (React Native):
1. Write integration tests that render real component trees (`render(<Screen />)`)
2. Use real hooks, real state, real context providers
3. Mock only: native modules, network requests (prefer MSW over manual mocks), navigation
4. Assert on: rendered output, user interaction results, state transitions
5. Run `npm test -- <test_file>` — tests MUST FAIL

**proof:dbt-test** (dbt):
1. Create/update schema.yml with tests matching scenarios
2. Run `dbt compile --select <model>` to validate
3. Run `dbt test --select <model>` — tests MUST FAIL

**proof:infra-runbook**:
1. Parse Gherkin scenarios as verification checklist (keep in memory, NOT as a file in repo)
2. Post initial progress comment to Linear issue (save the comment ID):
   ```markdown
   ## Verification Progress

   | # | Scenario | Status |
   |---|----------|--------|
   | 1 | [Scenario name] | ⬜ Pending |
   | 2 | [Scenario name] | ⬜ Pending |

   *Will reply to this thread with updates.*
   ```
3. No automated RED phase — proceed to implementation
4. **As you work:** Reply to the progress comment thread (use `parentId`) with updates:
   ```markdown
   **Scenario 1: [name]** ✅

   ```bash
   [command run]
   ```
   ```
   [verbatim output snippet]
   ```
   ```
5. **To check current state:** Read comments via `mcp__linear-server__list_comments` and find the progress thread
6. Post final summary reply when all scenarios verified, before proceeding to Step 8

**proof:manual**:
1. Parse Gherkin scenarios as verification checklist (keep in memory)
2. Post initial progress comment to Linear issue (same format as infra-runbook)
3. No automated RED phase — proceed to implementation/execution
4. Execute the task, verifying each scenario manually
5. **As you work:** Reply to progress thread with evidence for each scenario verified
6. Each verification must include: command run, verbatim output, and pass/fail result
7. Post final summary reply when all scenarios verified, before proceeding to Step 8

Use `proof:manual` for tasks that require human judgment or execution against live systems (e.g., data migrations, production validations) where automated testing isn't feasible.

### Step 5: Implement (GREEN)

1. Write minimal code to make tests pass
2. Follow YAGNI — no features beyond what scenarios specify
3. After implementation, run tests again
4. Tests MUST PASS (GREEN)
5. Report actual test output (exit code + summary)

### Step 5b: Lint & Format

Run the project linter/formatter on changed files:

**Python:**
1. `ruff check --fix <changed_files>` — auto-fix lint issues
2. `ruff format <changed_files>` — format code
3. If ruff reports unfixable errors, fix them manually before proceeding.

**TypeScript/JS:**
1. `npx eslint --fix <changed_files>`
2. `npx prettier --write <changed_files>`

Stage any formatting changes before proceeding to Step 6.

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

**MANDATORY — DO NOT SKIP THIS STEP.** Every work unit must be reviewed before completion, regardless of scope or complexity. Skipping review is a workflow violation.

**CRITICAL: Do NOT run the reviewer agent in the background.** It must run in the foreground because it needs interactive Bash tool approvals. Background execution will hang.

Launch the `reviewer` agent with this prompt template (fill in the bracketed values):

```
Review the implementation for Linear issue [ISSUE_ID]: "[TITLE]".

**Issue requirements (Gherkin scenarios):**
[list scenario names from the issue]

**Files changed:**
[list files from git diff --name-only]

**Standard quality checks (always verify):**
- Import hygiene: all at module level, grouped per CLAUDE.md, no unused
- Typing: function signatures annotated
- Mock paths: target lookup module, not source module
- No bare except without logging
- CLAUDE.md adherence: Deep Modules, Data Contracts, Semantic Failures
```

Wait for review verdict:
- **APPROVE** → proceed to Step 10
- **NEEDS_CHANGES** → fix issues, re-run reviewer
- **BLOCKED_FOR_HUMAN** → stop, report to human, wait for decision

Max 3 review iterations. If still failing, escalate to human.

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

### Step 11: Mark Issue Done

After completion comment is posted, update issue status:
```
mcp__linear-server__update_issue:
  id: [issue ID]
  state: "Done"
```

**The completion comment MUST be posted BEFORE marking done.** The comment is the audit trail that justifies the Done status.

#### 11.1: Check if Parent Outcome is Complete

If this Work Unit has a parent issue (the Outcome), check if all sibling Work Units are now Done:

1. Fetch all sub-issues of the parent:
   ```
   mcp__linear-server__list_issues:
     parentId: [parent ID]
   ```

2. Check status of each sibling:
   - Count total sub-issues
   - Count sub-issues with status "Done"

3. **If all sub-issues are Done**, mark parent Outcome as Done:
   ```
   mcp__linear-server__update_issue:
     id: [parent ID]
     state: "Done"
   ```
   Report:
   ```
   ✓ All Work Units complete (N/N)
   Parent Outcome [parent ID] marked as Done.
   ```

4. **If some sub-issues remain**, report progress:
   ```
   Work Unit complete. Outcome progress: X/Y Work Units done.
   Remaining:
   - [sibling ID]: [title] ([status])
   - [sibling ID]: [title] ([status])
   ```

### Step 12: Summary to User

Present the final summary to the user:

```markdown
## Work Unit Complete

**Issue:** [ID] - [Title]
**Parent Outcome:** [parent ID] - [parent title]
**Outcome Progress:** X/Y Work Units done [✓ Complete | In Progress]
**Scenarios:** N/N passing
**Lines changed:** X (budget: Y)
**Decisions logged:** N
**Learnings posted:** Y
**Review:** APPROVED

Completion comment posted to Linear.
Issue marked Done.
[Parent Outcome marked Done. | Remaining: A, B, C]
Ready for merge.
```

## Validation Rules

| Section | Required | Validation |
|---------|----------|------------|
| Want | Yes | Non-empty, < 50 words |
| Repo | Yes | Non-empty, must match current working directory |
| Done When | Yes | Contains valid Gherkin (Given/When/Then) |
| Approach | Yes | Non-empty (can be "TBD") |
| Proof label | Yes | Exactly one of: proof:tdd, proof:dbt-test, proof:rn-component, proof:infra-runbook, proof:manual |
| Budget label | Yes | Exactly one of: budget:small, budget:medium, budget:large |

## Error Handling

- **Missing section**: Report which section is missing, do not proceed
- **Invalid Gherkin**: Report parsing error, do not proceed
- **Repo mismatch**: Report expected vs actual repo, do not proceed
- **Dependency not complete**: Report which blocker is not Done, do not proceed
- **Approach invalidated**: Report invalidation reason and propose revision, wait for approval
- **Tests pass before implementation**: Report as error, scenarios may not be testing new behavior
- **Budget exceeded**: Report actual vs allowed, require split or justification
- **MCP failure**: Report error, do not claim success
