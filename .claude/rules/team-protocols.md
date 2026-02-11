# Agent Team Protocols

## File Ownership

**Default rule:** Each implementer gets exclusive file ownership. No two implementers touch the same file.

### Shared File Strategies

When files must be shared, use one of these:

1. **Interface-first (default for shared types):** One teammate defines shared types/interfaces FIRST as a blocking task (Phase 2 contracts). Other teammates implement against them after.
2. **Append-only (for registrations/config):** Teammates only ADD to shared files (new routes, new schema fields) — never modify existing lines. Lead resolves ordering in INTEGRATE.
3. **Lead handles shared files:** Lead makes surgical edits to shared files based on teammate outputs. This is why the lead is NOT in delegate mode — it needs the ability to code when handling cross-cutting concerns.

### Working Tree Conflict Mitigation

All teammates share the same working tree with no isolation. The lead must sequence tasks that touch adjacent code. If teammate A modifies imports that teammate B depends on, make B's task depend on A's via TaskUpdate addBlockedBy. The SPEC+PLAN phase must identify these dependencies explicitly.

## Build Order Dependencies

The lead must enforce this build order via task dependencies (TaskCreate + TaskUpdate addBlockedBy):

```
contracts/types (Task 1) ──blocks──→ everything else
data layer (Task 2) ──────blocks──→ API endpoints (Task 3)
API endpoints (Task 3) ───blocks──→ business logic (Task 4)
business logic (Task 4) ──blocks──→ frontend (Task 5)
all of the above ─────────blocks──→ integration glue (Task 6)
```

Tasks within the same layer CAN run in parallel (e.g., multiple API endpoints). Tasks across layers must respect dependencies.

## Micro-Validation Protocol

Every implementer must run quality gates after each change:

```bash
# TypeScript
npx tsc --noEmit 2>&1        # Type check
npm test 2>&1                 # Tests
npm run lint 2>&1             # Lint

# Python
python -m mypy . 2>&1        # Type check
pytest 2>&1                   # Tests
ruff check . 2>&1            # Lint

# Go
go build ./... 2>&1          # Build
go test ./... 2>&1           # Tests
golangci-lint run 2>&1       # Lint
```

If any gate fails: fix attempt 1 → fix attempt 2 (different approach) → BLOCKED. Two strikes and pivot.

## Reviewer Checklist

The reviewer (Phase 5: VALIDATE) must check all five points:

1. **Does it break anything?** Run full test suite. Check type compilation. Verify no regressions against existing behavior.
2. **Can I trace the data?** Pick a representative flow. Trace data from input to output through every layer. If any step is a black box, flag it.
3. **Is it over-engineered?** Abstractions nobody asked for? Patterns that add complexity without value? Spaceship when a bicycle would do?
4. **Security basics?** Auth on every endpoint that needs it. Input validation at boundaries. Parameterized queries. No secrets in code. No injection vectors.
5. **Is it testable?** Not "could someone theoretically test this" — is there actually a test? Does it cover the meaningful code paths?

Additionally run:
- Data trace test — pick a flow, trace input to output end-to-end
- Clean build — `rm -rf node_modules dist && npm install && npm run build && npm test`
- Check for debug leftovers: `console.log`, `debugger`, `print()`, `eslint-disable`, `@ts-ignore`

## Message Templates

```
# Teammate → Lead (task complete)
DONE: task-{id}
Files: [list of changed files]
Gates: tsc [PASS/FAIL] | test [PASS/FAIL] | lint [PASS/FAIL]
Data flow: [one-line description of what data goes in and comes out]
Notes: [one-line summary of what was done + any concerns]

# Teammate → Lead (blocked — two strikes exhausted)
BLOCKED: task-{id}
Issue: [one-line description]
Attempt 1: [what was tried, why it failed]
Attempt 2: [different approach tried, why it failed]
Need: [what I need to proceed]

# Teammate → Teammate (interface contract)
CONTRACT: task-{id} → task-{id}
Interface: [function signature / type definition / API endpoint]
Data in: [what this interface accepts]
Data out: [what this interface returns]
Notes: [any assumptions or constraints]

# Lead → User (progress report)
PHASE: [current phase] ([X/Y tasks complete])
Active: [teammate names and current tasks]
Complete: [completed task summaries]
Blocked: [any blocked tasks + reason]
Gates: [overall quality gate status]
Issues: [any concerns requiring user input]
```

## Action-Based Escalation

LLMs can't measure time, so escalation is action-based, not time-based:

- Teammate tried **2 different approaches** to same sub-problem without progress → message lead (two strikes and pivot)
- Teammate discovers work outside their assigned files → message lead, don't expand scope
- Teammate encounters failing tests they didn't cause → message lead
- Reviewer finds critical security issue → lead decides: fix now or flag for user
- Reviewer can't trace data through a layer → flag as black box, lead assigns observability task
- Any teammate unsure about approach → message lead, don't guess

## Progress Reporting

Lead reports to user:
1. Status summary after each phase transition
2. If user messages "status" → structured summary (phase, tasks complete/pending/blocked, gate status)
3. Final report at end: what was done, files changed, tests run, review checklist results

## Git Workflow

- All teammates work on the same branch (no git worktree isolation between team members)
- Teammates do NOT commit during EXECUTE — changes to working tree only
- Lead creates feature branch during INTEGRATE: `feat/<feature-name>`
- Lead commits during INTEGRATE with structured message after reviewing all changes
- If VALIDATE fails, lead can `git checkout -- <files>` for specific teammate outputs needing rework
- Commit message format: `<type>: <description>` (e.g., `feat: add user authentication`)
- Never push to main. Feature branches only. Never merge without user approval.

## LEARN Phase Protocol

After build completes (Phase 7):

1. **Run `/evaluate`** — scores the build, extracts observations, writes to chad-memory DB
2. **If user gives feedback** — record as additional observations via `chad-memory log-observation`
3. **Apply the feedback** — fix what needs fixing
4. **Understand the WHY** — don't just apply the change, understand what principle it reflects
5. **Re-validate** — run the full reviewer checklist (Phase 5) again
6. **Capture the lesson** — if the feedback reveals a pattern:
   - Suggest adding it to `~/.claude/rules/chad-engineering.md`
   - Format: what the feedback was, why the original was wrong, the general principle
7. **Run `/analyze` periodically** — consolidates patterns across builds, promotes rules, archives stale ones, regenerates `chad-memory.md`
8. **Report back** — "Feedback applied, tests pass, ready for final review"

The `/evaluate` step runs after every build. User feedback and `/analyze` are optional but recommended.

## Token Budget Awareness

Estimates include Opus lead + Opus reviewer + Sonnet implementers:

| Preset | Estimated Tokens | Estimated Cost |
|--------|-----------------|---------------|
| solo | 50K-200K | $1-$5 |
| pair | 200K-500K | $5-$15 |
| squad | 500K-1.5M | $15-$40 |
| full | 1M-3M | $30-$80 |

- If session cost exceeds $50, consolidate team (shut down idle agents)
- Lead monitors task completion rate — if progress stalls, consider reducing team size

## Recovery Protocol

**Teammate crash:** Lead spawns replacement via Task tool with team_name, assigns uncompleted tasks. Replacement picks up from git working tree state.

**Lead crash:** After `/resume`, lead checks `~/.claude/teams/` and `~/.claude/tasks/` to rediscover state. Review `git status` and `git diff` for working tree state. Spawn new teammates for incomplete tasks.

**Context overflow:** Teammates report partial progress via SendMessage before context limit. Lead captures state in task descriptions via TaskUpdate.
