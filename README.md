# Claude Workflow

Disciplined agent workflow system for Claude Code. Linear as source of truth, Gherkin specs, TDD execution, deterministic verification.

## Components

| Component | Purpose |
|-----------|---------|
| `/brainstorm` | Turn business goal into Outcomes and Work Units |
| `/work` | Execute single Work Unit with TDD |
| `/execute-outcome` | Batch execute Work Units with dependency-aware ordering |
| `reviewer` | Two-pass code review (deterministic + LLM quality) |

## Installation

```bash
# Clone repo
git clone <repo-url> ~/Code/claude-workflow

# Symlink to ~/.claude
ln -sf ~/Code/claude-workflow/skills/brainstorm ~/.claude/skills/brainstorm
ln -sf ~/Code/claude-workflow/skills/work ~/.claude/skills/work
ln -sf ~/Code/claude-workflow/skills/execute-outcome ~/.claude/skills/execute-outcome
ln -sf ~/Code/claude-workflow/agents/reviewer.md ~/.claude/agents/reviewer.md
```

## Workflow

```
/brainstorm "goal"     → Creates Outcomes + Work Units in Linear
/execute-outcome BHU-X → Executes all Work Units for an Outcome
/work BHU-Y            → Executes single Work Unit (TDD)
```

## Key Features

- **Git worktrees** for parallel Work Unit execution
- **Commit format**: `[BHU-XXX] <type>: <description>`
- **Budget gates**: small (<200 LOC), medium (200-400), large (>400)
- **Two-pass review**: deterministic checks + LLM quality review
