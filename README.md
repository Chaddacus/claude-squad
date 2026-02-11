# Claude Squad — Autonomous Claude Code Agent Configuration

An opinionated `~/.claude/` configuration that turns [Claude Code](https://docs.anthropic.com/en/docs/claude-code) into an autonomous software engineering agent with parallel worker teams, persistent memory, and a skill ecosystem.

This is **not** a library or package. It's a set of markdown files that define how Claude Code thinks, plans, and executes.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed (Anthropic's CLI for Claude)
- A Claude API key or Claude Max subscription

## How It Works

Claude Code reads configuration files from `~/.claude/` at the start of every session:

- **`~/.claude/CLAUDE.md`** — Global instructions loaded as system context. This is where you tell Claude Code how to behave across all projects.
- **`~/.claude/rules/*.md`** — Additional instruction files, all loaded automatically. This is where the team methodology, coding standards, and engineering philosophy live.
- **`~/.claude/agents/*.md`** — Custom subagent definitions. When Claude Code delegates work, these define how each subagent type behaves, what tools it has access to, and what model it runs on.
- **`~/.claude/skills/*/SKILL.md`** — Slash command definitions. Each `SKILL.md` becomes a `/command` you can invoke (e.g., `skills/build/SKILL.md` becomes `/build`). Skills contain structured protocols that Claude Code follows step-by-step.
- **`~/.claude/settings.json`** — Permissions, environment variables, and MCP server config.

The key insight: **Claude Code is a protocol executor.** Give it well-structured markdown protocols and it follows them reliably. The difference between "smart autocomplete" and "autonomous engineering agent" is entirely in these configuration files.

### Model Tiers

This config references two Claude model tiers:

- **Opus** — Most capable, best reasoning. Used for the lead agent (planning, supervision, code review). More expensive.
- **Sonnet** — Fast, capable, cheaper. Used for implementer agents (writing code in parallel). Best cost-to-performance for execution tasks.

## What's In Here

```
.claude/
├── CLAUDE.md                 # Global config (loaded every session)
├── settings.example.json     # Permissions, env vars, MCP servers
├── rules/                    # Always-loaded context
│   ├── team-methodology.md   # Workflow presets (solo/pair/squad/full)
│   ├── team-protocols.md     # Communication, ownership, git workflow
│   ├── coding-standards.md   # Universal quality standards
│   └── chad-engineering.md   # Engineering philosophy
├── agents/
│   └── chad2.md              # Solo builder subagent
├── skills/
│   ├── build/SKILL.md        # /build — Full build protocol with agent teams
│   ├── fix-issue/SKILL.md    # /fix-issue — TDD-driven issue resolution
│   ├── evaluate/SKILL.md     # /evaluate — Build scoring + observation extraction
│   └── analyze/SKILL.md      # /analyze — Cross-session pattern learning
└── memory/
    └── README.md             # How the persistent memory system works
```

## The System

### Agent Teams

Claude Code has an experimental feature that lets agents spawn other agents as parallel workers. This config defines structured presets for when and how to use them — from solo (no team, just do the work) to full (7 agents coordinating on a greenfield build).

| Preset | Team Size | When to Use |
|--------|-----------|-------------|
| **solo** | 0 (just the lead) | Single-file fixes, docs, < 3 files |
| **pair** | 1 teammate | Medium changes, 2-5 files, clear scope |
| **squad** | 2-4 teammates | Features, multi-file, parallelizable work |
| **full** | 4-7 teammates | Large features, refactors, greenfield projects |
| **explore** | 2-3 teammates | Debugging, architecture research, competing hypotheses |

Every team follows a phased execution model:

```
SPEC → SCAFFOLD → EXECUTE → VALIDATE → INTEGRATE → LEARN
```

The lead agent (Opus) plans the work, breaks it into tasks with exclusive file ownership (no two agents edit the same file), and delegates to implementer agents (Sonnet) that execute in parallel. A reviewer agent (Opus) then runs a 5-point checklist before anything gets committed. If an implementer fails twice on the same problem, it escalates instead of thrashing.

See `rules/team-methodology.md` for the full phase definitions and `rules/team-protocols.md` for communication and ownership rules.

### Skills

Skills are slash commands backed by structured protocols in `SKILL.md` files. You type `/build create a REST API for user management` and Claude Code executes the entire protocol defined in `skills/build/SKILL.md` — from writing a spec to spawning a team to delivering a PR.

- **`/build <task>`** — Takes a task description, auto-selects team size, and runs the full build lifecycle: spec, scaffold, parallel implementation, validation, PR creation
- **`/fix-issue <url>`** — Autonomous TDD-driven issue fixing with rollback safety, stuck detection, and escalation reporting
- **`/evaluate`** — Scores completed builds on measurable criteria, extracts observations as structured data
- **`/analyze`** — Aggregates observations across builds, detects recurring patterns, promotes proven observations to active rules

### Persistent Memory

Every build feeds a learning loop:

```
/build → /evaluate → chad.db → /analyze → rules/chad-memory.md → next /build
```

After a build completes, `/evaluate` scores it (files changed, tests passed, fix attempts, review findings) and extracts observations — things like "always add a placeholder test during scaffold or vitest exits with error." These get stored in a SQLite database.

`/analyze` periodically reviews all observations across builds. If the same pattern appears 2+ times, it gets promoted to an active rule. Active rules are written to `rules/chad-memory.md`, which Claude Code loads every session. Rules not seen in 10+ builds get auto-archived.

The result: the system gets measurably better at building software over time.

## Engineering Philosophy

The standards baked into every agent and skill:

- **Data in, data out** — Every layer observable. No black boxes. If you can't trace the data, you can't debug it.
- **API-first** — Define contracts before implementation. The spec is the source of truth.
- **Elegant simplicity** — If a junior dev can't read it and understand what it does, it's too clever.
- **Two strikes and pivot** — Same fix fails twice? Stop. Try a completely different approach.
- **Cut the delta** — The biggest productivity lever is reducing the time between attempts. Shrink the feedback loop.

These aren't suggestions — they're loaded into every session and enforced by the reviewer agent during validation.

## Setup

### 1. Copy the config

```bash
# Clone this repo
git clone https://github.com/Chaddacus/claude-squad.git
cd claude-squad

# Back up your existing config if you have one
[ -d ~/.claude ] && cp -r ~/.claude ~/.claude.backup

# Copy files (won't overwrite existing files)
cp -rn .claude/ ~/.claude/
```

### 2. Configure settings

```bash
# Copy the example settings
cp ~/.claude/settings.example.json ~/.claude/settings.json

# Edit with your API keys and preferences
# IMPORTANT: Never commit settings.json — it contains API keys
```

> **Note:** Agent teams require `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in your settings env — this is already set in `settings.example.json`. Agent teams are an experimental Claude Code feature and may change in future releases.

### 3. Set up persistent memory (optional)

The memory system requires a `chad-memory` CLI tool backed by SQLite. This tool is **not included** in this repo — you build it yourself. See `.claude/memory/README.md` for the schema and implementation guide. The `/evaluate` and `/analyze` skills won't work without it, but the rest of the system (teams, `/build`, `/fix-issue`) works independently.

## Cost Awareness

Agent teams consume tokens proportional to team size. Budget accordingly:

| Preset | Estimated Cost |
|--------|---------------|
| solo | $1-$5 |
| pair | $5-$15 |
| squad | $15-$40 |
| full | $30-$80 |

## Customization

This config is opinionated but modular. Fork it and make it yours:

- **Swap presets** — Edit `team-methodology.md` to change team sizes or phase gates
- **Change engineering standards** — Edit `chad-engineering.md` and `coding-standards.md` with your own principles
- **Add skills** — Create new `skills/<name>/SKILL.md` files following the existing pattern
- **Adjust models** — The config uses Opus for leads/reviewers and Sonnet for implementers. Adjust in `team-methodology.md` based on your budget

## License

MIT — use it, modify it, share it. If you build something cool with it, I'd love to hear about it.
