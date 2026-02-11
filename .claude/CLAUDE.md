# Global Configuration

Universal Claude Code configuration. Project-level CLAUDE.md and .claude/rules/ files take precedence over everything here.

## Agent Teams

Agent teams are enabled globally (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`). See `~/.claude/rules/` for the full methodology:

- `team-methodology.md` — Workflow presets and phased execution
- `team-protocols.md` — Communication, ownership, escalation, git workflow
- `coding-standards.md` — Universal coding standards
- `chad-engineering.md` — Engineering philosophy (data observability, API-first, two-strikes-and-pivot)

## Chad 2.0 — Autonomous Build Agent

Custom subagent and skill for autonomous end-to-end builds:

- **Subagent:** `~/.claude/agents/chad2.md` — Claude delegates to this for build tasks
- **Skill:** `/build <task>` — Invokes the full build protocol (spec -> scaffold -> build -> test -> deliver)

### When to use

- `/build <task>` — For medium-to-large features requiring the full protocol
- `/build --spec-only <task>` — Stop after spec/contracts (for planning)
- `/build --no-pr <task>` — Build and test but skip PR creation
- `/fix-issue <url>` — For existing GitHub issues (uses TDD loop)

## Memory System

Persistent build memory via `chad-memory` CLI and SQLite:

- **Database:** `~/.claude/memory/chad.db` — stores builds, observations, and rules
- **CLI:** `chad-memory` (globally linked) — init, log-build, log-observation, promote, archive, generate, stats, query
- **Generated rules:** `~/.claude/rules/chad-memory.md` — auto-generated from DB, loaded every session
- **Pipeline:** `/build` -> `/evaluate` (score + observe) -> `/analyze` (consolidate + promote)
- **Skills:** `/evaluate` scores a build, `/analyze` consolidates across builds

## Style Preferences

- No sycophancy. Skip "Great question!" or "That's a great idea!" — just do the work.
- Be concise. Say what needs to be said, nothing more.
- Show, don't tell. Code over prose. Diffs over descriptions.
- When uncertain, ask. Don't guess at requirements or make assumptions about intent.
- Prefer working code over theoretical discussions.

## Project-Level Overrides

Project-level CLAUDE.md takes precedence for:
- Framework-specific patterns and conventions
- Test configuration and commands
- Deployment workflows
- Team-specific coding standards
- Architecture documentation

If a project-level rule conflicts with a global rule, follow the project-level rule.

## Session Recovery

If resuming a session that had an active agent team, check `~/.claude/teams/` and `~/.claude/tasks/` to rediscover team state. Spawn new teammates for any incomplete tasks. Review the working tree (`git status`, `git diff`) to understand what was accomplished before the interruption.
