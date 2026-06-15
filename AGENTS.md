# Repository Guidelines

## Project Structure & Module Organization

This repository currently contains workflow requirements, domain context, ADRs, and reference material. When adding code, keep the layout predictable:

- `src/` for application or library source code.
- `tests/` for automated tests, mirroring `src/` where practical.
- `docs/` for design notes, ADRs, and contributor-facing references.
- `assets/` for static files such as images, fixtures, or sample inputs.
- `scripts/` for repeatable development and maintenance commands.
- `reference_doc/` for insight articles and summaries produced while researching comparable projects.
- `reference_project_when_insight/` for industry or open-source projects downloaded during research and analysis.
- `reference_project_when_coding/` for coding-time reference projects, especially when opencode SDK usage is uncertain.

Prefer small modules. Keep shared helpers close to the code that uses them until reuse is clear.

## Build, Test, and Development Commands

No build system is configured yet. When adding one, document the main commands in `README.md`. Common examples:

- `npm run dev` starts a local development server.
- `npm test` runs the test suite.
- `npm run lint` checks style and common defects.
- `make build` or `npm run build` produces distributable output.

Add scripts to the project manifest or `Makefile` instead of one-off shell commands.

## Coding Style & Naming Conventions

Use consistent formatting for the language or framework introduced. Prefer automated formatters such as Prettier, Black, `gofmt`, or `rustfmt` as appropriate. Use descriptive names, keep files focused, and avoid abbreviations that are not already common in the project domain.

Recommended defaults: two-space indentation for JavaScript/TypeScript and Markdown, four spaces for Python, `kebab-case` for CLI scripts and docs, and language-standard casing for source symbols.

## Testing Guidelines

Place tests under `tests/` or beside source files using the framework’s common convention. Name tests after behavior, not implementation details, for example `user-session.test.ts` or `test_user_session.py`. New behavior should include tests for the expected path and at least one failure or edge case.

Document any coverage target once a test runner is chosen. Until then, prioritize meaningful regression coverage over a numeric threshold.

## Commit & Pull Request Guidelines

This workspace has no Git history yet, so use a simple conventional style for future commits:

- `feat: add workflow parser`
- `fix: handle empty input`
- `docs: add setup guide`

Pull requests should include a concise summary, linked issue or motivation, test results, and screenshots or logs for user-visible changes. Keep PRs focused and call out any follow-up work explicitly.

## Security & Configuration Tips

Do not commit secrets, local credentials, generated private keys, or machine-specific configuration. Use ignored `.env` files for local settings and commit an `.env.example` when configuration is required.

## Requirement Analysis Rules

Use the `grill-with-docs` skill for requirement analysis when it is available. Use `rawrequest.md`, `CONTEXT.md`, `docs/adr/`, and `docs/workflow-domain-alignment.md` as the source of truth for requirement discussions. Clarify domain terms before proposing implementation details. When a term is resolved, update `CONTEXT.md`; when a hard-to-reverse architectural decision is made, add a short ADR under `docs/adr/`.

Ask one focused question at a time during requirement analysis. For each question, provide a recommended answer and the reasoning behind it. Do not keep expanding the design tree when the user asks to pause, summarize, or prepare alignment material.

Do not simply agree with the user. Compare each statement against the existing glossary, ADRs, opencode reference project, and current repository constraints. If a proposal conflicts with prior decisions, has unclear ownership, or creates avoidable complexity, call that out and recommend a better boundary.

Keep the current domain boundaries intact unless explicitly revisited: Workflow Runtime is embedded in opencode; Workflow Definitions describe structure and runtime policy; Workflow Skills own Step-level business execution; Workflow Targets describe what a Run acts on; Target Resolver provides advisory candidate targets, not final impact scope or authorization.

## Reference Material Rules

Keep research and coding references separate. When performing industry insight work, download comparable projects into `reference_project_when_insight/`, inspect them carefully, and write the resulting analysis into `reference_doc/`. When implementing code and needing concrete opencode SDK examples, consult `reference_project_when_coding/` instead. Do not mix exploratory research projects with coding-time reference projects.
