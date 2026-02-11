# Universal Coding Standards

These standards apply unless the project-level CLAUDE.md or .claude/rules/ specifies different standards. Project rules take precedence.

## Clean Code

- Single responsibility: one function does one thing.
- Functions stay under 30 lines. If longer, extract.
- Guard clauses over nested conditionals. Return early.
- Names describe intent: `getUserById` not `getData`, `isValid` not `check`.
- No magic numbers or strings. Use named constants.
- Delete dead code. Don't comment it out.

## Separation of Concerns

- Route / controller layer: request parsing, response formatting, auth checks.
- Service / business logic layer: orchestration, rules, transformations.
- Data / repository layer: database queries, external API calls, caching.
- Don't leak layers. Services don't know about HTTP status codes. Routes don't write SQL.

## API-First Design

- Define contracts (types, schemas, endpoints) before implementation.
- Schema validation at system boundaries.
- If the project uses a specific validation library (Zod, Joi, DRF serializers), use that. Otherwise default to Zod for TypeScript, Pydantic for Python.
- Consistent error response format across endpoints.
- Version APIs when breaking changes are unavoidable.

## Modularity

- Open/closed principle: extend behavior without modifying existing code.
- Strategy pattern over switch statements when behavior varies by type.
- Options objects over long parameter lists (>3 params).
- Dependency injection for testability. No hard-coded dependencies on external services.

## Testing

- If the project has a test framework configured, follow TDD: write failing test → implement → refactor.
- If the project has no test framework, write implementation and note that tests are needed.
- AAA pattern: Arrange, Act, Assert. One assertion concept per test.
- Co-locate tests with source files or in a parallel `__tests__`/`tests` directory matching project convention.
- Quality gates: no merging with failing tests. Fix or explicitly mark as known-failing with a reason.

## Security

- No secrets in code. Use environment variables or secret managers.
- Input validation at system boundaries. Don't trust external input.
- Parameterized queries. No string concatenation for SQL/NoSQL.
- Follow OWASP top 10. Pay attention to injection, XSS, CSRF, broken auth.
- Principle of least privilege for file access, API permissions, database roles.
