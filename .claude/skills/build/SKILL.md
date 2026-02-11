---
name: build
description: Execute Chad 2.0's autonomous build protocol with agent teams — takes a task description, selects the right team size, and builds end-to-end with spec, scaffold, parallel implementation, validation, and PR creation.
argument-hint: "[task description or spec]"
disable-model-invocation: true
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, Task, WebFetch, TaskCreate, TaskUpdate, TaskList, TeamCreate, TeamDelete, SendMessage
---

# /build — Autonomous Build Protocol (Agent Teams)

Take a task description and execute Chad 2.0's build protocol using agent teams for parallel execution.

## Usage

```
/build Create a CLI tool that fetches GitHub activity and generates a daily digest
/build Add user authentication with JWT tokens to the Express API
/build --spec-only Implement a billing tracker that monitors email invoices
/build --solo Fix the date parsing bug in src/utils/date.ts
```

## Flags

| Flag | Effect |
|------|--------|
| (none) | Auto-select preset based on task scope, full protocol |
| `--spec-only` | Stop after SPEC phase (contracts + data flow only) |
| `--no-pr` | Build and test but skip PR creation |
| `--solo` | Force solo mode (no team, lead does everything) |
| `--pair` | Force pair mode (1 implementer) |
| `--squad` | Force squad mode (2-4 implementers) |

## Step 0: Assess and Select Preset

Parse the task from `$ARGUMENTS`. If no preset flag is given, auto-select:

| Signal | Preset |
|--------|--------|
| Touches <3 files, one-sentence task | **solo** — no team |
| Clear scope, 2-5 files, splittable | **pair** — 1 implementer |
| Needs architecture, >5 files, parallelizable | **squad** — 2-4 implementers |
| Greenfield project OR major refactor | **full** — 4-7 teammates |

Read `~/.claude/rules/team-methodology.md` and `~/.claude/rules/team-protocols.md` for the full team methodology. Follow it exactly.

---

## Solo Mode (No Team)

For small tasks. Lead does everything directly.

1. **SPEC** — Define data flow, contracts, acceptance criteria (skip for trivial fixes)
2. **BUILD** — Implement with micro-validation loop:
   - Write failing test → implement → run gates (tsc + test + lint)
   - Pass → commit. Fail → fix attempt 1. Fail again → different approach. Fail again → report blocker.
3. **VALIDATE** — Run Chad's 5-point review checklist against own work
4. **DELIVER** — Feature branch, PR with structured summary

---

## Team Mode (Pair / Squad / Full)

### Phase 1: SPEC + PLAN (Lead solo)

1. If ambiguous, ask ONE round of clarifying questions (batch them)
2. Define:
   - **Data flow** — what comes in, what goes out, what transforms
   - **API contracts** — endpoints, request/response types, error formats
   - **Acceptance criteria** — specific, testable conditions for "done"
3. Write spec to `docs/SPEC.md` or project README
4. Break work into tasks following build order:
   1. Contracts/types (blocking — everything depends on this)
   2. Data layer (models, schemas, database)
   3. API endpoints (implement contracts)
   4. Business logic (services, transformations)
   5. Frontend (if applicable)
   6. Integration glue (auth, middleware, error handling)
5. Create team: `TeamCreate` with descriptive team name
6. Create tasks: `TaskCreate` for each work unit with clear acceptance criteria
7. Set dependencies: `TaskUpdate` with `addBlockedBy` to enforce build order

**If `--spec-only`:** Output the spec and stop here. No team created.

**Gate:** Spec exists. Data flow clear. Contracts defined. Task list created.

### Phase 2: SCAFFOLD (Lead or 1 Implementer — blocking)

1. Spawn first teammate (or lead does it directly for small projects)
2. Initialize project structure
3. Set up test harness from day one
4. Verify: build passes, test runner works
5. Commit: `init: project scaffold with test harness`
6. Mark scaffold task complete → unblocks implementation tasks

**Gate:** Clean build. Test runner executes. Proceed.

### Phase 3: EXECUTE (Lead + N Implementers)

1. Spawn implementers via Task tool with `team_name` parameter:
   - Each gets: specific task, file ownership list, API contracts, patterns to follow
   - Each runs micro-validation on every change (tsc + test + lint)
   - Two strikes and pivot — blocked after 2 failed approaches
2. Lead monitors via `TaskList`, redirects stuck teammates via `SendMessage`
3. Lead handles shared-file edits directly when needed
4. Implementers report DONE or BLOCKED via `SendMessage` to lead
5. Implementers do NOT commit — working tree changes only

### Phase 4: VALIDATE (Lead + 1 Reviewer)

1. Spawn reviewer (Opus model) via Task tool with `team_name`
2. Reviewer runs Chad's 5-point checklist:
   - Does it break anything?
   - Can I trace the data?
   - Is it over-engineered?
   - Security basics?
   - Is it testable?
3. Reviewer runs: unit tests → integration tests → e2e → data trace → security → clean build
4. Issues reported to lead → lead assigns fixes to original implementers
5. Iterate until all 5 review points pass

### Phase 5: INTEGRATE + DELIVER (Lead solo)

1. Review all changes holistically
2. Run full test suite one final time
3. Resolve any cross-file issues from parallel work
4. Create feature branch: `feat/<feature-name>`
5. Commit with structured message
6. Push and create PR (unless `--no-pr`):

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<what was built, 1-3 sentences>

## Architecture
<key decisions and why>

## Data Flow
<how data moves through the system, input to output>

## Testing
<what's tested and how — unit, integration, e2e>

## Review Focus
<what to look at closely>

## Limitations
<known shortcuts or gaps>

---
*Built by /build with agent team ([preset] preset, [N] teammates)*
EOF
)"
```

7. Send `shutdown_request` to all teammates
8. `TeamDelete` to clean up team resources
9. Report to user: PR link, files changed, tests run, review results
10. Remind user: "Run `/evaluate` to score this build and record learnings"

---

## Output Format

### Phase Transitions

```
════════════════════════════════════════
PHASE: SPEC + PLAN — COMPLETE
════════════════════════════════════════
Preset: squad (3 implementers + 1 reviewer)
Contracts: 4 endpoints defined
Tasks: 7 created, dependencies set
Spec: docs/SPEC.md
════════════════════════════════════════
```

### Team Status

```
────────────────────────────────────────
EXECUTE: 4/7 tasks complete
────────────────────────────────────────
impl-data:    DONE  — models + schemas (3 files)
impl-api:     DONE  — REST endpoints (4 files)
impl-logic:   ACTIVE — service layer (2/3 files done)
impl-frontend: WAITING — blocked by impl-logic
────────────────────────────────────────
```

### Final Report

```
════════════════════════════════════════
BUILD COMPLETE
════════════════════════════════════════
Task: <description>
Preset: squad (3 implementers + 1 reviewer)
PR: <url>

Files changed: 12
  - src/models/*.ts (+150)
  - src/routes/*.ts (+200)
  - src/services/*.ts (+180)
  - src/tests/*.test.ts (+300)

Review checklist:
  ✓ No regressions (all tests pass)
  ✓ Data traceable (input → output verified)
  ✓ No over-engineering
  ✓ Security basics covered
  ✓ All code paths tested

Tests: 24 passed, 0 failed
Build: Clean
════════════════════════════════════════
```
