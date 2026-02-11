---
name: analyze
description: Consolidate build observations across sessions, detect patterns, promote rules, and regenerate the chad-memory.md rules file.
argument-hint: "[--last N | --project X | --since YYYY-MM-DD]"
disable-model-invocation: true
allowed-tools: Read, Bash, Grep, Glob
---

# /analyze - Cross-Session Pattern Analysis & Rule Promotion

Aggregate observations across builds, detect recurring patterns, promote proven observations to active rules, archive stale rules, and regenerate the memory file.

## Usage

```
/analyze
/analyze --last 5
/analyze --project my-api
/analyze --since 2026-01-01
/analyze --last 10 --project my-api
```

---

## Step 1: Parse Arguments

From `$ARGUMENTS`, extract filters:

| Flag | Effect |
|------|--------|
| `--last N` | Analyze the last N builds only |
| `--project X` | Filter to builds for project X |
| `--since YYYY-MM-DD` | Filter to builds on or after this date |
| (none) | Analyze all builds in the database |

Flags can be combined. If `$ARGUMENTS` doesn't match any flag pattern, treat the entire argument as a project name filter.

---

## Step 2: Query the Database

```bash
# Get build data (with optional filters)
chad-memory query --project <X> --since <Y> --last <N>

# Get aggregate stats
chad-memory stats
```

If no filters apply, run without flags to get everything:

```bash
chad-memory query
chad-memory stats
```

Parse the output to extract:
- List of builds with their metrics
- List of observations with their categories, summaries, frequencies, and statuses

---

## Step 3: Aggregate Metrics

Compute these aggregates across the filtered builds:

| Metric | Calculation |
|--------|-------------|
| **Avg fix attempts per task** | sum(fix_attempts) / count(builds) |
| **Block rate** | count(builds with blocked_tasks > 0) / count(builds) * 100 |
| **Avg review findings per build** | sum(review_findings) / count(builds) |
| **Test pass rate** | sum(tests_passed) / sum(tests_total) * 100 |

### Trend Detection

If there are enough builds (>= 4), split into two halves (older vs recent) and compare each metric:

- If recent is better by >= 10%: trend is improving (arrow down for rates/counts, arrow up for pass rates)
- If recent is worse by >= 10%: trend is degrading
- Otherwise: stable

Use these symbols for display:
- Improving: (down arrow for bad metrics, up arrow for good metrics)
- Degrading: (up arrow for bad metrics, down arrow for good metrics)
- Stable: (right arrow)

---

## Step 4: Pattern Detection

### Find Recurring Observations

```bash
# Get all observations
chad-memory query
```

Group observations by similarity:
1. Exact summary match — same observation seen multiple times
2. Near-match — summaries that describe the same concept with different wording (e.g., "use guard clauses instead of nested ifs" and "prefer early returns over deep nesting")

For exact duplicates, `chad-memory log-observation` handles deduplication automatically (it bumps frequency). For near-matches, log a new observation with the clearer wording and note the frequency.

### Status Progression

Observations progress through statuses:
- `raw` — newly recorded, seen once
- `candidate` — seen 2+ times, eligible for promotion
- `active` — promoted to a rule, included in generated memory file
- `archived` — rule retired after not being seen in 10+ builds

---

## Step 5: Promotion Gate

For each observation with frequency >= 2 (candidates), evaluate ALL of these criteria:

| Gate | Question | Pass Condition |
|------|----------|----------------|
| **Frequency** | Seen enough times? | frequency >= 2 |
| **Actionable** | Does it specify a concrete action? | Contains a verb — "do X", "use Y", "avoid Z", "prefer A over B" |
| **Measurable** | Can compliance be verified? | Can check via code review, linting, or test output |
| **Not redundant** | Is there already an active rule covering this? | No existing active rule with same or overlapping scope |
| **Specific** | Does it apply to a concrete phase/step/tool? | Not vague like "write better code" — specifies what, where, when |

If ALL gates pass, promote the observation:

```bash
chad-memory promote <observation-id>
```

If any gate fails, leave the observation as `candidate` and note which gate(s) failed.

---

## Step 6: Archive Check

Review all currently active rules:

```bash
chad-memory query
```

For each active rule:
1. Check if any observation with a matching or related summary appeared in the last 10 builds
2. If the rule has NOT been referenced in 10+ consecutive builds, archive it:

```bash
chad-memory archive <rule-id>
```

This keeps the active rule set fresh and relevant. Archived rules are not deleted — they can be re-promoted if they reappear.

---

## Step 7: Regenerate Memory File

After all promotions and archives:

```bash
chad-memory generate
```

This writes the updated `~/.claude/rules/chad-memory.md` file with all active rules formatted for Claude Code consumption.

Verify the file was written:

```bash
cat ~/.claude/rules/chad-memory.md | head -20
```

---

## Step 8: Output Analysis Report

```
════════════════════════════════════════
ANALYSIS REPORT
════════════════════════════════════════
Builds analyzed: <n>
Period: <earliest build date> - <latest build date>

Trends:
  Fix attempts:     <n>/task (<direction> vs previous period)
  Block rate:       <n>% (<direction>)
  Review findings:  <n>/build (<direction>)
  Test pass rate:   <n>% (<direction>)

New Rules Promoted:
  [<category>] <rule text> (seen <n>x, from: <project list>)
  ...

Rules Archived:
  [<category>] <rule text> (not seen in <n> builds)
  ...

Active Rules: <current count>/15

Recommendations:
  - <process improvement suggestions based on trends>
  - <if block rate is high: suggest better upfront planning>
  - <if fix attempts are high: suggest more TDD discipline>
  - <if review findings are high: suggest pre-review self-checks>
  - <if test pass rate is declining: suggest investigating test reliability>
════════════════════════════════════════
```

### Recommendations Logic

Generate 1-3 actionable recommendations based on the data:

- **Block rate > 20%**: "Consider more detailed upfront planning in SPEC phase — high block rate suggests tasks are under-specified"
- **Avg fix attempts > 2**: "Increase TDD discipline — write failing tests before implementation to catch issues earlier"
- **Review findings > 3/build**: "Add a self-review step before VALIDATE — implementers should run Chad's 5-point checklist on their own work"
- **Test pass rate < 95%**: "Investigate test reliability — flaky or failing tests erode confidence and slow builds"
- **Active rules near 15**: "Approaching rule cap — review active rules for consolidation opportunities"
- **No observations in recent builds**: "Consider running /evaluate after each build to capture learnings"

---

## Error Handling

- If `chad-memory` is not installed: fail with message "chad-memory CLI not found. Install with `npm link` in the chad-memory project directory."
- If database is empty: output "No builds recorded yet. Run `/evaluate` after a build to start collecting data."
- If query returns no results for the given filters: output "No builds match the given filters." and suggest broadening the search.
- If `generate` fails: warn but still output the analysis report.
