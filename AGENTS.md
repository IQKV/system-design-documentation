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

## Minimal Change Discipline

Solve the whole problem with the least new code. The best code is the code never written.

### Scope before you write

Read the task and the code it touches. List every place the change must reach: callers, tests, fixtures, config, exports. Check what it could break for users: data it would destroy or expose, callers that stop working. That is scope. Extra features are not.

### Pick the smallest complete change

Take the first option that fully works:

1. Does it need to exist? Drop features, options and flexibility nobody asked for — name them in one line so the user can decide.
2. Already in this codebase (helper, component, service, pattern)? Use it the way the surrounding code does.
3. Standard library or platform feature? Use it, unless the project has its own.
4. An installed dependency? Use it. Never add a dependency for a few lines.
5. Can it be one line a reader gets at a glance? One line.
6. Otherwise: the minimum code that works.

Be lazy about the solution, never about the change itself:

- Finish every part the task needs, including the callers, tests and fixtures the change breaks.
- No abstraction, wrapper, option, config, boilerplate or "for later" code nobody asked for. Keep values in the form the platform already gives you. Deletion beats addition. Keep the layers, interfaces and conventions the codebase already has.
- The shortest working diff wins, once you know everything it must touch. A one-liner that needs decoding is not short.
- Comment only the why the code cannot show, in one line.
- Bug fix: grep every caller of the function you touch, then fix the root cause once in the shared code.
- Code you move or merge keeps its error handling and validation.
- Between options of equal size, take the one that is correct on edge cases.
- New non-trivial logic (a branch, a loop, a parser, money or security, or a whole new script) leaves one small test or an assert-based self-check. Trivial changes need none.
- A shortcut with a known limit gets a one-line comment: `shortcut: <the limit>, <when to upgrade>`.
- Push back when the need does not justify a part: state the smaller option in one line instead of building the bigger one silently.

Never cut: validation at trust boundaries, error handling that prevents data loss, security, accessibility, the calibration real hardware needs, anything the user explicitly asked for.

### Token discipline

- Locate with Grep/Glob, then read only the ranges you need — not whole trees or whole files.
- Prefer editing existing files to creating new ones; reuse code that is already there before writing.
- Never paste file contents back in a reply. Cite the path and lines.
- Keep replies to the outcome: what changed, what passed, any risk (see AI Communication Standards).

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

## Evaluation

How we judge whether an agent task in this repository is actually done. Evaluation is evidence-based — a claim without a result is not a pass.

> This section starts minimal and grows with the repository. Add a gate only when a change type repeatedly slips through without it.

### Definition of done

A task is evaluated as done only when all of the following hold:

- Every gate required by the change type below is satisfied.
- The final report names what was done and what was checked — per AI Agent Guidelines.
- Nothing required was skipped silently. Something that could not be checked is reported as a gap, never as a pass.

### Required gates by change type

| Change type | Required gates |
| --- | --- |
| Any change | No build or test gate — formatting is enforced by the pre-commit hook |
| New document | Placed under `docs/` and linked from the relevant index; content is not duplicated from another document |
| Cross-reference change | Both ends updated in the same change — fix the drift at the source |
| Wording-only edit | No gate |

### Ordering

Check the narrowest relevant item first, then the rest for the change type. On a problem, stop and report it rather than continuing on an unverified base (see Execution Discipline).

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
