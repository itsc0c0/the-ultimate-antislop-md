# Philosophy & How to Use This Document

## What "Slop" Means Here

"Slop" is not a synonym for "AI-generated." Plenty of AI-generated code, writing, and design is careful, intentional, and excellent — and plenty of human-generated code, writing, and design is generic, careless, and low-effort. "Slop" describes a *quality failure mode*, not an *origin*. It is what you get when output is produced to satisfy the shape of a request rather than the substance of it: code that compiles but wasn't reasoned about, prose that reads fluently but says nothing, a UI that looks "designed" but wasn't actually decided.

Three properties tend to co-occur in slop, in code, writing, or design alike:

- **Genericness.** The output converges on the statistical average of everything similar the author (human or model) has seen, rather than being shaped by the specific problem in front of them. A landing page that looks like every other AI-generated landing page. A code review comment that could be pasted onto any PR. A function name so generic it tells you nothing (`handleData`, `processItem`, `doStuff`).
- **Unverified confidence.** The output is presented as correct, complete, or tested without the work that would justify that presentation having actually been done. A claim that "this fixes the bug" without having reproduced the bug. A cited API that was never checked against real documentation. A sentence structured as a confident conclusion built on a guess.
- **Padding over substance.** Effort goes into making the output *look* thorough (length, hedging, boilerplate, decoration) rather than making it *be* thorough. Comments that narrate the obvious instead of explaining the non-obvious. A five-paragraph answer to a yes/no question. A UI with five feature cards because five looks complete, not because there are five features.

Slop is expensive precisely because it is fluent. A sloppy first draft that visibly struggles gets caught and fixed. Sloppy output that reads well, compiles, and looks finished gets shipped — and the cost shows up later, in bugs that "look fine," designs nobody can tell apart from a competitor's, and technical debt nobody flagged because nothing about the code waved a flag.

## Why This Document Exists

This document exists to make the difference between "shaped like the right answer" and "actually the right answer" checkable, in as many concrete situations as possible — across programming languages, frameworks, infrastructure, writing, visual design, and the behavior of AI coding agents themselves. It is written to be useful to two overlapping audiences at once:

- **Humans** who want a single, comprehensive reference for what disciplined, high-craft work looks like across a wide range of technical and creative domains, and who want concrete language for critiquing work (their own or someone else's) that technically works but isn't actually good.
- **AI coding agents** who can be pointed at all or part of this document (directly, or via a project's `AGENTS.md`/`CLAUDE.md`/`.cursorrules`) as an explicit set of constraints, to counteract the tendency of a model to default to the generic, statistically-average pattern unless told not to.

## How the Document Is Organized

Every content section follows the same shape:

- **DO** rules state a positive practice, with a short rationale for why it matters — not just "do X" but "do X, because Y."
- **DON'T** rules state an anti-pattern to avoid, with a short rationale and, where it isn't obvious, what to do instead.
- Rules are grouped into topic subsections within each part, and most parts close with a condensed **Quick Checklist** — a scannable, terser restatement of the section's highest-value rules, meant for a fast pre-ship pass rather than a first read.

The document moves from general to specific: universal principles that apply everywhere, then the patterns specific to AI-generated output (the actual "anti-slop" core of the project), then language-by-language and domain-by-domain detail, then design, then the conventions for the instruction files and skills that steer AI agents themselves.

## What This Document Is Not

- **It is not a single house style.** Rules here express widely-shared engineering and design judgment, not one team's arbitrary preferences (indentation width, brace placement, and similar bikeshed-prone choices are deliberately left alone). Where a project's existing conventions genuinely conflict with a rule in here, matching the existing codebase usually wins — consistency within a project is itself one of this document's rules (see Part 2).
- **It is not a substitute for judgment.** Every rule here has edge cases where the opposite is correct. A guideline that "avoid premature abstraction" doesn't mean "never abstract"; a guideline that "avoid excessive comments" doesn't mean "never comment." Treat every rule as a strong default that a specific, articulable reason can override — not as a rule to follow past the point it stops making sense.
- **It is not a checklist to satisfy mechanically.** The fastest way to produce a new kind of slop is to follow this document's letter while missing its point — padding a PR description with checkbox-shaped language, adding a comment to every line to "follow the commenting rule," inserting a token accessibility attribute without checking it actually works with a screen reader. The rules describe outcomes worth wanting, not incantations.
- **It does not replace testing, review, or running the code.** Nothing in this document substitutes for actually executing a program, actually reading a rendered page, actually having another person read a piece of writing. Several sections say this explicitly, because it's the single most common shortcut that produces confidently-wrong output.

## How to Use This With an AI Coding Agent

The most direct way to use this document with a coding agent is to point the agent's instruction file at it — for example, a line in `AGENTS.md` or `CLAUDE.md` such as "Follow the rules in `ANTISLOP.md`, especially Part 3 (AI-Generated Code Slop) and whichever language/framework parts apply to this repository." Part 19 of this document covers how to write and maintain agent instruction files well, including why pointing at a large reference document works better as a *targeted* pointer ("read Part 5 before touching Python code") than as an instruction to load the entire file into context on every turn.

A few practical notes for that use case:

- This document is deliberately large and organized into parts precisely so that only the relevant parts need to be in context for a given task — a request touching only Python and SQL doesn't need the Swift or Kubernetes sections loaded.
- The rules are written as constraints an agent can check its own output against, not as narrative to summarize back to a user. Treat a DON'T rule the way you'd treat a linter error: something to notice and fix before presenting work as finished, not something to mention having read.
- Where this document and a project's own conventions disagree, the project's conventions win, per the consistency principle in Part 2 — this document fills gaps and catches generic defaults, it doesn't override an explicit, deliberate project decision.

## How to Use This as a Human Reviewer

Used as a review checklist, this document works best applied narrowly rather than broadly: pick the two or three sections most relevant to what's actually being reviewed (a PR's language and domain, a draft's genre, a UI's component types) rather than attempting to run every part against every piece of work. The Quick Checklists at the end of each part exist for exactly this — a fast pass to catch the highest-value issues before a deeper review.

## A Living Document

Languages, frameworks, and the specific tells of AI-generated genericness all change over time — what reads as an obvious "AI slop" pattern today (a specific shade of purple gradient, a specific font pairing) may fade as models and defaults shift, and new patterns will emerge to replace it. Treat the specific tells named throughout this document (especially in Part 17) as illustrative of a *category* of failure — reflexive, unexamined defaulting to whatever is statistically common — rather than as a permanently fixed list. The underlying discipline this document argues for (be specific, verify before claiming, make deliberate choices instead of defaulting) outlasts any individual example of what "generic" currently looks like.
