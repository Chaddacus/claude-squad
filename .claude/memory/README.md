# Persistent Memory System

The memory system creates a feedback loop where every build contributes to improving future builds.

## Architecture

```
/build (any task)
    |
    v
/evaluate (score + extract observations)
    |
    v
chad.db (SQLite — builds, observations, rules)
    |
    v
/analyze (consolidate patterns, promote rules)
    |
    v
rules/chad-memory.md (auto-generated, loaded every session)
    |
    v
next /build (has accumulated knowledge)
```

## Database Schema

### builds
- `id`, `project`, `task`, `preset`, `files_changed`, `tests_total`, `tests_passed`
- `fix_attempts`, `blocked_tasks`, `review_findings`, `status`, `pr_url`, `notes`
- `created_at`

### observations
- `id`, `build_id`, `category` (pattern/mistake/tool/process)
- `summary` (actionable statement), `detail` (full context)
- `applies_to` (typescript/python/go/api/testing/frontend/database/all)
- `frequency` (auto-incremented on duplicates), `status` (raw/candidate/active/archived)

## Observation Lifecycle

1. **raw** — Newly recorded, seen once
2. **candidate** — Seen 2+ times, eligible for promotion
3. **active** — Promoted to a rule, included in generated memory file
4. **archived** — Not seen in 10+ builds, retired (can be re-promoted)

## Promotion Gates

All must pass for an observation to become an active rule:

| Gate | Requirement |
|------|-------------|
| Frequency | Seen >= 2 times |
| Actionable | Contains a concrete verb ("do X", "use Y", "avoid Z") |
| Measurable | Can be verified via code review, linting, or test output |
| Not redundant | No existing active rule covers the same thing |
| Specific | Applies to a concrete phase/step/tool (not vague) |

## Implementation

The `chad-memory` CLI is a Node.js tool backed by better-sqlite3. To build your own:

1. Create a SQLite database with the schema above
2. Build a CLI with these commands: `init`, `log-build`, `log-observation`, `promote`, `archive`, `generate`, `stats`, `query`
3. The `generate` command writes `~/.claude/rules/chad-memory.md` with all active rules formatted for Claude Code
4. Link it globally: `npm link`

The key insight: Claude Code loads all files in `~/.claude/rules/` every session. So `chad-memory.md` gets loaded automatically, giving every session access to accumulated knowledge.
