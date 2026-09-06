# Swift

## Idiomatic Style (Naming per Swift API Design Guidelines)

- **DO:** Name types, protocols, and enum cases with `UpperCamelCase`, and name properties, methods, functions, and enum associated-value labels with `lowerCamelCase`. Consistent casing is how Swift signals what kind of symbol you're looking at without needing to check a declaration.
- **DO:** Conform simple, closed-set enums to `CaseIterable` when code needs to enumerate every case (populating a picker, validating input against the full set), instead of hand-maintaining a separate array of all cases that can silently drift out of sync when a case is added or removed.
  ```swift
  enum Weekday: CaseIterable { case monday, tuesday, wednesday, thursday, friday, saturday, sunday }
  // Weekday.allCases stays correct automatically as cases are added/removed
  ```
- **DO:** Use `@available(*, deprecated, message: "Use newMethod(_:) instead")` with a specific, actionable message when deprecating an API, so callers (and the compiler's warning) point directly at the replacement instead of leaving them to guess what to migrate to.
- **DO:** Name boolean function parameters so the call site reads unambiguously even without an IDE's parameter-name hint — an argument label like `animated:` in `setVisible(true, animated: true)` disambiguates what each `true` means, whereas two adjacent unlabeled booleans are a common source of "which one was which" mistakes at the call site.
- **DON'T:** Use `_` to discard a value that a reader would actually want visible for clarity (discarding a `Result`'s error, ignoring a completion handler's status) — reserve underscore-discarding for values that are genuinely irrelevant (an unused loop index in `for _ in 0..<3`), and handle or at least name values that carry real information.
- **DO:** Break a long function signature across multiple lines, one parameter per line, once it stops fitting comfortably on one line, so each parameter's label, type, and default value are individually scannable rather than requiring horizontal scrolling or wrapping mid-parameter.
  ```swift
  func configure(
      title: String,
      subtitle: String? = nil,
      style: ButtonStyle = .primary,
      isEnabled: Bool = true
  ) { ... }
  ```
- **DO:** Prefer a well-named `struct`/`enum` over an untyped `[String: Any]` dictionary for structured internal data whose shape is known at compile time — the dictionary form throws away type checking, autocomplete, and refactoring-tool support that a proper type gets for free, and it should be reserved for genuinely dynamic, externally-sourced data (like raw JSON before decoding).
- **DO:** Organize a Swift Package Manager package's public API deliberately — mark internal implementation types `internal` (the default) and expose only the intended public surface as `public`, since everything left `public` in a library target becomes a permanent part of that package's contract for every consumer once it ships.
- **DO:** Use key paths (`\User.name`) with APIs designed to accept them (`sorted(by: \.name)` via a custom comparator, SwiftUI bindings, Core Data predicates) instead of writing an equivalent closure by hand, since a key path is both more concise and, for simple property access, more efficient than a closure capturing `self`.
  ```swift
  // Both work, but the key path is more concise for pure property access
  users.sorted { $0.name < $1.name }
  users.sorted(using: KeyPathComparator(\.name))
  ```
- **DO:** Use `String`'s native Unicode-correct APIs (`count`, `Character`, grapheme-cluster-aware iteration) rather than assuming a `String` behaves like a fixed-width array of bytes/UTF-16 code units the way strings do in several other languages — Swift's `String` is intentionally Unicode-correct by default, and reaching for `NSString`/UTF-16-index-based APIs to "simplify" string handling often reintroduces bugs with emoji, combining characters, and non-Latin scripts that Swift's default `String` API was specifically designed to avoid.
- **DO:** Use `Data`, `URL`, and `FileManager`'s modern async-friendly APIs (`Data(contentsOf:)` variants, `URLSession`'s `async` data/download methods) for file and network I/O instead of older synchronous, error-code-based file APIs, matching the concurrency model the rest of a modern Swift codebase uses.
- **DO:** Optimize names for clarity at the point of use, not brevity in isolation. A call site like `x.insert(y, at: z)` reads as a sentence; a name saved by dropping the argument label almost always costs more in readability than it gains.
  ```swift
  // BAD — ambiguous at call site
  func insert(_ item: Element, _ index: Int)
  employees.insert(newHire, 3)

  // GOOD — reads like English at the call site
  func insert(_ item: Element, at index: Int)
  employees.insert(newHire, at: 3)
  ```
- **DO:** Name methods and properties according to their side effects: mutating verbs like `sort()`, `append()`, `remove()` for in-place mutation, and the corresponding noun/participle forms like `sorted()`, `appending()`, `removing()` for non-mutating variants that return a new value. This pairing is a load-bearing convention in the standard library and callers rely on it to reason about mutation without reading the implementation.
- **DON'T:** Prefix types or protocols with Hungarian-style markers (`CJUserManager`, `IUserRepository`, `TUser`). Swift's module system and type inference already disambiguate namespaces; such prefixes are a carryover from other languages and read as noise.
- **DO:** Give protocols that describe capabilities an `-able`, `-ible`, or `-ing` suffix (`Equatable`, `Comparable`, `ProgressReporting`), and give protocols that describe a role or a thing a noun name (`Collection`, `Sequence`). This distinguishes "this type can do X" protocols from "this type is an X" protocols at a glance.
- **DO:** Prefer `let` over `var` for every declaration unless the value is actually reassigned or mutated later. Defaulting to immutability documents intent and lets the compiler catch accidental mutation.
- **DON'T:** Abbreviate identifiers to save keystrokes (`usr`, `mgr`, `cfg`, `idx` outside of the tightest local loop scope). Swift favors descriptive names over C-style abbreviation, and an unfamiliar abbreviation costs every future reader more time than it ever saved the author.
- **DO:** Name Boolean properties and methods so they read as assertions (`isEmpty`, `hasSuffix(_:)`, `canEdit`), not as ambiguous nouns or verbs. `user.isActive` is unambiguous; `user.active` invites confusion with a non-Boolean property.
- **DO:** Follow the standard capitalization convention for acronyms: capitalize them fully when they start a name segment that isn't the very first word (`userID`, `htmlParser` → `HTMLParser` as a type, `urlString`), matching Apple's own frameworks so mixed codebases stay consistent.
- **DON'T:** Use `AnyObject`-typed or stringly-typed APIs as a substitute for real types when a concrete Swift type or enum would express the same intent more safely. A `String` "status" parameter with implicit magic values (`"pending"`, `"done"`) throws away the compiler's exhaustiveness checking that an `enum Status` would give for free.
- **DO:** Mark every declaration with the narrowest access level that works — `private`/`fileprivate` for implementation details, `internal` (the default) for module-internal API, `public`/`open` only for what other modules genuinely need. Wide-open access by default turns every stored property into part of your public contract, which makes later refactors riskier than they need to be.
- **DO:** Favor protocol-oriented composition — small, focused protocols plus extensions supplying default implementations — over deep class inheritance hierarchies. Swift's standard library itself is built this way (`Sequence`, `Collection`, `Equatable`), and it keeps behavior reusable without forcing an "is-a" relationship that doesn't really hold.
- **DON'T:** Use force casts (`as!`) to coerce a type when the cast isn't structurally guaranteed to succeed. Prefer `as?` with proper handling of the failure case, exactly as with force-unwrapping optionals — a failed force cast crashes the same way a failed force-unwrap does.
- **DO:** Use trailing closure syntax for the final closure argument, and multiple trailing closures (Swift 5.3+) when a call takes more than one closure parameter, instead of naming every closure argument inline when the call already reads clearly without labels.
- **DO:** Prefer computed properties over methods that take no arguments and simply return a derived value with no side effects (`var isValid: Bool { ... }` rather than `func isValid() -> Bool`). This matches how the standard library distinguishes "compute a value" from "perform an action."
- **DO:** Give initializer and function parameters sensible default values where a single "usual" case exists, so most call sites can omit them, rather than forcing every caller to pass every argument explicitly.
  ```swift
  // Callers that don't need custom retry behavior stay simple:
  func fetch(url: URL, retries: Int = 3, timeout: TimeInterval = 30) async throws -> Data
  ```
- **DON'T:** Let a single file or type grow into a dumping ground for unrelated functionality ("God object" view controllers, massive `Utils.swift` files with dozens of unrelated free functions). Split by responsibility into extensions or separate types even within the same module, so related code stays discoverable and unrelated code stays decoupled.
- **DO:** Use `some Protocol` (opaque return types) for functions that return "a specific but unspecified conforming type," such as SwiftUI `body` properties or generic factory functions, instead of returning a boxed `AnyProtocol` existential when the concrete type is stable and known at compile time. It preserves type identity and avoids the indirection cost of existential boxing.
- **DO:** Run a linter (SwiftLint or SwiftFormat) with a shared, checked-in configuration so naming, spacing, and common anti-patterns are enforced automatically and consistently across contributors, rather than relying on every reviewer to catch style drift by eye.
- **DO:** Use property observers (`willSet`/`didSet`) sparingly and only for genuine side effects tied to a property's own change (updating a dependent cache, triggering a UI redraw), not as a place to hide unrelated business logic that would be clearer as an explicit method call — an observer fires on every assignment, including ones a reader might not expect to trigger it.
- **DO:** Mark a stored property `lazy` when its initial value is expensive to compute and isn't always needed, so the computation is deferred until first access instead of running unconditionally at initialization time — but remember `lazy var` isn't thread-safe by default, so guard concurrent first-access with an actor or a lock if the property can be touched from multiple tasks simultaneously.
- **DO:** Conform types to `Codable` (or the narrower `Encodable`/`Decodable` as needed) for straightforward JSON/plist serialization, and use `CodingKeys` to map between a wire format's naming convention (often `snake_case`) and Swift's `camelCase` property names, rather than hand-writing parsing logic.
  ```swift
  struct User: Codable {
      let id: Int
      let fullName: String

      enum CodingKeys: String, CodingKey {
          case id
          case fullName = "full_name"
      }
  }
  ```
- **DO:** Write a custom `init(from:)`/`encode(to:)` only when the wire format genuinely can't be expressed by synthesized `Codable` conformance plus `CodingKeys` (nested flattening, conditional decoding based on a discriminator field) — reach for the manual implementation as the exception, not the default starting point.
- **DO:** Use extensions to organize a type's conformances and grouped functionality (`extension User: Equatable { ... }`, `extension User { // MARK: - Formatting ... }`) rather than cramming every protocol conformance and every category of method into the primary type declaration.
- **DO:** Use `// MARK: -` comments to divide a file into navigable sections in Xcode's jump bar, especially once a type's declaration grows past a screenful, mirroring the discipline of `#pragma mark` in Objective-C.
- **DON'T:** Define a custom operator (`<+>`, `~=` overloads, etc.) for anything other than a well-established mathematical or domain notation the whole team will recognize on sight. A cryptic custom operator saves a few characters at the cost of every future reader needing to look up what it means.
- **DO:** Use generics (`func merge<T: Mergeable>(_ a: T, _ b: T) -> T`) to write one algorithm that works across multiple concrete types safely, instead of duplicating near-identical logic per type or falling back to `Any`/`AnyObject` and losing compile-time type checking.
- **DO:** Prefer `@discardableResult` explicitly on a function whose return value is often intentionally ignored (a builder-pattern method returning `self`, a logging call returning a status) rather than leaving callers to see a "result unused" warning on every ordinary call site.
- **DO:** Use protocol extensions to provide a default implementation shared across every conforming type, reserving the protocol requirement itself for the parts that genuinely vary per conforming type — this is the core mechanism behind Swift's "protocol-oriented programming" style and avoids duplicating the same boilerplate implementation in every conformer.
  ```swift
  protocol Loggable {
      var logTag: String { get }
      func log(_ message: String)
  }
  extension Loggable {
      func log(_ message: String) { print("[\(logTag)] \(message)") }
  }
  ```
- **DO:** Use `Self` (capitalized) inside a protocol or class hierarchy to refer to "whatever concrete type actually conforms/subclasses here," enabling patterns like a factory method that returns the correct subclass, distinct from `self` (lowercase), which refers to the current instance.
- **DON'T:** Over-genericize a type or function with type parameters "just in case" future flexibility is needed, when only one concrete type is ever actually used in practice. Unused generality adds cognitive overhead for every reader without a corresponding real benefit; add the generic parameter when a second concrete use case actually arrives.
- **DO:** Use result builders (`@resultBuilder`, the mechanism behind SwiftUI's `ViewBuilder`) when designing a genuinely declarative DSL of your own, but recognize this is an advanced, rarely-needed tool for typical application code — reach for it deliberately, not as a default way to make an API "feel nicer."
- **DO:** Prefer `if case`/`guard case` for matching a single enum case with associated values inline, rather than a full `switch` statement, when only one case actually needs to be handled and the rest should simply fall through.
  ```swift
  guard case .success(let value) = result else { return }
  ```

## Optionals Handling (Avoiding Force-Unwrap Abuse)

- **DON'T:** Force-unwrap an optional with `!` unless the surrounding code has already proven, structurally, that the value cannot be `nil` at that point (e.g., immediately after an `if let` in the same scope, or a compile-time constant). A force-unwrap on a value whose nil-ness depends on runtime state — network responses, user input, dictionary lookups, array indices — is a crash waiting for the one input path nobody tested.
  ```swift
  // BAD — crashes the app the first time the key is missing
  let age = json["age"] as! Int

  // GOOD — degrades gracefully, and the failure is visible
  guard let ageValue = json["age"] as? Int else {
      throw ParsingError.missingField("age")
  }
  ```
- **DO:** Use `guard let` for early-exit unwrapping at the top of a function, and `if let` when the unwrapped value is only needed inside a conditional branch. `guard` keeps the "happy path" unindented and puts the failure handling exactly where the precondition is stated.
- **DO:** Use optional chaining (`user?.address?.city`) to safely traverse a chain of optional properties, and pair it with nil-coalescing (`?? "Unknown"`) to supply a fallback in one expression instead of nesting several `if let`s.
- **DON'T:** Reach for implicitly unwrapped optionals (`var label: UILabel!`) as a general-purpose way to "unwrap without syntax." Reserve them for the narrow, well-understood cases where a framework guarantees initialization before use (Interface Builder outlets, values set in `viewDidLoad` before any other method can run) — everywhere else they reintroduce the exact crash risk `!` has, just deferred to declaration time.
- **DO:** Use `if let shortName = longOptionalName` (same-name shadowing, available since Swift 5.7) to unwrap without inventing throwaway names, keeping the unwrapped value's name meaningful at every use site.
- **DO:** Combine multiple optional bindings in a single `guard`/`if let` with commas, and add a `where` clause for extra conditions on the same line, rather than nesting several unwrap blocks.
  ```swift
  guard let user = currentUser,
        let email = user.email,
        email.contains("@") else {
      return
  }
  ```
- **DON'T:** Use `try?` to silently swallow an error you have no intention of handling, just to get an optional you can force-unwrap or ignore. This converts a specific, debuggable error into an untraceable `nil`; catch the error and log or handle it, or propagate it with `throws`.
- **DO:** Model "value that is genuinely absent" with `nil` and "value that failed to compute" with a thrown `Error`; conflating the two by returning `nil` from a throwing-shaped operation hides failure reasons that callers and future maintainers need.
- **DO:** Use `map`/`flatMap`/`compactMap` on `Optional` to transform a wrapped value in one expression instead of unwrapping just to re-wrap the result.
  ```swift
  // BAD — unwrap, transform, re-wrap by hand
  var displayName: String?
  if let name = user.name {
      displayName = name.uppercased()
  }

  // GOOD — one expression, same result
  let displayName = user.name.map { $0.uppercased() }
  ```
- **DON'T:** Check `if value != nil` and then force-unwrap the same value on the next line. This throws away the type-checker's ability to bind the unwrapped value directly and reintroduces the exact crash risk a `guard let`/`if let` eliminates.
- **DO:** Use `compactMap` when transforming a collection where some elements may fail to produce a value, so `nil` results are dropped automatically rather than needing to be filtered out separately.
- **DON'T:** Model a tri-state condition ("not answered yet" vs. "answered false") with a plain `Bool?` sprinkled through business logic without a clear contract for what `nil` means at each use site — it's easy to accidentally treat `nil` and `false` as equivalent in one branch and as distinct in another. Prefer a three-case `enum` when the "unknown" state has real behavioral meaning.
- **DO:** Use nil-coalescing chains (`primary ?? secondary ?? fallback`) to express a priority list of optional sources in one readable line, rather than a nested `if let` ladder that does the same thing less clearly.
- **DO:** Use `switch` with pattern matching on an optional (`switch value { case .some(let x): ...; case .none: ... }`, or more idiomatically `switch value { case let x?: ...; case nil: ... }`) when unwrapping is just one of several cases being handled, rather than layering a separate `if let` on top of an already-present `switch`.
- **DO:** Provide a sensible default via a computed property or method rather than sprinkling the same `?? someDefault` expression at every call site — centralizing the default in one place means changing it later doesn't require hunting down every duplicate.
  ```swift
  // Centralize the fallback once instead of repeating `?? "Guest"` everywhere
  extension User {
      var displayName: String { name ?? "Guest" }
  }
  ```
- **DON'T:** Return an optional from a function whose caller will unconditionally force-unwrap the result at every call site — if every caller treats the value as always present, the function's signature should not be advertising that it might not be, and if it genuinely can fail, the failure should be handled at each call site rather than force-unwrapped away.
- **DO:** Use `Optional`'s `zip`-like composition (pairing two optionals together, e.g. via `if let a, let b`) rather than force-unwrapping one after checking `!= nil` on the other, keeping the "both present" invariant enforced by the compiler rather than by a runtime assumption.
- **DO:** Reserve `nil` array/collection values for "the collection itself is genuinely absent" and prefer an empty collection (`[]`) for "there are zero items" — conflating "no items yet" with "the whole collection wasn't loaded" behind the same optional-vs-nil signal makes call sites guess which case they're in.
- **DON'T:** Use `Optional<Optional<T>>` (a doubly-nested optional, which can arise from chaining certain generic APIs) without explicitly flattening it — Swift's type system allows it, but the two levels of "absent" almost never both carry distinct meaning, and code that pattern-matches on it incorrectly is a common source of confusing bugs.
- **DO:** Use `Dictionary`'s `default:` subscript form (`counts[key, default: 0] += 1`) instead of a manual `if let`/nil-coalescing dance when incrementing or accumulating into a dictionary value that might not yet have an entry.
  ```swift
  // BAD — verbose manual existence check before accumulating
  if let existing = counts[key] { counts[key] = existing + 1 } else { counts[key] = 1 }

  // GOOD — the default: subscript handles the missing-key case inline
  counts[key, default: 0] += 1
  ```
- **DON'T:** Use force-unwrap as a substitute for asserting a precondition that should be checked explicitly and given a meaningful message — prefer `precondition(condition, "message")` or `guard condition else { fatalError("message") }` when a genuine programmer-error invariant is being enforced, since both produce a debuggable message instead of `!`'s bare "unexpectedly found nil."
- **DO:** Use `assert`/`assertionFailure` for invariant checks that should only run in debug builds (stripped from release builds for performance), reserving `precondition`/`fatalError` for invariants serious enough to enforce even in production.

## Value Types vs Reference Types

- **DO:** Default to `struct` for models that represent data without identity — DTOs, view state, coordinates, configuration — and let Swift's value semantics (copy-on-assignment, copy-on-write for collections) prevent the shared-mutable-state bugs that plague reference types.
- **DO:** Reach for `class` only when you need reference semantics: shared mutable state across multiple owners, identity comparison (`===`), inheritance from a non-protocol base, or interoperability with Objective-C/Cocoa APIs that expect objects.
- **DON'T:** Make a type a `class` purely out of habit from other object-oriented languages when nothing about it needs identity or shared mutation. An unnecessary class introduces heap allocation, ARC retain/release traffic, and the possibility of aliasing bugs a struct would have made impossible by construction.
- **DO:** Conform value types to `Equatable` and `Hashable` (often free via compiler synthesis when all stored properties already conform) so they work naturally with `==`, `Set`, and dictionary keys, instead of hand-rolling comparison logic elsewhere.
- **DO:** Be aware that large value types with many stored properties are still cheap to copy in practice because Swift's collections and most compiler-optimized structs use copy-on-write — but a struct holding a reference-type property (e.g., a class-based cache) only shallow-copies that reference, so mutating it through one copy can be visible through another. Design mutable reference properties out of otherwise-value types, or wrap them in a dedicated reference-counted box you control.
- **DON'T:** Mix value and reference semantics in the same type's public surface (a struct that exposes a mutable class property directly) without documenting it — callers who assume struct semantics ("copies are independent") will be surprised when mutating a copy mutates the original through the shared reference.
- **DO:** Use `final class` by default for reference types unless the class is explicitly designed as a base for subclassing. `final` communicates intent, and it lets the compiler devirtualize method calls for a real performance win.
- **DO:** Mark struct methods that mutate `self` with the `mutating` keyword, and understand that this is what makes value semantics enforceable — a `let` struct instance simply cannot call a mutating method, which is the compiler protecting you from an accidental in-place edit of something meant to be immutable.
- **DO:** Reach for `indirect enum` (or `indirect case`) when a value type needs to be recursive — a JSON tree, an AST node, a linked-list-like structure — so the compiler can box the recursive case on the heap while everything else about the type keeps value semantics.
  ```swift
  indirect enum JSONValue {
      case string(String)
      case array([JSONValue])
      case object([String: JSONValue])
  }
  ```
- **DO:** Prefer a protocol with an associated type (or generics) over a class hierarchy when different implementations of a capability don't need shared mutable state or identity — it keeps each conforming type a lightweight, independently testable value type instead of forcing a common heap-allocated base class.
- **DON'T:** Assume value types are automatically "free" regardless of size or usage pattern — copying a large struct in a hot loop (e.g., inside `reduce` building up big intermediate values) still costs something even with copy-on-write optimizations, particularly once the struct's stored properties themselves aren't COW-backed. Profile before assuming either types is faster in a specific hot path.
- **DO:** Remember that `actor` is a reference type (it has identity, like `class`) that additionally enforces mutual-exclusion on its mutable state — don't reach for a `class` plus manual locking when what you actually want is an `actor`.
- **DO:** Pass value types by value freely between threads/tasks without synchronization concerns for the value itself — a genuinely immutable `struct` with no reference-type properties is inherently safe to share across concurrency domains, which is a large part of why Swift's concurrency model treats value types as `Sendable` by default when their contents are.
- **DON'T:** Assume two structs compare equal with `==` just because the compiler synthesized `Equatable` — synthesis compares stored properties field-by-field, so a struct containing a reference-type property compares that property by *reference identity* through its own `Equatable` conformance (or fails to compile if the reference type isn't `Equatable`), not by the referenced object's contents, unless that type defines value-based equality itself.
- **DO:** Use `Set` and `Dictionary` with value-type keys/elements whenever possible instead of class-based ones, since value semantics let the collection itself be copied and compared predictably — a `Set<SomeClass>` behaves correctly only if `SomeClass` implements `Hashable`/`Equatable` based on genuinely meaningful value equality, which for identity-based classes is often not what's wanted.
- **DO:** Recognize when a class's reference semantics are the entire point of the design (a shared cache multiple objects read and write, a singleton coordinating global state, a delegate relationship) and lean into it explicitly, rather than trying to retrofit struct-based value semantics onto a fundamentally shared-mutable-state problem.
- **DON'T:** Pass a large, deeply nested value type by value into a tight, performance-critical loop without considering whether a `class` wrapper (traded deliberately for reference semantics and single-allocation reuse) would actually perform better — value semantics are usually the right *safety* default, but profile before assuming they're free in every hot path.

## Error Handling (do/try/catch, Result)

- **DO:** Model domain errors as an `enum` conforming to `Error` (and `LocalizedError` when the message needs to reach the UI), with cases that carry the context needed to handle or report the failure. A typed error enum gives callers exhaustive `switch` handling and gives you self-documenting failure modes instead of an opaque `NSError` or `String`.
  ```swift
  enum NetworkError: Error, LocalizedError {
      case invalidURL
      case unauthorized
      case decodingFailed(underlying: Error)

      var errorDescription: String? {
          switch self {
          case .invalidURL: return "The request URL was invalid."
          case .unauthorized: return "You are not signed in."
          case .decodingFailed: return "The server response could not be read."
          }
      }
  }
  ```
- **DO:** Use `do { try ... } catch { ... }` for synchronous and `async` throwing calls, and prefer `catch` clauses that pattern-match specific error cases over one catch-all block that inspects the error with `if`/`as?` chains.
- **DON'T:** Use `try!` outside of contexts where failure is provably impossible (a hardcoded, compile-time-valid regular expression; a unit test asserting a precondition already guaranteed by the test setup). `try!` is a force-unwrap for errors and crashes the process exactly the same way.
- **DO:** Use the `Result<Success, Failure>` type for APIs that must return success-or-failure through a completion handler or store the outcome for later inspection — anywhere `async`/`await` isn't available or a first-class `throws` signature doesn't fit (e.g., a stored property representing a cached outcome). For any new API where the caller can simply `await` the call, prefer `async throws` over a `Result`-returning completion handler; it collapses two competing error-handling styles into one.
- **DO:** Convert between the two idioms deliberately at the boundary: `Result(catching:)` to capture a throwing call's outcome, and `try result.get()` to re-throw a `Result`'s failure — rather than writing manual `switch` boilerplate every time.
- **DON'T:** Swallow errors silently with an empty `catch {}` block. At minimum, log the error; in user-facing code, surface a meaningful message or a retry path. A silently discarded error turns a debuggable failure into a mysterious support ticket.
- **DO:** Rethrow with added context when crossing an abstraction boundary (e.g., wrapping a low-level `URLError` in a domain-specific `RepositoryError.fetchFailed(underlying:)`), so a bug report shows what the app was trying to do, not just what the network stack said.
- **DO:** Use `defer` to guarantee cleanup (closing a file handle, ending a signpost, unlocking a resource) runs on every exit path out of a function — including early `return`s and thrown errors — instead of duplicating the cleanup call before each exit point.
  ```swift
  func processFile(at url: URL) throws {
      let handle = try FileHandle(forReadingFrom: url)
      defer { handle.closeFile() }
      // any early return or thrown error below still closes the handle
  }
  ```
- **DO:** Write multiple, specific `catch` clauses that pattern-match individual error cases (`catch NetworkError.unauthorized { ... } catch NetworkError.decodingFailed { ... } catch { ... }`) when different failures need different recovery behavior, rather than one generic `catch` with an internal `switch`.
- **DON'T:** Use thrown errors for expected, frequent control flow (e.g., "no more items" in a normal loop, or a common validation branch hit on most calls). Reserve `throws` for genuinely exceptional or boundary-crossing failures; model expected alternate outcomes with a return type like an `enum` or `Optional` instead, since throwing/catching has real overhead and reads as "something went wrong," not "this is a normal branch."
- **DO:** When bridging from Objective-C/Foundation APIs that surface `NSError`, convert it into a specific Swift error type at the boundary rather than propagating raw `NSError` (with its stringly-typed `userInfo` and `domain`/`code`) deep into Swift-only code.
- **DO:** Implement retry-with-backoff for transient failures (network timeouts, rate limits) explicitly and visibly — a small loop with an attempt counter and increasing delay — rather than silently retrying inside a `catch` block in a way that hides the extra attempts from logs and callers.
- **DO:** Use Swift's typed `throws` (`func fetch() throws(NetworkError) -> Data`, available in newer Swift versions) when a function has exactly one well-defined error type it can throw, so callers get a precise error type in `catch` without needing to `as?`-cast a generic `any Error` — fall back to untyped `throws` when a function can genuinely propagate errors from multiple unrelated sources.
- **DO:** Distinguish between errors a caller can meaningfully recover from (invalid input, a resource temporarily unavailable) and programmer errors that indicate a bug (`preconditionFailure`, `fatalError`, a force-unwrap the code's own logic should have already guaranteed succeeds) — use `throws` for the former and let the latter crash loudly rather than being caught and silently handled, since silently recovering from a genuine logic bug just hides it until it causes worse damage elsewhere.
- **DO:** Test both the success and failure paths of every throwing function explicitly, including verifying the *specific* error case thrown (`XCTAssertThrowsError` with a closure inspecting the caught error), not just that "an error was thrown" — a test that only checks for "any error" doesn't catch a regression where the wrong failure reason is now being reported.
- **DON'T:** Catch an error only to immediately rethrow it unchanged with no added value (`catch { throw $0 }` with nothing else in the block) — if nothing is being logged, transformed, or cleaned up, the `do`/`catch` is pure noise and letting `throws` propagate the error would do the same thing without it.
- **DO:** Use `Result`'s `map`/`flatMap`/`mapError` to transform a success or failure value functionally within a `Result`-typed pipeline, instead of unwrapping via `switch` at every step just to re-wrap the transformed value in a new `Result`.
  ```swift
  let parsed: Result<Int, ParsingError> = rawResult
      .mapError { ParsingError.underlying($0) }
      .flatMap { string in Int(string).map(Result.success) ?? .failure(.notANumber) }
  ```
- **DO:** Group cleanup that must run on both the success and failure paths at the top of a function using `defer`, rather than duplicating the same cleanup call at the end of the success path and inside every `catch` clause.

## SwiftUI vs UIKit Basics

- **DO:** Default to SwiftUI for new features when the minimum deployment target supports it — it needs less boilerplate, has state-driven updates instead of manual view refreshes, and previews iterate faster. Reach for UIKit (or SwiftUI wrapped around UIKit via `UIViewRepresentable`) when a feature needs fine-grained scroll/gesture control, a component SwiftUI doesn't yet expose, or must support an OS version below SwiftUI's baseline.
- **DO:** Choose the right SwiftUI property wrapper for the ownership it implies: `@State` for view-local value-type state, `@Binding` for a two-way reference to state owned by a parent, `@StateObject` for a reference-type model a view creates and owns, `@ObservedObject` for one passed in from outside, and `@EnvironmentObject`/`@Environment` for values injected from an ancestor. Picking the wrong one causes either lost state on redraw or duplicate object instances.
  ```swift
  // BAD — @ObservedObject on a freshly created object: it gets
  // re-created (and its state lost) every time the parent redraws.
  struct ProfileView: View {
      @ObservedObject var viewModel = ProfileViewModel()
  }

  // GOOD — @StateObject owns the instance for the view's lifetime.
  struct ProfileView: View {
      @StateObject private var viewModel = ProfileViewModel()
  }
  ```
- **DON'T:** Put business logic directly inside a SwiftUI `View`'s `body`. Keep `body` a declarative description of layout driven by state; push data fetching, validation, and transformation into an `ObservableObject` view model or a plain service type that the view observes.
- **DO:** Keep SwiftUI views small and composed — extract a repeated visual chunk into its own `View` struct rather than growing one `body` past a couple hundred lines. Small views also make SwiftUI's diffing cheaper and previews faster to iterate on.
- **DO:** In UIKit, build layout with Auto Layout constraints (or a layout anchor/DSL wrapper) rather than manually computing `frame` rectangles. Manual frame math breaks on rotation, Dynamic Type, and different device sizes in ways constraints handle automatically.
- **DO:** Separate UIKit view controllers from their business logic using MVVM, MVC-with-thin-controllers, or a coordinator pattern — a "massive view controller" that owns networking, persistence, and layout in one file is a maintenance and testability liability regardless of which architecture label you put on it.
- **DON'T:** Force a SwiftUI-only feature into a UIKit codebase (or vice versa) purely because it's what the assistant defaults to; check the project's actual UI framework and existing conventions before generating new screens, and use `UIHostingController`/`UIViewRepresentable` deliberately at the seams when mixing is genuinely required.
- **DO:** Extract repeated view styling into a custom `ViewModifier` (and a `View` extension exposing it) instead of copy-pasting the same chain of `.padding()`/`.background()`/`.cornerRadius()` modifiers across many views.
  ```swift
  struct CardStyle: ViewModifier {
      func body(content: Content) -> some View {
          content.padding().background(.thinMaterial).cornerRadius(12)
      }
  }
  extension View { func cardStyle() -> some View { modifier(CardStyle()) } }
  ```
- **DO:** Use `NavigationStack` (iOS 16+) with type-safe, value-based navigation (`navigationDestination(for:)`) for new SwiftUI navigation code instead of the deprecated `NavigationView`, which has layout quirks on iPad/macOS and no type-safe path-based API.
- **DO:** Give `List`/`ForEach` elements stable, unique identity — conform the model to `Identifiable` with a real, stable `id`, not an array index — so SwiftUI can diff insertions, deletions, and reorders correctly instead of recycling rows onto the wrong data.
- **DON'T:** Overuse `AnyView` to paper over "these branches return different view types." `AnyView` erases the underlying type, which disables several of SwiftUI's diffing and layout optimizations; prefer `@ViewBuilder` with `if`/`switch` branches, or a `some View`-returning helper, which preserve type identity.
- **DO:** Wrap a UIKit view controller for use inside SwiftUI with `UIViewControllerRepresentable` (or a `UIView` with `UIViewRepresentable`) rather than reimplementing UIKit-only functionality from scratch in SwiftUI when no SwiftUI equivalent exists yet.
- **DO:** Use `#Preview` (or the legacy `PreviewProvider`) with realistic sample data — including edge cases like empty lists, long text, and error states — so visual regressions are caught in the canvas before they reach a simulator or device.
- **DO:** Use `.task { }` (rather than `.onAppear { Task { ... } }`) to run async work tied to a SwiftUI view's lifetime, since `.task` automatically cancels its work when the view disappears — `.onAppear` combined with a manually created `Task` requires you to store and cancel that task yourself to get the same behavior.
- **DO:** Use `GeometryReader` sparingly and only when a layout genuinely needs to read its container's size or position — it opportunistically expands to fill available space and can produce unexpected layout results when used as a default container, and newer APIs (`containerRelativeFrame`, alignment guides) cover many cases that used to require it.
- **DO:** Prefer SwiftUI's built-in animation system (`withAnimation`, `.animation(_:value:)`, matched geometry effects) over manually driving UIKit-style `CADisplayLink`/timer-based animation from within a SwiftUI view, since the built-in system integrates correctly with state-driven redraws and interruption/cancellation.
- **DON'T:** Force a UIKit view controller's entire navigation and presentation stack to be reimplemented by hand inside `UIViewControllerRepresentable` wrapper code when the two navigation systems (UIKit's and SwiftUI's) need to coordinate — use a `UIHostingController` embedded in a UIKit navigation stack, or a `UIViewControllerRepresentable` coordinator with clear, minimal responsibility, rather than fighting both frameworks' navigation models simultaneously.
- **DO:** Use `@FocusState` for managing keyboard/field focus declaratively in SwiftUI forms, rather than falling back to `UIResponder`-based `becomeFirstResponder()` calls bridged in from UIKit, when the whole form is already SwiftUI-native.
- **DO:** In UIKit, reuse cells correctly in `UITableView`/`UICollectionView` (dequeuing via `dequeueReusableCell(withIdentifier:for:)` and resetting any cell state that might carry over from a previous reuse, like clearing an image view before an async image load starts) — a common bug class is stale content from a recycled cell showing briefly before the correct content loads.
- **DO:** Use `UICollectionViewCompositionalLayout`/`UICollectionViewDiffableDataSource` for new UIKit collection view work instead of manually managing `UICollectionViewFlowLayout` and hand-written `reloadData()` calls with manual diffing — the newer APIs handle complex layouts and animated diffing with substantially less boilerplate and fewer state-mismatch bugs.
- **DO:** Use `@Environment(\.colorScheme)`/`@Environment(\.dynamicTypeSize)` in SwiftUI (or the UIKit trait-collection equivalents) to adapt a view's appearance to the system's current light/dark mode and accessibility text size, rather than hardcoding colors and font sizes that ignore those user preferences.
- **DO:** Use the `App`/`Scene` protocol-based app lifecycle (`@main struct MyApp: App { ... }`) for new SwiftUI-first projects instead of an `AppDelegate`/`SceneDelegate` pair, reserving `UIApplicationDelegateAdaptor` only for the specific delegate callbacks (push notification registration, certain lifecycle hooks) SwiftUI's native lifecycle doesn't yet expose directly.
- **DO:** Use SwiftData (`@Model`, `@Query`) for new persistence code targeting a SwiftData-compatible deployment target, and Core Data for projects that already have an established Core Data model or need capabilities SwiftData doesn't yet cover — check which persistence framework a project has already standardized on rather than introducing a second one.
  ```swift
  @Model
  final class Note {
      var title: String
      var body: String
      init(title: String, body: String) { self.title = title; self.body = body }
  }
  ```
- **DON'T:** Perform Core Data/SwiftData context saves or fetches from a background thread without going through the framework's designated concurrency mechanism (`NSManagedObjectContext.perform`, a background `ModelContext`) — both frameworks are not thread-safe for direct cross-thread access to the same context, and violating this produces intermittent, hard-to-reproduce crashes or data corruption.
- **DO:** Use `NavigationSplitView` for adaptive multi-column layouts (a sidebar plus detail view that collapses appropriately on compact-width devices) instead of manually branching layout code based on size class, since it's built specifically to handle that adaptive behavior across iPhone, iPad, and Mac.

## Memory Management (ARC, Avoiding Retain Cycles)

- **DO:** Understand that Automatic Reference Counting deallocates a class instance only when its strong-reference count reaches zero — any cycle of strong references (A holds a strong reference to B, B holds a strong reference back to A) will never reach zero and leaks for the app's lifetime.
- **DO:** Capture `self` as `[weak self]` in closures stored by a class instance for longer than the immediate call (completion handlers, `Combine` sinks, `NotificationCenter` observers held as properties, or any closure assigned to a property), and unwrap it with `guard let self else { return }` at the top of the closure body.
  ```swift
  // BAD — self strongly captured by a closure self owns: a cycle.
  class ImageLoader {
      var onComplete: (() -> Void)?
      func load() {
          onComplete = { self.finishUp() }
      }
  }

  // GOOD — breaks the cycle; the closure doesn't keep self alive.
  class ImageLoader {
      var onComplete: (() -> Void)?
      func load() {
          onComplete = { [weak self] in
              guard let self else { return }
              self.finishUp()
          }
      }
  }
  ```
- **DO:** Use `unowned` instead of `weak` only when the referenced object is guaranteed to outlive the closure or property that references it (e.g., a child object referencing a parent that owns it and cannot outlive it) — `unowned` avoids the optional-unwrapping ceremony but crashes immediately if the assumption is ever violated, so use it deliberately, not as a shortcut to skip `weak`'s unwrap.
- **DON'T:** Reach for `unowned` by default to avoid writing `guard let self`. If there's any doubt about relative lifetimes, `weak` fails safely (the closure becomes a no-op) while `unowned` fails as a crash.
- **DO:** Watch for retain cycles through delegate properties — a delegate reference should almost always be declared `weak var delegate: SomeDelegate?` so the delegate (typically a parent that owns the delegating object) isn't kept alive by the very object it owns.
- **DO:** Use Xcode's Memory Graph Debugger and the Instruments Leaks/Allocations templates to confirm suspected retain cycles rather than guessing; a purple exclamation-mark badge in the memory graph points directly at the cycle.
- **DON'T:** Assume ARC "just handles it" for closures the way it handles ordinary property references — closures capture variables they reference by default, and a closure capturing `self` (even indirectly, through a captured local variable that holds a strong reference) is a common source of cycles the compiler will not warn about.
- **DO:** Remove `NotificationCenter` observers (unless using the block-based API's automatically-scoped token pattern correctly) and invalidate `Timer`s in `deinit` or when the owning object no longer needs them — a repeating `Timer` retains its target by default and will keep an object alive indefinitely if never invalidated.
- **DO:** Capture `[weak self]` in Combine `.sink` closures assigned to a `Cancellable`/`AnyCancellable` stored on `self`, exactly as with any other closure stored on `self` — Combine pipelines are just as capable of creating retain cycles as completion handlers.
- **DO:** Use `autoreleasepool { }` around tight loops that create many Objective-C-bridged temporary objects (common in image-processing or bulk `Foundation` API calls) to release intermediate autoreleased objects promptly instead of letting them accumulate until the end of the current run loop iteration.
- **DON'T:** Assume a closure captured inside another captured closure "flattens out" the capture list — nested closures each need their own `[weak self]` where appropriate, since an outer `[weak self]` doesn't automatically make an inner closure's capture of `self` weak too.
- **DO:** Understand the difference in retain-cycle risk between a `struct`-based delegate/callback pattern (no risk at all, since structs aren't reference-counted) and a `class`-based one — when a design has flexibility, a value-type callback pattern (a stored closure on a struct, or a protocol requirement satisfied by a struct) sidesteps the entire category of ARC retain-cycle bugs, though it trades away reference identity where that's actually needed.
- **DO:** Watch specifically for retain cycles through parent-child view controller relationships in UIKit (a child view controller holding a strong reference back to its parent via a callback closure or delegate property) — this is one of the most common leak patterns in non-trivial UIKit apps, especially once child view controllers are added/removed dynamically.
- **DO:** Break a cycle between two objects that must both hold references to each other by making exactly one direction of the reference `weak` — decide which object is conceptually the "owner" and which is the "observer/dependent," and make the dependent's back-reference weak.
- **DON'T:** Treat every retain cycle as equally harmful without considering its actual lifetime impact — a cycle between two objects that live for the entire app lifetime anyway (e.g., a persistent singleton and its permanently-registered observer) is a leak in the strict technical sense but often not a practically significant one; prioritize fixing cycles involving short-lived objects (view controllers, per-request objects) that are supposed to be deallocated but aren't.
- **DO:** Verify a suspected deallocation with a lightweight `deinit` log statement (`deinit { print("MyViewController deallocated") }`) during development on classes with a history of leaking, as a cheap sanity check alongside (not instead of) the Memory Graph Debugger for confirming a fix actually worked.
- **DO:** Watch for retain cycles introduced by third-party SDKs (an analytics SDK holding a strong reference to a view controller it was handed for context) — audit any API that accepts `self` as a parameter for whether it retains that reference longer than the call itself needs.

## Concurrency (async/await, Actors)

- **DO:** Prefer `async`/`await` over completion-handler-based asynchrony for new code — it removes the pyramid of nested callbacks, lets errors propagate with ordinary `throws`, and makes cancellation and structured concurrency (`async let`, task groups) available.
  ```swift
  // BAD — callback pyramid, easy to leak or double-call the handler
  func fetchProfile(id: String, completion: @escaping (Profile?, Error?) -> Void) { ... }

  // GOOD — linear, composable, and errors propagate naturally
  func fetchProfile(id: String) async throws -> Profile { ... }
  let profile = try await fetchProfile(id: userID)
  ```
- **DO:** Use `actor` types to protect mutable state that's accessed from multiple concurrent tasks — the actor serializes access automatically, eliminating a whole class of data races without manual locks.
  ```swift
  actor Cache {
      private var storage: [String: Data] = [:]
      func set(_ data: Data, for key: String) { storage[key] = data }
      func get(_ key: String) -> Data? { storage[key] }
  }
  ```
- **DON'T:** Access actor-isolated state from outside the actor without `await` — the compiler enforces this, but a common AI-generated mistake is designing around it with `nonisolated(unsafe)` or by copying state out into a non-isolated global just to avoid writing `await`, which defeats the actor's entire safety guarantee.
- **DO:** Mark any type or function that must run on the main thread (view models backing UI, UIKit/AppKit calls) with `@MainActor`, either on the whole class or on individual methods, rather than manually dispatching to `DispatchQueue.main.async` inside `async` functions.
- **DON'T:** Block a thread waiting on async work with `DispatchSemaphore.wait()`, `DispatchGroup.wait()`, or a busy `while` loop bridging into `async` code. This can deadlock Swift's cooperative thread pool (which has a limited number of threads) and defeats the purpose of structured concurrency; use `await` or bridge with a `Task` instead.
- **DO:** Use `Task { }` to bridge synchronous contexts (like a SwiftUI `.onAppear` or a button action) into `async` code, and use `Task.isCancelled` checks or `try Task.checkCancellation()` inside long-running loops so cancellation (e.g., a view disappearing) actually stops the work instead of running to completion pointlessly.
- **DO:** Use `async let` for a fixed, known set of independent concurrent operations, and a `TaskGroup` (`withTaskGroup`/`withThrowingTaskGroup`) for a dynamic collection of concurrent child tasks — reach for `TaskGroup` rather than manually fanning out with a `DispatchGroup` in new async code.
- **DO:** Conform types shared across concurrency domains to `Sendable` (or mark them `@unchecked Sendable` only after manually verifying thread-safety, with a comment explaining why) so the compiler can actually check for data races at the boundary, rather than suppressing every "not Sendable" warning with a blanket `@preconcurrency import`.
- **DO:** Use `await MainActor.run { }` (or mark the enclosing function `@MainActor`) to hop onto the main actor for a specific block of UI-updating work from a background context, rather than reaching for `DispatchQueue.main.async` inside `async` code, which doesn't compose with structured concurrency's cancellation and error propagation.
- **DON'T:** Spawn unstructured, unmanaged `Task { }` instances inside loops or view lifecycle callbacks without a way to cancel them (storing a reference and cancelling in `onDisappear`, or using `.task { }` in SwiftUI which auto-cancels). Orphaned tasks keep running and mutating state after the view or object that spawned them is gone, which is its own category of bug distinct from a memory leak.
- **DO:** Understand that actors are reentrant across `await` suspension points — other calls can interleave with an in-progress actor method at any `await`, so don't assume an actor method runs atomically from start to finish just because it's an actor; re-check invariants after each `await` if they matter.
- **DON'T:** Mix Grand Central Dispatch (`DispatchQueue.async`) and `async`/`await` arbitrarily within the same function without a clear reason. Bridging between them (e.g., via `withCheckedContinuation`) is sometimes necessary at a legacy API boundary, but new code within a single function should pick one concurrency model and use it consistently.
- **DO:** Use `withCheckedContinuation`/`withCheckedThrowingContinuation` to bridge a completion-handler-based legacy API into `async`/`await` exactly once at the boundary, and be careful to resume the continuation on every possible code path exactly once — resuming it zero times leaks the awaiting task forever, and resuming it more than once is a runtime crash in debug builds (`withCheckedContinuation` specifically checks for this misuse).
  ```swift
  func legacyFetch() async throws -> Data {
      try await withCheckedThrowingContinuation { continuation in
          legacyAPI.fetch { data, error in
              if let error { continuation.resume(throwing: error) }
              else { continuation.resume(returning: data!) }
          }
      }
  }
  ```
- **DO:** Set an appropriate `Task` priority (`.userInitiated`, `.background`, `.utility`) when spawning work whose urgency differs meaningfully from the default, so the system's scheduler can prioritize interactive work over background bookkeeping appropriately, rather than leaving every task at the same default priority regardless of how visible its result is to the user.
- **DON'T:** Assume `@MainActor` isolation automatically makes a type's non-actor-isolated properties or methods thread-safe too — isolation applies to what's explicitly marked (or inferred at the type level), and a `nonisolated` member on an otherwise `@MainActor` type still needs its own thread-safety consideration if it touches mutable state.
- **DO:** Use `Task.detached` deliberately and rarely — it explicitly opts out of inheriting the calling context's actor isolation, priority, and task-local values, which is occasionally correct (genuinely independent background work) but is usually the wrong default compared to a plain `Task { }`, which inherits context sensibly.
- **DO:** Use `AsyncSequence`/`AsyncStream` to model a source of asynchronously produced values (a stream of location updates, incoming WebSocket messages) that callers consume with `for await`, instead of a manual callback-based observer pattern that doesn't compose with structured concurrency's cancellation.
  ```swift
  for await location in locationUpdates {
      handle(location)
  }
  ```
- **DON'T:** Assume every `async` function runs concurrently with its caller — calling `await someAsyncFunc()` still suspends the calling task until the callee completes, exactly like a synchronous call, unless the call is explicitly wrapped in its own `Task`/`async let`/task group to actually run alongside other work.

## Testing (XCTest)

- **DO:** Structure tests as `XCTestCase` subclasses with clearly named test methods (`func test_login_withInvalidPassword_returnsUnauthorizedError()`), using the `XCTAssert*` family (`XCTAssertEqual`, `XCTAssertTrue`, `XCTAssertThrowsError`, `XCTAssertNil`) so failures report the expected vs. actual values automatically.
- **DO:** Design production code for testability by injecting dependencies (network clients, clocks, persistence) through initializers or protocols, rather than reaching for global singletons inside the type under test — a hardcoded `URLSession.shared` call is untestable without hitting the real network.
  ```swift
  protocol ProfileFetching { func fetch(id: String) async throws -> Profile }

  final class ProfileViewModel {
      private let service: ProfileFetching
      init(service: ProfileFetching) { self.service = service }
  }
  // Tests substitute a fake conforming to ProfileFetching — no network needed.
  ```
- **DO:** Use `async` test methods (`func test_fetchProfile_succeeds() async throws`) to test `async` production code directly with `await`, rather than wrapping it in `XCTestExpectation`/`waitForExpectations` boilerplate that's now largely unnecessary.
- **DO:** Keep each test focused on one behavior and one assertion path — a test named for a single expected outcome that actually checks five unrelated things makes failures ambiguous about what broke.
- **DON'T:** Write tests that assert on implementation details (private method call counts, internal storage layout) instead of observable behavior; such tests break on harmless refactors and stop being a useful safety net.
- **DO:** Use `setUp()`/`tearDown()` (or `setUpWithError()` for throwing setup) to establish and clean up shared fixtures, keeping individual test bodies focused on arrange-act-assert for their specific case.
- **DO:** Use `XCTUnwrap(_:)` to unwrap an optional inside a test and fail with a clear message if it's `nil`, instead of force-unwrapping (`!`) in test code — a force-unwrap failure in a test reports an opaque crash, while `XCTUnwrap` reports a proper, attributable test failure.
- **DO:** Use hand-written or lightly generated test doubles (stubs that return fixed data, mocks that record calls for verification, fakes that reimplement simplified real behavior) behind the same protocol the production dependency conforms to, and keep them simple enough that the test's intent stays legible.
- **DON'T:** Make async tests wait with `Thread.sleep` or a fixed-duration `DispatchQueue.asyncAfter` "just to be safe." This makes tests slow and still flaky under load; use `await` directly on the async call under test, or `XCTestExpectation` with a specific fulfillment condition and a bounded timeout for callback-based APIs that haven't been migrated to `async`.
- **DO:** Track code coverage (Xcode's built-in coverage reports, or `xccov`) to find untested branches, especially in error-handling paths and edge cases, but treat the percentage as a signal to investigate, not a target to game with shallow tests.
- **DO:** Structure test bodies with a clear arrange/act/assert (or given/when/then) shape — set up inputs, perform the one action under test, then assert on the outcome — so a reader can tell at a glance what's being verified without tracing the whole method.
- **DO:** Write tests that cover boundary and edge-case inputs explicitly (empty collections, the largest/smallest representable values, `nil`/absent optional inputs, malformed data) in addition to the "happy path," since these are exactly the inputs most likely to trigger a force-unwrap crash or an off-by-one error in the code under test.
- **DO:** Use snapshot testing (comparing a rendered SwiftUI/UIKit view against a stored reference image) for UI code where behavior is hard to assert against programmatically but easy to verify visually — pair it with a deliberate, reviewed process for updating the reference snapshots when a change is intentional.
- **DON'T:** Leave flaky tests (ones that intermittently fail without a code change, often due to timing assumptions or shared mutable test state) disabled or ignored indefinitely instead of fixing their root cause. A flaky test that's silently skipped erodes trust in the whole suite and eventually gets ignored even when it's catching a real regression.
- **DO:** Isolate tests from each other's state — avoid shared mutable singletons, static properties, or a shared on-disk/UserDefaults state that one test's execution can leave in a state that affects a different test's result depending on run order.
- **DO:** Use XCUITest (UI testing) sparingly, for a small number of critical end-to-end user flows, rather than trying to cover every interaction path at the UI-test level — UI tests are slow and comparatively brittle (sensitive to layout/timing changes) relative to unit and integration tests, so they belong at the top of the testing pyramid, not as the primary coverage mechanism.
- **DO:** Set accessibility identifiers (`.accessibilityIdentifier("loginButton")`) on interactive elements specifically to give UI tests a stable way to locate them, independent of visible text that might change with localization or copy edits — relying on visible label text to locate elements in UI tests makes the tests fragile to routine copy changes.
- **DO:** Run the test suite with Address Sanitizer/Thread Sanitizer enabled periodically (not necessarily on every CI run, given the performance cost) to catch memory-safety and data-race bugs that ordinary test assertions wouldn't surface even when the test's functional outcome looks correct.

## Common AI-Assistant Mistakes

- **DON'T:** Force-unwrap optionals throughout generated code as a default habit (`response.data!`, `dict["key"] as! String`) to make the code compile quickly. This is the single most common Swift anti-pattern in AI-generated code and turns normal runtime conditions (missing keys, failed network calls, nil view controller references) into crashes; use `guard let`/`if let` and propagate failure properly instead.
- **DON'T:** Capture `self` strongly by default inside closures assigned to properties or passed to long-lived async APIs, creating retain cycles. Default to `[weak self]` for any closure stored on `self` or handed to a framework API that outlives the immediate call, and only use a strong capture when the closure is provably short-lived (e.g., a `Task` that completes and is discarded).
- **DON'T:** Invent SDK APIs, method names, or parameter labels that sound plausible but don't exist in UIKit, SwiftUI, or Foundation (a hallucinated `View.onDisappearAsync { }` modifier, a nonexistent `URLSession.shared.data(for:completion:)` overload, a `List` initializer parameter that was never added). Verify method signatures against current documentation or existing code in the project rather than pattern-matching from a similar-looking but different API family.
- **DON'T:** Mix outdated completion-handler-based concurrency patterns into a codebase that has already adopted `async`/`await`, or vice versa, without being asked — check what the surrounding code already uses and match it instead of defaulting to whichever pattern is more familiar.
- **DON'T:** Ignore `@MainActor` isolation requirements and add `@unchecked Sendable` or `nonisolated` annotations reflexively just to silence a concurrency-checking warning, without verifying the code is actually safe to run off the main thread. This trades a compiler-caught bug for a runtime data race or a UI-update-off-main-thread crash.
- **DON'T:** Generate SwiftUI state management that mismatches ownership (`@ObservedObject` for an object the view itself instantiates, or a `@State` var holding a reference type expected to be shared) — verify which property wrapper matches who owns and who merely observes the state before writing it.
- **DON'T:** Assume a project uses Combine, RxSwift, or a specific DI framework without checking — SwiftUI's native tooling has made several third-party patterns (particularly Combine-heavy view models) less necessary, and grafting one framework's idioms onto a codebase that doesn't use it creates an inconsistent mess.
- **DON'T:** Generate calls to Combine operators or overloads that don't exist on the type in question (a hallucinated `.retry(delay:)` variant, a `Publisher` method borrowed from RxSwift's API surface) — Combine's operator set is precise and mismatched pattern-matching from a similar reactive framework produces code that simply doesn't compile.
- **DON'T:** Omit `#available`/`@available` checks when using an API newer than the project's deployment target, or conversely, add unnecessary `#available` guards around APIs that have been available since a target well below the project's actual minimum. Check the project's deployment target (in the Xcode project settings or `Package.swift`) before assuming any particular API's availability.
- **DON'T:** Assume a completion-handler-based API always calls its handler exactly once, synchronously, or on the main thread. Many system APIs call back asynchronously on a background queue and some can call back more than once or not at all on cancellation — read the actual documented contract instead of assuming the most convenient behavior.
- **DON'T:** Wire `@Published` properties or bindings that mutate SwiftUI state from a background thread/task without hopping to the main actor first. Mutating `@Published` off the main thread doesn't always crash immediately but produces undefined UI update behavior and intermittent glitches that are hard to reproduce.
- **DON'T:** Generate `Codable` conformances with a mismatched or missing `CodingKeys` mapping when the wire format's key names differ from Swift's naming convention, and then paper over the resulting decode failures with a force-unwrapped fallback or `try?` that silently produces `nil`. Verify the actual JSON shape (from documentation or a real response) before writing the model.
- **DON'T:** Leave stale, commented-out, or dead code paths (an old completion-handler-based method left in place "just in case" alongside its new `async` replacement) in generated diffs — either the old code is still needed and should be tested and used somewhere, or it should be removed rather than left as clutter that suggests uncertainty about which version is actually correct.
- **DON'T:** Default to `class` for every new type without considering whether the type's actual requirements (no shared mutable state, no identity needed) call for a `struct` instead — this is one of the more common "generated code looks like code from a different language" tells, especially from patterns learned primarily from Java/C#/Objective-C-style examples.
- **DON'T:** Generate SwiftUI code that recreates a `@StateObject`-owned object on every parent redraw by initializing it as a plain default value instead of via the `@StateObject` wrapper's initializer semantics, silently resetting the view's state on every unrelated parent update.
- **DON'T:** Omit accessibility identifiers and accessibility labels from generated SwiftUI/UIKit view code as an afterthought — treat `.accessibilityLabel`/`.accessibilityIdentifier` on custom controls as a default part of "complete" UI code, the same way alt text is expected on generated HTML images.
- **DON'T:** Generate code assuming a fixed device size/orientation (hardcoded frame dimensions, assuming a specific safe-area inset) — use adaptive layout (Auto Layout constraints, SwiftUI's layout system, safe-area-relative modifiers) that responds correctly across the range of supported device sizes and orientations.

## Quick Checklist
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
