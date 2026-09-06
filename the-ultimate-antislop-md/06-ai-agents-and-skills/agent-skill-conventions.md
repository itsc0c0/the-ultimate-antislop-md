# AI Agent & Skill Conventions

## Repo-Level Agent Instruction Files (CLAUDE.md / AGENTS.md / Cursor Rules)

### Purpose and Mental Model

- **DO:** Treat the instruction file as a machine-readable briefing an agent reads at the start of every session, not as a wiki page a new human hire skims once. An agent has no persistent memory of the last session and no tenure at the company, so anything it needs to act correctly this run has to be re-derivable from the file (or the code) every single time.

- **DO:** Write the file to answer the question "what would a competent engineer need to know in the first sixty seconds to avoid breaking this repo?" rather than "what would a new hire want to understand about this system over their first month." Those are different documents with different content, and conflating them is the single most common failure mode in instruction files.

- **DO:** Assume the file will be read in full on every invocation and re-read partway through long sessions when the agent needs to recheck a rule, so every line has an ongoing token cost. Content that is read hundreds of times across a team's usage should earn its place by preventing a mistake, not by being merely informative.

- **DON'T:** Model the instruction file on a README or an onboarding doc. A README sells the project and explains what it is for humans discovering it; an agent instruction file gives operational constraints to something that is about to start editing files and running commands. Put the "what is this project" narrative in the actual README and link to it once, briefly, rather than reproducing it.

- **DON'T:** Assume the agent has read any other document unless the instruction file explicitly tells it to. If a contributing guide, architecture doc, or style guide exists and matters, name it and its path so the agent can decide whether to open it — don't assume institutional context transfers automatically.

### What to Include: Exact Stack and Framework Versions

- **DO:** Pin the exact language runtime version, package manager, and major framework versions the project actually uses, including patch-relevant details when a known behavior differs across minor versions. Agents default to whatever is statistically most common in their training data when a version isn't specified, which is frequently stale, frequently wrong for a codebase that adopted something recent, and a common source of code that "looks right" but uses removed or renamed APIs.

- **DO:** State versions as facts pulled from the actual lockfile or manifest, not as aspirations or ranges copied from a `package.json` caret range. A rule like "Node 20, pinned via `.nvmrc`; do not use `??=` patterns assuming Node 22 semantics" is more useful than "we use a modern version of Node."

- **DO:** Call out any deliberately old or unusual version pin and why it exists, so the agent doesn't try to "helpfully" upgrade it. If a project is stuck on React 17 because of a third-party dependency, say so — otherwise an agent asked to add a feature may reach for `useId` or other post-17 APIs and produce code that fails to build.

- **DON'T:** List versions the agent can already discover with one command (e.g., restating the exact npm version, or the OS the CI machine happens to run) unless a specific incompatibility depends on it. Cluttering the version block with irrelevant precision buries the versions that actually matter.

```markdown
## Stack

- Runtime: Node 20.11 (see `.nvmrc`); do not assume Node 22+ APIs.
- Package manager: pnpm 9 only — never generate a `package-lock.json` or `yarn.lock`.
- Framework: Next.js 14 App Router (not Pages Router — there is no `pages/` directory).
- Database: PostgreSQL 15 via Prisma 5.18. Prisma 6 syntax (e.g. `omit`) will not compile here.
- React 18.2, pinned below 19 because of a third-party charting library; do not use the `use()` hook.
```

### What to Include: Executable Commands Near the Top

- **DO:** Put the exact install, build, test, and lint commands — with the flags actually used in CI — within the first screen of the file. These are the commands an agent needs before it can do anything else, and if they're buried under three sections of prose the agent will either guess wrong or spend a tool call rediscovering `package.json`.

- **DO:** Include the flags that make output usable for an agent, not just the flags a human would type interactively: non-interactive mode, machine-readable/verbose output, a way to run a single test file or a single test by name, and a way to run just the changed files' checks when the full suite is slow.

- **DO:** Distinguish between the fast local loop and the full CI-equivalent check, and say which one to run when. An agent that only knows `npm run test:ci` (which takes fourteen minutes and boots four containers) will either avoid testing altogether during iteration or burn enormous time re-running it after every one-line change.

- **DON'T:** Give a command without its required working directory, environment variables, or setup step if getting it wrong produces a misleading error rather than a helpful one. "Run `pytest`" is incomplete if it actually requires `cd services/api && poetry run pytest` with a `.env.test` file sourced first — say the whole invocation, not the aspirational short form.

- **DON'T:** List commands that don't work, are aspirational, or were true of a previous tooling setup. A stale `make test` line that now 404s against a deleted Makefile actively teaches the agent to expect commands to fail and to work around errors rather than trust the file — this compounds into general distrust of the whole document.

```markdown
## Commands

- Install: `pnpm install --frozen-lockfile`
- Dev server: `pnpm dev` (http://localhost:3000, hot reload)
- Full test suite (slow, ~6 min): `pnpm test`
- Single file: `pnpm test src/lib/pricing.test.ts`
- Single test by name: `pnpm test -t "applies bulk discount"`
- Lint (matches CI exactly): `pnpm lint --max-warnings=0`
- Typecheck: `pnpm typecheck`
- Before opening a PR, CI equivalent locally: `pnpm ci:check` (runs lint + typecheck + test + build)
```

### What to Include: Concrete Code Snippets Over Prose

- **DO:** Show one real, working snippet of house style rather than three paragraphs describing it in the abstract. Agents are language models trained heavily on code; a snippet in the actual project's idiom transfers pattern-matching far more reliably than adjectives like "we prefer functional style" ever will.

- **DO:** Pull the snippet from real code in the repo (or a faithful minimal reduction of it) rather than inventing a toy example that doesn't match how the codebase actually looks. A snippet that doesn't match what `grep` would turn up elsewhere teaches the wrong lesson.

- **DO:** Pair a "do it this way" snippet with a short "not this way" counter-example when the wrong way is a pattern the agent is statistically likely to reach for by default (a very common library idiom that this codebase deliberately avoids). Seeing both boundaries in code is far more precise than a sentence of prose trying to describe the boundary.

- **DON'T:** Write multi-paragraph essays describing an architectural pattern in words when a ten-line snippet would communicate the same constraint more precisely and in fewer tokens. One real snippet beats three paragraphs of description almost every time for this audience.

```markdown
## Error handling style

Always return a typed `Result<T, AppError>`, never throw across module boundaries:

    // Do this:
    async function loadUser(id: string): Promise<Result<User, AppError>> {
      const row = await db.users.findUnique({ where: { id } });
      if (!row) return err({ code: "NOT_FOUND", message: `user ${id}` });
      return ok(toUser(row));
    }

    // Not this — throwing leaks past the API boundary and skips our error logging middleware:
    async function loadUser(id: string): Promise<User> {
      const row = await db.users.findUnique({ where: { id } });
      if (!row) throw new Error("not found");
      return toUser(row);
    }
```

### What to Include: Testing, Mocking, and Determinism Rules

- **DO:** State the project's actual testing philosophy as enforceable rules: which layers get unit tests versus integration tests, whether network calls must be mocked or hit a real test double service, how database state is set up and torn down, and whether snapshot tests are welcome or banned. These rules vary enormously between codebases and an agent left to its own defaults will invent its own philosophy, often incompatible with the existing suite.

- **DO:** Specify how randomness, time, and external IDs are made deterministic in tests (a fixed clock helper, a seeded RNG, a fixture UUID generator) if the project has one, by name. An agent unaware of an existing `freezeTime()` helper will call `Date.now()` directly in a new test and produce a test that is flaky one day a year, or reach for `jest.useFakeTimers()` when the codebase actually uses a different clock abstraction everywhere else.

- **DO:** State the minimum bar for "done" on tests explicitly — for example, whether new code requires new tests at all, whether coverage thresholds are enforced by CI and will hard-fail the build, and whether tests must pass locally before commit or only before merge.

- **DO:** Name any test data conventions (a shared factory/fixture module, a naming convention for test-only fixtures, a policy against committing real user data into fixtures) so the agent uses the existing pattern instead of hand-rolling ad hoc mock objects that drift from the real schema.

- **DON'T:** Leave testing rules implicit and assume the agent will "figure out the pattern" by reading a few existing test files. It often will, correctly — but only if it happens to read representative files first; stating the rule directly removes that dependency on file-read order and covers cases the sampled files didn't happen to show.

```markdown
## Testing rules

- Every new exported function in `src/lib/` needs a unit test in the same directory (`foo.ts` → `foo.test.ts`).
- Never call `Date.now()` or `new Date()` directly in code under test — use `clock.now()` from `src/lib/clock.ts` so tests can call `clock.set(...)`.
- Mock outbound HTTP with the `nock` fixtures in `test/fixtures/http/`; do not add a new mocking library.
- Do not write snapshot tests (`toMatchSnapshot`) — they are banned in this repo, they rot silently.
- `pnpm test` must pass with zero skipped tests before any PR; CI fails on `.only` or `.skip`.
```

### What to Include: A Clear Three-Tier Permission Model

- **DO:** Define explicitly, in one place, three tiers: actions the agent may take autonomously without asking, actions that require asking first and waiting for a yes, and actions that are off-limits entirely regardless of how the request is phrased. Ambiguity here is the single biggest source of both over-cautious agents that ask permission for everything (wasting the human's time) and over-eager agents that do something destructive without being asked.

- **DO:** Make the "always ask first" tier specific to real risks in this project — force-pushing, deleting migrations, rotating secrets, modifying CI/CD config, touching billing code, deploying to production — rather than a generic list copied from elsewhere. The right list depends entirely on what's actually dangerous in this particular codebase.

- **DO:** Make the "never do this" tier absolute and phrased as hard constraints, not preferences — "never commit directly to `main`," "never modify files under `legal/`," "never add a new production dependency without approval" — so the agent treats them as constraints to satisfy rather than defaults to override when convenient.

- **DO:** Explicitly grant the freedoms in the "always OK" tier too, not just the restrictions. An agent that isn't told it may freely run the test suite, create new files, or install dev dependencies will sometimes ask permission for harmless, expected actions, which is its own form of friction.

- **DON'T:** Bury permission rules inside prose paragraphs scattered across the file. A rule an agent needs to check before every risky action should be scannable as a short, dedicated, itemized list — not something it has to remember from a sentence three sections back.

- **DON'T:** Phrase restrictions as soft suggestions ("try to avoid force-pushing when possible") when they are meant as hard rules. Soft language reads as negotiable to a model optimizing for task completion under time pressure; use unambiguous imperatives for anything genuinely non-negotiable.

```markdown
## Permissions

**Always OK, no need to ask:**
- Run tests, lint, typecheck, build.
- Create/edit/delete files inside `src/`, `test/`, `docs/`.
- Install and remove *dev* dependencies.
- Create a local branch and commit to it.

**Ask first, wait for explicit yes:**
- Adding or upgrading a *production* dependency.
- Any change to `.github/workflows/`, `Dockerfile`, or `infra/`.
- Any schema migration under `db/migrations/`.
- Deleting more than 3 files in one change.

**Never do this, no exceptions:**
- Force-push to `main` or `release/*`.
- Commit secrets, API keys, or `.env` contents.
- Bypass pre-commit hooks with `--no-verify`.
- Modify anything under `legal/` or `compliance/`.
```

### What to Include: Calling Out Non-Standard or Uncommon Tooling

- **DO:** Explicitly flag any tool, framework, internal library, or workflow that is unusual enough that the agent's training data is unlikely to have good coverage of it — an internal build system, a homegrown ORM, a company-specific CLI, a fork of a popular library with a different API. The whole point of the instruction file is to compensate for exactly this kind of blind spot; common, well-known tools need little or no explanation, but the further a tool is from mainstream usage, the more explicit the file must be.

- **DO:** Give the non-standard tool's basic command surface directly in the file (the two or three commands actually used day to day) rather than just naming it and expecting the agent to intuit correct usage from a similar-sounding public tool. An internal tool named `depctl` that superficially resembles `npm` but has entirely different flags will otherwise get called with `npm`-shaped flags that don't exist.

- **DO:** Note when a common tool is present but deliberately not the primary one, to prevent the agent defaulting to what it recognizes. If both `pip` and an internal `pkg` wrapper exist but only `pkg` is sanctioned, say that outright — an agent that recognizes `pip` from long experience will otherwise reach for it out of habit.

- **DON'T:** Assume that naming an internal tool once, without examples, is sufficient. If the tool has a non-obvious invocation pattern, show the actual invocation the way the code-snippet rule above recommends — a bare tool name is not enough context for correct usage.

```markdown
## Non-standard tooling

- Dependency management uses our internal `depctl` wrapper around pip, NOT raw `pip install`.
  Add a dependency: `depctl add requests==2.32.0`. Never edit `requirements.txt` by hand — it's generated.
- Database migrations go through `migrate.sh`, a thin wrapper over Alembic that also updates our schema
  registry. Never call `alembic revision` directly — use `./scripts/migrate.sh new "add users.email index"`.
- Internal RPC framework is `grail`, not gRPC — service definitions live in `*.grail` files, not `.proto`.
```

### What to Omit: Don't Restate What's Inferable

- **DO:** Leave out anything the agent can reliably and cheaply discover by reading the code itself — the exact directory layout of a conventional project, the presence of a `package.json` and what it contains, obvious naming conventions visible from a single file listing. Every line in the instruction file competes for attention with every other line; spending that budget on facts one `ls` or one file open would reveal is a net loss.

- **DO:** Prefer pointing at a canonical source over duplicating its content. If the exact list of environment variables lives in `.env.example`, say "see `.env.example` for required environment variables" rather than copying the list into the instruction file, where it will inevitably drift out of sync with the real file.

- **DO:** Reserve the instruction file for facts that are either non-discoverable by reading (intent, history, unwritten conventions, external constraints) or are cheap to state but expensive to rediscover by trial and error (a gotcha that costs a full test run to learn the hard way).

- **DON'T:** Write an architecture essay describing what each directory contains when a one-line description per top-level directory, or nothing at all for genuinely self-explanatory ones, would do. A `src/components/` directory holding React components does not need a paragraph explaining that components live there.

- **DON'T:** Copy-paste content from the README, the package manifest, or other config files "just to be thorough." Duplicated content doubles the maintenance burden and doubles the chance of the two copies silently disagreeing, which is worse than omitting the fact and trusting the agent to look it up.

### File Length: Concise Beats Comprehensive

- **DO:** Keep the file short enough that it is fully read and actually internalized every time, favoring a tight, high-signal document over an exhaustive one. Practically, this tends to mean something in the range of one to a few hundred lines for most projects — long enough to cover real gotchas, short enough that nothing in it gets skimmed past.

- **DO:** Cut ruthlessly during review: for every section, ask whether removing it would actually cause a mistake next session. If the honest answer is "probably not," cut it or compress it into a single line.

- **DO:** Push genuinely deep material — a full architecture writeup, a philosophy document, a detailed migration history — into a separate linked document rather than inlining it, and reference that document by path only where it's actually relevant to a task the agent might do. The main instruction file should read like a table of load-bearing constraints, with deep dives one hop away for when they're needed.

- **DON'T:** Let the file grow into a comprehensive architecture essay under the belief that more context always helps. Long instruction files with deep prose sections on history, philosophy, and design rationale measurably tend to be followed less faithfully than short, dense ones — agents are prone to skimming, deprioritizing, or losing track of rules buried on page four of a document, and a rule that isn't followed provides no more value than a rule that was never written, while still costing tokens every session.

- **DON'T:** Treat file length as a proxy for thoroughness. A 40-line file that states the five rules that actually prevent real mistakes is more valuable — and more likely to be followed — than a 600-line file that states those same five rules once, buried among two hundred lines of general software-engineering advice equally true of any codebase.

### Write for Machine Parsing, Not Human Onboarding

- **DO:** Structure the file with short, scannable headers, itemized lists, and fenced code blocks over long paragraphs, so a model doing retrieval-style attention over the document can locate the relevant rule quickly. Prose paragraphs bury discrete rules inside sentences that have to be parsed for the actionable clause.

- **DO:** Write each rule as a standalone, complete statement that makes sense without needing the surrounding paragraph for context, since the agent may need to recall or re-read just that one line rather than the whole section.

- **DO:** Use consistent, literal terminology throughout — if the project calls something "the ingest pipeline" in one place, don't rename it "the ETL job" two paragraphs later. Humans resolve synonyms effortlessly; treating terms as consistent identifiers removes a source of ambiguity for an agent trying to match a rule to the code path it's currently touching.

- **DON'T:** Write in a conversational, narrative voice aimed at making a new human teammate feel welcome ("Hey! Welcome to the team, we're so glad you're here..."). That framing wastes tokens and signals nothing actionable; state facts and rules directly.

- **DON'T:** Bury the actionable instruction inside a rhetorical question, a joke, or a hedge ("you might want to consider maybe running the linter at some point"). State the rule plainly as an imperative.

### Avoiding Auto-Generated, Uncurated Instruction Files

- **DO:** Have a human who actually knows the codebase review and edit any instruction file before it's committed, whether it was drafted from scratch, generated by an agent summarizing the repo, or copied from a template. The file is making claims that will be trusted and acted on; unverified claims in it are worse than no file, because they're followed with false confidence.

- **DO:** Treat an LLM-generated first draft as exactly that — a first draft to prune, correct, and fact-check against the real commands, real versions, and real constraints of the project, not a finished artifact to commit as-is.

- **DO:** Spot-check any auto-generated command list by actually running each command before committing it. A generated "test command" that was pattern-matched from a common convention rather than read from the actual `package.json` scripts block is a frequent source of confidently wrong instructions.

- **DON'T:** Run a "generate CLAUDE.md for my repo" prompt once and commit the output unedited. Uncurated, LLM-authored instruction files tend to be simultaneously too long (padded with generic best-practice prose that applies to any codebase) and too thin on the specific, non-obvious facts that would have actually prevented a mistake — a combination that measurably drags down agent task success while inflating token cost, the opposite of the file's purpose.

- **DON'T:** Let an agent regenerate or substantially rewrite its own instruction file autonomously mid-task without human review, even if it noticed something stale. Have it propose the specific edit as a diff for a human to approve, the same way it would propose a code change — instruction files are policy, and policy changes deserve a review step.

### Keeping Instructions Fresh

- **DO:** Update the instruction file in the same PR that changes the fact it describes — a version bump, a renamed script, a changed test command, a new required environment variable. Treat drift between the instruction file and reality as a bug, exactly like a broken test.

- **DO:** Periodically audit the file against the current codebase (a quarterly pass is a reasonable cadence for an actively developed project) specifically hunting for commands that no longer work, versions that have since been bumped, and rules that describe a pattern the codebase has since moved away from.

- **DO:** Treat an agent reporting "this instruction doesn't match what I found in the code" as a signal worth act­ing on immediately — file a follow-up to fix the instruction file, since if it fooled the agent once it will fool it again next session.

- **DON'T:** Let the instruction file become historical fiction — describing a migration that's since been reverted, a framework that's since been replaced, or a directory structure from before a refactor. A stale instruction file is worse than none, because it actively steers the agent toward code that no longer exists or patterns the team has deliberately abandoned, and it teaches the agent (correctly) to distrust the rest of the document too.

- **DON'T:** Let a big rewrite or migration land without a corresponding pass over the instruction file. If a project migrates its test runner, its ORM, or its package manager, the instruction file is a first-class artifact of that migration, not an afterthought to fix "eventually."

### Internal Consistency: Avoiding Contradictions

- **DO:** Read the file end to end before committing changes to check that no rule in one section silently contradicts a rule in another — for example, a "permissions" section granting free rein to install dependencies while a "workflow" section elsewhere says all dependency changes require sign-off. When two sections disagree, an agent has no principled way to know which one governs, and will either pick unpredictably or freeze and ask a question the human considers already answered.

- **DO:** Resolve apparent tension explicitly rather than leaving both statements standing — if there's a genuine exception to a general rule, say so in the same place as the rule ("never add a production dependency without approval, except for well-known patch-level security updates, which may be applied directly").

- **DO:** Keep a single source of truth per fact. If the test command is stated in a "Quick Start" section and again in a "CI" section, make sure an edit to one is mirrored in the other, or better, state it once and reference it from the second location instead of duplicating it.

- **DON'T:** Let sections written at different times by different contributors quietly diverge — a common failure mode when several team members each add "their" section over months without re-reading the whole document. Assign the file an owner, or at minimum treat any addition to it as requiring a full read-through, the same discipline applied to any other shared source of truth.

### Concrete, Checkable Rules Over Vague Adjectives

- **DO:** Phrase every rule so that a specific piece of code or a specific action can be checked against it and unambiguously pass or fail. "Functions in `src/api/` must return `Result<T, E>`, never throw" is checkable; "write clean, maintainable code" is not — it means something different to every reader and gives an agent no way to know if it has satisfied the instruction.

- **DO:** Replace quality adjectives with the concrete practice that adjective was standing in for. If "good practices" meant "always validate input at the API boundary with `zod`," say that. If "clean code" meant "functions under 40 lines, no more than 3 levels of nesting," say that.

- **DO:** Prefer numeric, structural, or example-anchored rules over aesthetic ones wherever the underlying intent can be made concrete — line-length limits, a specific linter config to defer to, a named pattern to follow, a specific file to use as the canonical example of "the way we do this."

- **DON'T:** Write rules using words like "clean," "elegant," "idiomatic," "modern," "robust," "professional," or "best practices" without immediately cashing them out into something checkable. These words carry real meaning to an experienced human engineer steeped in a particular team's taste, but to a model with no access to that unstated taste they resolve to whatever pattern is statistically dominant across all training data — frequently not what this specific team means.

- **DON'T:** Assume a vague rule is harmless because "the agent will figure out what we mean from context." Sometimes it will; the failure mode when it doesn't is silent, plausible-looking code that satisfies the letter of a vague rule while violating its unstated intent, and nobody notices until review.

```markdown
<!-- Vague, unenforceable -->
Write clean, idiomatic code following best practices.

<!-- Concrete, checkable -->
- Max function length: 40 lines (enforced by `eslint max-lines-per-function`).
- No `any` in TypeScript; use `unknown` and narrow, or a generic.
- Prefer early returns over nested if/else; max nesting depth is 3 (see `.eslintrc`).
- Validate all external input at the API boundary with `zod`; never trust a request body downstream.
```

### Monorepos and Nested Instruction Files

- **DO:** Use a top-level instruction file for facts true of the whole repository (global commands, org-wide permission rules, shared tooling) and nested instruction files inside individual packages or services for facts scoped to that unit, when the agent's tooling supports resolving both. This mirrors how a human would want the same information laid out and keeps each file focused on what's actually true in its own directory.

- **DO:** Make explicit, in the top-level file, that nested files exist and take precedence for their subtree, so the agent knows to look for and prefer the more specific file when working inside a given package rather than only ever reading the root file.

- **DO:** Keep package-level files scoped strictly to what's different about that package — its own test command if it differs from the monorepo default, its own non-standard dependency, its own permission exception — rather than re-explaining monorepo-wide facts already stated at the root.

- **DON'T:** Let package-level files contradict the root file instead of refining it. A nested file should narrow or add to the parent's rules, never silently reverse a root-level "never" into a local "always" without explicitly flagging the override and why it's safe in that specific context.

- **DON'T:** Duplicate the entire root file into every package directory "for convenience." That multiplies the staleness problem across every copy and guarantees the copies will disagree with each other within a few months.

```markdown
repo-root/CLAUDE.md          # global commands, org-wide permission tiers, shared conventions
packages/api/CLAUDE.md       # api-specific: its own test DB setup, its non-standard RPC framework
packages/web/CLAUDE.md       # web-specific: its own component conventions, its Storybook workflow
packages/legacy-billing/CLAUDE.md  # narrow permission override: "never touch without human review"
```

### File Naming and Tool-Specific Variants

- **DO:** Match the exact filename and location convention each tool expects (`CLAUDE.md` at the repo root for Claude Code, `AGENTS.md` for tools that follow that emerging convention, `.cursor/rules/*.mdc` or `.cursorrules` for Cursor) so the tool actually picks the file up automatically rather than the content going unread because it lived at the wrong path.

- **DO:** When a team uses more than one agent tool and the underlying rules are genuinely identical, keep one canonical file and have the tool-specific files be thin pointers or symlinks to it, rather than maintaining near-duplicate content in three places that will drift apart.

- **DO:** Where a tool's rule format has structural requirements the others don't (for example, frontmatter fields, glob-scoped rule files, or a size limit), respect that tool's format precisely rather than reusing prose written for a different tool's conventions verbatim — a `.mdc` file with malformed frontmatter may simply fail to load.

- **DON'T:** Assume every agent tool reads every possible filename. An unread file provides zero value regardless of how well written it is; verify, for each tool actually in use on the project, that its instruction file is in the location and format that tool expects.

### Example: A Well-Formed Instruction File Skeleton

- **DO:** Order sections by how urgently the agent needs the information: commands and permissions first (needed before doing anything), stack and conventions next (needed while doing the work), and deeper context or links to further reading last (needed occasionally). This mirrors the actual order in which an agent needs facts during a session.

```markdown
# CLAUDE.md

## Commands
(install / dev / test / lint / typecheck / build — exact flags)

## Stack
(pinned versions, framework, DB, non-standard tooling)

## Permissions
(always OK / ask first / never)

## Code style
(one or two real snippets, not prose)

## Testing rules
(mocking, determinism, coverage bar)

## Gotchas
(the 3-5 things that have bitten people before, stated as rules)

## Further reading
(links to ARCHITECTURE.md, CONTRIBUTING.md, etc. — not inlined)
```

- **DON'T:** Front-load the file with company history, team structure, or philosophy. None of that changes what the agent should do in the next five minutes; put it in `docs/` and link it once from "Further reading" if it must exist at all.

### Full Worked Example: Before and After

- **DO:** Compare a flawed instruction file against a corrected one side by side when auditing or teaching this convention — the difference is usually not any single sentence but the cumulative effect of several small failures (vagueness, staleness, missing permissions, prose bloat) compounding into a file that reads fine but doesn't actually steer behavior.

The flawed version below is realistic: every individual sentence in it looks reasonable in isolation, and it would pass a casual skim. It fails for reasons this section covers throughout — vague adjectives instead of checkable rules, no pinned versions, no permission tiers, a stale command, and prose where a snippet or a list would work better.

```markdown
<!-- FLAWED: reads fine, steers nothing -->
# Welcome to Acme Platform!

This is the main repository for Acme's platform. It's a modern web application
built with a lot of care by our amazing engineering team over the past few years.
We believe in writing clean, maintainable, well-tested code and following
industry best practices at all times.

## Getting started

Clone the repo and install dependencies, then you should be able to run the
dev server. If you run into issues, ask in #eng-help.

To run tests, use the test command. Please make sure your code is well tested
and follows our style guide before submitting a PR.

## Architecture

Our system uses a microservices architecture with several services that
communicate via events. We chose this architecture because it gives us
flexibility to scale different parts of the system independently as the
company grows, which was an important consideration given our trajectory...
[... 40 more lines of architecture history and rationale ...]

## A note on quality

Please write clean, idiomatic, professional code. We care a lot about code
quality here and expect all contributions to meet a high bar.
```

The corrected version states the same underlying facts — plus the ones actually missing above — as checkable, scannable rules an agent can act on immediately:

```markdown
<!-- CORRECTED: short, checkable, current -->
# CLAUDE.md

## Commands
- Install: `pnpm install --frozen-lockfile`
- Dev server: `pnpm dev` (localhost:3000)
- Tests: `pnpm test` (full, ~5 min) · `pnpm test <path>` (single file) · `pnpm test -t "<name>"` (single case)
- Lint (must be zero warnings, matches CI): `pnpm lint --max-warnings=0`

## Stack
- Node 20.11 (`.nvmrc`), pnpm 9 only. Next.js 14 App Router. PostgreSQL 15 via Prisma 5.18.
- Non-standard: internal `depctl` wraps pnpm for licensing checks — use `depctl add <pkg>`, not raw `pnpm add`.

## Permissions
- Always OK: tests, lint, typecheck, files under `src/`, `test/`; dev-dependency changes.
- Ask first: new production dependencies; anything under `infra/` or `.github/workflows/`; DB migrations.
- Never: force-push to `main`; commit secrets; bypass pre-commit hooks; touch `legal/`.

## Code style
    // Errors are typed Results, never thrown across module boundaries:
    async function loadUser(id: string): Promise<Result<User, AppError>> { ... }

## Testing rules
- Use `clock.now()` from `src/lib/clock.ts`, never `Date.now()` directly, in code under test.
- Mock HTTP with `nock` fixtures in `test/fixtures/http/`. No snapshot tests.

## Gotchas
- "Connection terminated unexpectedly" in tests → stale transaction, run `pnpm test:reset-db`, don't just retry.
- Local `pnpm build` passing does not guarantee CI's stricter `tsconfig.build.json` passes — run `pnpm typecheck:strict`.

## Further reading
- Architecture rationale: `docs/ARCHITECTURE.md` · Contributing: `CONTRIBUTING.md`
```

- **DON'T:** Mistake the flawed version's friendly tone or apparent thoroughness for quality. It is longer than the corrected version, reads as more welcoming, and contains not a single false statement — and it is still strictly worse, because none of its content is checkable or immediately actionable, its one real instruction ("use the test command") doesn't say what that command is, and its closing quality guidance ("clean, idiomatic, professional") is exactly the class of vague adjective this section warns against.

### Environment Variables, Secrets, and Local Setup

- **DO:** List which environment variables are required to run the project locally and where to get sanctioned values for them (a `.env.example` file, a secrets manager, a team wiki link) without ever putting a real secret value inside the instruction file itself. The file is committed to source control and typically has broad read access; it must never be the place a credential lives.

- **DO:** State explicitly that secrets are never to be printed, logged, committed, or included in a diff, even for debugging purposes, and name any tooling that would catch or prevent that (a secret-scanning pre-commit hook, a `.gitignore` entry) so the agent knows a safety net exists but doesn't rely on it as the sole line of defense.

- **DO:** Note any local infrastructure a task might need (a database that must be running, a queue, a mock third-party service) and the exact command to bring it up, since an agent that doesn't know a Postgres container needs to be running before tests will pass wastes a cycle debugging a misleading connection-refused error instead.

- **DON'T:** Paste a real API key, database URL with credentials, or token into the instruction file "temporarily" for convenience. Convenience secrets in a text file read by an agent and committed to history are a standing security liability, not a shortcut.

```markdown
## Local setup

- Copy `.env.example` to `.env.local`; ask a team lead for real values, never invent placeholders that look real.
- Start local infra: `docker compose up -d db redis` (required before `pnpm dev` or `pnpm test:integration`).
- Never print the contents of `.env*` files in output, logs, or commit messages.
```

### Git Workflow, Commit, and PR Conventions

- **DO:** State the branch naming convention, the commit message format (a specific style like Conventional Commits, or an internal convention), and any required PR template sections, if the project enforces them, since these are exactly the kind of arbitrary-but-mandatory convention an agent has no way to guess correctly on its own.

- **DO:** Say explicitly whether the agent is expected to create commits and branches as part of its normal workflow, or whether it should stage changes and let a human commit — this varies by team and materially changes what "finishing a task" looks like.

- **DO:** State any required pre-merge steps beyond passing CI — a required reviewer, a linked ticket ID in the PR description, a changelog entry — so a PR isn't left in a state that looks done but is blocked on a step nobody flagged.

- **DON'T:** Leave commit conventions unstated and let each session invent its own commit message style, if the project actually enforces one via a commit-lint hook or a documented convention — an agent unaware of the convention will produce commits that fail CI or need to be rewritten.

```markdown
## Git conventions

- Branch names: `<type>/<short-desc>` e.g. `fix/pagination-off-by-one`.
- Commit messages: Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`, `test:`), imperative mood.
- Every PR description must reference a ticket ID (`Closes ENG-1234`) — CI checks for this pattern.
- Do not create commits directly on `main`; always work on a feature branch.
```

### Domain Glossary for Non-Obvious Terminology

- **DO:** Define project- or domain-specific terms, internal codenames, or overloaded words that carry a specific meaning in this codebase different from their everyday meaning — a "tenant," a "workspace," a "run," a "shard" — whenever getting the definition wrong would lead to code that operates on the wrong concept or the wrong scope.

- **DO:** Keep glossary entries short and definitional, not explanatory essays — one line stating what the term means and, where relevant, which type or table represents it in code.

- **DON'T:** Assume terminology is self-evident from context when a project has quietly overloaded a common English word to mean something narrower or different than usual. An agent that doesn't know "account" specifically means "billing account" and not "user login" in this codebase can write code that conflates the two.

```markdown
## Glossary

- **Workspace**: a billing/permissions boundary containing one or more Projects (table: `workspaces`).
  Not the same as a user's personal settings — see `personal_settings` for that.
- **Run**: one execution of a pipeline (table: `pipeline_runs`), not a CI run — CI runs are `ci_jobs`.
- **Draft**: an unpublished Document version; a Draft is never shown to end users, only to its author.
```

### Debugging Tips and Known Pitfalls

- **DO:** Record the handful of gotchas that have actually bitten people before — a misleading error message that means something other than what it says, a race condition in the test suite, a caching layer that makes a change look like it didn't take effect — as explicit, named rules rather than leaving each session to rediscover them the hard way.

- **DO:** Keep this section short and specific to real, repeated incidents, not a general troubleshooting guide for the underlying framework — the value is in the project-specific gotchas nothing else would tell the agent.

- **DON'T:** Let this section balloon into a general debugging tutorial for the language or framework in use. That's well covered by the model's training data already; the only content worth the tokens here is what's specific to this codebase's own sharp edges.

```markdown
## Known pitfalls

- "Connection terminated unexpectedly" from the test DB almost always means the previous test run
  left a transaction open — run `pnpm test:reset-db`, don't just retry.
- The dev server caches API responses for 60s even with hot reload; hard-refresh or hit `/api/__clear-cache`
  if a backend change doesn't seem to show up.
- `pnpm build` succeeding locally does NOT mean the Vercel build will succeed — Vercel uses a stricter
  TypeScript config (`tsconfig.build.json`); run `pnpm typecheck:strict` to catch what local `build` misses.
```

### CI/CD Awareness

- **DO:** State what CI actually checks, in what order, and which checks are required to merge versus advisory, so the agent's own pre-submission verification actually mirrors what will gate the PR rather than a guessed subset of it.

- **DO:** Note any CI behavior that differs meaningfully from local runs (a stricter config, a different environment, a matrix of versions tested) so a local "it passes" claim is calibrated to what CI will actually enforce, not a false signal of full readiness.

- **DON'T:** Assume the agent can infer the CI pipeline's exact checks from a cursory glance at a workflow file. If specific steps are commonly missed or misunderstood (a required status check with a non-obvious name, a check that only runs on certain paths), say so directly instead of relying on the agent to reverse-engineer CI configuration under time pressure.

### Delegating to Existing Linters and Formatters Instead of Restating Their Rules

- **DO:** Point at the actual linter/formatter config (`.eslintrc`, `ruff.toml`, `.prettierrc`) as the source of truth for style rules already encoded there, rather than re-describing those same rules in prose in the instruction file. A linter config is enforced automatically and can't drift from itself the way a prose restatement of it can.

- **DO:** Reserve prose style guidance in the instruction file for conventions the linter genuinely cannot express — naming philosophy, file organization, when to extract a helper versus inline logic — the judgment calls a formatter has no opinion on.

- **DON'T:** Duplicate a linter's rule set into the instruction file as a bullet list ("use 2-space indentation," "always use semicolons," "prefer single quotes"). This is both redundant with something already mechanically enforced and a fresh source of drift the moment the lint config changes and the prose copy doesn't.

## SKILL.md & Agent Skill Design

### What a Skill Is, and What It Isn't

- **DO:** Think of a skill as a packaged, reusable procedure the agent should follow for one recurring kind of task — closer to a runbook or a well-tested macro than to a general-purpose personality tweak. A skill earns its existence by encoding know-how that would otherwise have to be re-explained, re-discovered, or re-derived by the agent every time the task comes up.

- **DO:** Reserve skills for tasks with real recurring structure — a specific multi-step procedure, a specific output format with real constraints, a specific tool sequence — not for one-off requests that happen once and never recur. A skill written for a task that will only ever be asked once is a maintenance cost with no repeated payoff.

- **DON'T:** Use a skill as a dumping ground for general tone or personality preferences that apply to everything the agent does regardless of task. That belongs in a standing system-level instruction, not in a triggerable, task-scoped skill — packaging it as a skill means it only applies when the trigger fires, silently disabled the rest of the time.

### Frontmatter: Name and Trigger Description

- **DO:** Give the skill a short, specific, literal name that reflects the task it performs, not a marketing name or an abbreviation only the author would recognize. A name like `pdf-form-fill` communicates instantly; a name like `DocMaster` or `sk_047` communicates nothing and forces whoever is choosing between skills to open each one to find out what it does.

- **DO:** Write the one-line description to state exactly *when* to use the skill — the triggering situation, in the user's own likely words — not just a restatement of what the skill does internally. A description is a routing signal read by something deciding whether this skill is relevant right now; it needs to describe the situation that should trigger it, not summarize its implementation.

- **DO:** Include concrete trigger phrases or synonyms a user would plausibly say, especially ones that don't share vocabulary with the skill's internal name, so semantic or keyword matching against the description actually catches real phrasing. If users say "quarterly numbers deck" as often as "financial report," both should appear somewhere in the description.

- **DO:** State any hard prerequisite or scope boundary directly in the description if it changes whether the skill should fire — for example, that a skill only applies to a specific file type, a specific existing document, or a specific stage of a workflow.

- **DON'T:** Write a description that only describes mechanism ("uses pandas to transform CSV data") without describing the triggering situation ("when the user has a messy CSV/Excel file that needs cleaning before analysis"). A mechanism-only description fails to fire on natural user phrasing that never mentions the mechanism.

- **DON'T:** Make the description so generic that it reads as applicable to almost any request ("helps with documents," "assists with data tasks"). An overly broad description causes the skill to fire — or be considered — far more often than it's actually the right tool, adding noise to every unrelated request.

```markdown
---
name: invoice-reconciliation
description: >
  Match a batch of vendor invoices (PDF or CSV) against purchase orders and flag
  discrepancies. Use when the user has invoices to reconcile, mentions "matching
  invoices to POs," "invoice discrepancies," or uploads accounts-payable exports
  needing a three-way match against POs and receiving records.
---
```

### One Coherent Capability Per Skill

- **DO:** Scope a skill to a single coherent capability with a clear beginning and end — one job that can be described in one sentence without an "and" joining two unrelated things. A skill that reconciles invoices is one capability; a skill that reconciles invoices *and* generates monthly board decks *and* drafts vendor emails is three capabilities wearing one name.

- **DO:** Split a skill that has organically accumulated multiple unrelated responsibilities into separate skills, each independently triggerable, once the grab-bag nature becomes apparent — usually visible as an unwieldy frontmatter description trying to cover several distinct triggering situations at once with "or."

- **DO:** Let closely related sub-tasks live inside one skill when they're genuinely steps of the same procedure (e.g., "extract data from the PDF" and "validate the extracted data against schema" as two steps of one document-processing skill) — the test is whether they're always invoked together as part of one job, not whether they happen to touch similar file types.

- **DON'T:** Build a single mega-skill meant to be "the one skill for all spreadsheet work" or "the one skill for anything data-related." Broad umbrella skills produce diluted, generic instructions inside them (because they have to cover many unrelated cases) and make the trigger description impossible to write precisely, which drags down both triggering accuracy and instruction quality at once.

- **DON'T:** Duplicate logic across multiple skills instead of factoring out a shared sub-procedure, if a tool's skill system supports composition or shared reference material. Copy-pasted procedure steps rot independently and drift apart exactly like copy-pasted code.

### Calibrating Trigger Specificity

- **DO:** Aim the trigger description at the narrowest phrasing that still reliably covers every real situation the skill should handle — specific enough to avoid false-positive firing on unrelated requests, broad enough to still catch the range of ways a user actually asks for this task.

- **DO:** Test the trigger description against both the requests it should catch and adjacent requests it should *not* catch, and refine wording until both hold. A financial-report skill's description should fire on "build me the quarterly board deck" but not on every unrelated request that happens to mention a number.

- **DO:** Prefer a description anchored in the situation and artifact ("the user has a spreadsheet of sales data and wants trend analysis") over one anchored in abstract capability ("performs data analysis"), since situational descriptions map more precisely onto real requests and produce fewer accidental matches.

- **DON'T:** Write a trigger so broad it fires on nearly every request in its general domain, effectively becoming the default handler for anything vaguely related. Constant, low-precision firing trains users (and the routing layer) to distrust or route around the skill, defeating its purpose.

- **DON'T:** Write a trigger so narrow — tied to one exact phrase or one specific file name a single user happened to use once — that it never fires again for anyone else asking for the same underlying task in slightly different words. Over-narrow triggers are a common result of writing the description by copying the one request that prompted the skill's creation, instead of generalizing from it.

### Versioning Skills as the Underlying Process Changes

- **DO:** Version a skill explicitly (in a changelog section, a version field, or a dated note) whenever the underlying process it encodes changes — a new required step, a changed tool, a changed output format required by a downstream consumer. Skills that quietly drift out of sync with the real process they were meant to encode become a source of consistently wrong output.

- **DO:** Record why a version changed, not just that it did, so a future edit can tell whether an old instruction inside the skill is stale by design or by accident. "v3: switched from `xlsx` output to `csv` because downstream ingestion changed" is far more useful later than a bare version bump.

- **DO:** Re-validate a skill against a real example task after any nontrivial edit, the same way a code change gets tested, rather than assuming a plausible-looking instruction edit will produce correct behavior the first time it's actually invoked.

- **DON'T:** Let a skill silently fall out of sync with a process change happening elsewhere (an API this skill calls gets deprecated, a template it fills in gets redesigned, a compliance rule it enforces gets updated) without an explicit process for someone to notice and update it. An outdated skill is worse than no skill, because its confident, procedural framing makes wrong output look authoritative.

- **DON'T:** Treat "the skill has always worked" as evidence it's still correct if the environment around it has changed. A skill can produce plausible-looking output that satisfies its own internal logic while being wrong against a process that moved on without it.

### Concrete Examples and Templates Inside a Skill

- **DO:** Embed at least one concrete worked example or a fillable template directly inside the skill — a sample input and the exact expected output shape, a boilerplate file to start from, a reference table of valid values — rather than relying solely on abstract prose instructions to convey the target shape.

- **DO:** Make the example realistic enough to disambiguate genuine edge cases the skill needs to handle correctly (a field that's sometimes missing, a value that needs special-casing), not just a happy-path toy case that leaves the hard part unaddressed.

- **DO:** Keep templates in their own reference files inside the skill's directory (rather than pasted inline as prose) when a tool's skill format supports bundling reference files, so the exact template can be copied or extended precisely instead of being re-derived from a description of it.

- **DON'T:** Describe the desired output format only in prose ("the report should have a summary section, then a table, then recommendations") when providing the literal template achieves the same result with far less room for the agent to improvise a structurally different but individually plausible-looking alternative.

- **DON'T:** Let the embedded example go stale relative to the actual current template or schema used downstream — an example is a promise about what correct output looks like, and a promise that's silently broken teaches wrong output with full confidence.

### Avoiding Skills That Duplicate the Base Model's Own Competence

- **DO:** Build a skill only where it adds real value beyond what the underlying model already does well unprompted — a specific required format, a specific sequence of tool calls, domain knowledge the model doesn't reliably have, or a compliance/consistency requirement that needs to be enforced every time rather than left to per-session judgment.

- **DO:** Test whether the task the skill targets is actually done well without it, on a few representative examples, before investing in building and maintaining the skill. If the model already produces essentially the same correct output unprompted, the skill is pure overhead: another file to maintain, another trigger to tune, another thing that can go stale, for no behavior change.

- **DON'T:** Write a skill that just re-states general good practice the model already reliably follows ("write clear, well-organized prose," "double-check your arithmetic," "use proper grammar"). These add token cost and a firing surface with no corresponding improvement in output, and they crowd out the skill's usefulness signal for the genuinely load-bearing skills nearby.

- **DON'T:** Build a skill purely to "make sure" the model does something it already reliably does, out of an abundance of caution. If a specific failure mode is actually observed in practice, encode the fix for that specific failure mode — don't pre-emptively wrap already-solid behavior in ceremony.

### Skill Composition, Scope, and Boundaries

- **DO:** Design skills to compose cleanly when a task legitimately spans more than one — a report-generation skill and a chart-styling skill should be able to both apply to one request without their instructions conflicting or duplicating each other's guidance.

- **DO:** Make each skill's scope boundary explicit when it's easy to confuse with a neighboring skill — state directly what this skill does *not* cover and, where possible, name the sibling skill that does, so the routing decision between look-alike skills is unambiguous rather than left to guesswork.

- **DON'T:** Let two skills silently overlap in what they claim to handle, each with its own slightly different instructions for the same underlying task. Overlapping skills create nondeterministic output depending on which one happens to fire, and neither maintainer notices the other exists to keep them in sync.

### Testing and Iterating on Skills

- **DO:** Run a skill against a handful of realistic example requests — including edge cases and near-miss requests that shouldn't trigger it — before considering it finished, the same way a function gets tested before being called done.

- **DO:** Collect and review real triggering failures (fired when it shouldn't have, didn't fire when it should have, fired correctly but produced wrong output) and feed them back into either the trigger description or the body instructions, treating skill quality as something iterated on with evidence rather than finalized once at authoring time.

- **DON'T:** Ship a skill based purely on how it reads, without running it against at least one real task end to end. A skill description and body can look complete and well-written while still producing subtly wrong output the first time it's actually exercised — the only way to know is to run it.

### Skill Directory Structure and Bundled Resources

- **DO:** Bundle a skill's supporting material — templates, reference tables, small scripts, sample fixtures — alongside its main instructions in the skill's own directory when the tooling supports it, so the skill is a self-contained unit rather than a set of instructions that silently depend on files living elsewhere that could move or be deleted without the skill noticing.

- **DO:** Keep the main instruction file focused on the procedure itself, with reference material split into clearly named companion files the instructions point to by name, rather than inlining a large reference table or a long template directly into the main flow where it interrupts the procedural steps.

- **DO:** Name bundled files descriptively enough that their purpose is clear from the filename alone (`invoice_template.csv`, `field_mapping_reference.md`) rather than generic names (`data.csv`, `notes.md`) that give no signal about what's inside without opening them.

- **DON'T:** Reference an external file path, URL, or shared resource from inside a skill without bundling or clearly pinning it, if that resource is likely to move, be renamed, or go stale independent of the skill. A skill that silently breaks because a linked resource moved is a hard failure to diagnose, since the skill's own instructions will look complete and correct in isolation.

### Scoping Tool Access and Permissions Per Skill

- **DO:** Grant a skill access only to the tools and permissions it actually needs for its stated procedure, when the platform supports scoping tool access per skill, following the same least-privilege reasoning applied to the three-tier permission model in the repo-level instruction file.

- **DO:** Flag inside the skill itself any step that requires an elevated or risky permission (a network call to an external system, a destructive file operation, a payment action) so a human reviewing the skill — or an agent about to invoke it — can see the risk surface without having to trace through every step of the procedure first.

- **DON'T:** Grant a skill broad, unscoped tool access "to be safe" or "in case it's needed," mirroring the anti-pattern of registering speculative tools. A skill with narrower access is both safer by default and easier to reason about when auditing what it's capable of doing.

### Background/Autonomous Skills vs. Interactive Ones

- **DO:** Design a skill meant to run unattended or in the background (a scheduled task, a fully autonomous multi-step job) with more conservative defaults and stricter internal checks than an interactive skill that has a human watching each step and able to interrupt — the cost of a wrong turn is much higher when nobody is present to catch it early.

- **DO:** Build an explicit stopping condition or checkpoint into any skill meant to run for many steps or a long duration unattended, so it doesn't compound an early mistake across dozens of subsequent steps before anyone notices.

- **DON'T:** Reuse an interactive skill's instructions verbatim for an unattended/background context without re-examining which steps assumed a human would be present to catch an error or answer a clarifying question. A step that says "ask the user if unsure" has no meaningful behavior in a fully unattended run and needs an explicit fallback.

### Gallery of Common Skill-Authoring Mistakes

- **DON'T:** Write a trigger description that's a direct copy of the skill's own internal jargon rather than the language a real user would type — the fix is to write the description from the outside, imagining the actual request, not from the inside, describing the implementation.

- **DON'T:** Let a skill's body instructions silently assume a specific starting state (a specific file already exists, a specific tool is already authenticated) without checking for it or stating it as a precondition — a skill that fails cryptically because an unstated precondition wasn't met looks like a bug in the skill rather than a documentation gap, and gets debugged as one.

- **DON'T:** Write a skill so rigidly procedural that it can't handle a minor, foreseeable variation in the input (a slightly different file format, an optional field that's sometimes absent) without breaking entirely — build in the small amount of judgment the procedure actually needs rather than treating every input as if it will exactly match one canonical shape.

- **DON'T:** Leave a skill's instructions written entirely in the imperative voice with no rationale anywhere, so that when an input doesn't quite fit the documented steps, there's no stated intent to reason from in order to improvise correctly — a line or two of "why this step exists" at key junctures lets a skill degrade gracefully instead of failing opaquely on the first unanticipated case.

### Full Worked Example: Before and After

The flawed skill below has a plausible-looking frontmatter and body — it would likely pass a quick read — but combines several of the mistakes above: a mechanism-only description with no triggering situation, a grab-bag of unrelated capabilities, and prose instructions with no concrete example of the target output.

```markdown
<!-- FLAWED -->
---
name: doc-helper
description: Helps with documents using various techniques.
---

# Doc Helper

This skill helps the user with documents. It can create reports, format
spreadsheets, clean up data, and also help write emails about the documents
if needed. Try to produce good, professional-looking output that the user
will be happy with. Use your best judgment on formatting.
```

This fails to fire reliably (the description gives no situational trigger), fires on the wrong requests when it does fire (it claims four unrelated capabilities), and gives an agent nothing concrete to match against for "good, professional-looking output." The corrected version narrows to one capability, states an exact trigger, and includes a real template:

```markdown
<!-- CORRECTED -->
---
name: expense-report-builder
description: >
  Build a formatted expense report from a batch of receipts (images, PDFs, or a
  CSV export). Use when the user has receipts to reconcile, says "expense report,"
  "reimbursement summary," or uploads a folder of receipt images/PDFs needing totals
  by category.
---

# Expense Report Builder

## When this applies
Receipts or a receipt export need to become one formatted report with totals
by category, ready to submit for reimbursement. Not for general spreadsheet
cleanup — use `spreadsheet-cleanup` for that.

## Steps
1. Extract vendor, date, amount, and category from each receipt (see
   `category_mapping.md` in this skill's directory for the category list).
2. Fill `report_template.csv` (bundled) — one row per receipt, one summary
   row per category.
3. Flag any receipt where amount or date couldn't be confidently read, rather
   than guessing a value.

## Example output shape
See `example_output.csv` in this skill's directory for a fully worked sample
with three categories and a flagged unreadable receipt.
```

- **DO:** Notice that the corrected version is not merely "better worded" — it is scoped to one capability, states the one sentence that would make it fire at the right time, names its sibling skill to resolve the obvious adjacent-skill ambiguity, and points to bundled concrete artifacts instead of describing the target shape in adjectives. Each of those is a distinct fix for a distinct failure mode, not a single stylistic polish pass.

## Tool Design for Agents

### Single Clear Responsibility Per Tool

- **DO:** Design each tool to do exactly one well-defined thing, with a name and behavior that map cleanly onto that one thing. A tool called `search_issues` should search issues; it should not also, depending on which parameters are set, create an issue, close an issue, and post a comment — each of those is a different responsibility and belongs in its own tool.

- **DO:** Split a tool that has accumulated multiple modes behind a discriminator parameter (an `action: "get" | "create" | "delete"` style parameter) into separate tools once the modes have meaningfully different parameter shapes, error conditions, and risk profiles. A single overloaded tool forces the agent to reason about which mode applies before it can even reason about the parameters for that mode, adding an extra layer of indirection to every call.

- **DO:** Let a small number of tools remain intentionally multi-mode only when the modes are genuinely trivial variations of the same operation with near-identical parameters and risk (e.g., a `list` tool that takes an optional filter) — the line to hold is whether a human reading the tool name alone would correctly guess what it does in every mode.

- **DON'T:** Build a single do-everything tool ("manage_project") that internally branches into a dozen unrelated operations to minimize the number of tools registered. Fewer tools is not free — a tool that hides ten operations behind one name and a mode parameter forces the agent to first discover the hidden operation surface (often by trial and error) before it can act, which is slower and more error-prone than picking the right narrowly-named tool from a list.

- **DON'T:** Conflate "read" and "write" behavior in one tool parameterized by a boolean flag such as `dry_run` unless the tool is specifically designed around a preview/commit pattern. A tool whose destructiveness is toggled by an easy-to-miss boolean is a standing invitation for the flag to be forgotten exactly when it mattered most.

```
# Overloaded, ambiguous:
manage_ticket(action: "create"|"update"|"close"|"comment"|"assign", ...)

# Single responsibility, unambiguous:
create_ticket(title, description, labels)
update_ticket_status(ticket_id, status)
add_ticket_comment(ticket_id, body)
assign_ticket(ticket_id, assignee)
```

### Naming Tools and Parameters for Pattern-Matching

- **DO:** Give every tool a descriptive, literal, verb-first name that unambiguously signals its effect — `delete_file`, `send_email`, `create_pull_request` — since agents select tools largely by pattern-matching the task at hand against tool names and descriptions, not by deeply reasoning about every candidate's full schema every time.

- **DO:** Name parameters after the domain concept they represent, using the same terminology the tool's description and the rest of the toolset use, so an agent that has learned the vocabulary from one tool can transfer it correctly to a related one — a `user_id` in one tool and a `userId` or `uid` in a sibling tool for the same underlying concept invites mismatched calls.

- **DO:** Make destructive or irreversible tools sound as consequential as they are — `permanently_delete_record` reads very differently from `remove_record`, and the more accurate, weightier name measurably reduces careless invocation.

- **DO:** Keep parameter names self-explanatory enough that an agent rarely needs to consult the full description just to guess a parameter's shape — `format: "json" | "csv"` over `mode: string`, `max_results: number` over `limit: any`.

- **DON'T:** Use internal jargon, abbreviations, or codenames as tool or parameter names when a plain descriptive alternative exists. A tool named `zeta_sync` or a parameter named `flg2` communicates nothing to pattern-matching and forces a full read of the description just to form a hypothesis about what it does.

- **DON'T:** Give near-duplicate names to tools with meaningfully different behavior (`update_record` vs `update_record_v2` vs `update_record_full`) — near-identical names collapse into indistinguishable options during selection, and the agent has no reliable signal for which one is intended for the current situation without opening every schema.

### Writing Descriptions: When to Use, Not Just What It Does

- **DO:** Open the tool description with the situation in which it should be reached for, not only a summary of its mechanism — "Use this to look up a customer's current subscription tier before answering billing questions" is more actionable than "Returns the customer's subscription record."

- **DO:** State explicit boundaries — what this tool does *not* do, and which other tool to use instead for an adjacent but different need — whenever confusion with a neighboring tool is plausible. This is the same principle as scoping a skill, applied to individual tools: an explicit boundary prevents mis-selection far more reliably than hoping the name alone disambiguates.

- **DO:** Document parameter constraints precisely and completely inside the schema — required vs. optional, valid enum values, format expectations, units (seconds vs. milliseconds, cents vs. dollars) — since a model filling in parameters from a description has no other source of truth for these details and will otherwise guess plausibly and sometimes wrongly.

- **DO:** Include a short example call in the description for any tool whose correct usage isn't obvious from the name and parameter list alone, particularly ones with structured or nested input.

- **DON'T:** Write a description that only restates the tool's name in slightly more words ("get_user: gets a user") without adding any information not already implied by the name — a description like that costs context budget for zero disambiguation value.

- **DON'T:** Leave units, formats, timezones, or coordinate systems ambiguous in a parameter's description ("start_time: the start time") when the tool actually requires a specific format (ISO 8601 UTC, a Unix timestamp in milliseconds, a specific timezone-naive local time). Ambiguous units are a leading cause of subtly wrong tool calls that pass validation but do the wrong thing.

```
# Weak description — restates the name, omits the constraint that actually matters:
"resize_image: resizes an image"

# Strong description — states when to use it and the constraint that would otherwise be guessed:
"resize_image: Use when the user needs an existing image scaled to specific pixel
dimensions (not cropped — use crop_image for that). max_dimension caps the longer
edge in pixels and preserves aspect ratio; output format defaults to the source
format unless output_format is set."
```

### Structured, Actionable Errors Over Silent Failures

- **DO:** Return a clear, structured error that states what went wrong and, where possible, what a valid retry would look like, whenever a tool call fails or a request can't be satisfied as given. A precise error lets the agent self-correct on the next call; an opaque or missing one leaves it guessing or, worse, assuming success.

- **DO:** Distinguish, in the error itself, between categories that call for different agent behavior — a permission/auth failure (retrying with the same parameters will never succeed, a human needs to act), a validation failure (the call itself was malformed and a corrected retry is likely to work), and a transient failure (a retry with the same parameters might succeed later). Collapsing all three into one generic "error occurred" message forces the agent to guess which kind it's facing.

- **DO:** Make partial-success outcomes explicit rather than reporting them as either a clean success or a clean failure. If a batch operation processed 8 of 10 items and 2 failed, say exactly that, with enough detail per failed item to retry just those — not a bare "success" that hides two silently dropped items, and not a bare "failure" that discards the eight that worked.

- **DON'T:** Swallow an internal error and return an empty result, a default value, or a bare `null` where a real error occurred. An agent has no way to distinguish "there were legitimately zero results" from "the call actually failed" if both produce the identical empty response — and it will typically treat the empty response as the former, silently proceeding on a false premise.

- **DON'T:** Return an error message so generic it provides no actionable signal ("something went wrong," "invalid request," a raw stack trace with no summary). A generic error forces either a blind retry with the same doomed parameters or an escalation to the human with nothing useful to relay.

```
# Silent failure — indistinguishable from "no results found":
{ "results": [] }

# Structured, actionable error:
{
  "error": {
    "type": "invalid_parameter",
    "parameter": "date_range.end",
    "message": "end date must be after start date (got end=2026-01-01, start=2026-03-01)",
    "retryable": true
  }
}
```

### Avoiding Destructive-by-Default Behavior

- **DO:** Require an explicit, hard-to-trigger-by-accident confirmation step or flag before a tool performs an irreversible action — permanently deleting data, force-overwriting a file, sending an external communication, spending money — so the default, easy-to-reach call path is always the safe, reversible one.

- **DO:** Prefer soft-delete, versioning, or a recoverable trash/archive state as the default behavior for "remove" operations wherever the underlying system supports it, reserving true permanent deletion for a separate, more clearly named and more deliberately gated tool.

- **DO:** Make any bypass of a safety check (a `force` flag, a `skip_confirmation` flag) require an explicit true value rather than defaulting to on, and name it so its danger is legible at a glance rather than buried in a generically named boolean.

- **DO:** Scope the blast radius of a single tool call as tightly as the task allows — a tool that deletes one named record by ID is safer by construction than one that deletes "everything matching a filter," even if the latter is more convenient to build; where a bulk-destructive operation is genuinely needed, make it return a preview of what would be affected before requiring a second call to actually commit.

- **DON'T:** Design a tool where the shortest, most obvious call is also the most dangerous one — a `delete(id)` tool with no confirmation, no soft-delete, and no dry-run path invites exactly the careless one-line call that causes real damage, especially under time or context pressure where an agent is more likely to skip caution it wasn't explicitly required to exercise.

- **DON'T:** Make irreversible actions symmetrically easy to invoke as reversible ones. If creating a resource takes one simple call, permanently destroying it should not also take one simple call with no additional friction — the asymmetry in real-world consequence should be reflected in the asymmetry of how easy each is to trigger.

### Idempotency Where Possible

- **DO:** Design tools so that calling them twice with the same input produces the same end state as calling them once, wherever the underlying operation allows it — a `set_status(id, "closed")` call is naturally idempotent; a `create_ticket(...)` call, called twice with identical arguments, is not, unless the tool is explicitly designed to deduplicate.

- **DO:** Accept an idempotency key or a natural unique identifier for "create"-style operations where duplicate creation is a realistic risk (an agent retrying after an ambiguous timeout, for instance), so a retried call with the same key updates or no-ops instead of creating a second duplicate resource.

- **DO:** Make retry-safety an explicit, stated property of the tool in its description when it's non-obvious, so the agent (and the human reviewing its tool calls) knows whether a retry after an uncertain failure is safe to issue or risks a duplicate side effect.

- **DON'T:** Leave an agent to guess whether retrying a failed or timed-out call is safe. An agent that doesn't know a `create_invoice` call is non-idempotent may retry after a timeout and silently generate two invoices for one order; either make the operation idempotent or make the non-idempotency and its risk explicit enough that the agent checks state before retrying.

### Avoiding Ambiguous Overlap Between Tools

- **DO:** Audit the full tool set for pairs or clusters of tools that could plausibly both apply to the same request, and either merge them, sharpen their descriptions to make the boundary unambiguous, or clearly rank one as preferred for the overlapping case.

- **DO:** Prefer one well-designed general tool with clear, well-documented parameters over two narrow tools whose scopes blur into each other at the edges — ambiguity at tool-selection time is often costlier than the small ergonomic loss of a slightly more general single tool.

- **DON'T:** Register two tools that both plausibly satisfy the same request with no principled way to tell which one to use — for example, both a `search_files` and a `find_files` tool with subtly different but overlapping semantics. The agent will select between them inconsistently across otherwise-identical requests, producing behavior that looks nondeterministic from the outside even though each individual tool works correctly.

- **DON'T:** Let two tools drift toward overlap over time as each is independently extended to cover "just one more case" that happens to be the other tool's job. Treat a newly proposed capability that overlaps an existing tool's scope as a signal to extend that existing tool, not to bolt on a parallel one.

### Return Value and Output Design

- **DO:** Return the minimum information the agent actually needs to proceed, in a predictable, consistently-shaped structure, rather than the full raw payload of an underlying API call. A tool that dumps an entire unfiltered API response forces the agent to spend context re-parsing and re-filtering data it didn't ask for, on every single call.

- **DO:** Keep the shape of a tool's return value consistent across calls and across success/failure branches wherever practical (same field names, same nesting, same types) so the agent can rely on a stable mental model of the response instead of re-learning the shape situationally.

- **DO:** Include enough context in a returned identifier or reference (a human-readable name alongside a raw ID, for instance) that the agent can sensibly refer back to it in conversation or in a follow-up call without a second lookup.

- **DON'T:** Return deeply nested, sparsely documented raw API responses verbatim as the tool's entire output when a flattened, purpose-built summary would serve the actual task better. Passing through raw complexity offloads the work of interpreting the response onto every future call site instead of doing it once, correctly, inside the tool.

### Tool Count and Context Budget

- **DO:** Keep the number of registered tools proportionate to what the agent actually needs for the tasks at hand, since every registered tool's name, description, and schema occupies context budget on every single turn regardless of whether it's ever called that session.

- **DO:** Group closely related, rarely-needed capabilities behind a smaller number of well-parameterized tools rather than registering a large flat list of narrow one-off tools for every minor variation, when doing so doesn't compromise the single-responsibility principle above — the two principles are in tension and the right balance depends on how genuinely distinct the operations are.

- **DON'T:** Register speculative tools "in case they're useful someday" that no current task actually exercises. Unused tools are pure context tax with no offsetting benefit, and a larger tool list also increases the odds of an ambiguous-overlap mis-selection among tools that are individually fine.

### Pagination and Large-Result Handling

- **DO:** Design any tool that can return an unbounded number of results with built-in pagination (a page size and a cursor or offset) and a sane default page size, rather than returning an entire result set unconditionally — an unbounded return can silently blow past a usable context size on exactly the query that matters most.

- **DO:** Return a clear signal that more results exist beyond the current page (a `has_more` flag, a `next_cursor`) so the agent knows whether to stop or continue rather than mistaking a truncated first page for the complete answer.

- **DON'T:** Silently truncate a large result set without indicating that truncation happened. An agent that receives a bare, quietly-cut-off list has no way to know it isn't the whole answer, and may proceed to reason or report as if it were complete.

### Read/Write Separation and Least Privilege

- **DO:** Keep read-only tools structurally incapable of causing side effects, and make that guarantee legible in the tool's name and description, so an agent (and a human skimming the available tools) can immediately tell which calls are safe to make freely for exploration and which require the more careful handling reserved for mutating calls.

- **DO:** Scope a given tool's underlying credentials or access level to the minimum needed for that tool's stated job — a tool meant only to read a calendar should not hold write access to it merely because the same API key happens to grant both.

- **DON'T:** Build a single tool that both reads and writes behind an innocuous-sounding read-like name ("get_or_create_user" silently creating a user on a lookup miss) — a name implying safety on a call that can mutate state invites careless, unreviewed use in a context where only a read was intended.

### Signaling Rate Limits and Backoff

- **DO:** Return a structured, explicit signal when a call fails due to rate limiting or quota exhaustion — including, where available, how long to wait before retrying — so the agent can back off correctly instead of hammering the same failing call in a tight loop or, worse, mistaking the failure for a different, unrelated error.

- **DON'T:** Let a rate-limit failure surface as an undifferentiated generic error indistinguishable from a validation failure. Without a distinguishing signal, an agent has no principled way to choose between "retry after a pause" and "this call is wrong and retrying won't help."

### Tool Deprecation and Versioning

- **DO:** Mark a deprecated tool as deprecated directly in its description, including what to use instead, for as long as it must remain registered for backward compatibility, so an agent selecting between an old and a new tool has the signal to prefer the replacement.

- **DO:** Version a tool's schema deliberately when its parameters change in a breaking way, and retire the old version cleanly rather than letting two incompatible shapes coexist under the same tool name across different deployments — an agent has no way to know which parameter contract is live for a given call unless the tool identity or its description reflects the current one precisely.

- **DON'T:** Silently change a tool's parameter meaning (a unit, a default, a required/optional status) between versions without a corresponding change to its name or description. An agent relying on a previously-learned calling pattern for the same tool name will produce calls that are syntactically valid but semantically wrong against the new contract.

### Testing and Dry-Running Tools Before Shipping Them

- **DO:** Exercise a new or changed tool with a range of realistic and edge-case calls — missing optional parameters, boundary values, malformed input — before it's made available to an agent in production use, the same way a function is tested before being merged.

- **DO:** Verify that a tool's error paths actually produce the structured, informative errors intended, not just that its happy path works — error-path testing is exactly the part most likely to be skipped and most likely to matter the first time a real failure occurs.

- **DON'T:** Ship a tool having only verified its documented example call. Real usage will exercise combinations the author didn't anticipate, and an untested tool's failure mode under those combinations is unknown at exactly the moment an agent is depending on it mid-task.

## Agentic Coding Workflow Anti-Patterns

### False Completion Claims

- **DO:** Actually execute the build, the test suite, and the specific check being claimed, and read its real output, before stating that tests pass, that a build succeeds, or that a task is complete. A claim of success is a factual assertion about something checkable; it should never be produced from a prediction of what the output probably would be.

- **DO:** Quote or summarize the actual command output that supports a completion claim ("ran `pnpm test`, 142 passed, 0 failed") rather than a bare assertion, so the claim is auditable and so producing a false one requires actively fabricating output rather than just omitting a step.

- **DO:** Report a failed or not-yet-run check exactly as such. "I made the change; I have not yet run the test suite" is a complete, honest, and useful status — it is not a worse answer than a false "tests pass," it is a categorically better one.

- **DON'T:** Say "tests pass" or "this works" based on the change looking correct on inspection, matching a familiar pattern, or being similar to code that worked before. Code that looks right is not the same claim as code that has been run and verified, and presenting the former as the latter is a fabrication regardless of how confident the visual inspection felt.

- **DON'T:** Report a task as done when a step was skipped, blocked, or partially completed, even if finishing it would have been easy or is likely to work. Partial completion reported as full completion removes the human's ability to notice and finish the remaining part, which is strictly worse than an honest partial-completion report.

```
# False completion — never ran anything:
"I've implemented the discount logic and all tests pass."

# Honest, verified completion:
"Implemented the discount logic. Ran `pnpm test src/pricing`: 14 passed, 0 failed.
Also ran the full suite (`pnpm test`) to check for regressions: 187 passed, 0 failed."
```

### Stub Code and Placeholder Data Presented as Finished

- **DO:** Deliver code that is fully wired up and functional for the stated task, or explicitly and visibly flag exactly which parts are stubbed and why, rather than letting a stub blend in as if it were real, working logic.

- **DO:** Use loud, greppable markers (`TODO`, `FIXME`, `NotImplementedError`, an explicit thrown error with a clear message) for any intentionally incomplete piece, so it surfaces in a code search and cannot be mistaken for finished behavior by a future reader — human or agent.

- **DO:** State any remaining stub, mock, or placeholder explicitly in the summary of the work, with enough specificity that the human knows exactly what still needs real implementation before this is usable — not just a vague "some parts may need refinement."

- **DON'T:** Leave a function returning hardcoded or mock data, an empty catch block, or a `pass`/no-op body in code that is presented as the finished deliverable, with no comment and no mention in the summary. A silent stub is functionally a lie about what the code does — it will pass a casual read and fail in the exact place a user first exercises it for real.

- **DON'T:** Fabricate plausible-looking sample data to make a demo or a test look like it's exercising a real path when it's actually hitting a stub. This actively hides the incompleteness rather than merely omitting a mention of it, and is a strictly worse failure than an honest stub — the fabricated data makes the stub harder to detect, not easier.

```python
# Silent, undisclosed stub — looks finished, isn't:
def calculate_tax(amount, region):
    return amount * 0.08  # hardcoded, ignores `region` entirely

# Disclosed and loud:
def calculate_tax(amount, region):
    # TODO(unimplemented): region-specific tax tables are not wired up yet.
    # Currently applies a flat 8% for all regions — do not ship without
    # replacing this before launch in any region other than the default.
    raise NotImplementedError("Region-specific tax rates not yet implemented")
```

### Scope Creep and Drive-By Changes

- **DO:** Limit a change to what was actually asked, and treat anything beyond that — a refactor of adjacent code, a rename, a dependency bump, a formatting pass over untouched files — as a separate, explicitly proposed follow-up rather than something bundled silently into the requested change.

- **DO:** Surface an opportunistic improvement noticed along the way as a suggestion in the summary ("while in this file, I noticed X could be cleaned up — want me to do that separately?") rather than acting on it unasked. Naming the opportunity preserves its value without imposing the cost of an unreviewed, unrequested diff.

- **DO:** Keep the diff reviewable — a reviewer should be able to map every changed line back to the stated task. A change that mixes the actual fix with unrelated drive-by edits makes the diff harder to review, harder to revert cleanly, and harder to attribute if something breaks later.

- **DON'T:** "Fix while I'm in here" — renaming variables, reformatting untouched code, upgrading an unrelated dependency, or restructuring a neighboring function that wasn't part of the request, just because the opportunity was visible during the change. Each of those is a legitimate task on its own, but bundling it into an unrelated change removes the human's ability to evaluate, scope, or decline it independently.

- **DON'T:** Expand a narrowly-scoped bug fix into a broader refactor of the surrounding system on the theory that the fix "should really" be done at a higher level of abstraction. If a genuinely better approach exists at a larger scope, say so explicitly and let the human decide whether to expand scope — don't decide unilaterally mid-task.

```
# Asked: "fix the off-by-one error in the pagination calculation"

# Scope creep — touches 6 unrelated files, renames things, reformats:
diff --stat
 src/pagination.ts     |  4 +--
 src/api/users.ts      | 40 ++++++++--------   <- unrelated rename
 src/api/orders.ts     | 38 ++++++++--------   <- unrelated rename
 src/utils/format.ts   | 12 +++---            <- unrelated reformat

# Correct scope — one file, the actual fix:
diff --stat
 src/pagination.ts | 4 +--
```

### Silently Reverting or Deleting the User's Uncommitted Work

- **DO:** Treat any uncommitted change already present in the working tree as the user's in-progress work to be preserved, unless explicitly told to discard it. Check `git status` and `git diff` for pre-existing local changes before starting, and factor them into the plan rather than overwriting or discarding them as a side effect of an unrelated action.

- **DO:** Ask before running any command that would discard uncommitted changes (`git checkout -- .`, `git reset --hard`, `git clean -f`, force-overwriting a file that already has local edits) if there's any chance uncommitted work exists that the current task didn't create.

- **DO:** Stash, branch, or otherwise explicitly preserve pre-existing uncommitted work before performing an operation that might conflict with it, and say clearly what was preserved and where, so the human can find it again.

- **DON'T:** Run a broad reset, clean, or checkout command to "get back to a known state" after hitting an error, without first checking whether doing so destroys work that isn't the agent's own and isn't yet committed anywhere. This is one of the most damaging failure modes precisely because it's silent and often irreversible — the user discovers the loss only later, with no warning it happened.

- **DON'T:** Overwrite a file with uncommitted local edits just because a generated version needs to go there, without first checking whether those edits exist and are wanted. A file's on-disk content should never be treated as disposable scratch space by default.

### Overusing Destructive Escape Hatches

- **DO:** Diagnose the actual root cause of a failing test, a rejected commit, or a blocked push, and fix that cause, treating any bypass flag as a last resort reserved for situations where the human has explicitly authorized skipping the check.

- **DO:** Read the actual error a hook, a linter, or a check produced before reaching for any flag that suppresses it. Most "blocking" failures are pointing at a real problem; understanding what it's pointing at is usually faster than working around it, and is the only path that doesn't leave the underlying issue live in the codebase.

- **DO:** Explain, if a bypass is genuinely warranted and authorized, exactly what check was skipped and why, so the human retains visibility into what protection was turned off and can restore it deliberately later.

- **DON'T:** Reach for `--force`, `--no-verify`, `git reset --hard`, `git push --force`, or similar irreversible or check-bypassing flags as a default way to get past a failure, error, or rejected operation. These exist for narrow, deliberate use, not as a general "make the red thing go away" button, and normalizing their use as a workaround erodes exactly the safety net they were installed to provide.

- **DON'T:** Bypass a pre-commit hook, a required status check, or a branch protection rule without explicit human authorization for that specific instance, even under time pressure or when the check seems obviously wrong. If the check really is wrong, the correct fix is to fix or disable the check itself, deliberately and visibly — not to route around it silently on one commit while leaving it armed (and now proven bypassable) for everyone else.

```
# Escape-hatch reflex:
$ git commit -m "fix" --no-verify   # pre-commit hook was failing, so... skip it?

# Root-cause diagnosis:
$ git commit -m "fix"
  pre-commit: eslint failed — 3 errors in src/pricing.ts
$ pnpm lint src/pricing.ts   # read the actual errors
  3:1  error  'discount' is defined but never used
$ # fix the real issue, then commit normally
```

### Fabricating Citations, Links, APIs, Flags, or Config Options

- **DO:** State an API signature, a CLI flag, a config option, or a library behavior only when it has been verified against real documentation, the actual installed version's source, or a real successful invocation — not from a general impression of how a similar tool "usually" works.

- **DO:** Say explicitly when something is uncertain — "I believe this flag exists but haven't verified it against this exact version; worth checking `--help` before relying on it" — rather than stating a guess with the same confident phrasing used for a verified fact.

- **DO:** Prefer checking the actual installed version's help output, type definitions, or source over recalling the API from memory, whenever the tool to do so cheaply is available (`--help`, a `.d.ts` file, an actually-installed package's source) — verifying is usually faster than the eventual cost of debugging a hallucinated call.

- **DON'T:** Invent a plausible-sounding function name, parameter, CLI flag, or config key because it fits the pattern of similar real ones, and present it as if it were verified. A hallucinated API call is often syntactically and stylistically indistinguishable from a real one until it's actually run, which makes it a particularly costly kind of error — it passes review by eye and fails only at runtime, or worse, fails silently.

- **DON'T:** Fabricate a citation, a URL, a paper title, or a quoted source to support a claim. An invented but plausible-looking reference is worse than no reference, because it appears to be verifiable and, once trusted, actively misleads anyone who builds on the claim without independently re-checking it.

### Comment Noise vs. Signal

- **DO:** Write comments that explain the *why* behind a non-obvious decision — a business rule, a workaround for a specific bug, a reason a simpler-looking approach was rejected — information that isn't recoverable just by reading the code itself.

- **DO:** Let well-named functions, variables, and clear structure carry the *what* and *how* on their own, reserving comments for the parts that genuinely need a explanation beyond what the code already states.

- **DON'T:** Narrate obvious code line by line ("// increment i by 1", "// loop through the list", "// return the result"). This kind of comment adds visual noise without adding information, and its presence throughout a file trains readers to skip all comments, including the rare ones that actually matter.

- **DON'T:** Pad a diff with a high density of comments as a way of making a small or uncertain change look more thorough or more carefully considered than it is. Comment density is not a proxy for code quality, and reviewers who notice the pattern discount the comments (and the effort they imply) accordingly.

```python
# Noise — restates what the code already says:
# create an empty list
items = []
# loop over the input
for x in raw_input:
    # append x squared to items
    items.append(x ** 2)

# Signal — explains something the code alone can't:
# Squaring here (not abs()) intentionally amplifies outliers before the
# threshold check below — see INC-4021 for why abs() under-flagged fraud.
items = [x ** 2 for x in raw_input]
```

### Preamble/Postamble Padding

- **DO:** Open directly with either the action being taken or the answer being given, reserving any framing for cases where context genuinely changes how the response should be read.

- **DO:** End a response when the actual work or answer is stated, rather than appending a restatement of what was just said, a generic offer of further help, or a summary that adds no information beyond what preceded it.

- **DON'T:** Preface routine work with throat-clearing ("Great question! Let's dive into this. I'll start by...") or a restated version of the request before actually doing anything. This delays the actually useful content and adds nothing a direct start wouldn't already convey.

- **DON'T:** Close out with generic filler ("I hope this helps! Let me know if you have any other questions!") after already delivering the substantive answer. Padding at the end costs the reader's attention for zero added value, and in an agentic context it costs tokens on every single turn across a long session.

### Miscalibrated Clarifying Questions

- **DO:** Proceed directly, using reasonable defaults and stating the assumption made, for decisions that are low-stakes, easily reversible, or have an obviously correct answer given context already available — the exact button-copy wording, a minor naming choice, a formatting detail with an established convention already visible in the codebase.

- **DO:** Stop and ask before proceeding on decisions that are high-stakes, hard to reverse, or genuinely ambiguous given the information available — a choice that affects data integrity, a choice that's expensive to undo, a requirement that could reasonably be interpreted two structurally different ways with very different amounts of work behind each.

- **DO:** When asking, ask a specific, answerable question with the actual options laid out, rather than an open-ended "what do you want me to do?" that pushes the entire thinking burden back onto the human. A good clarifying question already shows that real thought went into narrowing the space of plausible answers.

- **DON'T:** Interrupt for confirmation on trivial, obviously-inferable decisions — the exact wording of a log message, which of two equally valid variable names to use, whether to add a blank line — when a reasonable default clearly exists and stating the assumption after the fact costs less than stopping to ask before.

- **DON'T:** Barrel ahead silently on a genuinely ambiguous, high-stakes decision — which of two incompatible database schema migrations to run, whether a destructive cleanup should include a specific directory, how to interpret a spec that contradicts itself — guessing one interpretation and proceeding as if it were the only reasonable one. The calibration failure runs in exactly the wrong direction when trivial choices get interrupted and consequential ones get silently guessed.

### Sycophancy and Over-Apologizing

- **DO:** Give a direct, technically grounded assessment of a proposed approach, a piece of code, or an idea, including a plainly stated disagreement or a stated tradeoff, when that's the honest technical read — regardless of how the request was phrased or how confident the human sounded about their own approach.

- **DO:** Acknowledge a genuine mistake once, clearly, and then move straight to the fix, treating the acknowledgment as informational rather than as an emotional performance requiring escalating apology.

- **DON'T:** Praise a mediocre or flawed idea as good, or preface disagreement with excessive hedging and validation, to soften the interaction. Agreement that isn't earned by the actual merits of the proposal is a disservice — it optimizes for the human feeling good about the exchange over the human getting a technically accurate signal to act on.

- **DON'T:** Apologize repeatedly or with escalating intensity for a single mistake, or apologize reflexively for things that aren't actually errors (asking a clarifying question, taking a reasonable amount of time, correctly pointing out a constraint the human didn't like). Repeated apology adds no corrective value beyond the first acknowledgment and shifts focus away from the fix.

```
# Sycophantic:
"That's a great approach! Though I might gently suggest, if you don't mind me
mentioning it, that there could possibly be a tiny edge case here, but your
call either way — you know best!"

# Direct:
"This will break on empty input — `items[0]` throws when the list is empty.
Add a length check before indexing, or use `items[0] if items else None`."
```

### Not Verifying Work Before Declaring Success

- **DO:** Run the relevant tests, execute the changed code path, and read the actual diff end to end before declaring a task finished — verification is a required step of the task, not an optional nicety appended if time allows.

- **DO:** Check the output of a change against the actual request, not just against "did the code run without throwing" — a script that executes cleanly but produces the wrong numbers has not been verified just because it didn't crash.

- **DO:** Re-read the full diff before presenting it, specifically looking for exactly the failure modes covered in this section — a leftover stub, an unrelated drive-by change, a debug print left in, a hardcoded test value that should have been a parameter.

- **DON'T:** Treat "the code compiles" or "the code looks right" as equivalent to "the code has been verified to do what was asked." Compilation and visual plausibility are necessary, weak signals, not a substitute for actually exercising the behavior.

- **DON'T:** Skip verification because the change looks small or the task looks routine. Small changes fail in small, easy-to-miss ways precisely because they don't trigger the same scrutiny a large change would — an off-by-one, a swapped argument order, an inverted boolean are exactly the kind of thing that survives a glance but fails a real run.

### Architecture Astronaut Behavior

- **DO:** Match the solution's complexity to the actual, current requirements of the task, choosing the simplest design that correctly solves the stated problem and is easy to extend later if a real need arises.

- **DO:** Add abstraction — an interface, a plugin system, a configuration layer, a generic framework — only in response to a concrete, currently-known need for the flexibility it provides, not in anticipation of a hypothetical future one.

- **DON'T:** Introduce a generic plugin architecture, a configurable strategy pattern, or a new abstraction layer for a task that has exactly one required implementation and no stated need for a second one. Speculative extensibility built for a future that may never arrive adds real, immediate cost — more code to read, more indirection to trace, more surface for bugs — in exchange for a benefit that may never be collected.

- **DON'T:** Rewrite or restructure a working, simple piece of code into a "more proper" or "more scalable" architecture as an unrequested side effect of a small feature addition. If the existing structure is genuinely inadequate for the requested change, say so and propose the restructuring as its own explicit, scoped step — don't fold an architectural opinion into a task that didn't ask for one.

```
# Asked: "add a function to export the report as CSV"

# Architecture astronaut — nobody asked for a plugin system:
class ExportStrategy(ABC): ...
class CsvExportStrategy(ExportStrategy): ...
class ExportStrategyFactory:
    def create(self, format: ExportFormat) -> ExportStrategy: ...
# ...200 lines of scaffolding for a format nobody has asked to add yet

# Right-sized:
def export_report_csv(report: Report) -> str:
    ...
```

### Silently Downgrading or Removing a Requested Feature

- **DO:** Implement the feature as actually requested, and if a genuine blocker makes that infeasible within reasonable effort, say exactly what the blocker is and what the options are — implement it a different way, implement a reduced version explicitly labeled as such, or flag that it needs more time or a different approach — rather than quietly shipping less than what was asked.

- **DO:** Treat "this part is hard" as information to surface, not a reason to route around the requirement. The human asked for the hard part because they need the hard part; a silently simplified substitute may satisfy the letter of the request while missing the actual reason it was asked for.

- **DON'T:** Drop a requirement, replace it with an easier approximation, or silently narrow its scope because the full version was difficult, time-consuming, or hit an unexpected obstacle, without flagging that this happened. A summary that describes the reduced version as if it were the full request is a misrepresentation of what was delivered, even if every individual sentence in it is technically true.

- **DON'T:** Let a feature quietly regress to a simpler fallback path (a client-side-only validation instead of the requested server-side enforcement, a synchronous poll instead of the requested real-time push, an approximate calculation instead of the requested exact one) without calling out that the fallback was used and why, so the human can decide whether the gap actually matters for their use case.

```
# Asked: "add real-time push notifications when a task status changes"

# Silent downgrade — ships polling, calls it done, doesn't mention the gap:
"Implemented notifications — the UI now updates when a task's status changes."
(actually: polls every 30s, was described in the summary as if it were push)

# Honest surfacing of the blocker:
"True real-time push would need a WebSocket or SSE channel, which this stack
doesn't currently have. I implemented 30-second polling as an interim version —
it works but isn't instant. Want me to look into adding a push channel, or is
polling acceptable for now?"
```

### Ignoring Existing Conventions in Favor of the Agent's Own Defaults

- **DO:** Read a handful of representative, similar existing files before writing new code, and match their patterns — naming style, error handling shape, import ordering, how tests are structured — even where a different pattern is more familiar from training data in general.

- **DO:** Treat an established local convention as authoritative over a generic best practice whenever the two conflict. A codebase's own consistency is worth more than any individual file's adherence to an abstractly "better" pattern the rest of the codebase doesn't share.

- **DON'T:** Introduce a different state-management pattern, error-handling style, or file-organization scheme in one new file or module just because it's the agent's own statistically preferred default, leaving the codebase with two competing conventions side by side. This is a subtler form of the scope-creep problem — it doesn't touch unrelated files, but it still imposes an unrequested stylistic choice, and it compounds every time it happens because the next look-alike file now has two precedents to choose between.

### Reinventing Utilities That Already Exist

- **DO:** Search the codebase for an existing helper, utility function, or shared module before writing a new implementation of common logic — date formatting, a debounce helper, a validation function, an HTTP client wrapper — since duplicated logic is a maintenance and consistency cost even when each copy is individually correct.

- **DON'T:** Write a fresh implementation of something the codebase already solves elsewhere, purely because it was faster than searching for the existing one. A second, slightly different date-formatting function is a bug waiting to happen the day the two diverge on an edge case (a timezone, a locale) and callers get inconsistent behavior depending on which one they happened to import.

### Losing Track of the Original Task Over a Long Session

- **DO:** Periodically restate, at least to oneself, what the original request actually was and check current progress against it, particularly after a long detour through debugging, exploration, or an unexpected complication — long sessions are exactly where drift from the original ask accumulates unnoticed.

- **DO:** Return explicitly to the stated goal after resolving a tangent, and note in the summary if the tangent changed the scope of what was delivered relative to the original ask.

- **DON'T:** Let an extended debugging detour or an interesting side-discovery quietly become the de facto task, such that what ships addresses the detour but only partially — or not at all — the thing that was originally asked for. A session that ends with "I fixed the interesting bug I found" instead of "I did the thing you asked for" has failed regardless of how good the interesting fix was.

### Leaving Debug Artifacts and Generated Files Behind

- **DO:** Remove temporary debug output — stray `print`/`console.log` statements, temporary test files, scratch scripts used to explore the problem — before presenting a change as finished, and check the diff specifically for these before submitting it.

- **DO:** Respect the project's `.gitignore` and existing conventions for build artifacts, caches, and generated files; never add a generated file to version control just because it happened to be created locally during the session.

- **DON'T:** Ship a diff that includes leftover debugging instrumentation, a commented-out block of the previous failed attempt, or an accidentally-staged build artifact. Each of these is noise a reviewer has to notice and ask about, and their presence is itself a signal that the change wasn't given a final read-through before being presented.

### Assuming Success of Asynchronous or Fire-and-Forget Operations

- **DO:** Check the actual outcome of an operation that could fail asynchronously — a background job enqueued, an email queued for delivery, a webhook fired — rather than treating "the call to start it didn't throw" as equivalent to "it definitely completed successfully."

- **DON'T:** Report a task as complete based only on successfully *initiating* an operation whose actual completion or failure happens later or elsewhere, without checking (where checkable) that it actually landed. "The job was enqueued" and "the job ran successfully" are different claims, and only the first is verifiable at the moment of enqueuing.

### Suppressing Flaky Tests Instead of Fixing Them

- **DO:** Investigate a genuinely flaky or intermittently failing test to find its root cause — a race condition, shared mutable test state, an unmocked source of nondeterminism — and fix that cause, treating flakiness as a real bug in the test or the code under test rather than a nuisance to route around.

- **DON'T:** Silence a flaky test by adding a retry wrapper, increasing a timeout blindly, or marking it skipped, as a way to make a red CI run turn green without understanding why it was red. This is a specialized case of the destructive-escape-hatch anti-pattern — it makes the symptom disappear while leaving the underlying nondeterminism live, ready to resurface as a real production bug or a harder-to-diagnose failure later.

### Editing Before Reading Enough Context

- **DO:** Read the surrounding function, the file's imports, and at least one caller or one sibling implementation before changing code, so a change is grounded in how the code is actually used, not just how it looks in isolation.

- **DO:** Search for other call sites of a function before changing its signature or behavior, and check each one is still correct under the new behavior, rather than assuming a change is self-contained because the diff is small.

- **DON'T:** Edit a function based on a narrow, single-file view when its behavior is depended on elsewhere in ways that a quick search would have revealed. A change that's locally correct but breaks an unexamined caller is a failure of insufficient context-gathering, not bad luck — and it is a failure the agent had the means to prevent by reading one more file before editing.

### Editing Outside the Stated Scope of Permission

- **DO:** Stay within whatever scope was explicitly granted for the current task — a named file, a named directory, a stated set of changes — and treat any need to go beyond it as a reason to check in, not a reason to quietly proceed because it seemed helpful.

- **DON'T:** Touch a file, a system, or a resource that falls under an "ask first" or "never" tier of the permission model while pursuing an in-scope task, on the reasoning that it was necessary to get the actual task done. If a permitted task turns out to require a restricted action, that's exactly the situation the permission tiers exist to catch — surface it rather than working around it silently.

### Reintroducing Bugs That Were Already Fixed

- **DO:** Check the recent history of a function or file (a quick `git log` / `git blame` on the area being touched) before reintroducing a pattern that looks locally reasonable, when there's a realistic chance a past fix specifically addressed a bug in that exact area.

- **DON'T:** Revert an intentional fix by editing the same code back toward its previous, buggy shape — for example, removing a null check that was added to fix a specific crash, because the check "looks unnecessary" from a narrow read of the current code without its history. If a defensive check seems redundant, verify why it's there before removing it rather than assuming it was left over by accident.

## Writing Good Instructions for Agents (for humans directing them)

### Specify Constraints and Non-Negotiables Up Front

- **DO:** State the hard constraints — technology choices that are fixed, files or systems that are off-limits, performance or compatibility requirements, deadlines that affect scope — at the start of the request, before describing the desired outcome. An agent that learns a hard constraint only after building toward a solution that violates it has to redo real work, which is strictly more expensive than stating the constraint once at the outset.

- **DO:** Distinguish explicitly between a hard requirement and a soft preference in the same request ("must use the existing auth middleware; would prefer TypeScript but plain JS is fine if it's simpler"), since an agent given an undifferentiated list of desires has no way to know which ones it may trade off against each other under pressure.

- **DO:** Mention known constraints that aren't obvious from the codebase alone — a regulatory requirement, an unstated backward-compatibility need, a downstream consumer that would break — since these are exactly the kind of fact an agent cannot discover by reading the code and has no way to account for unless told.

- **DON'T:** Discover and communicate a hard constraint only after reviewing a first attempt that violates it. If the constraint was known in advance, stating it upfront costs one sentence and saves an entire redo cycle.

- **DON'T:** Bury a genuinely load-bearing constraint in the middle of a long, otherwise low-priority paragraph where it's easy to skim past. State non-negotiables as a short, distinct list, the same way a well-formed instruction file separates its permission tiers from its general prose.

```
# Constraint discovered too late, after a full implementation:
"Oh — I should have mentioned, this can't use any new npm dependencies,
company policy. Can you redo it without the library you just added?"

# Constraint stated up front:
"Add CSV export to the report page. Hard constraint: no new npm dependencies
(security review backlog is 6 weeks) — build it with what's already installed.
Prefer keeping it in the existing `exporters/` module if that fits."
```

### Provide Examples of Desired Output

- **DO:** Include a concrete example of the desired output — a sample of the target format, a similar piece of existing code to match the style of, a mock of the expected result — whenever the request has a specific shape in mind that would otherwise have to be inferred from a description alone.

- **DO:** Point at a real, existing instance in the codebase or the project when one already embodies the desired pattern ("format it like the existing report in `reports/monthly.ts`") rather than describing that pattern in the abstract — this is the same "one real snippet beats three paragraphs" principle applied to a single request, not just to a standing instruction file.

- **DO:** Show a counter-example alongside the desired one when there's a specific wrong-but-plausible interpretation to rule out — "like the summary in `X`, not like the more verbose one in `Y`" resolves an ambiguity that "make it a good summary" leaves entirely open.

- **DON'T:** Describe a target format purely in adjectives ("make the output look professional," "format it nicely") when a one-paragraph example or a pointer to an existing instance would remove the ambiguity entirely and produce a first attempt far closer to what's actually wanted.

### State the Definition of "Done" Explicitly

- **DO:** State what condition marks the task as complete — which tests need to pass, which specific behavior needs to be demonstrated, which edge cases need to be handled, whether a specific manual check or specific stakeholder sign-off is part of "done" — so both the human and the agent are checking the work against the same bar.

- **DO:** Include the negative space of "done" when it matters — what is explicitly out of scope for this request, so it isn't quietly bundled in (scope creep) or, in the opposite failure, silently assumed to be someone else's problem when it was actually expected.

- **DO:** Make "done" verifiable by something concrete wherever possible (a specific command that should pass, a specific scenario that should work end to end) rather than a subjective judgment call, since a verifiable bar is something the agent can actually check itself against before declaring success.

- **DON'T:** Leave "done" implicit and trust that a vague request and a vague sense of completion will happen to converge. Divergent definitions of done are a common source of a delivered task that technically satisfies the literal words of the request while missing what was actually needed.

```
# Implicit, subjective "done":
"Add validation to the signup form."

# Explicit, verifiable "done":
"Add validation to the signup form: email must be a valid format, password
must be 8+ characters with at least one number, and both fields show an
inline error on blur if invalid. Done = these three cases plus the existing
happy-path test in signup.test.ts all pass, and submit is disabled while
either field is invalid."
```

### Prefer Iterative Small Requests Over One Giant Ambiguous One

- **DO:** Break a large, multi-part, or exploratory piece of work into a sequence of smaller requests, each reviewed before the next is issued, whenever the full scope is complex or the requirements are still being worked out. Small steps surface a wrong turn after one unit of wasted work instead of after the whole thing is built the wrong way.

- **DO:** Use an early, small, cheap step to resolve genuine uncertainty (a rough draft, a spike, a single representative case handled end to end) before committing to a large request that assumes a particular direction is correct — this is the same principle as verifying a low-cost assumption before betting a large amount of work on it being true.

- **DO:** Reserve a single large, comprehensive request for well-understood, mechanical, low-ambiguity work — a broad rename, a repetitive migration following an already-agreed pattern — where the risk of the large batch going in the wrong direction is genuinely low.

- **DON'T:** Hand over one large, ambiguous request covering an entire feature, an entire refactor, or an entire new system in one shot when the requirements are still fuzzy or likely to be refined through seeing a first attempt. The larger and more ambiguous the single request, the more work is at risk of being built on a misunderstanding that a smaller checkpoint would have caught early.

- **DON'T:** Treat "give a big request so the agent has full context" and "give a small request so mistakes are caught early" as the same strategy. Full context can be communicated up front (through a good instruction file, background documents, or a clearly stated overall goal) while the actual units of work requested and reviewed still stay small — the two concerns are independent, and conflating them is a common reason people over-scope a first request.

### Stating Priority When Goals Conflict

- **DO:** State explicitly which goal wins when two stated goals are in tension — speed versus thoroughness, minimal diff versus best long-term design, shipping today versus getting a second opinion — since an agent left to resolve that tension on its own will pick a default that may not match what's actually wanted for this particular request.

- **DO:** Rank multiple simultaneous asks by priority when not all of them can be fully satisfied within reasonable effort, so the agent knows what to protect first if a tradeoff becomes necessary partway through.

- **DON'T:** Hand over a request with two silently competing goals ("make this fast, and also make it fully backward compatible, and also keep the diff small") with no indication of which matters most, then treat whichever one the agent happened to prioritize as a wrong guess. If the priority genuinely matters, it has to be stated — it cannot be inferred correctly from a flat, unordered list of desires.

### Communicating Real Urgency and Stakes

- **DO:** Say plainly when a task is genuinely high-stakes — touches production data, affects billing, is going out in the next release, has compliance implications — so the level of care, the amount of double-checking, and the threshold for asking a clarifying question before proceeding are calibrated to the actual consequences of a mistake.

- **DO:** Say plainly, just as usefully, when a task is low-stakes and exploratory — a throwaway prototype, a proof of concept that will be discarded — so time isn't spent on production-grade polish, extensive testing, or cautious incremental steps that the task doesn't warrant.

- **DON'T:** Let every request default to an ambiguous middle level of implied urgency and stakes, forcing a guess about how much caution is warranted. Miscalibrated caution in either direction has a real cost: too much care on a throwaway prototype wastes time, too little on a production billing change risks real damage.

### Giving Feedback That Improves the Next Iteration

- **DO:** Give specific, actionable feedback on a completed piece of work — what was wrong, ideally why it was wrong, and what the corrected version should look like — rather than a vague rejection that leaves the actual gap unspecified.

- **DO:** Point at the specific line, function, or behavior that needs to change when giving corrective feedback, the same way a good code review comment anchors to a specific diff line rather than commenting generically on the whole PR.

- **DON'T:** Respond to unsatisfactory work with only "this isn't right, try again" and no further detail. Vague rejection feedback produces another guess with the same odds of missing the actual issue as the first attempt, and wastes an iteration that specific feedback would have resolved directly.

### Providing Negative Examples Alongside Positive Ones

- **DO:** Show what *not* to do, concretely, whenever there's a plausible wrong interpretation that a purely positive description wouldn't rule out — a specific pattern the codebase deliberately avoids, a naive-but-wrong approach that's tempting for this exact problem, a previous attempt that didn't work and why.

- **DON'T:** Describe only the desired outcome and assume the space of wrong interpretations is obvious. Often the single most useful piece of information is exactly the wrong turn most likely to be taken — naming it explicitly closes off far more ambiguity than another sentence describing the right answer in different words would.

## Quick Checklist
- Pin exact language, framework, and package-manager versions in the instruction file — never leave the agent to assume "whatever's common."
- Put install/build/test/lint commands, with real flags, in the first screen of the instruction file.
- Give a fast local test loop and a slower CI-equivalent one, and say which to use when.
- Show one real code snippet of house style instead of a paragraph describing it.
- State testing, mocking, and determinism rules explicitly — don't rely on the agent inferring them from sampled test files.
- Define a three-tier permission model: always OK, ask first, never — as a scannable list, not prose.
- Explicitly flag any non-standard or internal tooling the agent wouldn't recognize from training data alone.
- Omit anything the agent can cheaply discover by reading the code itself.
- Keep the instruction file short enough to actually be read and followed in full, every time.
- Write for machine parsing: short headers, itemized lists, consistent terminology, no narrative onboarding voice.
- Point at `.env.example` and a secrets manager for credentials — never paste a real secret into the file.
- State branch naming, commit message format, and PR requirements if the project enforces them.
- Define project-specific jargon (overloaded domain terms) in a short glossary if getting them wrong misdirects code.
- Record the handful of real, repeated gotchas and pitfalls as explicit rules, not a general debugging tutorial.
- State what CI actually checks and how it differs from local runs, so local "it passes" claims are calibrated correctly.
- Point at the linter/formatter config as the source of truth for style rules instead of restating them in prose.
- Never commit an LLM-generated instruction file without a human curating and fact-checking it first.
- Update the instruction file in the same PR as the change it describes; audit it periodically for drift.
- Read the whole instruction file for internal contradictions before adding to it.
- Replace vague quality adjectives ("clean," "robust," "best practices") with concrete, checkable rules.
- In monorepos, scope nested instruction files to what's different in that subtree; don't duplicate the root file.
- Match instruction filenames and formats exactly to what each agent tool actually expects to find.
- Give each skill a literal name and a description that states exactly when to trigger it, not just what it does.
- Keep every skill scoped to one coherent capability; split a grab-bag skill into separate ones.
- Calibrate trigger specificity: not so broad it fires on everything, not so narrow it only matches one past phrasing.
- Version a skill and record why whenever the process it encodes changes.
- Embed a concrete example or template inside a skill rather than relying on prose description alone.
- Don't build a skill for something the base model already reliably does well unprompted.
- Test a skill against real example requests, including near-misses that shouldn't trigger it, before shipping.
- Bundle a skill's templates, references, and scripts alongside it rather than depending on files that could move.
- Scope a skill's tool access and permissions to only what its procedure actually needs.
- Give unattended/background skills stricter defaults and explicit checkpoints, since no human is watching each step.
- Write skill trigger descriptions from the outside (how a user would ask), not from the inside (internal jargon).
- Give each tool a single clear responsibility; split overloaded multi-mode tools into separate ones.
- Name tools and parameters descriptively and consistently — agents pattern-match on names.
- Write tool descriptions that state when to use the tool, not only what it mechanically does.
- Document units, formats, and constraints precisely in tool parameter schemas.
- Return structured, specific errors that distinguish permission/validation/transient failures — never a silent empty result.
- Report partial success explicitly; never collapse it into a bare success or bare failure.
- Require explicit confirmation or a flag for irreversible actions; make the safe path the easy path.
- Prefer soft-delete/recoverable states over true permanent deletion as the default.
- Design "create" operations to be idempotent, or accept an idempotency key, wherever retries are plausible.
- Audit the tool set for ambiguous overlap between tools and resolve it — merge, sharpen boundaries, or rank.
- Keep tool return values minimal, consistently shaped, and purpose-built rather than raw API dumps.
- Keep the registered tool count proportionate to actual need — unused tools are pure context tax.
- Paginate any tool that can return unbounded results, and signal clearly when more results exist.
- Keep read-only tools structurally side-effect-free and legible as safe by name, separate from mutating ones.
- Return a distinct, explicit signal for rate-limit/quota failures so retries can back off correctly.
- Mark deprecated tools as deprecated in their description, naming the replacement, until they're retired.
- Test a tool's error paths and edge cases, not just its documented happy-path example, before shipping it.
- Never claim tests pass, a build succeeds, or a task is complete without having actually run the check.
- Never leave an undisclosed stub, mock data, or placeholder in code presented as finished.
- Never make unrelated drive-by changes beyond the scope of what was asked — propose them separately instead.
- Never discard or overwrite a user's uncommitted work without checking for it and asking first.
- Diagnose root causes instead of reaching for `--force`, `--no-verify`, or a hard reset to get past a failure.
- Never fabricate an API, a CLI flag, a config option, or a citation — verify or say it's unverified.
- Write comments that explain non-obvious "why," not comments that narrate obvious "what."
- Skip preamble and postamble padding — open with the work, end when the work is stated.
- Ask before proceeding on genuinely ambiguous, high-stakes decisions; don't ask about trivial, obvious ones.
- Give direct, honest technical assessments instead of sycophantic praise or excessive apology.
- Verify work — run it, read the diff, check the actual output — before declaring success.
- Match solution complexity to the actual current requirement; don't build speculative abstraction nobody asked for.
- Never silently downgrade or drop a requested feature because it was hard — surface the difficulty and the tradeoff.
- Match existing codebase conventions over the agent's own default style, even when the default is generically fine.
- Search for an existing utility before writing a new implementation of common logic.
- Periodically re-check progress against the original request, especially after a long debugging detour.
- Strip stray debug prints, scratch files, and commented-out dead attempts from the diff before presenting it.
- Verify the actual outcome of async or fire-and-forget operations instead of assuming the call succeeded.
- Fix the root cause of a flaky test instead of adding retries, longer timeouts, or a skip to silence it.
- Read surrounding code and check other call sites before changing a function's behavior or signature.
- Stay within the granted scope of a task; surface a needed out-of-scope action instead of quietly taking it.
- Check history before removing a defensive-looking check — it may be a past fix, not leftover clutter.
- As a human directing an agent, state hard constraints and non-negotiables before work starts, not after a first attempt.
- Provide a concrete example or a pointer to an existing pattern instead of describing desired output only in adjectives.
- State an explicit, verifiable definition of "done," including what's out of scope.
- Break large, ambiguous work into small reviewed steps rather than one big speculative request.
- State which goal wins when two requested goals conflict (speed vs. thoroughness, minimal diff vs. best design).
- Communicate real urgency and stakes so caution is calibrated to actual consequences, not guessed at.
- Give specific, anchored feedback on unsatisfactory work instead of a vague "try again."
- Show a concrete negative example when a wrong-but-plausible interpretation needs to be explicitly ruled out.
