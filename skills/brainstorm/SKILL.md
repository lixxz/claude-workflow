---
name: brainstorm
description: Brainstorm Business Goal into Outcomes and Work Units
allowed-tools: Read, Grep, Glob, mcp__linear-server__get_project, mcp__linear-server__create_project, mcp__linear-server__list_projects, mcp__linear-server__list_issues, mcp__linear-server__get_issue, mcp__linear-server__create_issue, mcp__linear-server__update_issue, mcp__linear-server__list_teams
---

# Brainstorm: Business Goal → Outcomes → Work Units

Help user turn a business goal into well-defined Outcomes and Work Units.

**Linear Hierarchy:**
- Project = Business Goal (few, meaningful groupings)
- Issue = Outcome (user-facing value with success metric)
- Sub-issue = Work Unit (implementation with Gherkin specs)

## Usage
```
/brainstorm [initiative-name or goal description]
/brainstorm --status "Project Name"
```

## Important: Using Linear MCP

Use Linear MCP tools as direct function calls, NOT bash commands.

---

## Status Mode: `--status`

When invoked with `--status "Project Name"`, skip the brainstorm workflow and show current project state.

### How to Fetch Status

1. Find the project:
   ```
   mcp__linear-server__list_projects
   ```
   Match by name to get project ID.

2. Fetch all issues for the project:
   ```
   mcp__linear-server__list_issues:
     project: [project ID]
     includeArchived: false
   ```

3. For each issue, fetch details including relations:
   ```
   mcp__linear-server__get_issue:
     id: [issue ID]
     includeRelations: true
   ```

### Status Report Format

Present the status in this format:

```
## Project Status: [Project Name]

**Progress:** X of Y Work Units complete (Z%)

### Completed ✓
| Issue | Title | Completed |
|-------|-------|-----------|
| BHU-XXX | [title] | [date] |

### In Progress 🔄
| Issue | Title | Assignee |
|-------|-------|----------|
| BHU-YYY | [title] | [name] |

### Ready to Start (Unblocked) 🟢
| Issue | Title | Blocked By (all done) |
|-------|-------|----------------------|
| BHU-ZZZ | [title] | BHU-XXX ✓ |

### Blocked 🔴
| Issue | Title | Waiting On |
|-------|-------|------------|
| BHU-AAA | [title] | BHU-YYY (in progress) |

### Critical Path
BHU-YYY → BHU-AAA → BHU-BBB (final)

**Next action:** Run `/work BHU-ZZZ` to start the next unblocked item.
```

### Status Logic

1. **Completed**: status = "Done" or "Canceled"
2. **In Progress**: status = "In Progress"
3. **Ready to Start**: status = "Backlog"/"Todo" AND all blockedBy issues are Done
4. **Blocked**: status = "Backlog"/"Todo" AND some blockedBy issues are not Done

---

## Workflow

### Phase 0: Identify Project

1. Fetch existing projects via `mcp__linear-server__list_projects`
2. Ask user:
   ```
   Which project should this work go into?
   1. [Project A] (existing)
   2. [Project B] (existing)
   ...
   N. Create new project

   Or specify: /brainstorm --project "Project Name" ...
   ```
3. If new project, get name and create via `mcp__linear-server__create_project`
4. Store project ID for Phase 4

### Phase 1: Understand the Goal

1. If user provided goal description, clarify it
2. Ask clarifying questions ONE AT A TIME:
   - What problem are we solving?
   - Who benefits? (user segment)
   - What does success look like?
   - Any constraints? (time, tech, scope)
3. Summarize understanding, confirm with user

### Phase 2: Propose Outcomes

**Before proposing, run a Necessity Check for each potential Outcome.**

Every line of code is a liability. The best solution is often no new code at all.

1. For each potential Outcome, investigate:
   - **Existing solution?** Search codebase for similar functionality
   - **Config change?** Could this be achieved by changing settings/env vars?
   - **Library/tool?** Is there an existing package that solves this?
   - **80/20 alternative?** Could we get 80% of the value with 20% of the effort?

2. Propose 3-5 Outcomes (user-facing value chunks)
3. Present in this format:
   ```
   ## Proposed Outcomes

   1. **[Outcome name]**
      - Want: [1-2 sentences]
      - Success metric: [measurable]
      - Complexity: Low/Medium/High
      - Necessity check:
        - Existing solution: [None found / Found X but doesn't cover Y]
        - Config alternative: [Not applicable / Considered but Z]
        - Library option: [None suitable / X exists but Y reason not to use]
        - 80/20: [This IS the minimal approach / Could do X instead for less]

   2. **[Outcome name]**
      ...
   ```
4. Ask user to approve, modify, or remove each
5. Confirm final set before proceeding

### Phase 3: Break Down into Work Units

For each approved Outcome:

1. Explore codebase to understand technical scope
2. Propose Work Units (one per bounded context/repo):
   ```
   ## Outcome: [name]

   Work Units:
   1. WU: [title]
      - Repo/area: [e.g., Django API, React Native, dbt]
      - Proof type: [tdd/rn-component/dbt-test/infra-runbook]
      - Budget estimate: [small/medium/large]
      - Key scenarios (draft):
        - [scenario 1]
        - [scenario 2]
   ```
3. Get user approval on breakdown

### Phase 3.5: Map Dependencies

After Work Units are approved, identify execution order:

1. Analyze dependencies between Work Units:
   - Which WUs produce artifacts others need?
   - Which WUs require foundational work first?
   - Which WUs can run in parallel?

2. Present dependency graph:
   ```
   ## Execution Order

   Phase 1 (No blockers - can start now):
   - BHU-XXX: [title]
   - BHU-YYY: [title]

   Phase 2 (After Phase 1):
   - BHU-ZZZ: [title] ← blocked by BHU-XXX

   Critical Path:
   BHU-XXX → BHU-ZZZ → BHU-AAA → BHU-BBB (final)
   ```

3. Present table with recommended order:
   ```
   | Order | Issue | Title | Blocked By |
   |-------|-------|-------|------------|
   | 1 | BHU-XXX | [title] | — |
   | 2 | BHU-YYY | [title] | BHU-XXX |
   ```

4. Get user approval on dependencies before creating in Linear

### Phase 4: Create in Linear

**Hierarchy:** Project (Business Goal) → Issue (Outcome) → Sub-issue (Work Unit)

1. Get team ID via `mcp__linear-server__list_teams`

2. Check if Project exists for this Business Goal, or create one:
   ```
   mcp__linear-server__create_project:
     name: [Business Goal / Initiative name]
     team: [team ID]
     description: [1-2 sentence goal]
   ```

3. For each Outcome, create parent Issue:
   ```
   mcp__linear-server__create_issue:
     title: [Outcome name]
     team: [team ID]
     project: [project ID]
     description: |
       ## Want
       [from approved outcome]

       ## Success Metric
       - Primary: [metric]
       - Secondary: [metric] (optional)

       ## Status
       - [ ] Work Units defined
       - [ ] Work Units complete
       - [ ] Success metric verified
   ```

4. For each Work Unit, create Sub-issue under Outcome:
   ```
   mcp__linear-server__create_issue:
     title: WU-N: [title]
     team: [team ID]
     project: [project ID]
     parentId: [Outcome issue ID from step 3]
     labels: [proof type, budget]
     description: |
       ## Want
       [1-2 sentences]

       ## Done When
       ```gherkin
       Scenario: [from draft scenarios]
         Given ...
         When ...
         Then ...
       ```

       ## Approach
       TBD - agent proposes during /work

       ## Decisions
       <!-- Added during implementation -->
   ```

5. Add blocking relationships from Phase 3.5:
   ```
   mcp__linear-server__update_issue:
     id: [Work Unit issue ID]
     blockedBy: ["BHU-XXX", "BHU-YYY"]  # Issues that must complete first
   ```

   Note: Update each Work Unit that has dependencies. Use issue identifiers (e.g., "BHU-123") in the blockedBy array.

### Phase 5: Summary

Report what was created:
```
## Brainstorm Complete

Project: [Business Goal name] ([Linear URL])

Outcomes created: N
- [Outcome 1] ([Linear URL])
  - WU-1: [title] ([Linear URL])
  - WU-2: [title] ([Linear URL])
- [Outcome 2] ([Linear URL])
  - WU-1: [title] ([Linear URL])
  ...

## Start Here (No Blockers)

| Issue | Title |
|-------|-------|
| BHU-XXX | [title] |
| BHU-YYY | [title] |

## Critical Path

BHU-XXX → BHU-ZZZ → BHU-AAA → BHU-BBB (final validation)

Next: Run `/work [first-unblocked-issue]` to begin execution.
```

## Principles

- **One question at a time** — Don't overwhelm
- **User approves each phase** — Don't create without approval
- **Outcomes are user-facing** — Not technical tasks
- **Work Units are implementation** — Technical, with Gherkin
- **YAGNI** — Remove anything not essential
