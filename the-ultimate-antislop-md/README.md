# The Ultimate Anti-Slop Reference

A comprehensive DO/DON'T rulebook for avoiding generic, low-effort, hallucinated, and low-quality output — in code, in writing, and in design. Written for both human developers/designers and AI coding agents.

This repository ships the reference two ways:

- **`ANTISLOP.md`** — the complete document in one file (front matter, table of contents, and all 22 parts), for reading top to bottom or pointing an AI agent's instruction file at the whole thing.
- **The folders below** — the same content split into focused, single-topic files, for when you only need one language, one domain, or one category (e.g. only `02-languages/python.md`, or only the design-related files).

Every rule is either:

- **DO:** a positive practice, with the reasoning behind it.
- **DON'T:** an anti-pattern to avoid, with the reasoning behind it and, where it isn't obvious, what to do instead.

Most files close with a **Quick Checklist** — a condensed, scannable restatement of that file's highest-value rules for a fast pass before shipping.

Start with [`00-start-here/philosophy.md`](00-start-here/philosophy.md) — it explains what "slop" means in this document, how to use the reference (as a human reviewer or by pointing an AI coding agent at it), and what the document deliberately is *not*.

## Folder Structure

### [`00-start-here/`](00-start-here/)
- [`philosophy.md`](00-start-here/philosophy.md) — What "slop" means, how this document is organized, how to use it with an AI coding agent or as a human reviewer.

### [`01-core-principles/`](01-core-principles/) — language-agnostic foundations
- [`universal-code-quality.md`](01-core-principles/universal-code-quality.md) — Naming, functions, comments, error handling, abstraction, state, dependencies, code review, refactoring, consistency, technical debt.
- [`ai-generated-code-slop.md`](01-core-principles/ai-generated-code-slop.md) — The core of the project: hallucination, unverified completion claims, placeholder/mock leakage, comment noise, over-engineering, sycophancy, scope creep, destructive shortcuts, silently weakened safeguards, dependency bloat, ignored conventions, symptom-chasing, partial implementation.

### [`02-languages/`](02-languages/) — one file per language
`javascript-typescript-nodejs.md` · `python.md` · `go.md` · `rust.md` · `c.md` · `cpp.md` · `java.md` · `kotlin.md` · `csharp-dotnet.md` · `swift.md` · `objective-c.md` · `ruby.md` · `php.md` · `bash-shell.md` · `sql.md` · `html-css.md`

### [`03-frameworks-and-domains/`](03-frameworks-and-domains/) — one file per domain
- `frontend-frameworks.md` — React, Vue, Angular, Svelte, state management, component architecture, accessibility.
- `backend-api-design.md` — REST/GraphQL/gRPC design, backend frameworks, microservices, API documentation.
- `mobile-architecture.md` — iOS, Android, React Native, Flutter.
- `devops-git.md` — Git, Docker, Kubernetes, CI/CD, Infrastructure as Code.
- `databases-data-engineering.md` — Schema design, query performance, migrations, NoSQL, transactions, caching, ETL.
- `testing-qa.md` — Test philosophy, unit/integration/e2e testing, mocking, TDD, coverage.
- `security-appsec.md` — Injection, auth, secrets, dependency security, cryptography, OWASP-aligned practices.

### [`04-writing/`](04-writing/)
- [`ai-generated-writing-slop.md`](04-writing/ai-generated-writing-slop.md) — Generic openings, overused vocabulary ("delve," "leverage," "not just X but Y"), structural tells, hedging, filler transitions, empty conclusions, formatting tics, tone problems.

### [`05-design/`](05-design/)
- [`ai-generated-design-slop.md`](05-design/ai-generated-design-slop.md) — The specific visual/UX tells that mark an interface as AI-generated (the "AI purple" gradient, glassmorphism-by-default, centered-hero-with-badge templates, icon-topped feature-card grids, and more) and how to actually fix them.
- [`general-design-principles.md`](05-design/general-design-principles.md) — Foundational, non-AI-specific design: typography, color, layout, components, motion, accessibility, content design, iconography, design systems, information architecture.

### [`06-ai-agents-and-skills/`](06-ai-agents-and-skills/)
- [`agent-skill-conventions.md`](06-ai-agents-and-skills/agent-skill-conventions.md) — Writing good `AGENTS.md`/`CLAUDE.md`/cursor-rules files, `SKILL.md` design, tool design for agents, agentic coding workflow anti-patterns, writing good instructions for agents.

### [`07-reference/`](07-reference/)
- [`standards-digest.md`](07-reference/standards-digest.md) — A map to the formal, publicly maintained standards and style guides this document draws on (PEP 8, WCAG, OWASP Top 10, Conventional Commits, and more).
- [`master-checklist.md`](07-reference/master-checklist.md) — Every file's Quick Checklist collected in one place, in document order.
- [`sources-further-reading.md`](07-reference/sources-further-reading.md) — Further reading and attribution.

## Using This With an AI Coding Agent

Point your project's `AGENTS.md` / `CLAUDE.md` / `.cursorrules` at whichever files apply to the current work, rather than the whole reference — for example:

```
Follow the rules in the-ultimate-antislop-md/01-core-principles/ai-generated-code-slop.md
and the-ultimate-antislop-md/02-languages/python.md for all Python work in this repo.
```

`00-start-here/philosophy.md` covers this in more detail, including why a targeted pointer works better than loading the entire reference into context on every turn.

## License / Attribution

This is an original synthesis of widely-shared, publicly-taught engineering and design knowledge — it does not reproduce text from any single source. See `07-reference/sources-further-reading.md` for the further-reading list and a note on attribution.
