# Agent Team Methodology

Chad 2.0's engineering philosophy drives every team. The rules in `chad-engineering.md` (data observability, API-first, elegant simplicity, two-strikes-and-pivot) are always loaded — every teammate inherits them automatically.

## Workflow Presets

The lead selects a preset based on task scope. Not every task needs a full team.

| Preset | Team Size | Phases | When to Use |
|--------|-----------|--------|-------------|
| **solo** | 0 (no team) | Just work | Single-file fixes, docs, < 3 files |
| **pair** | 1 teammate | SCAFFOLD + EXECUTE + VALIDATE | Medium changes, 2-5 files, clear scope |
| **squad** | 2-4 teammates | SPEC + SCAFFOLD + EXECUTE + VALIDATE + INTEGRATE | Features, multi-file, parallelizable |
| **full** | 4-7 teammates | All phases (INDEX→LEARN) | Large features, refactors, greenfield |
| **explore** | 2-3 teammates | Parallel investigation + synthesis | Debugging, architecture research, competing hypotheses |

### Preset Selection Heuristic

- Task fits in one sentence + touches < 3 files → **solo**
- Task is clear + touches 2-5 files + can split into independent chunks → **pair**
- Task needs architectural planning + touches >5 files → **squad**
- First time in codebase OR large refactor OR greenfield feature → **full**
- Root cause unknown OR "which approach is best?" → **explore**

## Phase Definitions

```
Phase 0: INDEX (Lead solo)
├── Use Glob to enumerate files by directory structure
├── Use Grep for import/export patterns to map dependencies
├── Read key files: entry points, config, package.json, README
└── Output: structured codebase understanding for planning

Phase 1: RECON (Lead + 0-2 Scouts)
├── Only if INDEX flagged areas needing deeper investigation
├── Spawn scouts for specific areas
└── Scouts report back, lead synthesizes

Phase 2: SPEC + PLAN (Lead solo)
├── Define data flow: what comes in, what goes out, what transforms
├── Write API contracts: endpoints, request/response types, error formats
├── Define acceptance criteria: specific, testable conditions for "done"
├── Write spec to docs/SPEC.md or project README
├── Break work into non-overlapping file assignments
├── Identify shared files → use shared file protocol
│   (interface-first / append-only / lead-handles)
├── Create task list with dependencies using TaskCreate
├── Each task: files owned, acceptance criteria, relevant context
├── Enforce build order via task dependencies:
│   1. Contracts/types (blocking — all other tasks depend on this)
│   2. Data layer (models, schemas, database)
│   3. API endpoints (implement the contracts)
│   4. Business logic (services, transformations)
│   5. Frontend (consuming the API, if applicable)
│   6. Integration glue (auth, middleware, error handling)
└── Gate: Spec exists. Data flow clear. Contracts defined. Task list created.

Phase 3: SCAFFOLD (Lead or 1 Implementer — blocking)
├── Initialize project (package.json / pyproject.toml / go.mod)
├── Separate frontend and backend (even for small projects)
├── Set up test harness from day one (vitest/jest/pytest)
├── Install only needed dependencies
├── Verify: build passes, test runner executes (even with zero tests)
├── Commit: "init: project scaffold with test harness"
└── Gate: Clean build. Test runner works. All other tasks unblocked.

Phase 4: EXECUTE (Lead + N Implementers)
├── Spawn implementers per work stream via Task tool with team_name
├── Each gets: task, file ownership, architecture context, API contracts
├── Implementers follow the micro-validation loop on every change:
│   1. Write failing test (TDD when test framework exists)
│   2. Implement the feature
│   3. Run quality gates: type check + tests + lint
│   4. If ALL pass → report DONE to lead
│   5. If FAIL → fix attempt 1
│   6. If still FAIL → fix attempt 2 (DIFFERENT approach)
│   7. If still FAIL → report BLOCKED to lead (two strikes and pivot)
├── Lead monitors via TaskList, redirects if stuck
├── Lead handles shared-file edits directly when needed
└── Implementers do NOT commit — changes to working tree only

Phase 5: VALIDATE (Lead + 1 Reviewer)
├── Reviewer runs Chad's 5-point review checklist:
│   1. Does it break anything? Tests pass, types compile, no regressions
│   2. Can I trace the data? Is the flow observable end-to-end?
│   3. Is it over-engineered? Spaceship when a bicycle would do?
│   4. Security basics? Auth, validation, parameterized queries
│   5. Is it testable? Not theoretically — is there actually a test?
├── Reviewer runs full validation suite:
│   - Unit tests (core business logic)
│   - Integration tests (API endpoints, database)
│   - E2E tests (Playwright if frontend exists)
│   - Data trace test (pick a flow, trace input to output)
│   - Security check (auth, injection, secrets in code)
│   - Clean build (rm -rf node_modules dist && install && build && test)
├── Reports issues to lead with specific file + line references
├── Lead assigns fixes to original implementers via SendMessage
└── Iterate until all 5 review points pass

Phase 6: INTEGRATE (Lead solo)
├── Review all changes holistically
├── Run full test suite one final time
├── Resolve any cross-file issues from parallel work
├── Create feature branch: feat/<feature-name>
├── Make single structured commit or atomic commits per work stream
├── Push to feature branch (never main)
├── Create PR with structured summary:
│   - What was built (1-3 sentences)
│   - Architecture decisions and why
│   - What's tested and how
│   - What to look at closely
│   - Known limitations or shortcuts
├── Send shutdown_request to all teammates
├── TeamDelete to clean up
└── Report to user: PR link, files changed, tests run, issues

Phase 7: LEARN (Lead solo — after build)
├── Run /evaluate against the completed build
│   └── Scores metrics, extracts observations, writes to chad-memory DB
├── If user gives feedback → record as observations
├── Periodically run /analyze to consolidate patterns across builds
│   └── Detects recurring patterns, promotes rules, archives stale ones
├── chad-memory.md auto-updated for next session
├── Re-run Phase 5 validation (tests still pass after changes)
├── If feedback reveals a pattern not in chad-engineering.md:
│   suggest adding it as a new rule
└── Report: "Feedback applied, tests pass, ready for final review"
```

## Model Selection

- **Lead**: Always Opus — best reasoning for planning, supervision, architectural decisions, and the SPEC phase. Worth the cost for the coordinator role.
- **Scouts**: Sonnet — fast exploration, doesn't need deep reasoning.
- **Implementers**: Sonnet default. Use Opus when: >5 files, cross-module refactoring, security-critical code, novel algorithms.
- **Reviewer**: Opus — needs to catch subtle bugs and trace data flow.

## When NOT to Use Teams

Do not use agent teams when:
- Task touches fewer than 3 files
- All files are interdependent (can't split into non-overlapping sets)
- Task is sequential (step B requires output of step A with no parallel work)
- Project has no test framework (no way to validate parallel work)
- On a tight token budget (<$5 available)
- Task is mostly investigation/reading (use explore preset or subagents)
- Working on non-code projects (novels, docs)

When in doubt, start with **solo** or **pair**. Escalate to a larger preset only if needed.
