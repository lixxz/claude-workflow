# Claude Workflow

Disciplined agent workflow system for Claude Code. Linear as source of truth, Gherkin specs, TDD execution, deterministic verification.

## Philosophy

**Writing code is cheap. Trust, correctness, and maintainability are expensive.**

This workflow addresses:
- Agents start fresh each session (no persistent mental model)
- Specs drift from implementation
- No rationale trail for decisions
- Hard to verify agent output is correct

**Solution:** Linear is source of truth, Gherkin specs are executable, verification is deterministic, humans review decisions—not implementation.

---

## Linear Hierarchy

```
Project (Business Goal)
└── Milestone (Epic - major deliverable, optional target date)
    └── Issue (Outcome - user-facing value with success metric)
        └── Sub-issue (Work Unit - implementation with Gherkin specs)
```

| Level | Linear Object | What It Represents | Example |
|-------|---------------|-------------------|---------|
| Project | Project | Business Goal | "Land Intelligence App V1" |
| Milestone | Milestone | Epic / Major Deliverable | "User Authentication" |
| Outcome | Issue | User-facing value | "User can register" |
| Work Unit | Sub-issue | Implementation task | "WU: Registration API" |

---

## Core Principles

| Principle | Implementation |
|-----------|----------------|
| **Linear as source of truth** | No markdown specs. Issue IS the spec. |
| **Gherkin = spec = test** | Given/When/Then. No drift possible. |
| **Deterministic verification** | Trust tool output (exit codes), not LLM claims. |
| **Human reviews decisions** | You approve approach and exceptions. Machine verifies correctness. |
| **Enforced brevity** | Templates with limits. If you're skimming, it's too long. |
| **Budget gates** | Large changes require upfront approval. |

---

## Skills & Agents

| Component | Purpose | When to Use |
|-----------|---------|-------------|
| `/poc` | Structured proof-of-concept exploration | Testing ideas before committing |
| `/brainstorm` | Turn goal into Milestones, Outcomes, Work Units | Starting new work |
| `/work` | Execute single Work Unit with TDD | Implementing a task |
| `/execute-outcome` | Batch execute Work Units with dependencies | Running multiple tasks |
| `reviewer` | Two-pass code review | Called automatically by `/work` |

### Workflow

```
Idea
  ↓
/poc "hypothesis"              ← Explore in isolated worktree
  ↓                              Capture learnings
  ↓                              Decide: proceed or drop
  ↓
/brainstorm "goal"             ← Create in Linear:
  ↓                              Milestones, Outcomes, Work Units
  ↓
/execute-outcome BHU-X         ← Execute all Work Units for Outcome
  ↓                              (or /work BHU-Y for single WU)
  ↓
Done                           ← Verify success metric
```

---

## Your Role vs Agent Role

```
┌─────────────────────────────────────────────────────────────┐
│                     YOU (Human)                             │
├─────────────────────────────────────────────────────────────┤
│  STRATEGIC                                                  │
│  ├── Define business goals                                  │
│  ├── Approve milestones and outcomes                        │
│  └── Verify success metrics achieved                        │
│                                                             │
│  DECISIONS                                                  │
│  ├── Approve approach for Work Units                        │
│  ├── Approve large budget (> 400 LOC)                       │
│  ├── Approve critical dependency updates                    │
│  └── Resolve ambiguity and security issues                  │
│                                                             │
│  TASTE                                                      │
│  ├── Set standards in CLAUDE.md                             │
│  ├── Define budget thresholds                               │
│  └── Choose proof types per domain                          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   AGENT (Machine)                           │
├─────────────────────────────────────────────────────────────┤
│  EXPLORATION                                                │
│  ├── Run POC experiments                                    │
│  ├── Log findings with reproducible commands                │
│  └── Recommend proceed/drop with rationale                  │
│                                                             │
│  PLANNING                                                   │
│  ├── Break goals into Milestones                            │
│  ├── Break Milestones into Outcomes                         │
│  ├── Break Outcomes into Work Units                         │
│  └── Create issues in Linear                                │
│                                                             │
│  EXECUTION                                                  │
│  ├── Generate tests from Gherkin                            │
│  ├── Implement to pass tests                                │
│  ├── Update issues with decisions                           │
│  └── Handle dependencies in batch execution                 │
│                                                             │
│  VERIFICATION                                               │
│  ├── Run tests (exit code)                                  │
│  ├── Check types and linter                                 │
│  ├── Verify budget (git diff)                               │
│  └── Two-pass review                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Budget Gates

| Size | LOC | Requirement |
|------|-----|-------------|
| Small | < 200 | None |
| Medium | 200-400 | Justification in commit |
| Large | > 400 | Human approval BEFORE starting |

---

## Proof Types

| Label | Domain | Verification |
|-------|--------|--------------|
| `proof:tdd` | Python/Django | pytest-bdd |
| `proof:dbt-test` | dbt models | dbt test |
| `proof:rn-component` | React Native | Jest |
| `proof:infra-runbook` | Terraform | Plan applied |
| `proof:manual` | UI/exploratory | Checklist + screenshots |

---

## Deterministic Verification

**Trust tool output, not LLM claims.**

| Claim | Verified By |
|-------|-------------|
| "Tests pass" | pytest exit code 0 |
| "Issue created" | MCP response with ID |
| "Budget OK" | git diff --stat parsed |
| "Linter clean" | ruff/eslint exit code 0 |
| "Types check" | mypy/tsc exit code 0 |

---

## Installation

```bash
# Clone repo
git clone <repo-url> ~/Code/claude-workflow

# Symlink skills to ~/.claude
ln -sf ~/Code/claude-workflow/skills/brainstorm ~/.claude/skills/brainstorm
ln -sf ~/Code/claude-workflow/skills/work ~/.claude/skills/work
ln -sf ~/Code/claude-workflow/skills/execute-outcome ~/.claude/skills/execute-outcome
ln -sf ~/Code/claude-workflow/skills/poc ~/.claude/skills/poc

# Symlink agents
ln -sf ~/Code/claude-workflow/agents/reviewer.md ~/.claude/agents/reviewer.md
```

---

## Issue Templates

### Outcome (Issue)

```markdown
## Want
[1-2 sentences: user-facing value]

## Success Metric
- Primary: [metric + target]

## Status
- [ ] Work Units defined
- [ ] Work Units complete
- [ ] Success metric verified
```

### Work Unit (Sub-issue)

```markdown
## Want
[1-2 sentences: what outcome, not how]

## Repo
[exact repo name, e.g., bhume-platform]

## Done When
```gherkin
Scenario: [Clear behavior description]
  Given [precondition]
  When [action]
  Then [expected result]
```

## Approach
TBD - agent proposes during /work

## Decisions
<!-- Added during implementation -->
```

**Multi-repo features:** Create separate Work Units for each repo with dependencies.
Example: `BHU-100: API endpoint` (backend) → `BHU-101: Mobile integration` (mobile, blocked by BHU-100)

---

## Quick Reference

| Question | Answer |
|----------|--------|
| "I have an idea to explore" | `/poc "hypothesis"` |
| "I know what I want to build" | `/brainstorm "goal"` |
| "What should I work on?" | `/brainstorm --status "Project"` |
| "Execute this task" | `/work BHU-XXX` |
| "Execute all tasks for outcome" | `/execute-outcome BHU-XXX` |
| "Continue a POC" | `/poc --resume .worktrees/poc-name` |
