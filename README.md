# Subagent-Driven Development

A Claude Code skill that executes implementation plans by dispatching fresh subagents per task with mandatory two-stage review gates (spec compliance, then code quality). Supports parallel mode for independent tasks using worktree isolation.

> **Based on** [subagent-driven-development](https://github.com/obra/superpowers/tree/main/skills/subagent-driven-development) from [@obra](https://github.com/obra)'s [superpowers](https://github.com/obra/superpowers) framework. Parallel mode adapted from [spawn-tasks](https://github.com/theradengai/spawn-tasks) by [@theradengai](https://github.com/theradengai).

## Quick Start

```
You: Here's my implementation plan with 5 tasks. Execute it.

Claude: I'm using Subagent-Driven Development to execute this plan.

[Reads plan, extracts all tasks]
[Analyzes dependencies between tasks]
[Tasks 1-3 share files → sequential mode]
[Tasks 4-5 are independent → parallel mode]

Task 1: Add authentication middleware
[Dispatches implementer subagent with full task context]

Implementer: "Should sessions be stored in Redis or memory?"
You: "Redis — we need horizontal scaling"

Implementer: Implemented, 6/6 tests passing, committed.

[Dispatches spec reviewer]
Spec reviewer: ✅ All requirements met

[Dispatches code quality reviewer]
Code reviewer: ✅ Approved

[Tasks 4 & 5: launches both in parallel with worktree isolation]
[Both complete → reviews run in parallel → cross-task integration review]

Done! 5 tasks completed, all reviews passed.
```

## Installation

**Claude Code:**
```bash
# Git clone
git clone https://github.com/wan-huiyan/subagent-driven-development.git ~/.claude/skills/subagent-driven-development
```

**Cursor** (2.4+):
```bash
# Per-project rule (most reliable)
mkdir -p .cursor/rules
# Create .cursor/rules/subagent-driven-development.mdc with SKILL.md content + alwaysApply: true

# Manual global install
git clone https://github.com/wan-huiyan/subagent-driven-development.git ~/.cursor/skills/subagent-driven-development
```

## What You Get

- **Fresh subagent per task** — no context pollution between tasks
- **Two-stage review gates** — spec compliance first (did they build what was asked?), then code quality (is it well-built?)
- **Review loops** — reviewers found issues? Implementer fixes, reviewer re-reviews, repeat until approved
- **Parallel mode** — independent tasks launch simultaneously in isolated worktrees, then reviews run after all complete
- **Cross-task integration review** — after parallel mode, one reviewer checks all changes together
- **Question handling** — subagents can ask clarifying questions before and during work

## Sequential vs Parallel

| Aspect | Sequential Mode | Parallel Mode |
|--------|----------------|---------------|
| **When to use** | Tasks share files or have ordering dependencies | Tasks touch completely different modules |
| **Execution** | One task at a time, review after each | All tasks simultaneously |
| **Isolation** | Single worktree | Separate worktree per task |
| **Wall-clock time** | Sum of all tasks | Slowest task |
| **Review timing** | After each task | After all tasks complete |
| **Extra safeguard** | N/A | Cross-task integration review |

The skill auto-detects which mode to use based on dependency analysis. You can also mix modes: parallel for independent tasks, sequential for dependent ones.

## How It Works

| Step | What Happens |
|------|-------------|
| 1. Extract | Read plan, extract all tasks with full text and context |
| 2. Analyze | Check dependencies between tasks to determine mode |
| 3. Implement | Dispatch implementer subagent(s) — answer questions if asked |
| 4. Spec review | Dispatch spec compliance reviewer — verify code matches requirements |
| 5. Fix spec gaps | Implementer fixes any missing/extra work, re-review until ✅ |
| 6. Quality review | Dispatch code quality reviewer — verify clean, tested, maintainable |
| 7. Fix quality issues | Implementer fixes, re-review until ✅ |
| 8. Integration review | (Parallel mode) One reviewer checks all changes together |
| 9. Finalize | Mark complete, move to next task or finish branch |

## Key Design Decisions

**Why two-stage review instead of one?**
Spec compliance and code quality are different concerns. A beautifully written function that solves the wrong problem still fails. Checking spec first prevents wasted effort on quality-reviewing code that needs to be rewritten.

**Why fresh subagent per task?**
Context pollution is real. A subagent that just implemented Task 1 carries assumptions into Task 2. Fresh subagents get exactly the context they need — nothing more.

**Why not skip reviews in parallel mode?**
Parallel mode saves time on implementation, not on quality. The cross-task integration review is actually *more* important in parallel mode because tasks were developed in isolation.

**Why require user confirmation for parallel mode?**
Dependency analysis can miss subtle coupling (shared database tables, implicit ordering via API calls). The user is the final authority on whether tasks are truly independent.

## Prompt Templates

The skill includes three prompt templates that define how subagents behave:

| Template | Role | Key Behavior |
|----------|------|-------------|
| `implementer-prompt.md` | Builds the feature | Asks questions first, implements, tests, self-reviews, commits |
| `spec-reviewer-prompt.md` | Verifies correctness | Reads actual code (doesn't trust implementer's report), checks for missing/extra work |
| `code-quality-reviewer-prompt.md` | Verifies quality | Reviews after spec compliance passes, checks clean code and test coverage |

## Attribution

The **sequential mode** (core skill) is based on [**subagent-driven-development**](https://github.com/obra/superpowers/tree/main/skills/subagent-driven-development) from the [superpowers](https://github.com/obra/superpowers) framework by [Jesse Vincent (@obra)](https://github.com/obra). The original skill established the pattern of fresh-subagent-per-task with two-stage review gates. This repo adapts and extends that foundation with parallel mode.

The **parallel mode** feature was inspired by and adapted from [**spawn-tasks**](https://github.com/theradengai/spawn-tasks) by [@theradengai](https://github.com/theradengai). spawn-tasks introduced the approach of launching parallel Claude Code sessions with worktree isolation for independent tasks.

This skill combines both ideas: spawn-tasks' parallel worktree execution integrated into superpowers' two-stage review workflow — adding dependency analysis, mandatory review gates after parallel completion, and cross-task integration review.

**Key differences from spawn-tasks:**
- spawn-tasks is fire-and-forget (no review gates) — this skill enforces spec + quality review after parallel completion
- spawn-tasks supports tmux pane spawning — this skill uses the Agent tool with `isolation: "worktree"`
- spawn-tasks writes task files to `.tasks/spawn/` — this skill passes full task text directly to subagent prompts
- spawn-tasks is standalone — this skill integrates into a broader workflow with worktree setup and branch finishing

## Limitations

- **Not for tightly coupled tasks** — if tasks share files or depend on each other's output, sequential mode is the only safe option
- **No tmux integration** — unlike spawn-tasks, this skill doesn't spawn visible tmux panes (uses background Agent tool instead)
- **Review overhead** — two reviewers per task adds cost; for trivial tasks, this may be overkill
- **Parallel mode requires truly independent tasks** — subtle dependencies (shared DB tables, implicit API ordering) can cause merge conflicts
- **Controller stays busy** — the main session coordinates everything, so you can't do other work while it's running

## Dependencies

**Required:**
- Claude Code with Agent tool support (for dispatching subagents)

**Optional (enhance the workflow):**
- Git worktree support (for parallel mode isolation)
- A plan file to execute (the skill expects an implementation plan with discrete tasks)

<details>
<summary>Quality Checklist</summary>

What this skill guarantees:

- [ ] Every task gets a fresh subagent (no context pollution)
- [ ] Spec compliance review before code quality review (correct order)
- [ ] Review loops continue until reviewer approves (no "close enough")
- [ ] Parallel mode only when tasks are truly independent
- [ ] User confirmation before launching parallel mode
- [ ] Cross-task integration review after parallel completion
- [ ] Subagent questions answered before implementation proceeds
- [ ] Full task text provided to subagents (not file references)

</details>

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-03-24 | Initial release: sequential mode with two-stage review + parallel mode with worktree isolation (adapted from spawn-tasks) |

## License

MIT
