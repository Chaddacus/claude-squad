# Chad's Engineering Philosophy

These augment the universal coding standards with Chad's specific engineering principles.

## Core Principles

**Data in, data out.** Every layer should be observable — what came in, what went out, what transformed in between. No black boxes. If you can't see the data flowing through the system, you can't debug it.

**API-first, specification-driven.** Define the contract before writing the implementation. Separated frontend and backend with clear API contracts. The spec is the source of truth.

**Elegant simplicity.** "What's the most straightforward way to tackle this problem?" If a junior dev can't read it and understand what it does, it's too clever.

**Everything is testable.** Not smoke-test testable — real integration and e2e testable. Backend AND frontend. Playwright/Browserbase for the UI. Every code path, every combination.

**Security by default, not by obstruction.** Auth on API endpoints. Prevent injection. Validate inputs. But don't over-secure during development to the point where iteration becomes painful. Security should be a guardrail, not a wall.

**Cut the delta.** The biggest productivity lever is reducing the time between attempts. Shrink the feedback loop above all else.

## Debugging Rules

1. **Follow the data.** Trace the full path — request in, transformations, response out. The bug is where the data diverges from what you expect.
2. **Two strikes and pivot.** If you try the same fix twice and it fails, stop. Try a completely different approach.
3. **Pinpoint, then unravel.** Find the exact layer/function/line where things go wrong. Then work outward. Don't shotgun-debug across the whole stack.
4. **Make it reproducible first.** Before you fix anything, get a reliable reproduction.

## Review Priorities

1. **Does it break anything?** Tests pass, types compile, no regressions.
2. **Can I trace the data?** Is the flow observable? Can I debug this at 2am?
3. **Is it over-engineered?** Did they build a spaceship when a bicycle would do?
4. **Security basics.** Auth, validation, parameterized queries. The OWASP stuff.
5. **Is it testable?** Not "could someone theoretically test this" — is there actually a test?

## Anti-Patterns

- Over-engineering. Abstractions nobody asked for.
- Happy-path-only code. What happens when the API returns 500? When the user sends garbage?
- Trying the same broken fix three times.
- "It works on my machine" without a reproducible test.
- Black-box modules where you can't see what's happening inside.
- Premature optimization before you've even measured.

## Communication

Constructive and direct. When something's wrong, say what's wrong AND how to fix it. No vague "this could be improved" — show the improvement. Respect the person, critique the code.
