# Master Quick-Reference Checklist

A single scannable pass-through of every part's closing checklist, in document order, for a fast pre-ship review without paging through the full text. This part adds no new rules — every item here is drawn directly from its part above; treat this as the condensed companion, not a replacement for the reasoning and examples in the full sections.

## Part 2 — Universal Code-Quality Principles

- Names reveal intent without needing an explanatory comment alongside them.
- Names are searchable, pronounceable, and free of cryptic project-only abbreviations.
- One consistent casing convention per identifier category, applied everywhere.
- No type or implementation detail baked into a name that will go stale when the implementation changes.
- Booleans read as yes/no questions (`is`/`has`/`can`/`should`) and avoid negative phrasing.
- A rename touches every place the old name appeared — declaration, comments, logs, siblings.
- No name promises behavior (e.g., "read-only") that the code doesn't actually deliver.
- Quantities that could plausibly be in more than one unit (time, size, money) encode the unit in the name or use a dedicated type.
- Every function has a single, nameable responsibility and reads as one clear thing.
- Side effects are obvious from a function's name and are never hidden inside something that looks like a pure read.
- Function arguments stay few; related parameters are grouped into a single meaningful structure.
- No boolean flag parameter silently switches a function's internal behavior.
- Commands (state changes) and queries (reads) are kept separate, not mixed in one function.
- Required call ordering is enforced by the API's shape, not left to documentation or convention.
- Comments explain why, never restate what the code already says.
- No commented-out dead code lingers — version control already remembers it.
- Every `TODO`/`FIXME` carries enough context (or a ticket link) to be actionable later.
- A stale or now-incorrect comment is fixed in the same change that invalidates it.
- Confusing code gets renamed or refactored before it gets a comment explaining the confusion.
- Commit messages and PR descriptions explain why a change was made, not just what changed.
- READMEs and API docs are updated in the same change as the behavior they describe.
- Errors fail fast and loudly for programming errors and violated invariants.
- No `catch` block (or equivalent) silently swallows an error with no logging, handling, or re-throw.
- Recoverable, expected failures and unrecoverable, exceptional failures are modeled and handled differently.
- Every error message carries enough context (identifiers, inputs, operation) to debug without reproducing it.
- Wrapped errors preserve the original error instead of discarding it.
- Exceptions (or equivalents) are reserved for exceptional conditions, not used for routine, expected branching.
- Every call that waits on something outside the process (network, lock, queue) has an explicit, reasoned timeout.
- `try`/catch blocks stay narrow and catch only the specific error type actually expected, not a broad catch-all.
- No speculative abstraction is built for a need that doesn't concretely exist yet (YAGNI).
- Shared abstractions are extracted after a third genuine repetition, not the second.
- No interface, factory, or config layer exists for something that has exactly one implementation and no real second one coming.
- An abstraction stretched with special-case flags to fit a new caller is a candidate to split back apart, not to keep bending.
- Deeply nested conditionals are flattened with guard clauses and early returns.
- Complex boolean conditions are extracted into a well-named predicate.
- Mutable state is scoped as narrowly as possible and shared only when there's a clear, deliberate reason.
- No component silently mutates a shared object as its way of communicating with another component.
- Dependencies between components are explicit (parameters, constructors, defined events), not implicit ordering or shared globals.
- Values are treated as immutable by default where the language and performance situation reasonably allow it.
- Global/singleton state is avoided in favor of values passed explicitly through the call chain.
- Business logic is written as pure functions wherever feasible; side effects live at the system's edges.
- Acquired resources (files, connections, locks) are released on every exit path, including error paths.
- A new dependency is justified by real complexity or risk, not by saving a few lines of easy code.
- Dependencies are checked for maintenance activity, license compatibility, and security history before adoption.
- The dependency tree is periodically audited for unused, redundant, or abandoned packages.
- Code review focuses on correctness, readability, and maintainability, not just style nitpicks a linter should catch.
- Review feedback is specific and actionable, and marks clearly whether it's blocking or a suggestion.
- Diffs stay small and scoped to one logical change so a reviewer can actually reason about all of it.
- Unrelated changes (refactor, fix, dependency bump) are never bundled into the same diff.
- A reviewer only approves a change they actually understood — no rubber-stamping.
- Refactors proceed in small, independently verifiable steps, checked against tests after each one.
- A refactor and a behavior change are never mixed into the same commit or pull request.
- Tests covering current behavior exist (or are written first) before a non-trivial refactor begins.
- Small, low-risk cleanups are made in passing (boy-scout rule); large rewrites are proposed and planned separately, not slipped into an unrelated change.
- Existing codebase conventions are followed even when they differ from personal preference; disagreements go through team discussion, not unilateral deviation.
- Each recurring kind of problem has one established solution in the codebase, not several competing ones left to accumulate.
- The codebase uses one consistent representation for "no value" for a given kind of data, not a mix of null/undefined/sentinels.
- Deliberately taken-on technical debt is documented with its reasoning and a reference, not left as tribal knowledge.
- Debt is periodically revisited and consciously paid down or re-justified, never left open indefinitely by default.
- "Good enough for now" shortcuts are recognized as carrying an ongoing cost, not treated as free.


## Part 3 — AI-Generated Code Slop: The Core Anti-Patterns

- Never state a package, function, method, or config option exists without genuine grounds for confidence.
- Never claim "tests pass" without having actually run them.
- Never claim a bug is fixed without a mechanism connecting the change to the symptom.
- Flag uncertainty explicitly instead of presenting a guess with unwarranted confidence.
- Never fabricate a citation, link, issue number, or benchmark result.
- Never leave a TODO, stub, or fake/mock data in code presented as finished, without saying so.
- Label illustrative/example output clearly as not-actually-run when it wasn't.
- Comments explain *why*, not *what* — delete comments that just restate the next line.
- Delete commented-out code instead of leaving it "just in case."
- Don't comment every line uniformly regardless of whether it needs it.
- Don't introduce an abstraction, config option, or design pattern for a single current use case.
- Apply the rule of three before generalizing a repeated pattern.
- Prefer the simplest structure that solves the actual, current, stated problem.
- Don't silently expand a small fix into a larger rewrite.
- Skip unnecessary praise/preamble; get to the substance.
- State disagreement or a negative finding plainly, not hedged into vagueness.
- Match response length to what the question actually needs.
- Don't make unrelated "drive-by" changes inside a scoped fix — call them out separately.
- Don't silently under-deliver either — say explicitly what was and wasn't done.
- Never bypass a failing check (test, hook, linter, type error) without understanding why it failed first.
- Get explicit confirmation before any destructive/hard-to-reverse operation.
- Never silently discard a user's uncommitted changes.
- Never loosen a test assertion, validation rule, or auth check just to make something pass, without flagging it.
- Treat changes to safeguards (tests, validation, auth checks) as needing their own explicit justification.
- Don't add a dependency for something already available in the project or the standard library.
- Don't generate near-duplicated files/functions where one parameterized implementation would do.
- Match existing codebase conventions over a generically "more correct" alternative, absent a stated reason.
- Chase root causes, not just the symptom currently producing an error message.
- When a fix applies to multiple locations, find and address all of them, and report the actual count.
- Handle edge cases' behavior, not just their type signature.


## Part 4 — JavaScript, TypeScript & Node.js

- Names are meaningful, consistently cased, and searchable — no cryptic single-letter vars outside tight loop scopes.
- `const` by default; `let` only for genuine reassignment; no `var`.
- No `any` used as an escape hatch — use `unknown` plus narrowing, or a real type.
- Every `async` function's rejections are handled — no unhandled-promise-rejection warnings.
- No mixing of `require`/`import` within the same module; match the project's module system.
- Errors carry enough context (what failed, with what input) — no bare `throw new Error("failed")`.
- No swallowed `catch` blocks that discard the error without logging, rethrowing, or handling it.
- `package.json` dependencies are pinned/lockfile-controlled; no unpinned `latest`-tag installs in production configs.
- ESLint/Prettier (or the project's actual configured tooling) run clean before code is presented as done.
- Tests actually exercise behavior, not just call the function and assert nothing.
- No blocking synchronous I/O (`fs.readFileSync` and friends) on a hot server request path.
- User input is validated/sanitized before use in a query, shell command, or HTML output.
- No `eval`, `new Function(...)`, or `dangerouslySetInnerHTML`-style sinks fed with untrusted input.
- Promises are awaited or explicitly returned — no silently-dropped floating promises.
- No `==` where `===` is meant; equality checks are intentional.
- Large lists render/paginate rather than loading everything into memory or the DOM at once.
- No invented npm package, method, or config flag presented without genuine confidence it exists.
- No claim that "tests pass" or "this is fixed" without having actually run something.
- No stub/mock/placeholder data left in code presented as the finished answer.
- Refactors that touch error handling or validation are flagged explicitly, not slipped in silently.
- Near-duplicated files/handlers are consolidated into one parameterized implementation where practical.
- New syntax/APIs are checked against the project's actual Node/TypeScript/bundler target before use.


## Part 5 — Python

- **DO** follow PEP 8 naming: `snake_case` functions/variables, `PascalCase` classes, `UPPER_SNAKE_CASE` constants.
- **DON'T** shadow builtins (`list`, `dict`, `id`, `type`) or use ambiguous single-letter names outside tiny loop scopes.
- **DO** write imperative, PEP 257-style docstrings for public modules, classes, and functions — summary line first.
- **DON'T** pad code with comments that restate what the next line already says; comment the *why*, not the *what*.
- **DO** annotate public function signatures with type hints and run mypy/pyright in CI, ideally in strict mode.
- **DON'T** default to `typing.Any` as an escape hatch, or leave `# type: ignore` unexplained.
- **DO** validate external/untrusted data at runtime (Pydantic or equivalent) — static types alone don't enforce anything at runtime.
- **DO** prefer comprehensions and generator expressions over manual append-loops; use generators for large/lazy data.
- **DO** use `with` for every resource needing deterministic cleanup — files, locks, connections, transactions.
- **DON'T** use a mutable object (`list`, `dict`, `[]`, `{}`) as a default argument value — use `None` and initialize inside the function.
- **DO** use `@dataclass` for data-holding classes, with `field(default_factory=...)` for mutable defaults and `frozen=True` for immutable ones.
- **DO** use `pathlib.Path` instead of `os.path` string manipulation in new code, consistently within a file.
- **DON'T** use bare `except:`, and don't catch `Exception` broadly only to log-and-swallow it — catch specific types.
- **DO** build a custom exception hierarchy rooted in one base exception per package; use `raise ... from exc` to chain causes.
- **DON'T** use `assert` to validate user input or API arguments — it's stripped under `python -O`; raise a real exception instead.
- **DO** return `None`/sentinels only when genuinely ambiguity-free; prefer raising a specific exception for real failures.
- **DO** manage dependencies via `pyproject.toml` plus a lockfile (uv/Poetry/pip-tools); always work inside a virtual environment.
- **DON'T** install packages into the system/global Python, and don't commit `.venv`/`venv` directories to version control.
- **DO** run `ruff`/`black` (or `ruff format`) and a type checker in CI as required, non-skippable checks, configured in `pyproject.toml`.
- **DON'T** suppress a lint/type warning with a blanket, unexplained `# noqa` — scope it to the specific rule with a reason.
- **DO** write pytest-style tests with fixtures for setup/teardown, `@pytest.mark.parametrize` for repeated cases, and `tmp_path`/`monkeypatch` over ad hoc equivalents.
- **DON'T** over-mock a test until it no longer exercises real logic, or assert only on mock-call-happened rather than actual outcomes.
- **DO** keep tests isolated, order-independent, and fast; separate slow integration/E2E tests from the unit suite.
- **DON'T** use `time.sleep()` to wait on async/concurrent conditions in tests — poll with a timeout or use the library's sync primitives.
- **DO** use `async`/`await` for I/O-bound concurrency; never call a blocking synchronous function directly inside a coroutine.
- **DON'T** create a "fire and forget" `asyncio.create_task()` without keeping a strong reference to it — it can be garbage-collected mid-flight.
- **DO** use `asyncio.TaskGroup`/`gather()` to run independent coroutines concurrently instead of awaiting them sequentially.
- **DO** set explicit timeouts on all awaited I/O and all outbound HTTP requests — never await indefinitely on a hung connection.
- **DON'T** check membership against a large `list` repeatedly in a loop — use a `set`/`dict` for O(1) lookups.
- **DO** vectorize numeric work over large arrays (NumPy/pandas) instead of iterating element-by-element in pure Python.
- **DON'T** build large strings with repeated `+=` in a loop — use `str.join()`.
- **DO** profile (`cProfile`, `py-spy`, `line_profiler`) before optimizing; re-profile after each change to confirm impact.
- **DON'T** ever build SQL by interpolating user input into a query string — use parameterized queries/an ORM, always.
- **DON'T** pass `shell=True` to `subprocess` (or use `os.system`) with any externally-influenced input — pass an argument list instead.
- **DON'T** call `eval()`/`exec()` on anything not fully trusted and hardcoded; don't `pickle.loads()` untrusted data; use `yaml.safe_load()`, never `yaml.load()`.
- **DO** use the `secrets` module (not `random`) for tokens/keys, `hmac.compare_digest()` for secret comparisons, and a real password-hashing library (bcrypt/argon2) for credentials.
- **DON'T** hardcode secrets/credentials in source; read them from environment variables or a secrets manager, never commit `.env`.
- **DO** verify webhook signatures before trusting payloads, and allowlist fields explicitly on any endpoint that writes from client input (avoid mass assignment).
- **DO** use the `logging` module, not `print()`, for all runtime diagnostics; choose log levels deliberately and configure logging once at the entry point.
- **DON'T** log secrets, tokens, or full PII, even at `DEBUG` level; use lazy `%s` formatting in log calls on hot paths, not f-strings.
- **DON'T** hallucinate library functions, parameters, or method signatures — verify against the actual installed version before relying on them.
- **DON'T** mix Python 2 idioms into Python 3 code: no `print` statements, `xrange`, `.has_key()`, `raw_input()`, or old-style `except Err, e:`.
- **DON'T** write every task as a flat, top-level script with no functions and no `if __name__ == "__main__":` guard — structure and test real applications.
- **DON'T** reach for a global mutable variable as the default way to share state — pass parameters explicitly or use a class/context object.
- **DON'T** over-comment obvious code, or under-document genuinely non-obvious behavior, side effects, and workarounds.
- **DON'T** silently drop existing functionality, tests, comments, or error handling while making an unrelated change — call out every behavior change.
- **DO** match structural/architectural weight to actual scope — neither over-engineer a one-off script nor leave a real application unstructured.
- **DON'T** present unverified code as tested, optimized, or "handles all edge cases" — state plainly what was and wasn't actually checked.
- **DO** read and match a codebase's existing conventions (naming, error handling, typing discipline) instead of introducing a competing style.
- **DO** guard clauses over deep nesting; keep boolean conditions free of double negatives and redundant `== True`/`== False` comparisons.
- **DO** use `enum.Enum` for closed sets of related constants instead of loose module-level string/int literals.
- **DO** use timezone-aware `datetime` objects (`datetime.now(timezone.utc)`) everywhere; never compare or store naive datetimes across timezone boundaries.
- **DON'T** use floating-point arithmetic for money — use `decimal.Decimal` constructed from strings, or integer minor units.
- **DO** pin dependencies via a lockfile with hashes for reproducible, tamper-resistant installs; scan for known vulnerabilities (`pip-audit`) in CI.
- **DON'T** grant CI/CD credentials broad, unscoped access, and never pin a third-party CI action to a mutable tag instead of a commit SHA.
- **DO** use guard clauses and early returns to keep validation/edge-case handling flat instead of deeply nested `if` blocks.
- **DON'T** retry a non-idempotent operation (a payment, a resource-creating POST) without an idempotency key — a retried timeout can duplicate the side effect.
- **DO** design setup, migration, and seed scripts to be idempotent by default — safe to re-run after a partial failure.
- **DON'T** trust a client-declared `Content-Length`, MIME type, or file extension — enforce real size limits and validate actual content.
- **DO** treat a passing type checker and a green lint run as a floor, not a substitute for tests, correctness review, or actually running the code.


## Part 6 — Go, Rust, C & C++

**Go:**

- [ ] Every file passes `gofmt`/`goimports` with no manual alignment.
- [ ] Identifiers use `MixedCaps`; packages are short, lowercase, and named for what they provide.
- [ ] No package stutters its own name in exported identifiers (`user.Profile`, not `user.UserProfile`).
- [ ] `internal/` is used to enforce real API boundaries.
- [ ] `main` stays thin; business logic lives in testable packages.
- [ ] Every returned error is checked or its omission is explicitly justified.
- [ ] Errors are wrapped with `%w`, not `%v`, when context is added.
- [ ] Sentinel/typed errors are compared with `errors.Is`/`errors.As`, never string matching.
- [ ] Error strings are lowercase and unpunctuated.
- [ ] `panic` is reserved for unrecoverable programmer errors, not expected failure paths.
- [ ] `recover` is used at defined boundaries (middleware, worker wrappers), always with logging.
- [ ] Every goroutine has an explicit, reachable exit condition.
- [ ] Shared mutable state is protected by a mutex, atomic op, or channel — never accessed unsynchronized.
- [ ] `sync.Mutex` and similar types are never copied by value.
- [ ] `context.Context` is the first parameter of I/O-bound functions and is actually honored via `Done()`.
- [ ] Tests and CI run with `-race` whenever goroutines touch shared state.
- [ ] Interfaces are small, defined at the point of consumption, and not designed "just in case."
- [ ] Struct embedding is used for real delegation, not simulated inheritance.
- [ ] Tests are table-driven with named subtests via `t.Run`.
- [ ] Test assertions check error type/sentinel, not exact error strings.
- [ ] `go vet ./...` and `golangci-lint run` pass cleanly in CI, with no blanket-disabled linters.
- [ ] Resources (files, connections, bodies, locks) are closed via `defer` immediately after successful acquisition.
- [ ] No fabricated stdlib or third-party API calls — every call is verified against real documentation or compiled code.
- [ ] Plain maps are never written to concurrently without synchronization.
- [ ] No `Get`-prefixed simple accessors; no ALL_CAPS constants.

**Rust:**

- [ ] Function parameters borrow (`&T`, `&[T]`, `&str`) by default; ownership is taken only when needed.
- [ ] `.clone()` calls are each individually justified, not reflexive borrow-checker workarounds.
- [ ] Explicit lifetimes are added only where elision genuinely can't infer them.
- [ ] `Rc<RefCell<T>>`/`Arc<Mutex<T>>` are used deliberately, not as default escape hatches.
- [ ] Fallible functions return `Result<T, E>`; absence-only cases return `Option<T>`.
- [ ] `?` is used for propagation instead of manual `match`-and-return boilerplate.
- [ ] `.unwrap()`/`.expect()` are absent from reachable production/library failure paths.
- [ ] Custom error types implement `std::error::Error` (via `thiserror` or by hand).
- [ ] Application code uses an erased error type (`anyhow`); libraries use precise enums.
- [ ] Traits favor default method implementations; only the true minimum is required.
- [ ] Static vs. dynamic dispatch (`impl Trait` vs `dyn Trait`) is a deliberate choice.
- [ ] Newtypes distinguish values that share a representation but aren't interchangeable.
- [ ] Every `unsafe` block is minimal in scope and carries a `// SAFETY:` comment.
- [ ] `unsafe` is reached for only after safe alternatives are ruled out.
- [ ] `unsafe impl Send`/`Sync` is never added just to silence a compiler error.
- [ ] `Arc<Mutex<T>>` critical sections are kept short; `PoisonError` handling is deliberate.
- [ ] No `Mutex` guard (non-async-aware) is held across an `.await` point.
- [ ] Every spawned async task's handle is awaited, tracked, or explicitly detached with reason.
- [ ] `Cargo.lock` is committed for binaries, omitted for libraries.
- [ ] Dependency versions use sensible semver ranges, not `"*"` or unnecessary exact pins.
- [ ] `cargo fmt` and `cargo clippy --all-targets --all-features -- -D warnings` pass in CI.
- [ ] No blanket `#![allow(clippy::all)]`; suppressions are narrow and justified.
- [ ] Unit tests live in `#[cfg(test)] mod tests`; integration tests live under `tests/`.
- [ ] Public API examples are doc-tests, run by `cargo test`.
- [ ] `cargo audit` runs regularly against the RustSec advisory database.
- [ ] No fabricated crate names, functions, or APIs — every call is verified against real docs or a compile.

**C:**

- [ ] Every `malloc`/`calloc`/`realloc` has exactly one clearly owned matching `free`.
- [ ] Every allocation's return value is checked for `NULL` before use.
- [ ] Pointers are set to `NULL` immediately after being freed.
- [ ] No pointer is freed more than once; no pointer is freed that wasn't heap-allocated.
- [ ] `realloc`'s result is assigned to a temporary, never overwriting the only reference to the original block.
- [ ] No function returns a pointer to a local (stack) variable.
- [ ] Cleanup on multi-resource, multi-failure-path functions uses one consistent pattern (e.g., `goto cleanup`).
- [ ] Every buffer write uses a bounded function (`snprintf`, `strncpy`/`strlcpy`) with the real destination size.
- [ ] Buffer/array sizes from external input are validated against actual capacity before use.
- [ ] `sizeof` is never used on a decayed pointer parameter to infer array length.
- [ ] Array allocation multiplication uses `calloc(count, size)`, not `malloc(count * size)`.
- [ ] Every loop bound is checked against the real array size (no `<=` off-by-one).
- [ ] Every pointer parameter's nullability contract is documented and either checked or asserted.
- [ ] Pointer arithmetic stays within a single array object (plus one-past-the-end for comparisons only).
- [ ] `const` is applied to every pointer parameter a function only reads.
- [ ] No reliance on signed integer overflow, unsequenced side effects, or other undefined behavior.
- [ ] Build uses `-Wall -Wextra -Werror` (or equivalent) with warnings actually fixed, not suppressed.
- [ ] `-fsanitize=address,undefined` runs in CI or regularly in development.
- [ ] Every header has an include guard or `#pragma once`.
- [ ] Headers declare only what's needed; implementation details stay in `.c` files.
- [ ] Every fallible call's return value/status code is checked at the call site.
- [ ] `errno` is read immediately after the call that set it, never after intervening calls.
- [ ] `assert()` is never used to validate untrusted external input.
- [ ] No runtime/user-derived string is ever passed directly as a `printf`-family format argument.
- [ ] Structs/arrays are explicitly zero-initialized where any field might be read before being set.
- [ ] `gets()`/unbounded `strcpy`/`strcat`/`sprintf()` never appear, even in example code.

**C++:**

- [ ] Every resource's lifetime is bound to an object via RAII — no manual acquire/release pairing.
- [ ] Standard RAII types (`lock_guard`, `fstream`, smart pointers) are used before hand-rolling one.
- [ ] `std::unique_ptr` is the default for owned heap allocations; `shared_ptr` is used only for genuine shared ownership.
- [ ] No raw `new`/`delete` appears for ownership in application code.
- [ ] Non-owning access uses raw pointers/references, not smart pointers passed unnecessarily.
- [ ] `weak_ptr` breaks any `shared_ptr` reference cycle.
- [ ] `auto`, range-based `for`, `nullptr`, and `enum class` are used over their legacy equivalents where they aid clarity.
- [ ] No C-style casts; every cast is a named cast (`static_cast`, `dynamic_cast`, etc.) matching real intent.
- [ ] Every non-mutating member function and every read-only parameter is marked `const`.
- [ ] `mutable` is reserved for genuinely incidental state (caches, mutexes), not to bypass real const-correctness.
- [ ] Large/non-trivial parameters are passed by `const&` unless the function needs its own copy.
- [ ] Classes follow the Rule of Zero by default, or the full Rule of Five when they own a raw resource.
- [ ] Move constructors/assignment are `noexcept` where truly non-throwing, and leave the source valid but empty.
- [ ] No use of a moved-from object except to reassign or destroy it.
- [ ] Container/array access is bounds-checked (`.at()`) unless the index is already provably in range.
- [ ] Iterators are never used after an operation that may have invalidated them.
- [ ] Every member variable is initialized, either via default member initializer or every constructor.
- [ ] No virtual call from a constructor/destructor expecting derived-class dispatch.
- [ ] STL containers and `<algorithm>` are used before a hand-rolled equivalent.
- [ ] Container choice (`vector` vs `list` vs `map` vs `unordered_map`) matches the real access pattern, not habit.
- [ ] `.reserve()` is used before loops appending many known-or-estimated elements.
- [ ] CMake uses `target_`-scoped commands with explicit `PUBLIC`/`PRIVATE`/`INTERFACE`, not legacy global commands.
- [ ] C++ standard and minimum CMake version are set explicitly, not left to toolchain defaults.
- [ ] Debug and release configurations are kept properly separated.
- [ ] Build artifacts are gitignored, never committed.
- [ ] No fabricated STL/library API calls — every call is verified against real documentation or a compile.
- [ ] Every "memory-safe"/"exception-safe"/"no leaks" claim is backed by actual tracing or sanitizer runs, not asserted by default.


## Part 7 — Java, Kotlin & C#/.NET

**Java:**

- Package names are lowercase/dot-separated; types use `UpperCamelCase`, members `lowerCamelCase`, constants `UPPER_SNAKE_CASE`.
- A formatter and linter run in CI; no wildcard imports; a public style guide backs the codebase's conventions.
- Composition is the default for reuse; inheritance is reserved for true is-a/LSP-honoring relationships.
- No god classes; interfaces are segregated by consumer need, not one fat interface for everyone.
- Fields are private; mutable internal collections are never returned directly from getters.
- Value-like objects favor immutability (`record`, final fields, no setters); `equals`/`hashCode` are always overridden together.
- Checked exceptions are reserved for recoverable conditions; unchecked for programming errors.
- No empty catch blocks or blanket `catch (Exception e)` used to silence warnings; the original cause is preserved when wrapping.
- Exception types — not message-string matching — distinguish failure cases; retry/circuit-breaker logic uses a resilience library.
- Try-with-resources is used for every `AutoCloseable`/`Closeable`.
- The right collection type matches the access pattern; `List.of`/`Collections.unmodifiableX` back immutable collections.
- Collections are never mutated mid-iteration; `computeIfAbsent`/`merge` replace manual check-then-act map logic.
- `Optional` is used only for "may be absent" returns, never for fields, parameters, or collection elements.
- Streams are used where genuinely clearer than a loop, not forced onto imperative logic; pipelines stay readable and terminal operations are the most specific available.
- Parallel streams are used only after benchmarking, never as a default "for speed."
- Constructor injection is the Spring default; field `@Autowired` and `ApplicationContext.getBean` service-location are avoided.
- `@Transactional` is applied at the service layer, aware that self-invocation bypasses the AOP proxy.
- Dependency versions are centrally pinned/managed (BOM/version catalog); the Maven/Gradle wrapper is checked in and used.
- Tests follow Arrange-Act-Assert with descriptive names; mocks target real collaboration boundaries, not plain data objects.
- Tests are hermetic — no real network/DB/sleep, no shared mutable static state, no unmocked wall-clock time.
- Raw `Thread` is replaced by `ExecutorService`/structured concurrency with bounded, workload-appropriate pool sizing.
- Shared mutable state across threads uses `java.util.concurrent` types; blocking `Future`/`CompletableFuture` calls specify a timeout.
- A logging facade (SLF4J) replaces `println`; levels match severity; no sensitive data is ever logged; placeholders replace concatenation.
- PECS (`? extends`/`? super`) is applied consistently; `@SuppressWarnings("unchecked")` is scoped as narrowly as possible.
- SQL is always parameterized; untrusted data is never deserialized without an allowlist; secrets stay out of source control.
- Pattern matching (`instanceof`, switch expressions, record patterns) replaces manual casts and `break`-per-case fall-through.
- Deprecated public API members carry `@Deprecated(since, forRemoval)`; public signatures don't break outside a major version.
- NIO.2 `Path`/`Files` replaces legacy `File`; large files are streamed, never loaded fully into memory.
- `setAccessible(true)` is avoided in application code; expensive reflective lookups are cached, not repeated per call.
- Enum constants carry their own behavior instead of external `switch` logic; `ordinal()` is never used for business values.
- Generated code targets the project's actual configured Java version — no assuming unavailable features.
- No hallucinated library methods, class names, or dependency coordinates — verified before use.
- No `catch (Exception e) { e.printStackTrace(); }` presented as real error handling.
- Ceremony (patterns, abstractions, streams) matches actual problem complexity — no gold-plating trivial logic.
- Generated changes match the actual scope requested — no unrequested abstraction layers, no silently incomplete stubs.

**Kotlin:**

- Nullability is modeled honestly in types (`T?` vs `T`); nothing is silently treated as "nullable everywhere."
- `!!` is avoided by default — used only as a rare, deliberate, commented escape hatch, and never in `as? ... !!` combinations.
- Safe calls (`?.`), Elvis (`?:`), and `?.let { }` replace manual `if (x != null)` chains.
- `lateinit var` is reserved for framework-guaranteed initialization; ten-nullable-field classes become explicit sealed states instead.
- `by lazy { }` replaces manual nullable-backing-field lazy init; `init` blocks validate data class invariants.
- Data classes stay immutable (`val`) when used as hash keys/set elements; `copy()`'s shallow-copy behavior is understood, not assumed deep.
- Data classes are not used for identity-based entities (DB rows, JPA objects); sensitive data is excluded from generated `toString()`.
- Sealed classes/interfaces model closed, fully-known sets of alternatives — never given a reflexive `else -> {}` catch-all in a `when`.
- Enum classes are reserved for uniform-shape variants; sealed classes are used when each variant carries distinct data.
- Coroutines launch in a structured, lifecycle-tied `CoroutineScope` — never `GlobalScope`; blocking calls run under `withContext(Dispatchers.IO)`.
- `async` calls meant to run concurrently are all started before any is awaited; `CancellationException` is always rethrown, never swallowed.
- `supervisorScope`/`coroutineScope` is chosen deliberately; `runBlocking` stays at true entry points, never nested in other coroutine code.
- `Mutex.withLock` (not JVM `synchronized`) guards suspend-function state; rate-limited concurrent access is bounded with a `Semaphore`.
- `Flow` is used for multi-value async streams, plain `suspend` functions for single results; suspend functions await their own child work.
- Extension functions are narrowly scoped, clearly named, and don't shadow same-named members or hide external mutable state.
- Collection operators (`map`, `filter`, `groupBy`, `mapNotNull`) replace manual loops; the most specific operator replaces `filter().x` chains.
- `asSequence()` backs large multi-stage pipelines; `buildList`/`buildMap` replace manual mutable-then-convert construction.
- Default parameters and named arguments replace Java-style overloaded function variants; operator overloading has intuitive meaning only.
- Top-level functions replace an unnecessary `object Utils` wrapper; top-level `const val` replaces a companion object for plain constants.
- `@JvmOverloads`/`@JvmStatic`/`@JvmName` are applied at Java-interop boundaries; platform types get real nullability established immediately.
- DSL builders use lambda-with-receiver and `@DslMarker`, reserved for genuinely nested/structural configuration.
- Interface delegation (`by`) replaces hand-written forwarding methods; custom property delegates are used only for genuinely repeated patterns.
- `when` expressions replace assignment-producing `if`/`else if` chains and use the subject-based form when matching one value.
- Visibility defaults to the most restrictive level that satisfies real callers; `internal` scopes module-shared, non-public-API code.
- `private set` expresses publicly-readable/internally-mutable properties instead of a fully open `public var`.
- `inline` is reserved for frequently-called functions with lambda parameters; `reified` is used only where runtime generic access is needed.
- Tests use Kotlin-idiomatic style and MockK over raw Mockito; `suspend` functions are tested with `runTest`, not `runBlocking`.
- Floating-point test assertions use an explicit tolerance; `!!` is avoided even in test code.
- Generated code reads as idiomatic Kotlin, not "Java translated line-by-line"; scope functions are each used for their specific purpose.
- No fabricated stdlib/coroutine API names — verified against the real API surface; the project's existing DI framework is matched, not replaced.

**C#/.NET:**

- `PascalCase` for types/methods/properties, `camelCase` for locals/parameters, `I`-prefixed interfaces — enforced via `.editorconfig`/analyzers.
- Async methods are suffixed `Async`; `async void` is used only for genuine UI event handlers — everywhere else it's `async Task`.
- No `.Result`/`.Wait()`/`.GetAwaiter().GetResult()` blocking on async code from synchronous call sites; no sync-over-async wrapper methods.
- `ConfigureAwait(false)` is used deliberately in library code, not pasted reflexively everywhere.
- A single `CancellationToken`/`CancellationTokenSource` flows through the full async chain, is honored, and is disposed when no longer needed.
- Independent async operations run concurrently via `Task.WhenAll`; failures are inspected fully (`AggregateException`), not just the first rethrown.
- `Task.Run` doesn't wrap already-async I/O work on the server; cancellation exceptions are handled at the responsible boundary, not swallowed.
- LINQ chains stay readable and aren't materialized (`.ToList()`) prematurely against `IQueryable`; deferred execution is understood, not re-enumerated by accident.
- Predicate overloads (`First(predicate)`) replace `.Where().First()`; `FirstOrDefault`/`Single` are chosen based on whether "not found" is expected.
- Nullable reference types are enabled and warnings are resolved, not suppressed; `!` is a rare, deliberate exception, backed by runtime guards at public boundaries.
- Specific exception types are thrown, never bare `Exception`; no empty `catch {}` blocks; `throw;` (bare) rethrows, never `throw ex;`.
- Exceptions aren't used for expected, high-frequency control flow; Web API exception handling is centralized in middleware.
- DI lifetimes (`Transient`/`Scoped`/`Singleton`) match actual state/thread-safety; scoped services (especially `DbContext`) are never captured in a `Singleton`.
- Constructor injection with `readonly` fields is the default; `IOptions<T>` replaces scattered `IConfiguration` lookups, validated at startup.
- `HttpClient` comes from `IHttpClientFactory`; secrets and connection strings stay out of source control; TLS validation is never disabled outside local dev.
- Related entities are eager-loaded/projected — no N+1 lazy-loading in a loop; `.AsNoTracking()` is used for read-only EF queries.
- Multi-step writes use an explicit transaction; writes are batched into one `SaveChangesAsync()`; `DbContext` lifetime matches the real unit of work.
- Filtering/paging happens in the database query, not after a premature `.ToList()`; generated SQL is checked periodically for cartesian explosions.
- `record`/`record struct` is the default for immutable data; entities with real identity avoid record value-equality; pattern-matching `switch` replaces manual casts.
- `Span<T>` and `struct` are reserved for genuinely allocation-sensitive hot paths backed by profiling, not applied by default; `StringBuilder` replaces loop concatenation.
- `Func`/`Action`/`Predicate` are preferred over custom delegates/events for simple callbacks; events null-check via `?.Invoke` and are unsubscribed when appropriate.
- Generic constraints (`where T : ...`) express real requirements instead of `object` casts; `default(T)` isn't assumed to equal `null` for value types.
- String comparisons specify an explicit `StringComparison`; `CultureInfo.InvariantCulture` backs round-trip formatting, not user display culture.
- Public APIs return `IReadOnlyList`/`IReadOnlyCollection` instead of mutable concrete types; collection-returning methods return empty, never `null`.
- `[Flags]` enums use explicit power-of-two values; persisted enum members have explicit values; parsed enum input is validated as a defined member.
- Parameterized tests (`[Theory]`/`[TestCase]`/`MemberData`) replace copy-pasted or contorted test variants; test names/arrangement match what's asserted.
- `DbContext` isn't mocked directly for query tests; async assertions use `Assert.ThrowsAsync`, not `.Result`; integration tests use isolated infrastructure.
- `IDisposable` resources are always wrapped in `using`/`await using`, never left undisposed.
- No hallucinated NuGet packages or API members — verified against the actual referenced package versions and target framework.
- The project's existing serializer (`System.Text.Json` vs `Newtonsoft.Json`) and established cross-cutting patterns are matched, not replaced ad hoc.


## Part 8 — Swift, Objective-C, Ruby, PHP, Bash/Shell, SQL & HTML/CSS

**Swift:**

- Use `UpperCamelCase` for types/protocols, `lowerCamelCase` for everything else; no Hungarian prefixes.
- Default to `let`; use `var` only where mutation is real.
- Never force-unwrap (`!`) a value whose nilness depends on runtime state.
- Use `guard let`/`if let` for unwrapping; reserve implicitly unwrapped optionals for framework-guaranteed cases (IBOutlets).
- Combine multiple optional binds in one `guard`/`if let` with `where` clauses instead of nesting.
- Never use `try?` to silently discard an error you should be handling.
- Default new models to `struct`; use `class` only for identity/shared mutation/Cocoa interop.
- Mark non-subclassed reference types `final`.
- Model domain errors as `Error`-conforming enums with meaningful cases, not `String` or bare `NSError`.
- Prefer `async throws` over `Result`-returning completion handlers for new async APIs.
- Never use `try!` outside provably-infallible contexts.
- Never leave an empty `catch {}` block — log or handle every caught error.
- Pick SwiftUI property wrappers by true ownership: `@StateObject` for views that create the object, `@ObservedObject`/`@Binding` for objects passed in.
- Keep SwiftUI `body` declarative; push logic into view models/services.
- Use Auto Layout constraints, not manual frame math, in UIKit.
- Separate view controllers from business logic (MVVM/coordinator); avoid "massive view controller."
- Capture `[weak self]` in any closure stored on `self` or handed to a long-lived API; unwrap with `guard let self else { return }`.
- Use `unowned` only when the referenced object is provably guaranteed to outlive the closure.
- Mark `delegate` properties `weak`.
- Use the Memory Graph Debugger / Instruments to confirm suspected retain cycles.
- Prefer `async`/`await` and `actor` types over completion handlers and manual locks for new concurrency code.
- Never block a thread on async work with semaphores or `DispatchGroup.wait()`.
- Mark UI-touching code `@MainActor`; don't silence isolation warnings with `@unchecked Sendable` reflexively.
- Inject dependencies via protocols/initializers so code is testable without hitting real networks/singletons.
- Write focused `XCTestCase` tests on observable behavior, not private implementation details.
- Never force-unwrap by default in generated code — treat it as a deliberate, justified exception.
- Never invent UIKit/SwiftUI/Foundation APIs that don't exist — verify signatures before using them.
- Match the concurrency style (async/await vs. completion handlers vs. Combine) already used in the surrounding codebase.
- Use the narrowest access level (`private`/`fileprivate`) that works; don't leave everything `internal`/`public` by default.
- Prefer protocol composition over deep class inheritance hierarchies.
- Use `map`/`flatMap`/`compactMap` on optionals instead of manual unwrap-transform-rewrap.
- Give `List`/`ForEach` items stable `Identifiable` ids, never array indices.
- Avoid `AnyView` overuse; prefer `@ViewBuilder`/`some View` to preserve type identity.
- Remove `NotificationCenter` observers and invalidate `Timer`s to avoid indefinite retention.
- Conform cross-concurrency-domain types to `Sendable`; don't blanket-suppress with `@unchecked Sendable`.
- Never spawn unstructured, uncancellable `Task {}` instances inside loops or view lifecycle callbacks.
- Use `XCTUnwrap` instead of force-unwrapping in tests.
- Verify a project's actual deployment target before assuming any API's availability.

**Objective-C:**

- Prefix custom types/globals with a project namespace prefix; never reuse Apple's reserved prefixes (`NS`, `UI`, etc.).
- Label every method parameter at declaration and call site; avoid ambiguous unlabeled selectors.
- Never write manual `retain`/`release`/`autorelease`/`[super dealloc]` in ARC-enabled code.
- Declare delegate and back-reference properties `weak`, not `strong`, to avoid retain cycles.
- Use the `__weak`/`__strong` capture dance for blocks stored on `self` or handed to long-lived APIs.
- Confirm the target actually uses ARC before writing any MRC-style code.
- Expose only the minimal public API in `.h`; keep mutable/internal state in a class extension in the `.m`.
- Use `@class` forward declarations in headers instead of full framework imports where possible.
- Group implementation code with `#pragma mark -` sections; split sprawling classes into categories.
- Declare `NS_DESIGNATED_INITIALIZER` and route convenience initializers through it.
- Wrap headers in `NS_ASSUME_NONNULL_BEGIN`/`END`; mark only true exceptions `nullable`.
- Annotate block parameter/return types with their own nullability, not just the enclosing method's.
- Match nullability annotations to actual runtime behavior, not wishful thinking.
- Use lightweight generics on `NSArray`/`NSDictionary` properties so Swift imports them fully typed.
- Declare enums with `NS_ENUM`/`NS_OPTIONS`, not bare `enum`, for correct Swift bridging.
- Write new business logic in Swift where the project is Swift-forward; reserve Objective-C for legacy/interop needs.
- Use `NS_SWIFT_NAME`/`NS_SWIFT_UNAVAILABLE` deliberately to shape the generated Swift interface.
- Never invent Foundation/UIKit/AppKit selectors or signatures — verify against real documentation or codebase usage.
- Don't graft Swift-style API idioms (trailing closures, label omission) onto Objective-C code.
- Verify retain-cycle-prone patterns (delegates, stored blocks, parent/child references) with Instruments or the Memory Graph Debugger.
- Use `instancetype` for factory/`init` methods so subclasses inherit correctly-typed constructors.
- Declare `copy` (not `strong`) for `NSString`/`NSArray`/`NSDictionary`/block properties that should snapshot their value.
- Prefix category method names on classes you don't own to avoid silent collisions with another library's category.
- Remove `NotificationCenter` observers and invalidate other resources in `dealloc`, even though ARC handles memory alone.
- Never let a `nonnull`-annotated method actually return `nil` on any code path.
- Design a thin Swift-facing adapter type at the language boundary instead of forcing a rich Swift type to be fully Objective-C-compatible.

**Ruby:**

- Use `snake_case` for methods/vars, `CamelCase` for classes/modules, `SCREAMING_SNAKE_CASE` for constants.
- Suffix predicate methods with `?` and destructive/raising methods with `!`.
- Prefer implicit return, string interpolation, and `unless`/trailing modifiers for simple conditionals; never `unless...else`.
- Use safe navigation (`&.`) deliberately — not as a blanket substitute for deciding whether `nil` is legitimate there.
- Favor `Enumerable` methods (`map`, `select`, `reduce`, `each_with_object`) over hand-rolled loops with accumulators.
- Use keyword arguments for methods with several parameters, especially booleans, so call sites self-document.
- Never use global variables (`$foo`) to pass state.
- Use `{ }` for single-line blocks, `do...end` for multi-line; convert to a `Proc`/`lambda` only when it must be stored or reused.
- Prefer `lambda`/`->` over `proc` when strict arity checking and local `return` semantics matter.
- Pair every `method_missing` override with `respond_to_missing?`.
- Never monkey-patch core classes for general application convenience; use modules, refinements, or explicit helpers instead.
- Prefer composition (mixins, delegation) over dynamically generated inheritance/`class_eval` mutation.
- Keep controllers thin (orchestration only) and don't let models balloon into unrelated-responsibility dumping grounds.
- Extract multi-step business operations into service objects with a single clear entry point.
- Eliminate N+1 queries with `includes`/`preload`/`eager_load`; verify with the `bullet` gem or query logs.
- Use strong parameters on every mass-assignable controller action; never `params.permit!`.
- Keep side-effecting logic (emails, external calls, DB writes) out of views and out of far-reaching model callbacks.
- Use background jobs for slow, non-blocking-required work instead of running it synchronously in the request cycle.
- Raise specific custom exception classes; rescue the narrowest class that's actually expected.
- Never rescue bare `Exception`; reserve broad `rescue` for genuinely intended catch-alls, not as a default habit.
- Use `ensure` for guaranteed cleanup, not manual duplication before every return/raise path.
- Structure RSpec with `describe`/`context`/`it`; use `let`/`let!` instead of repeating setup.
- Use verifying doubles (`instance_double`) over plain `double`; stub external services with WebMock/VCR.
- Use FactoryBot with minimal per-test overrides instead of hand-built records or shared fixtures.
- Don't test private methods directly; test them through the public interface that exercises them.
- Match the project's existing conventions (test framework, Rubocop config, service-object patterns) instead of defaulting to generic style.
- Use destructuring/multiple assignment instead of positional array indexing for readability.
- Prefer `Data.define`/`Struct.new` for small immutable value objects over a hand-rolled class.
- Enable `# frozen_string_literal: true` project-wide to catch accidental in-place string mutation.
- Use `Module#prepend` (with `super`) to wrap/intercept existing method behavior instead of physically overwriting it.
- Use `find_each`/`find_in_batches` for iterating large tables; never `.all.each` on an unbounded table.
- Know the difference between `.count`, `.length`, and `.size` on an ActiveRecord association before choosing one.
- Pair model-level validations with database-level constraints (`NOT NULL`, FKs, unique indexes) as a backstop.
- Bound every `retry` with an attempt counter; never retry unconditionally forever.
- Define a base application error class so a boundary layer can rescue every domain error uniformly.
- Use shared examples for common behavior across multiple types instead of duplicating specs.

**PHP:**

- Declare parameter, return, and property types everywhere PHP 8+ allows; add `declare(strict_types=1);` to every new file.
- Use constructor property promotion and `readonly` properties for value objects/DTOs.
- Use enums for closed sets of values instead of string/int constants.
- Prefer `match` over `switch` for value-producing branches; it's strict and exhaustive-checked.
- Use named arguments for calls with several optional/boolean parameters.
- Never use loose comparison (`==`) where strict comparison (`===`) is intended.
- Never write PHP 5-era patterns (untyped properties, `mysql_*` functions, `array()` literal) in new code.
- Format to PSR-12; run `php-cs-fixer`/`phpcs` in CI rather than relying on manual discipline.
- Use PSR-4 autoloading via Composer; never manually `require`/`include` autoloadable classes.
- One class/interface/trait/enum per file, file name matching the type name.
- Throw specific custom exception classes instead of returning `false`/`null` sentinels on failure.
- Never suppress errors with `@`; check preconditions explicitly instead.
- Never leave a `catch` block empty or silently swallow an exception — log or handle it.
- Commit `composer.lock` for applications; separate dev tooling into `require-dev`.
- Run `composer audit` in CI to catch known-vulnerable dependencies.
- Use prepared statements with bound parameters for every query touching external input — no exceptions.
- Never build SQL via string concatenation/interpolation, even with `addslashes()`.
- Validate structural SQL inputs (table/column names) against a strict allow-list; they can't be bound parameters.
- Stay alert for raw-query escape hatches in ORMs (`whereRaw`, `DB::raw`) that reintroduce injection risk.
- Write PHPUnit tests with constructor-injected, mockable dependencies — no real DB/network access in unit tests.
- Use data providers instead of copy-pasted near-identical test methods.
- Separate markup/templating from business and data-access logic; never tangle raw SQL with `echo`-ed HTML.
- Escape all user-controlled output rendered into HTML (`htmlspecialchars()` or an auto-escaping template engine).
- Never use deprecated/removed PHP functions; verify against the project's actual target PHP version.
- Guard against path traversal and command injection, not just SQL injection, on any external input.
- Verify the project's actual framework and version before generating framework-specific code.
- Use union/intersection types instead of no type or a vague `mixed` whenever more than one concrete type is truly accepted.
- Use named placeholders over positional ones in PDO once a query has more than a couple of bound parameters.
- Set `PDO::ATTR_ERRMODE` to `PDO::ERRMODE_EXCEPTION` on every connection instead of relying on manual error checks.
- Generate the correct number of placeholders for a dynamic `IN (...)` clause built from an array.
- Build a small custom-exception hierarchy rooted in one application base exception.
- Distinguish PHP's `\Error` hierarchy (programmer mistakes) from `\Exception` (recoverable conditions) when deciding what to catch.
- Declare an explicit `"php"` platform requirement in `composer.json` matching the real minimum supported version.
- Use `assertSame` over `assertEquals` when strict type/identity comparison is what's actually being verified.

**Bash/Shell:**

- Double-quote every variable expansion, command substitution, and array expansion (`"$var"`, `"$(cmd)"`, `"${arr[@]}"`) by default.
- Use arrays to build multi-argument command lines instead of concatenating into one string.
- Start every non-trivial script with `set -euo pipefail`; understand its gaps (conditions, non-last pipeline commands without `pipefail`).
- Use `trap ... EXIT` for guaranteed cleanup instead of manual cleanup calls before every exit point.
- Send error/diagnostic output to stderr (`>&2`), keep stdout clean for piping.
- Validate arguments and required environment variables at the top of the script with a clear usage message.
- Never parse `ls` output for iteration or attributes; use native globbing or `find -print0 | xargs -0`.
- Guard globs that might match nothing (`shopt -s nullglob`, or an explicit existence check inside the loop).
- Use shell file-test operators (`[[ -e ]]`, `[[ -d ]]`) instead of parsing `ls -l`/`stat` output.
- Decide POSIX `sh` vs. Bash explicitly via the shebang, and don't use Bash-only syntax in a `#!/bin/sh` script.
- Use `$(...)` command substitution over backticks; use `[ "$a" = "$b" ]` (not `==`) in POSIX scripts.
- Don't assume `/bin/bash` exists on every target system; verify before relying on Bash-only features.
- Watch for GNU-vs-BSD flag differences (`sed -i`, `date -d`) in cross-platform scripts.
- Run `shellcheck` on every script, locally and in CI; fix the underlying issue rather than suppressing by default.
- Suppress a specific shellcheck warning narrowly, with a comment explaining why — never a blanket file-wide disable.
- Never use `eval` on a string built from external or user-supplied input.
- Test file-handling logic against filenames containing spaces before considering it correct.
- Add a safety check before any destructive command (`rm -rf`, mass overwrite) — confirm the target path first.
- Pin the shellcheck version in CI so warnings don't drift purely from linter upgrades.
- Prefer `command -v tool` over `which tool` to check for a command's existence.
- Quote `"$@"` (not `"$*"` or bare `$@`) when forwarding positional arguments.
- Never assume a variable is non-empty and safe to leave unquoted — an empty unquoted variable silently drops an argument position.
- Declare every function-local variable with `local` to avoid leaking into or clobbering the caller's scope.
- Use process substitution (`< <(cmd)`) instead of piping into `while read` when the loop must set variables the caller needs afterward.
- Always use `read -r` to avoid backslash-escape surprises when reading lines.
- Use `getopts` (or a manual `while`/`case` parser) for scripts accepting more than one or two flags.
- Use `#!/usr/bin/env bash`, not a hardcoded `#!/bin/bash` path, for portability.
- Don't assume Bash 4+ features (associative arrays) are available on every target — verify the actual minimum supported version.
- Run shellcheck against the script's actual declared dialect (`-s sh` vs. `-s bash`).
- Treat a shellcheck failure in CI as a hard, merge-blocking gate for any script with real-world privileges.

**SQL:**

- Use consistent keyword casing and one-clause-per-line formatting for any non-trivial query.
- Alias tables meaningfully and qualify columns once more than one table is involved.
- Use explicit `JOIN ... ON` syntax, never implicit comma joins with conditions folded into `WHERE`.
- Use CTEs (`WITH`) to name intermediate steps instead of deeply nested unreadable subqueries.
- Index columns used in `WHERE`, `JOIN`, and `ORDER BY`/`GROUP BY` on tables of meaningful size — but don't index everything reflexively.
- Remember a composite index only helps queries filtering on a leading-column prefix.
- Verify actual index usage with `EXPLAIN`/`EXPLAIN ANALYZE` rather than assuming.
- Avoid wrapping indexed columns in functions inside `WHERE`; keep predicates sargable.
- Never use `SELECT *` in application code, views, or anything feeding further processing — list explicit columns.
- Use parameterized queries/prepared statements for every value that isn't literal SQL text — no exceptions.
- Never build SQL via string concatenation with manual escaping as a substitute for real parameterization.
- Validate dynamic structural SQL (table/column names, sort direction) against a strict allow-list; it can't be a bound parameter.
- Apply least-privilege database credentials per service as defense in depth beyond parameterization.
- Default to a normalized schema for OLTP systems; denormalize deliberately, only where measured, not preemptively.
- Keep duplicated (denormalized) data's sync mechanism explicit — trigger, app write path, or documented eventual consistency.
- Wrap multi-statement writes that must succeed/fail together in an explicit transaction.
- Keep transactions short — never hold one open across a slow external call or user-interaction wait.
- Choose an appropriate isolation level deliberately; know what the database's default actually guarantees.
- Use `SELECT ... FOR UPDATE` (or equivalent) to avoid read-then-write race conditions under concurrency.
- Batch large bulk writes into reasonably sized transactions, not one giant transaction or one-row-per-transaction.
- Never generate an N+1 query pattern — batch or join instead of querying per row of an outer loop.
- Never generate a migration/bulk update without an explicit transaction and a batching/rollback strategy.
- Check existing schema indexes/constraints/naming conventions before generating new queries or migrations.
- Verify the target database engine before using dialect-specific syntax (upserts, string concat, `LIMIT`/`TOP`).
- Explicitly index foreign key columns; don't assume every database engine auto-indexes them.
- Use `EXISTS`/`NOT EXISTS` over `SELECT *` subqueries for presence checks, and prefer `NOT EXISTS` over `NOT IN` when a `NULL` could be present.
- Use window functions instead of correlated subqueries/self-joins for per-group calculations alongside individual rows.
- Never store a comma-separated or JSON-packed ID list as a substitute for a real join table.
- Use `SAVEPOINT` for partial rollback within a larger transaction instead of discarding all its work.
- Never assume a `ROLLBACK` undoes non-database side effects (emails sent, external API calls) performed mid-transaction.
- Acquire locks in a consistent order across the application to reduce deadlocks, and always retry on the deadlock error.
- Use bounded connection pooling with guaranteed connection release; watch for connection leaks under load.
- Compare against `NULL` with `IS NULL`/`IS NOT NULL`, never `= NULL`.
- Batch bulk inserts into multi-row statements instead of one `INSERT` per row in an application loop.

**HTML/CSS:**

- Use the semantically correct element (`<nav>`, `<button>`, `<article>`, `<ul>`) instead of a generic `<div>`/`<span>` with a class doing the semantic work.
- Never build a clickable control out of a `<div onclick>`; use `<button>` or `<a>`.
- Keep exactly one `<h1>` per page and never skip heading levels.
- Wrap tabular data in a real `<table>`; never fake a grid with styled `<div>`s or use `<table>` for layout.
- Give every meaningful image a descriptive `alt`; give decorative images `alt=""`.
- Associate every form input with a real, persistent `<label>` — a placeholder is not a label.
- Use landmark elements (`<header>`, `<nav>`, `<main>`, `<footer>`) so assistive tech can jump between regions.
- Verify full keyboard operability and visible focus states; never remove `outline` without an equally visible replacement.
- Meet WCAG AA contrast minimums (4.5:1 normal text, 3:1 large text); never convey meaning by color alone.
- Use ARIA only to fill a genuine semantic gap — don't add it redundantly to elements with correct native semantics.
- Test with an actual screen reader and keyboard-only navigation, not just automated linting.
- Never use `!important` to resolve a specificity fight — fix the underlying selector/specificity issue.
- Keep selector specificity flat; avoid ID selectors and deep descendant chains for styling.
- Adopt one consistent naming methodology (BEM, utility-first, etc.) and apply it consistently.
- Scope component styles so they can't leak into unrelated parts of the page.
- Use CSS custom properties for design tokens (color, spacing, type scale) instead of repeating literal values.
- Use relative units (`rem`, `%`, `fr`) for anything that should scale; reserve `px` for things that genuinely shouldn't.
- Design mobile-first with `min-width` media queries, not desktop-first squeezed down with `max-width` queries.
- Set the viewport meta tag on every page; never disable pinch-to-zoom.
- Use CSS Grid/Flexbox for layout instead of float/clearfix or absolute-positioning hacks.
- Make images/media fluid (`max-width: 100%; height: auto`) instead of fixed-pixel-width.
- Test responsive layouts with realistic (long, overflowing) content, not just short placeholder text.
- Never default to inline `style=""` attributes for static design decisions.
- Never omit accessibility basics (alt text, labels, landmarks) from generated markup as an afterthought.
- Match the project's existing CSS architecture/naming convention instead of introducing a new one per generation.
- Check color-contrast ratios explicitly rather than eyeballing readability.
- Use `<time>`, `<details>`/`<summary>`, and `<dialog>` where they fit instead of custom-built equivalents.
- Mark required form fields with the `required` attribute, not just a visual asterisk.
- Announce dynamic content changes with `aria-live`; manage focus explicitly after modal/view changes.
- Tie validation error messages to their specific field via `aria-describedby`, not a single generic banner.
- Never use `tabindex` values greater than `0`; fix tab order by reordering markup instead.
- Use `:where()` for low-specificity reset/default styles so component rules can override them cleanly.
- Use logical properties (`margin-inline-start`) over physical ones when RTL support matters.
- Use `srcset`/`sizes` or `<picture>` to serve appropriately sized images instead of one oversized file scaled by CSS.
- Set explicit `width`/`height`/`aspect-ratio` on media to prevent layout shift while loading.
- Prefer native elements and established ARIA patterns over hand-rolled modals/dropdowns/tabs with unmanaged focus.
- Test text reflow and readability at up to 200% browser zoom, not just default zoom.


## Part 9 — Frontend Frameworks & Web Accessibility

- Call hooks only at the top level, never conditionally, in loops, or after an early return; name a function `useX` only if it actually calls other hooks.
- Don't store derived values in state; compute them inline or with `computed`/`useMemo`/`$derived`. Never mutate state/props in place — always create new references.
- Use the updater-function form of a setter when the next value depends on the previous one; reach for `useReducer`/a state machine once transitions get non-trivial.
- Split large contexts into small, purpose-specific ones; memoize provider values, and don't reach for Context/a global store before local state is proven insufficient.
- Read/write refs only in effects and event handlers, never during render, to drive rendering logic.
- Enable and respect exhaustive-deps/hook lint rules; don't silence a warning without fixing the root cause.
- Profile before adding `React.memo`/`useMemo`/`useCallback`; verify the memoization isn't defeated by an inline object/array/function prop from the parent.
- Use `children`/slots/composition to keep fast- and slow-changing parts of the tree from re-rendering each other, and to avoid prop-drilling.
- Use a stable, unique data identity (`item.id`) as list `key`; never use an array index or a freshly generated random value for a mutable/reorderable list.
- Include every reactive value an effect actually reads in its dependency array; return a cleanup function from any effect that subscribes to something external.
- Guard async effects/fetches against race conditions (`AbortController`, a cancelled flag, or `switchMap`); never pass an `async` function directly as an effect callback.
- Ask whether an effect is needed at all before writing one — derived state and event-driven logic often don't need one.
- Wrap data-fetching/lazy subtrees in Suspense-and-error-boundary pairs sized to independent failure domains, not one global boundary for the whole app.
- Default Server-Component-capable pages to server rendering; push `'use client'` to interactive leaves only, and never pass non-serializable values or secrets across that boundary.
- Pick one Vue API style (Composition or Options) per project and per component; know `ref()` needs `.value` while `reactive()` doesn't, and never destructure a `reactive()` object without `toRefs`.
- Never mutate a Vue prop directly in a child — emit an event instead; prefer `computed()`/`watchEffect` over a manual watch-and-set-a-second-ref combo.
- Scope component styles (`scoped`, CSS Modules) and default new Angular code to standalone components; reserve `NgModule` for existing module-based code.
- Unsubscribe every manual RxJS subscription; never nest `subscribe()` calls — flatten with `switchMap`/`mergeMap`/`concatMap`/`exhaustMap`; catch pipeline errors with `catchError`.
- Use `OnPush` with immutable input updates on Angular components; never mutate an `@Input()`-bound object in place; avoid expensive calls directly in templates.
- In Svelte 4, reassign (don't just mutate) top-level variables to trigger reactivity; prefer Svelte 5 runes for new code, and always clean up store subscriptions.
- Prefer framework data loaders (SvelteKit `load`, RSC async components, Angular resolvers) over client-only `onMount` fetches for a route's initial data.
- Classify state as local/shared-client/server/URL before picking a tool; don't duplicate server-fetched data into a separately-synced store; put shareable state in the URL.
- Choose a state library (Redux Toolkit, Zustand, Jotai, a state machine) to match the actual shape of the problem, not by default habit; model finite flows as explicit state machines.
- Implement optimistic updates only where failures are rare/recoverable, and always build the rollback path alongside the optimistic-apply path.
- Separate container (data/state) components from presentational (props-only) ones; extract a component on real duplication, not preemptively from a sample size of two.
- Organize by feature/domain, colocate tests/styles/stories with their component, and watch for god-component warning signs (huge files, unrelated state, constant churn).
- Reference design tokens instead of hardcoded colors/spacing; verify contrast independently per theme; constrain utility CSS to the design system's scale.
- Route all user-facing text through an i18n layer with real pluralization/interpolation; use logical CSS properties and locale-aware `Intl` formatting.
- Check a new dependency's bundle-size cost before adding it; import specific functions, not whole libraries; code-split by route and lazy-load heavy conditional UI with a loading fallback.
- Batch DOM reads separately from writes to avoid layout thrash; animate `transform`/`opacity`, not layout properties.
- Virtualize lists that can grow past a few hundred rows using a maintained library; never combine virtualization with index-based keys.
- Reserve image dimensions and use `srcset`/lazy-loading; load fonts with `font-display: swap`; prefetch likely-next routes, not every link indiscriminately.
- Move genuinely expensive synchronous work off the main thread; don't leak listeners, timers, or observers — clean them up on unmount/navigation.
- Guard browser-only APIs (`window`, `localStorage`, random/time-based values) behind a mount check in SSR frameworks to avoid hydration mismatches.
- Prefer real `<button>`/`<a>`/form elements over a `<div onClick>` with a bolted-on role; never remove a focus outline without a visible `:focus-visible` replacement.
- Trap and restore keyboard focus correctly for modals; never create a keyboard trap; give every icon-only control an accessible name.
- Use ARIA only when native HTML can't provide the needed semantics; implement full keyboard operability (Tab/Enter/Space/Escape/arrows) for custom widgets.
- Use `aria-live="polite"` (not `assertive`) for routine dynamic announcements; associate form errors/required state programmatically, not just visually.
- Use semantic landmarks and correct heading hierarchy; set `lang`; write real `alt` text (or `alt=""` for decorative images); caption video/audio.
- Respect `prefers-reduced-motion`; never let autoplaying/moving content be unpausable; confirm before irreversible/destructive actions; warn before session timeout.
- Manually test complex interactive components with a real screen reader — don't rely on automated scans alone.
- Never invent a hook, API, prop, package, or CSS/utility class that hasn't been verified to exist in the project's actual dependencies and version.
- Always include explicit loading, error, and empty states in any component that renders fetched data.
- Check the existing design system for a component before hand-rolling a new button/modal/input from scratch; search for existing utilities before writing new ones.
- Match the surrounding codebase's conventions (style, exports, framework version/idioms) rather than defaulting to generic training-data patterns.
- Don't add an unrequested dependency or unrequested abstraction layer to satisfy a narrowly-scoped request.
- Make the smallest targeted edit for a small requested change — don't regenerate an entire component from scratch.


## Part 10 — Backend Frameworks & API Design

**Architecture**
- Controllers/route handlers are thin; business logic lives in services, not in the transport layer.
- Dependencies are injected (constructor injection), not hard-imported as singletons deep in business logic.
- Each layer (controller → service → repository) depends only downward; nothing calls back up the stack.
- Framework request/response objects never leak past the controller layer.
- Config is loaded from environment/secrets, validated at startup, and never hardcoded in source.
- Services have liveness and readiness health checks that verify real dependencies, not just "process is up."
- Shutdown drains in-flight requests on `SIGTERM` instead of dropping them abruptly.
- No shared mutable state lives in server process memory across requests.
**REST Design**
- Resource URLs use plural nouns, not verbs; nesting stays shallow.
- HTTP methods match their semantics: GET is safe/cacheable, PUT/DELETE are idempotent, POST creates/acts.
- Status codes are chosen deliberately per case: 200/201/204, 400/401/403/404/409/422, 429, 500/503 — never a blanket 200 or 500.
- Every list endpoint is paginated by default, with a max page size enforced server-side.
- Sort/filter parameters are validated against an explicit allow-list, never interpolated into raw queries.
- Non-idempotent write endpoints with real-world consequences support an `Idempotency-Key`.
- API versioning strategy is applied consistently; breaking changes go in a new version, not an existing one.
- Response field naming, date formats, null-handling, and envelope shape are consistent across the whole API.
- CORS allows an explicit origin allow-list, never a wildcard combined with credentials.
- Monetary values use integer minor units (or exact decimals) plus an explicit currency code; timestamps are UTC ISO 8601.
**GraphQL & gRPC**
- Every relation resolver batches its data fetching (DataLoader or equivalent) to avoid N+1.
- Query depth/complexity limits are enforced before resolvers execute.
- Mutations use structured input/payload types and report errors distinctly from unexpected failures.
- Protobuf field numbers are never reused after a field is removed; retired numbers are reserved.
- gRPC deadlines are set and propagated across service-to-service call chains.
**Framework Conventions**
- New code follows the project's existing framework idioms (validation library, DI pattern, error handling) rather than inventing a new one per endpoint.
- ORM/entity objects are never returned directly as API responses; explicit DTOs/serializers map them.
- N+1 queries are prevented with the framework's eager-loading mechanism (`select_related`/`prefetch_related`, `includes`, `with`, DataLoader).
- Mass assignment is blocked by an explicit allow-list of client-settable fields (strong params, Form Requests, DTO whitelisting).
- Debug modes (`DEBUG=True`, `APP_DEBUG=true`, Flask's debugger) are off in production.
**Microservices**
- Each service owns its own data; no service queries another service's tables directly.
- Synchronous call chains stay short; independent reactions to one event go through async messaging.
- Circuit breakers and bounded retries with backoff protect calls to downstream services.
- Writes across service boundaries use idempotency keys; consumers are built to handle at-least-once delivery.
- Multi-step workflows have an explicit plan for partial failure (sagas/compensating actions), not an assumption it won't happen.
- Service-to-service calls are authenticated; "it's internal" is never treated as sufficient.
- A distributed trace ID propagates across every hop of a request.
**Documentation, Rate Limits, Auth**
- OpenAPI/GraphQL schema is generated from or validated against the real code, not hand-maintained separately.
- Every endpoint's error responses are documented, not just its happy path.
- Every publicly reachable endpoint has a rate limit; sensitive operations (auth, password reset) have stricter limits.
- 429 responses include `Retry-After`; rate-limit state is shared across instances, not per-process.
- Authentication and authorization are checked separately; a valid token alone never implies permission for the specific resource.
- Object-level authorization is enforced server-side on every write (never trust a client-supplied user/tenant ID).
- Tokens are short-lived with a defined revocation/refresh strategy; nothing sensitive is stored in a JWT payload.
**Validation & Errors**
- All external input (body, query, path, headers) is validated server-side, regardless of client-side validation.
- Error responses use one consistent structured shape with a machine-readable code across the whole API.
- Field-level validation errors identify exactly which field/path failed, including in nested payloads.
- Stack traces, SQL errors, and internal paths never reach the client-facing error body.
- PATCH validates only provided fields; PUT validates the full resource.
- File uploads are size-capped early and their real content type is checked, not just the client-supplied header/extension.
**Observability**
- Logs are structured (JSON), include a request/trace ID, and never contain secrets or full credentials.
- Latency is tracked as percentiles (p95/p99), not only as an average.
- Alerts are tied to real user-facing impact (SLO burn), not noisy raw thresholds that train responders to ignore them.
- Logs, metrics, and traces share one correlation ID scheme.
**Batch Operations & Webhooks**
- Batch endpoints report per-item success/failure explicitly unless the operation is deliberately all-or-nothing.
- Batch size has a documented maximum; batch items are individually idempotent for safe retries.
- Outbound webhooks are signed (HMAC), delivered asynchronously, retried with backoff and a cap, and deduplicated by event ID on the receiving end.
**Common AI-Assistant Pitfalls**
- No invented endpoints, framework methods, or library APIs — verified against the project's actual installed versions and route definitions.
- No god objects and no ad hoc pattern invented per endpoint that ignores the codebase's established conventions.
- Async/background-job pattern used for slow operations instead of blocking the request cycle inline.
- New migrations, config keys, or environment variables introduced by a change are called out explicitly, not left silent.


## Part 11 — Mobile App Architecture (iOS, Android, React Native & Flutter)

- Pick MVVM as the default iOS/Android architecture for most apps; reserve TCA/Redux-style unidirectional-flow frameworks for apps with genuinely complex, deeply shared cross-screen state.
- Don't let a `UIViewController`, SwiftUI `View`, `Activity`, or `Fragment` accumulate networking, persistence, and business logic directly — extract it into a ViewModel/repository layer.
- Default new iOS work to SwiftUI and new Android work to Compose; bridge to UIKit/XML views only for specific capability gaps, not wholesale rewrites of stable screens.
- Use `NavigationStack` with a typed path (iOS) and the Navigation Component/Compose Navigation (Android) instead of ad-hoc boolean-driven presentation state.
- Use `UserDefaults`/`SharedPreferences`/DataStore only for small preferences; use SwiftData/Core Data/Room for structured, queryable local data.
- Never store auth tokens, passwords, or secrets in plain `UserDefaults`/`SharedPreferences`/an unencrypted file — use Keychain (iOS) or Keystore-backed encrypted storage (Android).
- Don't run database queries, file I/O, or heavy computation on the main/UI thread on either platform.
- Design around each OS's real background-execution limits (`BGTaskScheduler`/background URL sessions on iOS; WorkManager, with Foreground Service reserved for user-visible ongoing work, on Android) — don't assume arbitrary background execution.
- Request notification/runtime permissions contextually, at point of use, with a clear rationale — never batch every permission request at first launch.
- Add every required `Info.plist` usage-description key before calling a privacy-sensitive API on iOS, or the app crashes at runtime.
- Handle every permission state explicitly (granted, denied, permanently denied, one-time-granted-then-revoked on Android 11+) with a path to Settings when needed.
- Provide working demo credentials/demo mode for App Review; never ship placeholder content, broken links, or Lorem Ipsum in a submitted build.
- Route digital-goods purchases through Apple IAP / Google Play Billing; don't link out to an external payment page for purely digital content.
- Support in-app account deletion if the app supports account creation (Apple requirement).
- Don't request `QUERY_ALL_PACKAGES`, `MANAGE_EXTERNAL_STORAGE`, or broad background location without a Play-policy-compliant justified use case.
- Fill out the Play Console Data Safety section accurately, including third-party SDK data collection, not just first-party.
- Downsample images to their display size and use a real caching library (Kingfisher/Nuke/SDWebImage, or Coil/Glide on Android) rather than re-decoding full-resolution images on every reuse.
- Use `RecyclerView`/`LazyColumn`/`FlatList`/`ListView.builder` with stable keys/DiffUtil for any list that can grow — never render an unbounded list inside a plain scroll container.
- Give `LazyColumn`/`LazyRow` items and RN `FlatList` a stable `key`/`keyExtractor`; never use an array index for a reorderable/mutable list.
- Scope Compose/Flutter state to the smallest subtree that needs it; use `const` constructors in Flutter and avoid hoisting `setState` above where it's needed.
- Treat the local database (SwiftData/Core Data/Room) as the single source of truth the UI reads from, syncing from the network in the background — don't read directly from transient network responses.
- Persist offline write queues durably and make them idempotent; never keep a pending-writes queue only in memory.
- Surface connectivity/staleness state honestly in the UI ("offline — showing saved data," a last-synced timestamp) rather than silently failing or showing a blank screen.
- Guard any API introduced after the app's `minSdkVersion`/deployment target with an explicit version check (`SDK_INT`, `@available`) or a backport library.
- Use `dp`/`sp` (Android) and points/Dynamic Type-relative sizing (iOS), never hardcoded pixel values, for layout and text.
- Test on the smallest and largest supported screen sizes, at least one tablet/large-screen and one foldable posture if supported, and both orientations.
- Wrap native SDK integrations behind a narrow, well-typed module boundary (Turbo/Native Modules in RN, platform channels in Flutter); don't scatter ad-hoc platform calls through shared code.
- Don't send large or high-frequency payloads across the JS-native bridge/platform channel; batch or move the hot path fully native.
- Adapt navigation chrome, dialogs, and control styling per platform (Cupertino vs. Material) rather than shipping one identical design on both.
- Choose a state-management approach proportional to actual app complexity; use a dedicated data-fetching/caching layer for server data instead of ad hoc per-screen fetch logic.
- Budget separate App Store and Play Store submission review cycles, listing requirements, and compliance checks — passing one says nothing about the other.
- Test actual release-mode builds (not just debug) on both platforms before submission.
- Use staged/phased rollouts for meaningful releases and watch crash rate before expanding to 100%.
- Reserve forced/blocking updates for genuinely breaking changes; keep specific, non-generic release notes.
- Integrate crash reporting from day one, symbolicate reports (dSYMs/ProGuard mapping), and treat crash-free rate as a tracked metric.
- Never log PII, tokens, or payment data into crash reports or analytics events.
- Watch for crash signatures concentrated on one device/OS/app-version combination, not just overall crash percentage.
- Give every interactive element a real accessibility label; never leave icon-only controls unlabeled.
- Support Dynamic Type / Android font scale; don't hardcode text sizes or disable system text scaling without strong justification.
- Size touch targets to at least 44×44pt (iOS) / 48×48dp (Android), even for visually small icons.
- Never convey state through color alone; pair it with icon/text/shape.
- Test both platforms with a real screen reader (VoiceOver/TalkBack) on critical flows, not just an automated scanner.
- Provide an accessible alternative for any custom-gesture-only interaction.
- Implement verified universal links/App Links (domain-association files), not just an unverifiable custom URL scheme, for links that matter.
- Handle deep links arriving via cold launch, warm foreground, and background resume — all three, not just cold launch.
- Validate deep-linked content still exists and the user is authorized before routing into it.
- Respect Low Power Mode / Battery Saver and avoid continuous high-accuracy GPS or tight polling loops when a coarser or event-driven approach suffices.
- Don't silently burn cellular data on large downloads/syncs without a Wi-Fi-only preference or size-awareness.
- Never hand-roll local encryption for secrets — use Keychain/Keystore-backed APIs (or `flutter_secure_storage`/`react-native-keychain`/`expo-secure-store` cross-platform).
- Verify every generated SDK/API call actually exists for the target OS version and deployment target before presenting it as correct — don't pattern-match a plausible-sounding method name.
- Pair every privacy-sensitive API call in generated code with its required permission declaration/request in the same response.
- Default generated data-fetching UI to include loading, error, empty, and offline states — never generate only the happy path.
- Default generated layout code to relative/flexible sizing; never hardcode a screen dimension from one reference device.


## Part 12 — DevOps, CI/CD, Docker, Kubernetes, Infrastructure as Code & Git

- Write commit subjects in the imperative mood, with a body that explains why, not just what.
- Keep commits atomic — one logical change per commit — and squash "oops"/"wip" commits before opening a PR.
- Never rewrite git history that's already been pushed and pulled by others without coordinating first.
- Keep PRs small and focused on one logical change; self-review the diff before requesting review.
- Write PR descriptions that state what changed, why, and how to verify it, with a rollback note for risky changes.
- Review design and intent before nitpicking style in code review; save nits for a clearly-labeled "nit:" comment.
- Commit a `.gitignore` before the first commit; never `git add .` reflexively without checking `git status` first.
- Never commit secrets, `.env` files, or private keys; if one leaks, rotate it immediately regardless of history cleanup.
- Use Git LFS or object storage for large binaries instead of committing them directly to the repo.
- Prefer `git push --force-with-lease` over plain `--force` when force-pushing your own rebased branch.
- Resolve merge conflicts by understanding both sides, then re-run the test suite before trusting the resolution.
- Tag releases with SemVer and annotated tags; never move or retag an already-published version.
- Use multi-stage Docker builds to keep compilers and dev dependencies out of the shipped runtime image.
- Start Dockerfiles from a minimal, explicitly pinned base image; never build production images from `latest`.
- Order Dockerfile instructions from least- to most-frequently-changing to maximize layer cache hits.
- Always run application containers as a non-root `USER`, with ownership set correctly on writable paths.
- Add a `.dockerignore` that excludes `.git`, `.env`, `node_modules`, and other build-irrelevant files.
- Clean up package-manager caches in the same `RUN` layer that installed them, not a later one.
- Never bake secrets into image layers via `ARG`, `ENV`, or `COPY`; use BuildKit secret mounts or runtime injection.
- Pin service image versions in `docker-compose.yml`, and keep secrets in a gitignored `.env`, not the committed file.
- Add health checks and `depends_on: condition: service_healthy` to Compose services with startup dependencies.
- Set CPU and memory requests and limits on every Kubernetes container, based on real observed usage.
- Define both liveness and readiness probes; never point a liveness probe at a downstream dependency check.
- Set `runAsNonRoot`, drop all Linux capabilities, and use a read-only root filesystem in pod security contexts.
- Store configuration in ConfigMaps and sensitive values in Kubernetes Secrets, never hardcoded in manifests or images.
- Apply default-deny NetworkPolicies and per-namespace ResourceQuotas on any shared or multi-tenant cluster.
- Never deploy a floating `:latest` image tag to production; pin to a specific version tag or digest.
- Version Helm charts independently from application versions, and lint/template them in CI before every `helm upgrade`.
- Order CI pipeline stages fast-to-slow, and fail fast on the first failing required check.
- Run independent CI checks in parallel, and use reusable workflows or templates instead of copy-pasted job definitions.
- Cache CI dependencies keyed on a lockfile hash, with a fallback restore key for partial cache hits.
- Quarantine known-flaky tests with a tracked follow-up ticket; never silently tolerate "just re-run CI."
- Store CI secrets only in the platform's dedicated secrets store, and never echo or print them in pipeline logs.
- Prefer OIDC-based short-lived credentials over long-lived static cloud keys stored as CI secrets.
- Match deployment strategy — rolling, blue-green, or canary — to the service's actual risk tolerance and traffic pattern.
- Make every deployment revertible with a single known command, and rehearse the rollback before an incident forces it.
- Use expand/contract database migrations so an application rollback never breaks against an already-migrated schema.
- Keep dev, CI, staging, and production on the same pinned tool and runtime versions to avoid "works on my machine."
- Store Terraform or Pulumi state in a remote, locked backend; never hand-edit a `.tfstate` file.
- Never commit state files to git — they routinely contain secrets and sensitive resource attributes in plaintext.
- Split IaC state per environment and component; avoid one monolithic state file for all infrastructure.
- Never make manual console or CLI changes to IaC-managed resources; fix the source and re-apply instead.
- Run scheduled drift detection (`plan`/`preview` against live state) to catch configuration that diverged from source.
- Design IaC modules with a minimal, coherent interface, and pin every consumer to a specific module version.
- Always review `plan`/`preview` output before every `apply`, paying special attention to any destroy or replace action.
- Never use `-auto-approve` outside a fully automated, already-plan-reviewed pipeline.
- Source cloud credentials from the environment or OIDC federation; never hardcode them in `.tf`/Pulumi source files.
- Mark sensitive Terraform variables and outputs `sensitive = true`, and still treat the state file itself as sensitive.
- Design IaC configurations and deploy scripts to be idempotent — re-running against no change should be a safe no-op.
- Never invent a CLI flag, YAML key, or Terraform resource; verify against the real schema or say explicitly you're unsure.
- Default every generated Dockerfile to non-root, multi-stage, and free of baked-in secrets, unless told otherwise.
- Never suggest `--force`, `--no-verify`, or `-auto-approve` as a first fix; diagnose and fix the root cause instead.
- Always warn explicitly, in words, before generating a destructive command, stating exactly what will be lost.
- Match a repo's existing CI conventions instead of inventing a new pipeline structure or tool unasked.
- Pin every generated dependency, base image, and third-party action to a specific version — never default to `latest`/`@main`.
- Treat any secret that ever appeared in git history, an image layer, IaC state, or a CI log as compromised and rotate it.
- Prefer the narrowest, most reversible operation that accomplishes the goal over the most destructive available option.


## Part 13 — Databases & Data Engineering

- Schema has explicit primary keys, appropriate foreign keys, and correct not-null/unique/check constraints — not left to application-layer enforcement alone.
- No `SELECT *` in application code paths that only need specific columns.
- Every query that filters or sorts on a column has a supporting index, verified against an actual query plan, not assumed.
- Pagination uses keyset/cursor pagination for large or frequently-changing datasets, not unbounded `OFFSET`.
- No string-concatenated SQL built from user input — parameterized queries or a safe query builder only.
- Migrations are reversible (or have a documented forward-only rationale) and reviewed for production-scale lock/performance impact.
- No editing of a migration that has already been applied in shared/production history — a new migration corrects it instead.
- Destructive migrations (dropping a column/table) are preceded by a backup/rollback plan, not run directly against production data.
- NoSQL schema design matches actual access patterns (embed vs. reference decided by read/write shape, not by habit).
- Long-running transactions are avoided; transaction scope is as narrow as correctness requires.
- Retries on non-idempotent operations use an idempotency key, not a bare retry.
- Cache invalidation strategy is explicit and matches actual data-freshness requirements — no unexamined arbitrary TTLs.
- Data pipeline steps are idempotent and safe to re-run after a partial failure.
- Schema evolution in a pipeline is handled explicitly (versioned schemas, validation at ingestion), not assumed stable.
- No claim that a migration, backfill, or bulk update "succeeded" without re-querying to confirm the actual resulting state.
- Every step of a multi-step operation is verified, not just whether the last step reported success.
- Recommended syntax/APIs are checked against the actual database version in use, not assumed current from general familiarity.
- Any suggested database/engine version is checked for end-of-life/end-of-support status before being presented as an unremarkable choice.
- Seed/example data is clearly separated from real migration and production data paths.


## Part 14 — Testing & QA

- Shape the suite like a pyramid: many fast unit tests, fewer integration tests, few e2e tests.
- Don't chase a fixed unit/integration/e2e ratio as a KPI in itself.
- Test business logic, calculations, branches, and state transitions — not trivial getters/setters.
- Write a regression test for every bug fix, ideally one that fails before the fix.
- Don't re-test third-party library or framework internals — trust them, test your usage of them.
- Assert on observable outputs and public contracts, not private state or internal call order.
- Ask "would this test survive a full rewrite with the same behavior?" — if not, it's coupled to internals.
- Don't use reflection/monkey-patching just to peek at private state for an assertion.
- Name tests after behavior + condition + expected outcome, not `test1`/`testFoo`.
- Pick one test-naming convention and apply it consistently across the codebase.
- Structure tests as arrange-act-assert (or given-when-then) with clear, separated phases.
- Keep each test to one logical behavior/reason to fail, even with multiple physical assertions.
- Don't cram unrelated scenarios into one giant test — split by scenario instead.
- Use parametrized/table-driven tests instead of looping over cases with assertions inside the loop.
- Make every test independently runnable, in any order, with fresh fixtures per test.
- Never let one test depend on state left behind by another test.
- Know the test-double vocabulary: dummy, stub, fake, mock, spy — and use the right one deliberately.
- Mock only genuine external boundaries: network, filesystem, clock, randomness, third-party APIs.
- Don't mock the very collaborator whose logic you're actually trying to verify.
- Prefer real objects or in-memory fakes over mocks for fast, deterministic dependencies.
- Make dependencies injectable so real implementations can be swapped for test doubles.
- Prefer state-based assertions over interaction-based (mock-call) assertions when both are available.
- Reserve interaction verification for cases where the interaction itself is the actual contract.
- Replace the system clock, RNG, and UUID generation with controllable fakes in tests.
- Never let a unit test make a real network call.
- Match integration/e2e test environments to production as closely as practical.
- Use disposable, containerized real dependencies for integration tests instead of over-mocking them.
- Never use a hardcoded `sleep()` to wait for async work — poll for the actual condition instead.
- Give every test isolated data (unique IDs, transactional rollback, per-worker namespace).
- Treat a flaky test as a real bug, not background noise to re-run past.
- Set up and tear down test data explicitly per test; never assume leftover data from elsewhere.
- Use factory/builder functions for test data, with overridable defaults.
- Use contract tests at service boundaries instead of relying solely on full e2e coverage.
- Reserve e2e tests for the small number of truly critical user journeys.
- Abstract e2e UI interactions behind Page Objects or similar reusable helpers.
- In TDD, always watch the test fail (red) before making it pass — never skip that step.
- Don't skip the refactor step in red-green-refactor.
- Use TDD for well-understood logic, bug fixes, and API design — not for exploratory/spike work.
- Don't mandate TDD as blanket dogma regardless of the kind of problem being solved.
- Treat coverage as a diagnostic signal, not a target to maximize or gate merges on rigidly.
- Never write a test that executes code without asserting anything meaningful, just to raise coverage.
- Sanity-check important tests by breaking the implementation and confirming the test fails.
- Remember 100% coverage doesn't mean bug-free — it can't catch logic that was never written, or a weak assertion.
- Consider mutation testing occasionally as a stronger signal than line coverage.
- Keep the unit test tier fast (seconds); separate slow integration/e2e tests into their own tier.
- Design tests to run safely in parallel, with fully isolated data per test.
- Make tests deterministic: no unseeded randomness, no reliance on the real clock or network.
- Never assume tests run in file/declaration order — periodically randomize test order to catch violations.
- Keep snapshots narrow and specific; avoid snapshotting huge, noisy structures.
- Always actually read a snapshot diff before accepting it — an update looks identical to a regression.
- Quarantine and track flaky tests explicitly; don't paper over them with blanket auto-retry.
- Make failure output diagnostic (expected vs. actual, stack traces, artifacts) so red runs are fast to fix.
- Never write assertion-free or tautological tests that pass regardless of implementation correctness.
- Never derive a test's expected value by re-implementing the same formula the code under test uses.
- Never delete, skip, or weaken a failing test to reach green without investigating why it failed first.
- Never claim tests pass without actually running them — report real tool output, not assumptions.
- Don't default to snapshot testing everywhere; write explicit assertions where they communicate more.
- Always cover edge cases and error paths, not just the happy path.
- Don't inflate test count/coverage with near-duplicate or trivial tests — prioritize distinct behavior.
- Never fabricate a method signature, API shape, or expected value — verify against the real implementation.
- Surface pre-existing flaky or failing tests encountered incidentally, rather than silently working around them.


## Part 15 — Security & Application Security (AppSec)

**Injection**
- Every SQL query uses parameterized queries or an ORM's safe query builder — no string-concatenated SQL, ever.
- Dynamic identifiers (table/column/sort order) are mapped through a fixed allowlist, never taken directly from user input.
- Shell commands are invoked with argument arrays (`shell=False`), never through a shell string built from user input.
- Output is encoded for its actual context (HTML body, HTML attribute, JS string, URL) — not one generic "sanitize" pass.
- Rich-text/user HTML is cleaned with a vetted sanitizer library (allowlist tags/attributes), never a hand-rolled regex filter.
- Template engines only ever render fixed template files with user data bound as variables — never compile template source from user input.
- A Content-Security-Policy restricts script sources and disallows `unsafe-inline`/`unsafe-eval`.
**Authentication**
- Passwords are hashed with Argon2id/bcrypt/scrypt — never plaintext, reversible encryption, or a bare fast hash (MD5/SHA-1/SHA-256).
- MFA is available and app-based/WebAuthn is preferred over SMS-only for sensitive accounts.
- Session IDs are high-entropy random, rotated on login/privilege change, and invalidated server-side on logout.
- Session cookies set `HttpOnly`, `Secure`, and an appropriate `SameSite` value.
- JWTs pin the expected algorithm server-side and never trust an `alg` value from the token itself; `none` is rejected.
- Secret comparisons (tokens, signatures) use constant-time comparison, not `==`.
- Login/reset/MFA endpoints are rate-limited, and login failure messages don't reveal whether an account exists.
- No home-grown authentication, session, or token scheme replaces a maintained library.
**Authorization**
- Every request re-verifies that the authenticated user actually owns/may access the specific resource by ID (no IDOR).
- Authorization logic is centralized in one policy/guard layer, not duplicated ad hoc per handler.
- Every verb (GET/POST/PUT/PATCH/DELETE) and every nested/related resource has its own ownership/permission check.
- Server-side authorization is authoritative; UI-hidden buttons/routes are never treated as a security control.
- Role/permission comparisons use strict, type-safe equality against a validated enum, never loose or coerced comparison.
- Authorization checks fail closed (deny) on any unexpected error, not fail open (allow).
- Multi-tenant queries always scope by a server-derived tenant ID, never a client-supplied one.
**Secrets Management**
- No secret, API key, or credential is hardcoded in source code, "temporarily" or otherwise.
- Secrets load from environment variables or a secret manager at runtime; `.env` files are gitignored with an `.example` committed instead.
- Any secret ever committed to version control is treated as compromised and rotated immediately.
- Logs, error messages, and crash reports redact known-sensitive fields (passwords, tokens, auth headers) before writing.
- Client-side/frontend code never ships a server-only secret; only scoped, public-safe keys reach the browser.
- CI/CD secrets use the platform's masked-secret feature, scoped narrowly, never echoed in build logs.
- Secrets are rotated on a schedule and immediately on any suspected exposure, with the old value explicitly revoked.
**Dependency Security**
- Dependencies are scanned for known vulnerabilities in CI, and high/critical findings are triaged, not ignored.
- Lockfiles are committed and installs use the lockfile-strict command (`npm ci`, not `npm install`).
- New dependencies (and their maintainers/activity) are sanity-checked before adding; package names are verified character-by-character.
- Any AI-suggested or unfamiliar package is confirmed to actually exist on the official registry before installing.
- Container base images and CI actions are pinned to immutable versions/digests, not mutable tags like `latest`.
- Install-time scripts and CLI tools from public registries are treated with the same suspicion as any other untrusted code.
**Input Validation & Sanitization**
- All input is validated server-side in full, regardless of what client-side validation already ran.
- Validation uses allowlists (what's valid) rather than denylists (what's forbidden) wherever practical.
- Uploaded files are type-checked by content sniffing, not by trusting the extension or `Content-Type` header.
- Uploads are stored under generated filenames, size-limited, and served from a cookie-less, non-executing origin.
- Request bodies bind onto models through an explicit field allowlist — no mass-assignment of arbitrary client fields.
- Untrusted data is deserialized only with safe, non-code-executing parsers (e.g., JSON), never unrestricted native serialization formats.
- Field length, numeric range, array size, and JSON nesting depth all have explicit bounds.
**Cryptography**
- All cryptographic operations go through a vetted library — no custom encryption, hashing, or key-exchange logic.
- Symmetric encryption uses an authenticated mode (AES-GCM/ChaCha20-Poly1305); ECB mode is never used.
- Security-relevant randomness (tokens, keys, nonces) comes from a CSPRNG, never a general-purpose PRNG like `Math.random()`.
- Nonces/IVs are unique per encryption operation and never reused with the same key.
- Encryption keys are stored separately from the data they protect, in a secret manager/KMS, never hardcoded or co-located.
- Base64/hex/URL encoding is never mistaken for encryption or treated as providing confidentiality.
**Transport Security**
- The entire application is served over HTTPS, with HTTP requests redirected, not just the login/payment pages.
- HSTS is sent with a meaningful `max-age` (and `includeSubDomains` where appropriate).
- TLS certificate verification is never disabled to silence an error — the actual cause (expired cert, missing CA, wrong hostname) is fixed instead.
- No page loads mixed content (HTTP subresources on an HTTPS page).
- Outdated TLS versions (SSLv2/3, TLS 1.0/1.1) and weak cipher suites are disabled on the server.
**Common Web Vulnerabilities**
- Every state-changing request is protected against CSRF (anti-CSRF token and/or `SameSite` cookies).
- Server-side fetches of user-influenced URLs validate the destination (allowlist, block private/metadata IP ranges) to prevent SSRF.
- Sensitive-action pages send `X-Frame-Options`/`frame-ancestors` to prevent clickjacking.
- Redirect targets are validated against an allowlist or restricted to relative paths — no open redirects.
- CORS uses an explicit origin allowlist; `*` is never combined with credentialed (cookie-bearing) requests.
- A baseline security header set (CSP, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`) is sent on every response.
- Expensive and authentication-related endpoints are rate-limited per account and per client, not just per IP.
- Limited-use/limited-quantity operations (coupons, inventory, one-time codes) are enforced with atomic database operations, not check-then-act code.
**Security Logging & Error Handling**
- End users see a generic error message with a correlation ID; full stack traces and internals stay server-side only.
- Debug/verbose error pages are confirmed disabled in production, not merely assumed to be.
- A dedicated, append-only-where-possible audit log records authentication events, permission changes, and sensitive-data access.
- No password, full token, or other sensitive field is ever written to logs in plaintext, including failed-login attempts.
- User-controlled values written to logs are contained as structured fields, not concatenated into free-text log lines (avoiding log injection).
- Security-relevant log patterns (repeated failed logins, spikes in denied requests) trigger automated alerts, not just passive storage.
**AI-Assistant-Specific Review Points**
- Any suggestion to disable TLS verification, catch-and-ignore a security exception, or add a lint-suppression comment is treated as a signal to find the real cause, not accepted as the fix.
- Generated code is checked for hardcoded-looking secrets before being accepted, even in "temporary" or example form.
- Generated database code is checked for raw string-built SQL when the project already has a safe ORM/query-builder path.
- Every additional or newly generated code path touching a resource (bulk endpoints, alternate API versions) gets the same authorization check as the original path.
- Role/permission comparisons in generated code use strict equality against a validated enum, not loose or user-influenced string matching.
- Generated CORS and infrastructure (security group/IAM) suggestions are checked for overly broad access used only to silence an error.
- Any algorithm, library, or random-number choice in generated crypto code is cross-checked against current guidance, not assumed correct because it looks familiar.
- Every package a suggestion introduces is verified to exist on the official registry before it's installed.
- Diffs from generated changes are read for what was *removed* or *loosened*, not only what was added, especially around auth, validation, and crypto.


## Part 16 — AI-Generated Writing & Content Slop

- No contextless scene-setting opener ("in today's fast-paced world...").
- Lead with the most important/specific thing, not a warm-up.
- Cut delve, boast, testament, tapestry, realm, foster, elevate, unlock, unleash, leverage, utilize, robust, seamless, holistic, cutting-edge, game-changer wherever a plainer word works.
- Cut "it's important to note," "needless to say," and similar throat-clearing lead-ins.
- Use "not just X, but Y" sparingly — only where the contrast is real, not as a sentence template.
- Don't force every topic into a numbered/bulleted list structure.
- Don't default to exactly three examples/reasons/steps regardless of what's actually true.
- Vary section length and paragraph rhythm to match actual content weight.
- Calibrate hedging to actual uncertainty — plain language for confident claims, specific caveats for uncertain ones.
- Don't manufacture false "on the other hand" balance for settled questions.
- Lead with the direct answer, then add caveats — don't bury the answer in qualification.
- Cut filler transitions (moreover, furthermore, in conclusion) used as a rhythm rather than a real connector.
- Don't restate the previous paragraph before adding a new one.
- Don't summarize-restate the intro as the conclusion — end on something new or just stop.
- No vague, noncommittal closing lines ("only time will tell," "the future looks bright").
- Use em dashes deliberately, not as reflexive default punctuation in most sentences.
- Use bold/italic sparingly enough that emphasis still means something.
- Match heading capitalization to actual house style instead of defaulting to Title Case.
- Skip decorative emoji in headings/bullets unless the context genuinely calls for it.
- Skip exclamation points as a substitute for actually interesting content.
- Write in a specific, real voice rather than a flattened corporate-neutral register.
- Don't perform enthusiasm the content doesn't earn.
- Replace vague intensifiers ("very," "significantly") with real numbers when the number is known.
- Prefer one specific, real example over several generic ones.
- Don't keyword-stuff or pad length to hit a target word count.
- Don't write a headline that overpromises relative to the actual content.
- Read drafts aloud and test-delete the first/last paragraph before calling a piece done.


## Part 17 — AI-Generated Design & UI "Slop" Tells

An "AI Slop Detector" pass — run through this before calling any UI work done. Each item should have a real, stateable reason behind it; if the honest answer to any item is "no reason, it just ended up that way," that's the one to fix first.
**Typography**
- [ ] Primary typeface was chosen for a stated reason, not left at the Inter/Roboto/Arial default.
- [ ] If a second (serif/display) typeface is used, its role in the brand's actual voice is clear, not just "looks premium."
- [ ] No italic-serif accent word dropped into an otherwise plain sans-serif headline with no real emphasis purpose.
- [ ] A real type scale exists (5+ defined steps) — headings, body, captions, and labels are visibly distinct, not just varying shades of the same size.
- [ ] Line-height and letter-spacing are tuned per size step, not a flat `1.5` applied everywhere regardless of text size.
- [ ] Fallback font stacks are considered, not left as a bare single font name.
**Color**
- [ ] The palette wasn't defaulted to the violet/indigo/magenta "AI gradient" family without a brand-specific reason.
- [ ] Gradients appear in at most one deliberate spot per screen, not stacked across background, button, border, and text simultaneously.
- [ ] No gradient text-fill on headlines used as a reflexive "make it pop" move.
- [ ] Dark-mode body text passes 4.5:1 contrast against its background, checked with a real tool, not by eye.
- [ ] All-caps tracked labels are reserved for short metadata, not applied to every label and header by reflex.
- [ ] Colored glow/box-shadow effects follow one consistent system, not a different color and blur per element with no shared rule.
- [ ] The accent color is a deliberate brand choice, not the component library's unmodified default blue.
- [ ] Color alone is never the only carrier of meaning (status, category, error) anywhere in the UI.
**Layout & Components**
- [ ] The hero isn't a default centered-text-plus-badge-pill template chosen purely because it's fast to produce.
- [ ] Any badge/pill above a headline contains real, specific, true information — not vague filler like "✨ AI-Powered."
- [ ] Feature cards vary in length and visual weight according to actual importance, not forced to a uniform line count.
- [ ] Colored top/left borders on cards map to a real status or category system, not applied for arbitrary visual variety.
- [ ] Numbered "1, 2, 3" steps are used only where the process is genuinely sequential, not applied reflexively to independent actions.
- [ ] A numbered sequence isn't artificially compressed to exactly three steps when the real process has more.
- [ ] Every stat/metric displayed (user counts, ratings, uptime %) is real and currently true — none are placeholder numbers "to replace later."
- [ ] Navigation and persistent UI icons come from a real icon system (SVG/icon font), not emoji standing in as icons.
- [ ] Glassmorphism/backdrop-blur is only used where something meaningful sits behind it and a real z-axis relationship exists.
- [ ] The component library's default theme (radius, palette, shadows, type) has been intentionally overridden, not shipped as-is.
- [ ] Border-radius scales sensibly with element size, not one flat radius value applied to every component regardless of scale.
- [ ] Drop shadows are applied by actual elevation logic (flush vs. floating), not the same faint shadow on every surface.
- [ ] The feature grid isn't forced into exactly three icon-boxes purely because three fits a tidy row.
- [ ] Every testimonial, rating, and "as featured in" logo is real and attributable — none are fabricated placeholder social proof.
- [ ] Decorative background shapes (blobs/waves), if present, connect to the actual brand — not a generic downloaded filler asset.
- [ ] Scroll-triggered animations, if used, are applied selectively where motion adds real meaning, not uniformly to every section by default.
- [ ] `prefers-reduced-motion` is respected for any non-essential animation.
**Functional / UX**
- [ ] Every form field is validated (client- and server-side) with specific, actionable inline error messages next to the relevant field.
- [ ] Required fields are visibly marked before submission, not discovered only after a failed submit.
- [ ] Every view with async data has explicit loading, empty, and error states designed and implemented — not just the happy path.
- [ ] Empty states explain why the view is empty and offer a clear next action, not a bare "No results."
- [ ] User-facing error messages are in plain language with a next step — no raw stack traces or bare HTTP codes shown to end users.
- [ ] Every interactive element has a visible focus state (`:focus-visible`, not suppressed with `outline: none` and nothing replacing it).
- [ ] The entire primary user flow is operable by keyboard alone — tabbed through and verified, not just clicked through with a mouse.
- [ ] Elements styled to look clickable are actually wired up to real functionality — nothing that looks like a button does nothing.
- [ ] Elements that are functionally interactive have a real hover/focus affordance — nothing clickable looks static.
- [ ] Semantic, native interactive elements (`<button>`, `<a>`, form controls) are used instead of styled `<div>`s with manual click handlers.
**Process**
- [ ] The brief or prompt included explicit negative constraints (what to avoid), not only a description of the desired outcome.
- [ ] Any reference design shared came with an explanation of *why* it works, not just a link or screenshot.
- [ ] The work went through separate passes (structure, typography, color, motion) rather than being requested fully-polished in one shot.
- [ ] The brief gave a specific point of view or persona, not a generic "make it look good/modern/professional" instruction.
- [ ] Multiple genuinely divergent directions were considered before converging on one, rather than accepting the first output.
- [ ] A dedicated audit pass against this checklist happened after the work felt "finished," ideally by someone other than the builder.
- [ ] Every pattern used here has a stateable reason tied to this specific product/brand — this list is a filter for unconsidered defaults, not a mandate for one fixed aesthetic.


## Part 18 — General Design Principles

- A real type scale (modular ratio) is defined and used consistently — not arbitrary one-off font sizes.
- Body text line-height and measure (line length) are set for readability, not left at defaults.
- Body text meets at least 4.5:1 contrast; large text/UI components meet at least 3:1 — checked, not assumed.
- Color is never the only signal for meaning (status, error, required field) — paired with an icon, label, or pattern.
- Light and dark themes are each designed with real hierarchy, not one inverted onto the other.
- Spacing follows a consistent scale (e.g., 4pt/8pt grid) rather than arbitrary pixel values per component.
- Alignment is deliberate — edges line up; nothing floats at a random offset from its neighbors.
- Every interactive element has defined hover/focus/active/disabled/loading/error/empty states.
- Focus indicators are visible and never removed without a compliant custom replacement.
- Touch targets meet the ~44×44pt / 48×48dp minimum.
- Form fields have persistent visible labels, inline validation, and specific, actionable error messages.
- Motion has a functional purpose (state change, spatial relationship) and respects `prefers-reduced-motion`.
- Icons are drawn from one consistent style/weight/grid, and unclear icon-only actions carry a text label.
- Imagery is purposeful and specific to the context, not generic filler stock photography.
- Design tokens (spacing, color, type) are the single source of truth — no hardcoded one-off values bypassing them.
- Components have documented purpose, variants, and states rather than being one-off "snowflake" builds.
- Navigation is predictable and consistently placed; deep hierarchies have real wayfinding (breadcrumbs, clear back paths).
- Progressive disclosure is used for advanced/rare options instead of surfacing every option at once.
- Related-content or cross-link sections contain genuinely relevant items, placed where they're contextually useful — not a generic dump.
- Empty states, loading states, and error states are designed, not left as a blank screen or a raw stack trace.


## Part 19 — AI Agent & Skill Conventions

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
