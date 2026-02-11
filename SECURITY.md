# Security Policy

## Scope

This repository contains Claude Code configuration files (markdown, JSON). There is no executable application code, server, or API surface. Security concerns are limited to:

- Leaked secrets or credentials in committed files
- Configuration that could cause unsafe behavior when loaded by Claude Code
- Prompt injection vectors in skill or agent definitions

## Reporting a Vulnerability

If you discover a security issue, please report it privately:

1. **Do not open a public issue.**
2. Use [GitHub Security Advisories](https://github.com/Chaddacus/claude-squad/security/advisories/new) to report privately.
3. Include: what you found, which file(s) are affected, and the potential impact.

You should receive an acknowledgment within 72 hours.

## What This Repo Should Never Contain

- API keys, tokens, or secrets of any kind
- `settings.json` (excluded via `.gitignore` — contains permissions and env vars)
- Database files (`chad.db`) or auto-generated rule files (`chad-memory.md`)
- Personally identifiable information beyond the public GitHub username

If you find any of the above committed, please report it immediately.
