---
name: evaluate
description: Score a completed build against measurable criteria, extract observations, and write results to the chad-memory database for persistent learning.
argument-hint: "[project path or notes]"
disable-model-invocation: true
allowed-tools: Read, Bash, Grep, Glob, TaskList
---

# /evaluate - Build Evaluation & Observation Extraction

Score a completed build, extract lessons learned, and persist results to the chad-memory database.

## Usage

```
/evaluate
/evaluate /path/to/project
/evaluate "Notes about the build — what went well, what didn't"
```

If `$ARGUMENTS` is empty, auto-detect the project from the current working directory basename. Don't fail — gather and record whatever data is available.

---

## Step 1: Gather Build Data

Collect metrics from the current session and working tree.

### Git History

```bash
# Recent commits on this branch
git log --oneline -20

# Determine how many commits belong to this build
# Look for the first commit that isn't part of the current feature
# (e.g., commits before the feature branch diverged from main)
MERGE_BASE=$(git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null || echo "")
if [ -n "$MERGE_BASE" ]; then
  BUILD_COMMITS=$(git rev-list --count "$MERGE_BASE"..HEAD)
else
  BUILD_COMMITS=1
fi

# Files changed in this build
git diff --stat "HEAD~${BUILD_COMMITS}" 2>/dev/null || git diff --stat HEAD~1
```

### Test Results

Try test runners in order until one succeeds:

```bash
# Node/TypeScript projects
cd <project> && npx vitest run --reporter=json 2>/dev/null

# Fallback: npm test
npm test 2>&1

# Python projects
pytest --tb=short 2>&1

# Go projects
go test ./... 2>&1
```

Parse output for:
- `tests_total`: total number of tests run
- `tests_passed`: number of tests that passed

### Task History

Run `TaskList` to inspect the current session's tasks. For each task, note:
- **fix_attempts**: count tasks that had multiple status changes (pending -> in_progress -> pending -> in_progress counts as 1 retry)
- **blocked_tasks**: count tasks that were ever marked BLOCKED or had "BLOCKED" in their description/comments

### PR Description

```bash
# Check for open PR on current branch
CURRENT_BRANCH=$(git branch --show-current)
gh pr view "$CURRENT_BRANCH" --json title,body,url 2>/dev/null
```

---

## Step 2: Score Metrics

Compute these metrics from the gathered data:

| Metric | Source |
|--------|--------|
| `files_changed` | `git diff --stat` line count |
| `tests_total` | Test runner output |
| `tests_passed` | Test runner output |
| `fix_attempts` | Task history — retries across all tasks |
| `blocked_tasks` | Task history — tasks that hit two-strikes |
| `review_findings` | Count of issues found during VALIDATE phase (estimate from task notes, git log messages mentioning "fix", "review", "finding") |
| `preset` | Infer from task list (solo/pair/squad/full) or branch name pattern |
| `status` | `completed` (all gates pass), `blocked` (unresolved blockers), or `abandoned` (user stopped early) |

Default `status` to `completed` unless evidence suggests otherwise.

---

## Step 3: Extract Observations

For each observation, determine:

- **category**: one of `pattern`, `mistake`, `tool`, `process`
  - `pattern` — a good approach worth repeating
  - `mistake` — a bad approach to avoid
  - `tool` — learning about a library, CLI tool, or framework
  - `process` — workflow or methodology improvement
- **summary**: actionable statement in "do X instead of Y" format (or "always do X" / "never do Y")
- **detail**: full context — what happened, why, what the outcome was
- **applies_to**: one of `typescript`, `python`, `go`, `api`, `testing`, `frontend`, `database`, `all`

### Where to Find Observations

**Blocked tasks:** What caused the block? What finally unblocked it? Category is usually `mistake` (what failed) + `pattern` (what worked).

**Failed quality gates:** What pattern caused the failure? Category: `mistake`. What fixed it? Category: `pattern`.

**Clean first-pass successes:** What approach worked on the first try? Category: `pattern`.

**Review findings:** What did the reviewer catch that the implementer missed? Category: `mistake`.

**User feedback:** What principle was violated? Category: `process` or `mistake`.

**Tool discoveries:** Did a tool behave unexpectedly? Was a new tool/flag/option useful? Category: `tool`.

Generate at least 1 observation. If the build was trivial, note the pattern that made it trivial (e.g., "existing test coverage caught the regression immediately").

---

## Step 4: Write to Database

### Log the build

```bash
chad-memory log-build '{"project":"<name>","task":"<description>","preset":"<preset>","files_changed":<n>,"tests_total":<n>,"tests_passed":<n>,"fix_attempts":<n>,"blocked_tasks":<n>,"review_findings":<n>,"status":"<status>","pr_url":"<url or empty string>","notes":"<any notes from $ARGUMENTS or auto-generated>"}'
```

The `log-build` command returns a build ID. Capture it for the next step.

### Log each observation

For each observation extracted in Step 3:

```bash
chad-memory log-observation '{"build_id":<id>,"category":"<category>","summary":"<summary>","detail":"<detail>","applies_to":"<scope>"}'
```

---

## Step 5: Output Build Scorecard

Print the following report:

```
════════════════════════════════════════
BUILD EVALUATION
════════════════════════════════════════
Project: <name>
Task: <description>
Preset: <preset>

Metrics:
  Files changed:    <n>
  Tests:            <passed>/<total> passed
  Fix attempts:     <n>
  Blocked tasks:    <n>
  Review findings:  <n>

Observations:
  [pattern]  <summary>
  [mistake]  <summary>
  [tool]     <summary>
  ...

Score: <qualitative assessment>
════════════════════════════════════════
```

### Qualitative Score

Based on the metrics, assign one of:

- **Clean** — 0 fix attempts, 0 blocked tasks, 0 review findings, all tests pass
- **Smooth** — <=1 fix attempt, 0 blocked tasks, <=1 review finding
- **Rough** — 2+ fix attempts OR 1+ blocked tasks OR 2+ review findings
- **Struggled** — 3+ fix attempts AND blocked tasks AND review findings

---

## Error Handling

- If `chad-memory` is not installed: output the scorecard anyway, but warn that results were NOT persisted. Suggest `npm link` in the chad-memory project.
- If git is not available or not a repo: skip git metrics, note them as "N/A".
- If no test runner found: set `tests_total` and `tests_passed` to 0, note "no test runner detected".
- If TaskList returns empty: set `fix_attempts` and `blocked_tasks` to 0, note "no task history available".
- Never fail entirely. Always output whatever scorecard data was gathered.
