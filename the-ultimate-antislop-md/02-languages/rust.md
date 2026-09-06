# Rust

## Ownership & Borrowing Idioms

- **DO:** Design function signatures around borrowing (`&T`, `&mut T`) by default, and only take ownership (`T`) when the function genuinely needs to consume, store, or move the value. Borrowing first is the idiomatic default and keeps callers' data usable after the call.
  ```rust
  // DON'T — takes ownership for no reason, forcing callers to clone
  fn total_len(items: Vec<String>) -> usize {
      items.iter().map(|s| s.len()).sum()
  }

  // DO — borrow; caller keeps ownership
  fn total_len(items: &[String]) -> usize {
      items.iter().map(|s| s.len()).sum()
  }
  ```
- **DON'T:** Reach for `.clone()` as a first response to a borrow checker error. A clone that "makes the error go away" often just hides a design problem — a lifetime that should be restructured, a scope that's too wide, or a value that should be borrowed instead of moved. Understand *why* the borrow checker is objecting before working around it.
- **DO:** Accept `&str` instead of `&String`, and `&[T]` instead of `&Vec<T>`, in function parameters. These are the more general slice/borrowed forms and let callers pass owned, borrowed, or literal data without an unnecessary conversion.
- **DO:** Use `Cow<'a, str>` (or `Cow<'a, [T]>`) when a function usually returns borrowed data but occasionally needs to return an owned, modified copy — it avoids allocating in the common case while still supporting the rare mutation path.
- **DON'T:** Fight a lifetime error by adding `'static` to make the compiler stop complaining. `'static` means "lives for the entire program," which is rarely what's actually needed and often just relocates the real bug (e.g., leaking memory via `Box::leak`, or forcing an owned value where a borrow was intended).
- **DO:** Let the compiler's lifetime elision rules do their job — most functions don't need explicit lifetime annotations at all. Add explicit lifetimes only when elision genuinely can't infer the relationship (e.g., a struct holding a reference, or a function returning a reference derived from more than one input).
  ```rust
  // Elision handles this fine — no annotation needed
  fn first_word(s: &str) -> &str {
      s.split_whitespace().next().unwrap_or("")
  }

  // Explicit lifetime needed: struct holds a borrowed reference
  struct Parser<'a> {
      input: &'a str,
      pos: usize,
  }
  ```
- **DO:** Prefer `Rc<RefCell<T>>` (or `Arc<Mutex<T>>` across threads) only when shared, mutable ownership is a genuine requirement (graphs, observer patterns, caches) — not as a default escape hatch to avoid thinking through ownership. Reach for it deliberately, and document why plain ownership/borrowing didn't work.
- **DON'T:** Store a `&T` reference inside a long-lived struct just to avoid a small, cheap clone (e.g., cloning an `Rc<str>` or a small `Copy` type). The resulting lifetime parameter often infects the whole struct's API and every place that constructs it, for a "savings" that wasn't worth the complexity.
- **DO:** Use `std::mem::take` or `std::mem::replace` to move a value out of a struct field or `&mut` location that must remain valid (e.g., inside a method that needs to consume `self.buffer` and leave a fresh empty one behind), instead of cloning.
  ```rust
  fn flush(&mut self) -> Vec<u8> {
      std::mem::take(&mut self.buffer) // leaves an empty Vec in its place
  }
  ```

## Error Handling — Result, Option, and `?`

- **DO:** Return `Result<T, E>` from any function that can fail for a reason the caller might want to handle or report, and use `Option<T>` for a value that may simply be absent with no further explanation needed (a lookup that may miss, an optional config field).
- **DO:** Use the `?` operator to propagate errors instead of manual `match`-and-return boilerplate. It keeps the happy path readable while still making every fallible call visible at the call site.
  ```rust
  // DON'T — verbose manual propagation
  fn read_config(path: &str) -> Result<Config, ConfigError> {
      let text = match std::fs::read_to_string(path) {
          Ok(t) => t,
          Err(e) => return Err(ConfigError::Io(e)),
      };
      match toml::from_str(&text) {
          Ok(cfg) => Ok(cfg),
          Err(e) => Err(ConfigError::Parse(e)),
      }
  }

  // DO — `?` with `From` conversions
  fn read_config(path: &str) -> Result<Config, ConfigError> {
      let text = std::fs::read_to_string(path)?;
      let cfg = toml::from_str(&text)?;
      Ok(cfg)
  }
  ```
- **DON'T:** Call `.unwrap()` or `.expect()` on a `Result`/`Option` in library or production code paths where the failure is reachable from untrusted input, I/O, or any external condition. A panic there takes down the whole task/thread (or process, without `catch_unwind`) for a condition the caller had no chance to handle.
- **DO:** Use `.expect("message")` over bare `.unwrap()` when a panic genuinely is the right response (an invariant that must hold, e.g. a `Mutex` that should never be poisoned) — the message documents *why* the invariant should hold, which is invaluable when it doesn't.
- **DO:** Implement `std::error::Error` (and `Display`) for custom error types, or use a crate like `thiserror` to derive that boilerplate, so errors compose cleanly with `?`, `Box<dyn Error>`, and other tooling that expects the standard trait.
  ```rust
  #[derive(thiserror::Error, Debug)]
  enum ConfigError {
      #[error("reading config file: {0}")]
      Io(#[from] std::io::Error),
      #[error("parsing config: {0}")]
      Parse(#[from] toml::de::Error),
  }
  ```
- **DO:** Use `anyhow::Result` (or an equivalent "erased" error type) in application/binary code where the caller just needs to report the failure, and reserve precise, enumerated error types (via `thiserror`) for library crates whose callers need to match on specific failure modes.
- **DON'T:** Define a single catch-all error enum with dozens of unrelated variants spanning multiple subsystems. Keep error types scoped to the module or crate boundary they describe, and convert between them explicitly (or via `#[from]`) at the boundary.
- **DO:** Use `.ok_or(...)`/`.ok_or_else(...)` to convert an `Option<T>` into a `Result<T, E>` with a meaningful error at the exact point where absence becomes a reportable failure, rather than unwrapping the option and letting a panic stand in for that error.
- **DON'T:** Swallow an error with `let _ = fallible_call();` without a comment explaining why the failure is safe to ignore. This is the Rust equivalent of Go's silently discarded error and hides the same class of bug.
- **DO:** Prefer combinators (`.map`, `.and_then`, `.unwrap_or_default`, `.context(...)` from `anyhow`) for short transformations, but fall back to an explicit `match` when the branches genuinely differ in behavior — don't force a deeply chained combinator expression just to avoid writing `match`.

## Traits & Generics

- **DO:** Model shared behavior with traits and use generics (`fn f<T: Trait>(x: T)`) or trait objects (`fn f(x: &dyn Trait)`) depending on whether you need static dispatch (monomorphized, faster, larger binary) or dynamic dispatch (one compiled version, runtime vtable lookup, needed for heterogeneous collections).
  ```rust
  // Static dispatch: monomorphized per concrete type
  fn render<W: std::fmt::Write>(target: &mut W, msg: &str) {
      let _ = write!(target, "{msg}");
  }

  // Dynamic dispatch: one function, works with any Renderer at runtime
  fn render_all(items: &[Box<dyn Renderer>]) {
      for item in items { item.render(); }
  }
  ```
- **DO:** Implement standard traits (`Debug`, `Clone`, `PartialEq`, `Default`, `From`/`TryFrom`, `Iterator`) for your own types wherever they make sense, since doing so lets your types plug into the rest of the ecosystem — printing, comparison, collection APIs, error conversion — for free.
- **DON'T:** Implement `Deref`/`DerefMut` on a type purely to get field-access-like ergonomics for an unrelated wrapper. `Deref` is meant to model "smart pointer to T" (like `Box<T>`); using it to fake inheritance between unrelated types produces surprising, hard-to-trace method resolution.
- **DO:** Use `From`/`Into` to define clean, infallible conversions between types, and `TryFrom`/`TryInto` when the conversion can fail — this integrates with `?` and gives callers an idiomatic, discoverable conversion API instead of a bespoke `to_x()`/`from_x()` method name.
- **DON'T:** Over-parameterize a function with generics "for flexibility" when only one concrete type is ever actually passed in the codebase. Unnecessary generics increase compile times, obscure the real contract, and often force `where` clause sprawl that a concrete type or a small trait wouldn't need.
- **DO:** Use trait bounds with `where` clauses to keep complex generic signatures readable, rather than cramming every bound into the angle brackets.
  ```rust
  fn merge<T>(a: T, b: T) -> T
  where
      T: Clone + PartialEq + std::fmt::Debug,
  {
      // ...
  }
  ```
- **DO:** Prefer returning `impl Trait` (e.g., `impl Iterator<Item = u32>`) from a function over a boxed trait object when there's exactly one concrete return type per code path — it avoids a heap allocation and dynamic dispatch while keeping the return type unnamed and simple.
- **DON'T:** Design a trait with dozens of required methods and no default implementations, forcing every implementer to write boilerplate for behavior most of them share. Provide default method bodies on the trait wherever a sensible default exists, and keep the truly required subset small.
- **DO:** Use marker traits and the newtype pattern (wrapping a primitive in a tuple struct, e.g., `struct UserId(u64);`) to get compile-time distinctions between values that share a representation but shouldn't be interchangeable (an order ID and a user ID both being `u64` is a classic source of silent bugs without a newtype).

## Unsafe Code Discipline

- **DO:** Treat `unsafe` as an explicit, auditable escape hatch used only when there is no safe way to express the operation — FFI calls into C, certain low-level performance-critical data structures, implementing a safe abstraction over raw memory. Every `unsafe` block should be small and its invariants documented.
  ```rust
  /// SAFETY: `ptr` is non-null, properly aligned for `T`, and points to
  /// an initialized `T` that this function has exclusive access to for
  /// the duration of the call (guaranteed by the caller contract above).
  unsafe fn read_raw<T>(ptr: *const T) -> T {
      unsafe { std::ptr::read(ptr) }
  }
  ```
- **DON'T:** Reach for `unsafe` to silence a borrow checker error you don't fully understand. Nearly every legitimate data structure need (linked lists, graphs, caches) has a safe pattern (`Rc`/`RefCell`, arena/index-based references, `Vec`-backed slotmaps) that avoids `unsafe` entirely; use those first.
- **DO:** Attach a `// SAFETY:` comment directly above every `unsafe` block explaining exactly which invariants the surrounding code relies on and why they hold. This is close to a project-wide convention in serious Rust codebases (and enforced by `clippy::undocumented_unsafe_blocks` in strict configurations) precisely because unsafe code is where the type system stops checking for you.
- **DON'T:** Wrap a large function body in `unsafe { ... }` when only a few lines actually require it. Minimize the unsafe surface so reviewers (and future you) can audit exactly what's unchecked, rather than having to re-verify an entire function's safe-looking logic as if it might not be.
- **DO:** Wrap `unsafe` internals behind a safe public API whenever possible, upholding the invariants internally so callers of your crate never need `unsafe` themselves and can't misuse the internals to cause undefined behavior.
- **DON'T:** Use `unsafe` to bypass bounds checking (`get_unchecked`) or overflow checking as a default optimization. Reach for it only after profiling shows the checked version is a real bottleneck, and only with a clear, documented proof that the access is always in bounds.
- **DO:** Run `cargo miri test` on `unsafe`-containing crates where feasible — Miri detects undefined behavior (invalid memory access, misaligned reads, violated aliasing rules) that compiles cleanly but is still UB, which is exactly the class of bug `unsafe` opens the door to.
- **DON'T:** Assume `unsafe` code that "seems to work" in testing is actually sound. Undefined behavior in Rust (as in C/C++) can appear to work by accident on a given compiler version/optimization level and then break silently after a toolchain upgrade; soundness has to be reasoned about explicitly, not just observed.
- **DO:** Prefer well-audited crates (e.g., `bytemuck`, `zerocopy`) for common unsafe-adjacent operations like reinterpreting bytes as a struct, rather than hand-rolling `transmute`. These crates encode the alignment/validity invariants once, correctly, instead of every call site re-deriving them.

## Concurrency — Send/Sync, Arc/Mutex

- **DO:** Trust the `Send`/`Sync` auto-trait system rather than fighting it — if the compiler says a type isn't `Send`, that's telling you it genuinely isn't safe to move across threads (e.g., it contains an `Rc<T>` or a raw pointer), and the fix is usually to swap in the thread-safe equivalent (`Arc` instead of `Rc`), not to force it with `unsafe impl Send`.
- **DON'T:** Write `unsafe impl Send for MyType {}` (or `Sync`) to work around a compiler error without actually verifying the type's internals are safe to share/move across threads. This is one of the most dangerous shortcuts in Rust — it turns off a real safety guarantee and can introduce data races the type system would otherwise have caught.
- **DO:** Use `Arc<Mutex<T>>` for shared, mutable state across threads, and keep the critical section (the code holding the lock) as short as possible — acquire, mutate, drop the guard — mirroring the same discipline as any other language's locking.
  ```rust
  use std::sync::{Arc, Mutex};

  let counter = Arc::new(Mutex::new(0));
  let mut handles = vec![];
  for _ in 0..4 {
      let counter = Arc::clone(&counter);
      handles.push(std::thread::spawn(move || {
          let mut n = counter.lock().unwrap();
          *n += 1;
      }));
  }
  for h in handles { h.join().unwrap(); }
  ```
- **DON'T:** Call `.lock().unwrap()` and treat the `unwrap()` as boilerplate you don't need to think about. It's unwrapping a `PoisonError`, which occurs when another thread panicked while holding the lock — deciding whether to propagate that panic, recover the poisoned data, or use `into_inner()` is a real design decision, not a formality.
- **DO:** Prefer message passing (`std::sync::mpsc`, or `tokio::sync::mpsc` in async code) over shared-state locking when the concurrency pattern is naturally a pipeline or producer/consumer — it sidesteps lock contention and deadlock risk entirely for that shape of problem.
- **DON'T:** Hold a `Mutex` guard across an `.await` point in async code (this either won't compile with a non-`Send` guard, or will compile but block the executor thread for the lock's duration). Use an async-aware mutex (`tokio::sync::Mutex`) when the critical section genuinely spans an await, or restructure to drop the guard before awaiting.
- **DO:** Use `tokio::spawn` (or your async runtime's equivalent) for concurrent async tasks and make sure every spawned task's `JoinHandle` is either awaited, explicitly detached with a documented reason, or tracked by a supervising structure (`JoinSet`) — an unawaited, untracked handle silently drops any panic or error the task produced.
- **DO:** Reach for `RwLock` instead of `Mutex` only when reads genuinely dominate writes and profiling shows lock contention; `RwLock` has higher overhead per-acquisition than `Mutex` and can starve writers under sustained read load, so it isn't a free upgrade.
- **DON'T:** Assume that because a type is `Clone`, cloning it inside a hot concurrent loop (e.g., cloning an `Arc` is cheap, but cloning the `T` inside a `Mutex<T>` after locking is not) is free. Know specifically what's being cloned — an `Arc` clone is a cheap refcount bump; cloning the underlying data is not.

## Cargo & Crate Conventions

- **DO:** Keep `Cargo.toml` dependencies minimal and specific — add a crate because the code actually needs it, and prefer well-maintained, widely used crates (checked via download counts, recent commits, and `cargo audit`) over obscure or unmaintained ones for anything security- or correctness-sensitive.
- **DON'T:** Pin overly loose or overly strict version requirements without reason. `"*"` invites breakage from any future release; `"=1.2.3"` (exact pin) blocks legitimate patch updates. The Cargo default caret requirement (`"1.2.3"`, meaning `>=1.2.3, <2.0.0`) is correct for most dependencies under semver.
- **DO:** Commit `Cargo.lock` for binary crates/applications (to guarantee reproducible builds) and omit it from version control for library crates (so downstream consumers resolve dependencies against their own constraints) — this is the standard convention, not an arbitrary choice.
- **DO:** Organize a multi-binary or multi-purpose project as a Cargo workspace with clearly separated crates (`crates/core`, `crates/cli`, `crates/server`), rather than one monolithic crate with feature flags trying to do everything.
  ```toml
  # workspace Cargo.toml
  [workspace]
  members = ["crates/core", "crates/cli", "crates/server"]
  resolver = "2"
  ```
- **DON'T:** Put executable-only logic in `lib.rs` just because it was easier to iterate on, when the crate is meant to be reused as a library. Keep `main.rs` thin (argument parsing, wiring) and put testable logic in `lib.rs`/submodules, mirroring the same "thin entrypoint" discipline as Go's `main` package.
- **DO:** Use Cargo features to make optional functionality (and optional heavy dependencies) opt-in, and keep the default feature set minimal so consumers who don't need the extra functionality don't pay its compile-time or binary-size cost.
- **DO:** Write meaningful `///` doc comments on every public item, including a runnable example where practical — `cargo doc` and `cargo test --doc` turn those examples into both documentation and executable tests, so they can't silently rot.
- **DON'T:** Publish or rely on a crate's internal (non-`pub`) module layout as a stable API. Only what's actually `pub` (and not `pub(crate)`) is part of the crate's contract; use `pub(crate)`/private modules liberally to keep the real public surface intentional and small.
- **DO:** Run `cargo audit` (checks against the RustSec advisory database) and `cargo outdated` periodically in CI or on a schedule to catch known-vulnerable or stale dependencies before they become a liability.

## Clippy & Rustfmt

- **DO:** Run `cargo fmt` on every file before committing, using the project's `rustfmt.toml` if one is checked in, so formatting is consistent and never part of code review discussion.
- **DO:** Run `cargo clippy --all-targets --all-features -- -D warnings` in CI so lint warnings fail the build rather than silently accumulating. Clippy catches a wide range of non-idiomatic patterns and outright bugs that `rustc` alone won't flag.
  ```rust
  // clippy: `needless_return`
  fn square(x: i32) -> i32 {
      return x * x; // DON'T — trailing `return` on the last expression
  }
  fn square(x: i32) -> i32 {
      x * x // DO — implicit tail expression
  }
  ```
- **DON'T:** Blanket-allow an entire clippy lint category (`#![allow(clippy::all)]`) at the crate root to make warnings disappear. Address individual lints, or add a narrowly scoped `#[allow(clippy::specific_lint)]` with a comment justifying why that one instance is intentional.
- **DO:** Pay particular attention to clippy's correctness-leaning lints (`clippy::correctness`, enabled by default and denying by default in recent versions) — these flag patterns that are very likely actual bugs, not just style preferences, and should essentially never be suppressed.
- **DON'T:** Ignore `clippy::unwrap_used` / `clippy::expect_used` (available in the `restriction` lint group) in library crates or production services without a project-level decision to allow them; enabling this group forces a deliberate choice at every panic-capable call site instead of an accidental one.
- **DO:** Let clippy's suggested idioms replace verbose patterns — e.g., `.iter().any(...)` instead of a manual loop with a boolean flag, `if let`/`let else` instead of a `match` with one meaningful arm, `.map_or` instead of `.map(...).unwrap_or(...)`.
- **DO:** Configure `rustfmt` and `clippy` settings in versioned project files (`rustfmt.toml`, `clippy.toml`, or lint attributes in `lib.rs`) rather than relying on each contributor's local defaults, so CI and local runs agree.

## Testing

- **DO:** Write unit tests in a `#[cfg(test)] mod tests { ... }` block colocated with the code they test — this is the standard Rust convention and keeps tests close to the implementation they exercise, with access to private items.
  ```rust
  fn parse_port(s: &str) -> Result<u16, std::num::ParseIntError> {
      s.parse()
  }

  #[cfg(test)]
  mod tests {
      use super::*;

      #[test]
      fn parses_valid_port() {
          assert_eq!(parse_port("8080").unwrap(), 8080);
      }

      #[test]
      fn rejects_non_numeric() {
          assert!(parse_port("abc").is_err());
      }
  }
  ```
- **DO:** Put integration tests that exercise the crate's public API as an external consumer would under `tests/` at the crate root — each file there is compiled as a separate crate linking only against the library's public interface, catching accidental reliance on private internals.
- **DON'T:** Write a single sprawling `#[test]` function asserting many unrelated behaviors. Split tests by behavior so a failure's name immediately tells you what broke, the same rationale as Go's named subtests.
- **DO:** Use `Result<(), E>` as a test function's return type (with `?` inside) instead of `.unwrap()` chains, when the test's own readability benefits from early, clear propagation of a setup failure.
- **DO:** Write doc-tests (runnable code fenced in `///` doc comments) for public API examples — `cargo test` runs them automatically, so documentation examples can't silently go stale or stop compiling.
- **DON'T:** Rely solely on `assert_eq!`/`assert!` for property-style invariants (e.g., "serialization round-trips for any input") that are naturally suited to randomized testing. Use `proptest` or `quickcheck` for those, since a handful of hand-picked example cases won't reliably surface edge cases like empty strings, extreme integers, or Unicode boundaries.
- **DO:** Use `#[should_panic(expected = "...")]` sparingly and only to test a genuinely intended panic (an invariant violation), not as a substitute for testing a `Result::Err` path, which should be asserted directly instead.
- **DO:** Run tests with `cargo test` in CI across the same feature-flag combinations the crate actually ships (`--no-default-features`, `--all-features`), since a feature-gated code path with no dedicated CI run can silently break unnoticed.

## Common AI-Assistant Mistakes in Rust

- **DON'T:** Add `.clone()` reflexively every time the borrow checker complains, without diagnosing the actual ownership issue. This is the single most common "AI slop" pattern in Rust output — code that compiles but silently does far more allocation and copying than the equivalent, correctly borrowed version.
  ```rust
  // DON'T — clones to dodge a borrow error instead of restructuring
  fn process(items: &Vec<String>) -> Vec<String> {
      let items = items.clone(); // unnecessary clone
      items.into_iter().filter(|s| !s.is_empty()).collect()
  }

  // DO — borrow through, only allocate what's actually new
  fn process(items: &[String]) -> Vec<String> {
      items.iter().filter(|s| !s.is_empty()).cloned().collect()
  }
  ```
- **DON'T:** Sprinkle `unsafe` blocks to "make the borrow checker happy" or to hit a performance target without profiling first. Generated code that reaches for raw pointers, `transmute`, or `unsafe impl Send`/`Sync` to route around a compile error is very often introducing real undefined behavior, not just satisfying an overly strict checker.
- **DON'T:** Invent crate APIs, function names, or crate names that sound plausible for a popular ecosystem (a hallucinated method on `tokio`, `serde`, or `reqwest` that doesn't exist, or a crate name that resembles a real one but isn't) without verifying against actual documentation, `cargo doc`, or a successful compile.
- **DON'T:** Use `.unwrap()`/`.expect()` throughout example or "production-ready" code and label it production-ready. Generated code should distinguish between a quick illustrative snippet (where `.unwrap()` is acceptable shorthand) and code presented as deployment-ready (where fallible calls need real `Result` handling).
- **DON'T:** Generate an `unsafe impl Send for T {}` or `unsafe impl Sync for T {}` as a generic fix for a "cannot be sent between threads safely" compiler error. Verify the type's actual internals are thread-safe first; if they contain an `Rc`, a `RefCell` without synchronization, or a raw pointer to non-thread-safe data, this is actively unsound, not just unidiomatic.
- **DON'T:** Write async code that holds a `std::sync::Mutex` guard across an `.await` point (or, worse, add `unsafe` to force it to compile). Either restructure so the lock is dropped before awaiting, or switch to an async-aware mutex.
- **DON'T:** Overuse generics and trait bounds to make a function "maximally flexible" when the codebase only ever calls it with one concrete type. This inflates compile times and error message complexity for no real benefit — default to concrete types and generalize only when a second real caller appears.
- **DON'T:** Assume a `Result`-returning function has been fully handled just because `?` is present somewhere in the function. Check that the function's own return type actually propagates the error correctly to a caller equipped to handle it, rather than `?`-ing inside a function whose signature doesn't return `Result` at all (which won't even compile) or silently converting to a less specific error type that loses useful context.
- **DON'T:** Claim generated `unsafe` code, FFI bindings, or lock-free data structures are "sound" or "safe" without actually working through the invariants (or running Miri/sanitizers). State the specific invariants relied upon and flag anything unverified rather than asserting soundness as a default.

## Quick Checklist
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
