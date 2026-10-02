# AI Agent Development Guide

## Project Overview

**system-design-documentation** — System design and architecture documentation repository. Contains Markdown docs covering platform architecture, decisions, and technical specifications.

**Key characteristics:**

- Documentation only — no application code
- Node.js tooling (pnpm + Husky) for commit hooks and formatting
- All docs live under `docs/`

## Execution Discipline

- Read the relevant document before editing it.
- Fix at the source — do not duplicate content across documents; link instead.
- No speculative additions — add only what is directly requested.
- After two identical failures, change approach.

## Security

- Keep credentials, internal URLs, and unreleased product details out of commits.
- Do not include personal contact details or private infrastructure specifics in docs.
- Never bypass `--no-verify` unless explicitly requested.

## AI Agent Guidelines

- **Ask before applying**: describe the change, wait for approval.
- **Approval phrases**: "Yes", "Proceed", "Apply", "Do it", "Looks good"
- **Never create** auto-generated summary or review files.
- Keep documents factual, concise, and technically accurate.
- Do not duplicate content — link between docs instead.

## Commit Standards

Format: `type(scope): subject`

- Subject: imperative, lowercase, no trailing period, ≤ 72 chars
- Types: `docs`, `feat`, `fix`, `chore`, `ci`, `revert`
- Scope: affected document or section (e.g., `architecture`, `decisions`, `api`, `deployment`)
- For `fix`: symptom + trigger, not the line changed
  - ✅ `fix(architecture): diagram shows outdated service boundaries`
  - ❌ `fix(architecture): update diagram`

Examples:
- `docs(architecture): add event sourcing decision record`
- `feat(api): document new webhook payload format`
- `chore(deps): update formatting tools`
