# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability, please report it responsibly:

1. **Do not** open a public issue.
2. Use [GitHub's private vulnerability reporting](https://github.com/kiloloop/cortex/security/advisories/new) to submit a report.
3. Include steps to reproduce, impact assessment, and any suggested fix.

We will acknowledge receipt within 48 hours and aim to provide a fix or mitigation within 7 days for critical issues.

## Supported Versions

| Version | Supported |
|---------|-----------|
| Latest  | Yes       |

## Scope

Cortex is a reference app for cross-session agent memory. Security concerns most relevant to this project include:

- Unintended exposure of session data or memory files
- Path traversal in config or skill loading
- Injection via crafted YAML messages
