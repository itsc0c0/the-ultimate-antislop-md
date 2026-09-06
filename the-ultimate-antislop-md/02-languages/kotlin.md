# Kotlin

## Null Safety Idioms

- **DO:** Model nullability honestly in the type itself (`String?` vs `String`) and let the compiler enforce handling at every call site, rather than treating every type as implicitly nullable "just in case." Kotlin's whole null-safety value proposition depends on non-null types actually meaning non-null.
- **DON'T:** Use the not-null assertion operator (`!!`) as a default way to silence a compiler nullability error. `!!` converts a compile-time-preventable `NullPointerException` back into a runtime crash with no explanation — it should be a rare, deliberate escape hatch used only when nullability is genuinely impossible by invariant the compiler can't see, and even then a comment should say why.
  ```kotlin
  // DON'T
  val name = user.profile!!.name!!

  // DO
  val name = user.profile?.name ?: "Unknown"
  ```
- **DO:** Use the safe call operator (`?.`) chained with the Elvis operator (`?:`) to provide a fallback value, or `?.let { }` to run a block only when a value is present, instead of manual `if (x != null)` checks scattered through business logic.
- **DO:** Rely on Kotlin's smart-casting after an explicit null check (`if (value != null) { value.length }`) for a local `val` — the compiler tracks the non-null state within that scope automatically, so a redundant `!!` or `?.` afterward is unnecessary noise.
- **DON'T:** Use `?:` with a `throw` or `return` as an afterthought bolted onto unrelated logic in a way that obscures the actual early-exit condition. Prefer a clear, intention-revealing guard: `val order = repository.find(id) ?: throw OrderNotFoundException(id)` reads cleanly as "fetch or fail," which is the idiom's actual purpose.
- **DO:** Use `lateinit var` only for non-null properties that are guaranteed to be initialized before use by a framework lifecycle (Spring bean injection, Android `onCreate`, JUnit `@BeforeEach`) — never as a way to defer initialization logic that could instead be handled with a nullable type or lazy delegation.
- **DON'T:** Default every property to nullable with `= null` to sidestep constructor design. A class with ten nullable `var` properties initialized to `null` is really several different states smashed into one type; model those states explicitly (sealed classes, separate constructors, or a builder with required parameters) instead of relying on null as a stand-in for "not set yet."
- **DO:** Use `by lazy { }` for expensive properties that must be non-null but computed on first access, rather than a nullable backing field with manual null-checking initialization logic.
  ```kotlin
  // DON'T
  private var _config: Config? = null
  val config: Config
      get() {
          if (_config == null) _config = loadConfig()
          return _config!!
      }

  // DO
  val config: Config by lazy { loadConfig() }
  ```
- **DON'T:** Use platform types (values coming from Java interop with no Kotlin-visible nullability annotation) without immediately establishing their real nullability at the boundary. Treat every unannotated Java return value as potentially null until proven otherwise, and wrap the boundary call so the rest of the Kotlin codebase gets a real `T` or `T?`.
- **DO:** Prefer `requireNotNull(value) { "message" }` over `!!` when a null value genuinely is a programmer error that should fail fast — it produces the same crash-on-null behavior as `!!` but with a message that actually explains what went wrong.
- **DO:** Use nullable receiver extension functions (`fun String?.orEmptyTrimmed(): String`) sparingly and only where they genuinely simplify a common null-handling pattern used repeatedly — not as a way to avoid ever writing an explicit null check.

- **DO:** Use the contract-enforcing `contract` builder or, more commonly, simply lean on Kotlin's built-in smart-casting after `is`/`!is` checks and after calls to standard-library functions annotated with contracts (like `isNullOrEmpty`) — the compiler already tracks nullability precisely in these cases, so an extra manual cast or `!!` immediately afterward is redundant.
- **DON'T:** Use `as?` (safe cast) and then immediately follow it with `!!` (`(value as? Foo)!!`). This combination defeats the purpose of the safe cast — if the intent is "must be a `Foo` or this is a bug," use the direct unsafe cast `as Foo` instead, which fails with a clearer `ClassCastException` at the exact point of the invalid assumption.
- **DO:** Use `checkNotNull(value) { "message" }` (the Kotlin equivalent of `requireNotNull` for internal-state invariants rather than argument validation) to fail fast with a clear message when internal state that should never be null turns out to be null — again, preferable to a bare `!!` with no explanation.
- **DO:** Design nullable function parameters with sensible defaults (`fun greet(name: String? = null)`) rather than forcing every caller to pass an explicit `null` for "not provided," and handle the default inside the function with a single Elvis expression.
- **DON'T:** Overuse the safe-call chain (`a?.b?.c?.d?.e`) to the point that a single expression silently swallows a null anywhere along a long chain, when a null partway through actually represents a bug worth surfacing (a missing required relationship, not an optional one). Long safe-call chains deserve a moment's thought about whether every `?.` link is genuinely optional or whether some should be a hard failure instead.
- **DO:** Prefer sealed result types (`Result<T>` from the Kotlin standard library for simple cases, or a custom sealed class for richer domain errors) over encoding "success or failure" as a nullable return value when the failure case needs more information than "it wasn't there."

- **DO:** Use the Elvis operator combined with a scope function for validating and transforming a nullable value in one step (`val validated = input?.trim()?.takeIf { it.isNotEmpty() } ?: return null`) rather than splitting the same logic across several separate `if` statements.
- **DON'T:** Treat a nullable `Boolean?` as if it were a two-valued `Boolean` in a plain `if` condition — `if (nullableFlag)` doesn't compile, which is correct, but the fix should be a deliberate decision about what `null` means (`if (nullableFlag == true)`, or `?: false`), not a reflexive `!!`.
- **DO:** Use `filterNotNull()` on a `Collection<T?>` to produce a `List<T>` cleanly when nulls in the source collection should simply be dropped, rather than filtering with `{ it != null }` followed by an unsafe cast or a `!!` inside `.map { }`.
- **DO:** Reach for the `takeIf`/`takeUnless` functions to turn a boolean condition on a value into a nullable result inline (`value.takeIf { it > 0 }`), which composes cleanly with a following `?.let { }` or `?:` — this is often clearer than an equivalent `if`/`else` returning `null` for the failing branch.

## Data Classes

- **DO:** Use `data class` for any type whose primary purpose is holding a fixed, immutable set of values with structural equality — it generates `equals()`, `hashCode()`, `toString()`, `copy()`, and `componentN()` destructuring functions for free, all correctly matched to the declared properties.
  ```kotlin
  data class Money(val amount: BigDecimal, val currency: Currency)
  ```
- **DON'T:** Make data class properties `var` when the object is meant to represent an immutable value. Mutable data class instances defeat the safety of the generated `equals`/`hashCode` pair — mutating a field after inserting the instance into a `HashSet` or using it as a `HashMap` key silently breaks lookups, because the hash code changes but the object's bucket doesn't.
- **DO:** Use `copy()` with named arguments to derive a modified instance rather than hand-writing a new constructor call that repeats every unchanged field. `copy()` keeps derivation resilient to new properties being added later.
  ```kotlin
  val discounted = order.copy(total = order.total * 0.9)
  ```
- **DON'T:** Put a data class with more than a handful of properties directly on the wire as a JSON DTO without considering whether default values, versioning, and backward compatibility are actually being handled — a data class's generated `equals`/`copy` behavior is about identity and derivation, not serialization compatibility, and those are separate concerns that need separate attention.
- **DON'T:** Use `data class` for types with meaningful identity rather than structural equality (e.g., a JPA/Hibernate-managed entity, or anything with a mutable lifecycle tied to a database row). Entities should typically use identity-based equality (comparing a stable ID) rather than the generated field-by-field equality a data class provides, which can misbehave with lazy-loaded proxies and partially-populated instances.
- **DO:** Override `toString()` (or exclude a property from it) on a data class that holds sensitive data (passwords, tokens, PII) since the generated `toString()` prints every property verbatim — a stray log statement of the whole object otherwise leaks secrets into logs.
- **DO:** Use data classes with sealed hierarchies to model each variant's specific payload (see Sealed Classes below) rather than one large data class with many nullable fields representing "whichever variant this happens to be."

- **DO:** Use `data object` (Kotlin 1.9+) instead of a plain `object` for a no-argument singleton case of a sealed hierarchy that should still print a meaningful `toString()` (`data object Loading` prints `"Loading"` instead of the default `object`'s memory-address-style string).
- **DON'T:** Give a data class an unbounded number of properties as a way to avoid designing a proper aggregate — beyond roughly five to seven properties, consider whether the type is actually several smaller, named concepts that should be grouped into nested value objects instead of one flat, long parameter list.
- **DO:** Validate invariants in a data class's `init` block (`init { require(amount >= BigDecimal.ZERO) { "amount must not be negative" } }`) so it's structurally impossible to construct an invalid instance, rather than trusting every call site to validate before construction.
  ```kotlin
  data class Money(val amount: BigDecimal, val currency: Currency) {
      init {
          require(amount >= BigDecimal.ZERO) { "amount must not be negative: $amount" }
      }
  }
  ```
- **DON'T:** Rely on a data class's positional `componentN()` destructuring in contexts where the property order might plausibly change later (a public API surface) — positional destructuring silently reassigns to the wrong variable if two properties of the same type swap order, with no compiler error. Prefer named property access at public boundaries and reserve destructuring for tightly-scoped, easily-reviewed local usage (like `for ((key, value) in map)`).
- **DO:** Use `@JvmRecord` when a Kotlin data class needs to interoperate as a genuine Java `record` for callers on the Java side, rather than assuming Kotlin data classes and Java records are automatically interchangeable across the language boundary.

- **DO:** Use `copy()` cautiously on data classes that wrap a mutable collection property — `copy()` performs a shallow copy, so the new instance shares the same underlying mutable collection reference as the original unless the property itself is copied explicitly, which can produce surprising aliasing bugs.
  ```kotlin
  data class Cart(val items: MutableList<Item>)
  val original = Cart(mutableListOf(item1))
  val copy = original.copy()
  copy.items.add(item2) // also visible through `original.items` — shared reference
  ```
- **DON'T:** Use a data class to model a type where two logically different instances should never compare equal even with identical field values (e.g. two separately-issued but coincidentally identical tokens) — structural equality is the wrong tool there; use a regular class with identity semantics, or include a distinguishing field.
- **DO:** Prefer `toString()`'s auto-generated, readable output for logging/debugging data classes over manually formatting fields with string concatenation — but override it explicitly when the default field dump would be too verbose or expose sensitive data.
- **DO:** Combine data classes with `Comparable`/a `Comparator` built via `compareBy` for natural ordering, rather than implementing manual multi-field comparison logic by hand.
  ```kotlin
  data class Employee(val lastName: String, val firstName: String, val salary: Int)
  val byNameThenSalary = compareBy<Employee> { it.lastName }.thenBy { it.firstName }.thenByDescending { it.salary }
  ```

## Sealed Classes

- **DO:** Use `sealed class`/`sealed interface` to model a fixed, closed set of alternatives (e.g., a network result of `Success`, `Error`, `Loading`) so the compiler can enforce exhaustive handling in a `when` expression. This turns "you forgot to handle a new case" from a runtime bug into a compile-time error.
  ```kotlin
  sealed interface UiState {
      object Loading : UiState
      data class Success(val data: List<Item>) : UiState
      data class Error(val message: String) : UiState
  }

  fun render(state: UiState) = when (state) {
      is UiState.Loading -> showSpinner()
      is UiState.Success -> showItems(state.data)
      is UiState.Error -> showError(state.message)
      // no `else` needed — compiler verifies exhaustiveness
  }
  ```
- **DON'T:** Add an `else -> {}` branch to a `when` over a sealed hierarchy just to satisfy the compiler quickly. An `else` branch defeats the entire benefit of sealing the hierarchy — when a new subtype is added later, that new case silently falls into `else` instead of failing to compile until someone deliberately handles it.
- **DO:** Prefer `object` for sealed subtypes that carry no data (singletons like `Loading` above) and `data class`/`data object` (Kotlin 1.9+) for variants that do carry data, rather than giving every variant a full class declaration with an empty body.
- **DON'T:** Reach for a plain `enum class` when variants need to carry different associated data per case — an `enum` entry can hold fields, but every entry is forced to share the same shape; a sealed hierarchy lets each subtype declare its own distinct properties.
- **DO:** Use sealed classes to model domain error types instead of a generic exception with a string reason or an integer error code. A sealed `PaymentError` hierarchy (`InsufficientFunds`, `CardDeclined(reason: String)`, `NetworkFailure`) is exhaustively `when`-able and self-documenting, unlike a bag of magic strings.
- **DO:** Keep sealed hierarchies in the same file/module as their base type unless the language version's cross-file sealed support is deliberately used — this keeps the full set of permitted subtypes visible in one place, which is the entire point of sealing.
- **DON'T:** Model what is really a two-state boolean flag as a sealed class hierarchy "for consistency" — sealed classes earn their ceremony when there are meaningfully different associated payloads or more than two truly distinct states; for a plain true/false condition, a `Boolean` or a two-value `enum` is simpler and equally clear.

- **DO:** Use exhaustive `when` expressions (not statements) over a sealed hierarchy as the return value of a function whenever possible — a `when` used as an expression forces the compiler to verify exhaustiveness for the assignment/return to type-check, which is a stronger guarantee than a `when` statement that merely executes branches with no required return value.
- **DON'T:** Model a sealed hierarchy's variants as `enum class` entries implementing a shared `sealed`-like interface just to get associated behavior per case, when a genuine `sealed class` hierarchy with per-subtype properties would represent the same thing more directly and let each variant carry different data shapes.
- **DO:** Combine sealed classes with Kotlin's pattern-matching-style `when` on multiple conditions (guard conditions in `when` branches, Kotlin 2.1+) or nested `is` checks to express complex per-variant logic clearly, rather than falling back to a chain of `if`/`else if`/`instanceof`-style checks that the sealed hierarchy was meant to replace.
- **DO:** Place sealed subtypes as nested classes/objects inside the sealed parent (`sealed interface Result { data class Ok(...) : Result; data object Empty : Result }`) when the variants are small and tightly coupled to the parent's identity — this keeps `Result.Ok`/`Result.Empty` naming self-documenting at call sites versus top-level types that need a shared prefix to convey the same relationship.

- **DO:** Use sealed interfaces (rather than sealed classes) as the default when the hierarchy's variants don't need to share common constructor-initialized state — interfaces permit a subtype to also inherit from an unrelated class if ever needed, which a sealed class's single-inheritance restriction would block.
- **DON'T:** Reach for a sealed class to represent a hierarchy that's genuinely open-ended and expected to grow from outside the module (e.g. a plugin system where third parties register new variants) — sealed hierarchies are specifically for closed, fully-known sets; an open interface without sealing (or an enum-like registry pattern) fits an extensible set of variants better.
- **DO:** Use `when` expression exhaustiveness on sealed hierarchies as a deliberate design tool during refactoring — adding a new sealed subtype and letting the compiler enumerate every `when` that now needs updating is a safe, guided way to extend behavior consistently across a codebase.
- **DO:** Give each sealed subtype a name that reads naturally after the parent's name at call sites (`NetworkResult.Success`, `NetworkResult.Failure`) rather than a name that only makes sense in isolation — since callers usually reference subtypes qualified by their parent.

## Coroutines

- **DO:** Launch coroutines within a structured `CoroutineScope` tied to a well-defined lifecycle (`viewModelScope`, `lifecycleScope`, a scope created with `coroutineScope { }`/`supervisorScope { }`, or a scope owned by a service and cancelled on shutdown) so cancellation of the parent reliably cancels all child work. Structured concurrency is what makes coroutine cancellation and error propagation predictable.
- **DON'T:** Launch coroutines on `GlobalScope`. `GlobalScope.launch` creates work with no parent to cancel it, no structured error propagation, and a lifetime tied to the entire application process — it routinely causes leaked work that outlives the component that started it and silently swallowed exceptions.
  ```kotlin
  // DON'T
  fun refreshData() {
      GlobalScope.launch { repository.refresh() }
  }

  // DO
  class DataViewModel(private val repository: Repository) : ViewModel() {
      fun refreshData() {
          viewModelScope.launch { repository.refresh() }
      }
  }
  ```
- **DO:** Use `withContext(Dispatchers.IO)` to move blocking I/O work off the calling dispatcher, and `Dispatchers.Default` for CPU-bound work — but only wrap the specific blocking call, not the entire surrounding function, so the rest of the coroutine's logic stays on its original dispatcher.
- **DON'T:** Call a blocking function (JDBC, blocking HTTP client, `Thread.sleep`) directly inside a coroutine running on `Dispatchers.Main` or an unconfined dispatcher. This blocks the thread the coroutine dispatcher is using for other work — on Android it freezes the UI thread; in a server context it can starve the dispatcher's limited thread pool.
- **DO:** Use `async`/`await` for concurrent work whose results you actually need combined, and plain sequential `suspend` calls otherwise — reaching for `async { }` on every suspend call just to "be async" adds needless coroutine overhead and makes error handling harder to reason about for no benefit.
  ```kotlin
  // DO — concurrent when results are genuinely independent
  suspend fun loadDashboard(): Dashboard = coroutineScope {
      val user = async { fetchUser() }
      val orders = async { fetchOrders() }
      Dashboard(user.await(), orders.await())
  }
  ```
- **DON'T:** Swallow a `CancellationException` in a generic `catch (e: Exception)` block inside a coroutine. `CancellationException` is how coroutine cancellation propagates internally; catching and discarding it (rather than rethrowing) breaks the ability to cancel that coroutine's structured work tree, silently leaving orphaned work running.
  ```kotlin
  // DON'T
  try {
      doWork()
  } catch (e: Exception) {
      log.error("failed", e) // also swallows CancellationException
  }

  // DO
  try {
      doWork()
  } catch (e: CancellationException) {
      throw e
  } catch (e: Exception) {
      log.error("failed", e)
  }
  ```
- **DO:** Prefer `supervisorScope`/`SupervisorJob` when sibling coroutines should fail independently (one child's failure shouldn't cancel unrelated siblings), and plain `coroutineScope`/structured `Job` when a failure in any child should indeed cancel the whole group. Picking the wrong one either lets a real failure go unnoticed or cancels unrelated work that should have kept running.
- **DON'T:** Use `runBlocking` inside application code that already runs on a coroutine, or inside library code intended to be called from suspend functions. `runBlocking` blocks the calling thread until its coroutine completes, which — nested inside another coroutine's dispatcher — defeats the purpose of using coroutines at all and can deadlock a limited-thread dispatcher; it belongs at the true entry point (a `main()` function, a test's `runTest`), not deep in business logic.
- **DO:** Use `Flow` for asynchronous streams of multiple values over time (repeated emissions, reactive data sources) and plain `suspend` functions for a single asynchronous result — using a `Flow` that only ever emits once is unnecessary ceremony versus a plain suspend function.
- **DO:** Handle exceptions in a `Flow` with `.catch { }` and control concurrency/collection with the appropriate operators (`.flowOn` for dispatcher switching, `.collect` at the true consuming boundary) rather than wrapping the whole collecting `try`/`catch` around business logic that has nothing to do with the flow itself.
- **DON'T:** Forget to cancel or structure a coroutine started in response to a UI event (a button click starting a fire-and-forget `launch`) without a scope tied to that UI component's lifecycle — a user navigating away while the coroutine is still running should either cancel it or the coroutine should be explicitly designed to survive that navigation, not accidentally do either based on unrelated behavior.

- **DO:** Use `Mutex` from `kotlinx.coroutines.sync` instead of JVM `synchronized`/`ReentrantLock` to guard shared mutable state accessed from coroutines. Blocking JVM locks inside a suspend function can block the underlying dispatcher thread for other coroutines scheduled on it, while `Mutex.withLock` suspends without blocking the thread.
  ```kotlin
  // DON'T — blocks a dispatcher thread while held
  synchronized(lock) { sharedState += 1 }

  // DO
  mutex.withLock { sharedState += 1 }
  ```
- **DON'T:** Launch an unbounded number of concurrent coroutines against a shared, rate-limited resource (an external API, a small connection pool) without a concurrency limiter. Use a `Semaphore` or a bounded dispatcher/limited-parallelism context to cap how many coroutines can be doing that work simultaneously, just as you would bound a thread pool.
- **DO:** Prefer `channelFlow`/`callbackFlow` to bridge a callback-based API into a `Flow`, correctly awaiting cleanup via `awaitClose { }`, rather than manually wiring a `Channel` and forgetting to close it on cancellation.
- **DO:** Use `Dispatchers.Main.immediate` (in UI-facing coroutine code) rather than plain `Dispatchers.Main` when a coroutine should continue synchronously on the current thread if it's already the main thread, avoiding an unnecessary dispatch round-trip for what's effectively already-there work.
- **DON'T:** Treat `yield()` and `delay()` as equivalent ways to "give other coroutines a chance to run." `yield()` cooperatively yields without necessarily waiting; `delay()` suspends for a specific duration. Using `delay(1)` where `yield()` was intended adds unnecessary, arbitrary latency to code that should just cooperatively hand off control.
- **DO:** Test coroutine code that launches child coroutines by asserting on the parent scope's completion (`advanceUntilIdle()` in `runTest`) rather than adding real delays or manual polling in the test to "wait long enough."

- **DO:** Understand that `launch` returns a `Job` (fire-and-forget from the caller's perspective, but still structured under its parent) while `async` returns a `Deferred<T>` (a result must eventually be retrieved with `.await()`) — pick `launch` when the coroutine's only purpose is a side effect, and `async` only when you genuinely need its return value.
- **DON'T:** Call `.await()` on every `Deferred` immediately after creating it in sequence, which defeats concurrency just as surely as not using `async` at all — start every `async` you intend to run concurrently first, then await them together (individually or via `awaitAll()`).
  ```kotlin
  // DON'T — each async blocks on await before the next starts, so nothing overlaps
  val a = async { fetchA() }.await()
  val b = async { fetchB() }.await()

  // DO
  val deferredA = async { fetchA() }
  val deferredB = async { fetchB() }
  val (a, b) = awaitAll(deferredA, deferredB) // or deferredA.await(), deferredB.await()
  ```
- **DO:** Use `withTimeout`/`withTimeoutOrNull` to bound how long a suspend function is allowed to run, rather than relying on the caller to independently manage a timer — timeout logic belongs close to the operation whose latency it's bounding.
- **DO:** Prefer `Dispatchers.IO`'s bounded elasticity for blocking calls originating from coroutine code over manually spinning up raw threads — `Dispatchers.IO` is specifically sized and shared for exactly this kind of blocking-call offload.

## Extension Functions Discipline

- **DO:** Use extension functions to add cohesive, narrowly-scoped utility behavior to a type you don't own (a `String.toSlug()`, a `LocalDate.isWeekend()`) where the behavior reads naturally as if it were a member of that type. This is Kotlin's idiomatic alternative to a static `StringUtils.toSlug(str)` utility-class method.
  ```kotlin
  fun String.toSlug(): String = trim().lowercase().replace(Regex("\\s+"), "-")
  ```
- **DON'T:** Define extension functions with generic, ambiguous names on very common types (`Any.process()`, `String.handle()`) that pollute autocomplete for every value of that type across the whole codebase. A vague extension on a ubiquitous type like `Any` or `String` is discoverable by nobody and creates naming collisions as the codebase grows.
- **DO:** Scope extension functions as narrowly as their usage warrants — a private extension function local to the file that uses it, or one placed in a small, clearly-named utility file, rather than dumping every extension function ever written into one global `Extensions.kt` file that becomes an unstructured grab-bag.
- **DON'T:** Use an extension function where a genuine member function belongs — specifically, don't add extension functions to a type you *do* own as a workaround for "the class is getting too big," when the real fix is splitting the class's responsibilities. Extensions on types you control should be reserved for optional, context-specific behavior, not core class behavior artificially externalized.
- **DO:** Prefer extension functions over inheritance-based utility classes for cross-cutting formatting/validation helpers, since extensions don't require modifying or subclassing the original type and compose cleanly across unrelated type hierarchies.
- **DON'T:** Shadow or override the semantics of a member function with a confusingly similarly-named extension function. If a member function and an extension function have the same name and signature, the member always wins silently at the call site, and this can make an extension function appear to do nothing when called on an object that actually has a matching member.
- **DO:** Use extension functions on nullable receiver types (`fun Collection<T>?.isNullOrEmpty()` — actually part of the standard library, and a good pattern to follow) when a null-safe convenience genuinely reduces repeated boilerplate at many call sites across the codebase.

- **DO:** Use infix extension functions (`infix fun`) only for functions that read naturally as a binary operator or verb between two operands (Kotlin's own `to` for creating `Pair`s is the canonical example) — forcing infix notation onto a function that doesn't read like a natural verb phrase makes call sites more confusing, not less.
- **DON'T:** Define extension properties that hide expensive computation behind what looks like a cheap field access (`val List<Item>.total: BigDecimal get() = sumOf { it.price }` executed repeatedly in a hot loop). An extension property still looks like a field at the call site; if it does real work, name it as a function (`fun List<Item>.total(): BigDecimal`) so its cost is visible.
- **DO:** Scope an extension function as a member extension (declared inside another class) when it should only be callable within that class's context and receiver combination, rather than making it a top-level extension broadly visible to every file that imports it.
- **DO:** Reach for a top-level extension function on a third-party or JDK type before writing a static utility method with an explicit first parameter (`StringUtils.slugify(str)`), since the extension-function call site (`str.slugify()`) reads more naturally and chains cleanly with other calls on the same value.

- **DON'T:** Write an extension function whose behavior depends on hidden external mutable state rather than purely on its receiver and parameters — an extension function that appears to be a pure transformation but secretly reads/writes a shared field elsewhere surprises callers who reasonably expect extension functions on immutable receivers to behave predictably.
- **DO:** Document a non-obvious extension function with KDoc just as you would a member function, especially one added to a widely-used type — its usage sites won't have the declaration nearby for context, unlike a method visible in the class body it's called on.
- **DO:** Group related extension functions for a specific type into a clearly-named file (`StringExtensions.kt`, `LocalDateExtensions.kt`) so they're discoverable by filename, rather than scattering logically related extensions across unrelated files.

## Idiomatic Collections Operators

- **DO:** Use Kotlin's collection operators (`map`, `filter`, `associateBy`, `groupBy`, `fold`, `sortedBy`, `distinctBy`) for transformations, since they're expression-oriented, null-aware where relevant, and generally read more directly than an equivalent manual loop with a mutable accumulator.
  ```kotlin
  val activeEmailsByDomain = users
      .filter { it.isActive }
      .groupBy { it.email.substringAfter("@") }
  ```
- **DON'T:** Chain collection operators so deeply that the pipeline requires several read-throughs to understand, especially when each intermediate operator allocates a new list. For long pipelines, either break the chain into named intermediate `val`s, or switch to a `sequence { }` when eagerness/laziness actually matters.
- **DO:** Use `asSequence()` for large collections with multi-stage transformation pipelines where intermediate list allocation would be wasteful — sequences evaluate lazily, element by element, avoiding a fully-materialized intermediate list at every stage. For small collections, the eager, list-based operators are simpler and the laziness overhead isn't worth it.
- **DON'T:** Use `!!` inside a `map`/`filter` lambda to force-unwrap a nullable element instead of using `mapNotNull` to both filter out nulls and transform in one step.
  ```kotlin
  // DON'T
  val names = users.map { it.nickname!! }

  // DO
  val names = users.mapNotNull { it.nickname }
  ```
- **DO:** Reach for the specific operator that names the intent directly — `firstOrNull { }` instead of `filter { }.firstOrNull()`, `any { }`/`none { }` instead of `filter { }.isNotEmpty()`, `count { }` instead of `filter { }.size` — since these avoid building an intermediate collection just to check a condition, and are more directly readable.
- **DON'T:** Default to mutable collections (`mutableListOf`, `ArrayList`) for values that, once built, are never modified again. Build with a mutable collection only where genuinely needed, then expose/return the immutable `List`/`Map`/`Set` interface — Kotlin's read-only collection interfaces communicate intent to callers even though they don't provide the same deep runtime immutability guarantee as Java's `List.of`.
- **DO:** Use destructuring declarations for `Pair`/`Map.Entry`/data class returns (`for ((key, value) in map)`, `val (first, second) = pair`) where it improves readability over positional `.first`/`.second`/`.component1()` access.
- **DON'T:** Use `associate { }` when `associateBy { }` or `associateWith { }` says exactly what's intended — `associate` requires constructing a full `Pair` in the lambda for every element, whereas `associateBy`/`associateWith` communicate "keyed by this" or "paired with this computed value" directly and avoid the extra `Pair` allocation and syntax.
- **DO:** Prefer `buildList { }`/`buildMap { }`/`buildSet { }` for imperative-style collection construction inside a function, rather than declaring a mutable collection, populating it with a loop, and separately converting it to read-only at the end.

- **DON'T:** Chain `.filter { }.map { }` when `.mapNotNull { }` (combining a filter and transform into one null-aware pass) directly expresses the same intent, or when a single-pass `.fold`/`.sumOf` would avoid materializing an intermediate list that a two-stage `.filter().map()` builds and discards.
- **DO:** Use `.sumOf { }`, `.maxByOrNull { }`, `.minByOrNull { }`, and `.partition { }` for the specific aggregation they name directly, rather than hand-rolling the equivalent with `.fold` or a manual loop — the named operators communicate intent immediately and are already correctly handling edge cases (like an empty collection for the `OrNull` variants).
- **DON'T:** Call `.first()`/`.last()`/`.maxBy { }` (the non-nullable, throwing variants) on a collection whose emptiness is a real, expected possibility. Use the `OrNull` counterparts and handle the `null` case explicitly, reserving the throwing variants for collections that are a structural precondition to be non-empty.
- **DO:** Use `Sequence` builders (`sequence { yield(...) }`) for genuinely infinite or expensive-to-fully-generate sequences, matching Kotlin's lazy sequence semantics to the actual shape of the data source rather than forcing an eager `List` construction upfront.

- **DO:** Use `withIndex()` when both the element and its index are needed in an iteration (`for ((index, item) in list.withIndex())`), rather than manually tracking a separate counter variable alongside a `for` loop.
- **DON'T:** Use `+=`/`-=` on a `val`-declared read-only collection reference expecting in-place mutation — for a `List` (read-only interface) these operators actually create and reassign to a *new* list, which fails to compile against a `val` and can be a source of confusion about whether a collection is being mutated or replaced; use `MutableList` with real in-place `add`/`remove` when in-place mutation is genuinely intended.
- **DO:** Use `Map.forEach { (key, value) -> ... }` with destructuring directly in the lambda parameter list rather than accessing `.key`/`.value` on an `Map.Entry` repeatedly inside the body.

## Idiomatic Function & API Design

- **DO:** Use default parameter values instead of several overloaded functions that differ only by a trailing subset of parameters. One function declaration with defaults expresses the same API surface as multiple Java-style overloads, with less duplication to keep in sync.
  ```kotlin
  // DON'T — Java-style overloads
  fun connect(host: String) = connect(host, 443)
  fun connect(host: String, port: Int) = connect(host, port, timeoutMs = 5000)

  // DO
  fun connect(host: String, port: Int = 443, timeoutMs: Int = 5000) { ... }
  ```
- **DO:** Use named arguments for calls involving multiple parameters of the same type or several boolean flags, so the call site remains unambiguous without needing to check the function signature.
  ```kotlin
  createUser(name = "Ada", isAdmin = false, sendWelcomeEmail = true)
  ```
- **DON'T:** Wrap a single small utility function inside an `object Utils { fun ... }` singleton purely out of habit from languages that require every function to live inside a class. Kotlin supports top-level functions directly in a file — use them for standalone operations that don't belong to any particular type.
- **DO:** Use trailing-lambda syntax for a function's last parameter when it's a function type, enabling clean, DSL-like call sites (`transaction { ... }`, `repository.findAll { it.isActive }`) instead of an awkward parenthesized lambda argument.
- **DO:** Use higher-order functions to capture reusable control-flow patterns (`fun <T> retry(times: Int, block: () -> T): T`) instead of duplicating the same loop-and-try structure at every call site that needs retry behavior.
- **DON'T:** Overload operators (`plus`, `get`, `invoke`, `rangeTo`) for operations without an intuitive mathematical or collection-like meaning. A surprising operator overload (e.g. `+` triggering a network call) makes code harder to reason about, not easier, since readers bring strong pre-existing expectations about what these operators mean.
- **DO:** Reserve nullable parameter types for genuinely optional-or-absent values, and use non-null parameters with a sensible default for "optional with a standard fallback" — conflating the two (`fun greet(name: String? = null)` when a sensible non-null default like `"there"` exists) forces every caller to handle a null case that a default value would have avoided entirely.

## Interoperability with Java

- **DO:** Annotate Kotlin APIs meant to be consumed from Java with `@JvmStatic` (companion object members as real static methods), `@JvmOverloads` (generating overloads for default parameters), and `@JvmName` (choosing a Java-friendly method/file name) so the compiled bytecode is ergonomic to call from the Java side.
  ```kotlin
  class Config @JvmOverloads constructor(val timeout: Int = 5000, val retries: Int = 3)
  // without @JvmOverloads, Java callers must supply both parameters every time
  ```
- **DON'T:** Assume Kotlin's default parameter values are visible to Java callers without `@JvmOverloads`. Java sees only the single, full-parameter-list method/constructor by default — omitting the annotation forces every Java caller to pass every parameter explicitly, defeating the purpose of having defaults in the first place.
- **DO:** Treat every value returned from unannotated Java APIs as a platform type and establish its real nullability immediately at the boundary (an explicit `?` type, or a validated non-null assertion), rather than letting the ambiguous platform type propagate deeper into Kotlin code where its true nullability gets forgotten.
- **DO:** Use `@JvmField` on simple, non-null public properties that are meant to be accessed as plain fields from performance-sensitive Java call sites, avoiding the getter-method call overhead Kotlin properties otherwise compile to.
- **DO:** Decide deliberately, and consistently across the codebase, how checked exceptions from a wrapped Java library are handled at the Kotlin boundary — letting them propagate as effectively-unchecked (since Kotlin has no checked exceptions) is valid, but it should be a conscious choice, not an accident of how the wrapper happened to be written.

## Type-Safe Builders & DSL Design

- **DO:** Use lambda-with-receiver parameters (`fun html(block: Html.() -> Unit)`) to build type-safe DSLs, giving a builder block direct, scoped access to the receiver's members without needing an explicit prefix at every call — this is the mechanism behind Kotlin DSLs like `kotlinx.html` and Gradle's Kotlin DSL.
  ```kotlin
  fun html(block: HtmlBuilder.() -> Unit): String = HtmlBuilder().apply(block).build()

  val page = html {
      head { title("Home") }
      body { p("Welcome") }
  }
  ```
- **DON'T:** Over-engineer a DSL for a simple, flat configuration object that a plain data class with named-argument construction already serves well. DSLs earn their added complexity for genuinely nested, structural configuration (routing tables, UI trees, test specifications) — not for a handful of unrelated flat fields.
- **DO:** Apply `@DslMarker` to a DSL's marker annotation so nested builder scopes can't accidentally call an outer scope's builder methods from within an inner block — this closes off a common category of DSL misuse where implicit receivers from different nesting levels get confused with each other.
- **DO:** Keep the mutable builder class itself internal or private, exposing only the resulting immutable data class/object publicly, so the mutable construction-time state used while building can't leak out and be mutated again after the DSL block completes.
- **DON'T:** Nest DSL blocks more than two or three levels deep without extracting pieces into named, reusable functions. An overly deep DSL becomes just as hard to read as the imperative alternative it was meant to replace.
- **DO:** Provide sensible defaults within a DSL's builder scope so the common, simple case requires minimal boilerplate, while still allowing full customization for advanced cases — this balance is a large part of what makes a DSL worth its design investment over a plain constructor call.

## Delegation & Property Delegates

- **DO:** Use interface delegation (`class CachingRepository(repo: Repository) : Repository by repo { /* override only what changes */ }`) to compose behavior from an existing implementation, overriding only the specific methods that need different behavior — this avoids hand-writing a full set of manual forwarding methods for every unchanged member of the interface.
  ```kotlin
  interface Repository { fun findById(id: String): Item? }

  class CachingRepository(
      private val delegate: Repository,
      private val cache: MutableMap<String, Item?> = mutableMapOf()
  ) : Repository by delegate {
      override fun findById(id: String): Item? =
          cache.getOrPut(id) { delegate.findById(id) }
  }
  ```
- **DO:** Use `Delegates.observable`/`Delegates.vetoable` (`kotlin.properties`) for properties that need to react to or validate changes, instead of hand-writing a custom setter with manual old-value/new-value bookkeeping for what's ultimately a common, well-understood pattern.
- **DO:** Write a custom property delegate (implementing the `getValue`/`setValue` operator functions) when the same get/set behavior — reading from a shared cache, a thread-local, a preferences store — repeats across many unrelated properties, centralizing that logic in one reusable delegate class instead of duplicating custom accessors everywhere it's needed.
- **DON'T:** Reach for a custom property delegate for a one-off property whose special behavior is only used in a single place. The delegate abstraction pays off through reuse across multiple properties; for a single property, a plain custom getter/setter is simpler and equally clear.
- **DO:** Use `by Delegates.notNull<T>()` sparingly, for non-null properties that genuinely can't use `lateinit` (e.g. a non-reference/primitive type) but must still be assigned before first read outside the constructor — understanding it throws the same "read before assignment" style exception `lateinit` would.

## Idiomatic Control Flow

- **DO:** Use `when` as an expression for multi-branch logic that produces a value, replacing an `if`/`else if` chain that assigns to the same variable in every branch with one expression the compiler can verify covers every case (when the subject is a sealed type or `Boolean`).
  ```kotlin
  // DON'T
  val label: String
  if (status == Status.PAID) { label = "Paid" }
  else if (status == Status.PENDING) { label = "Pending" }
  else { label = "Unknown" }

  // DO
  val label = when (status) {
      Status.PAID -> "Paid"
      Status.PENDING -> "Pending"
      else -> "Unknown"
  }
  ```
- **DO:** Use range expressions (`in 1..10`, `until`, `downTo`, `step`) for range checks and iteration, instead of manual boundary comparisons (`x >= 1 && x <= 10`) that are easier to get subtly wrong at the edges.
- **DON'T:** Use a `while (true) { ... break ... }` loop where a more direct standard-library construct (`repeat(n) { }` for a fixed count, `generateSequence` for a lazily-produced sequence) already expresses the same intent without hand-managing a loop-and-break.
- **DO:** Use labeled breaks/continues (`outer@ for (...) { ... break@outer }`) sparingly, reserved for genuinely nested loop structures — and prefer extracting the nested loop into its own function with a plain early `return` when that reads more clearly than introducing a label.
- **DO:** Use early returns (guard clauses) for precondition checks at the top of a function instead of wrapping the entire function body in a single deeply-nested `if` block — flattening the "happy path" to the end of the function generally improves readability over nesting it inside successful-case conditionals.
- **DON'T:** Use the subject-less form of `when` (`when { condition1 -> ...; condition2 -> ... }`) as a reflexive substitute for `if`/`else if` when a subject-based `when (x)` matching directly on a single value would be clearer and avoid repeating that value's name in every branch condition.

## Companion Objects & Static-Like Members

- **DO:** Use a `companion object` for factory methods and constants genuinely tied to a specific class, giving them a natural, discoverable home (`Order.create(...)`, `Order.MAX_ITEMS`) instead of scattering related static-like helpers across a separate top-level utility file.
  ```kotlin
  class Order private constructor(val items: List<Item>) {
      companion object {
          const val MAX_ITEMS = 100
          fun create(items: List<Item>): Order {
              require(items.size <= MAX_ITEMS) { "too many items" }
              return Order(items)
          }
      }
  }
  ```
- **DON'T:** Cram unrelated static-like utility functions into a class's companion object just because "it needs to live somewhere." A companion object should hold members genuinely related to its enclosing class's construction or identity — unrelated helpers belong as top-level functions or in their own utility file.
- **DO:** Name a companion object explicitly (`companion object Factory { ... }`) when there's a natural, meaningful name for it, or when a class has multiple nested objects that need to be distinguished from each other — otherwise the default unnamed companion is fine and more concise.
- **DO:** Use `@JvmStatic` on companion object members intended for Java callers, so they compile to genuine static methods rather than requiring Java code to route through the generated `Companion` singleton instance.
- **DON'T:** Reach for a companion object purely as a reflexive workaround for "there's no `static` keyword in Kotlin," applied even to constants with no real tie to a specific class. A top-level `const val` is simpler and equally accessible for a genuinely standalone constant.

## Visibility Modifiers & Module Boundaries

- **DO:** Default to the most restrictive visibility that still satisfies actual callers — `private` for implementation details, `internal` for module-internal APIs not meant for external consumers, `public` reserved for genuine external API surface — rather than defaulting everything to `public` out of convenience.
- **DON'T:** Leave a class or property at Kotlin's default `public` visibility without a deliberate reason when `internal` would correctly scope it. Because Kotlin defaults to public (unlike some languages that default to package-private), an unreviewed declaration silently becomes part of the published API surface unless visibility is considered explicitly at the point it's written.
- **DO:** Use `internal` for classes and functions that need to be shared across files within a module but shouldn't be part of a published library artifact's public contract exposed to its consumers.
- **DO:** Use a `private` constructor paired with a public factory function or companion-object method when construction needs validation or a specific creation path, preventing callers from bypassing that validation by calling the raw constructor directly.
- **DON'T:** Expose mutable internal state through a `public var` property when a publicly-readable-but-internally-mutable property (`var foo: Int private set`) precisely expresses the intended access pattern — full `public var` grants external callers write access they were never meant to have.
  ```kotlin
  class Account {
      var balance: BigDecimal = BigDecimal.ZERO
          private set // readable from anywhere, only mutable from within Account

      fun deposit(amount: BigDecimal) { balance += amount }
  }
  ```

## Inline Functions & Reified Type Parameters

- **DO:** Mark small, higher-order functions taking a lambda parameter `inline` when they're called frequently, so the lambda body is inlined directly at the call site rather than allocating a `Function` object on every invocation — this is precisely why standard-library functions like `let`, `run`, `map`, and `filter` are all declared `inline`.
- **DON'T:** Mark a large function body `inline`. Inlining duplicates the function's bytecode at every call site; a large inlined function bloats the compiled output disproportionately to whatever performance benefit is actually gained.
- **DO:** Use `reified` type parameters on `inline` functions specifically when the function needs real runtime access to the generic type (`is T`, `T::class`) — `reified` is only possible on inline functions, since the compiler works around type erasure by substituting the concrete type at each individual call site during inlining.
  ```kotlin
  inline fun <reified T> Gson.fromJson(json: String): T = fromJson(json, T::class.java)
  val user: User = gson.fromJson(jsonString) // no Class<T> argument needed
  ```
- **DON'T:** Mark a function `inline` purely out of habit "for performance" when it has no lambda parameter that would actually benefit. Inlining a function with no function-type parameters provides no meaningful benefit and only bloats bytecode at every call site.
- **DO:** Use the `noinline`/`crossinline` modifiers deliberately on specific lambda parameters of an `inline` function when that particular lambda needs to be stored or passed elsewhere (`noinline`), or must not contain a non-local `return` because it executes in a different context — another lambda or a separate thread (`crossinline`).

## Testing

- **DO:** Use Kotlin idioms in tests just as in production code — trailing lambdas, named arguments for clarity in assertions, and a Kotlin-first testing library (Kotest, or JUnit 5 with `kotlin.test` / MockK) rather than writing Java-flavored Kotlin test code.
- **DO:** Use MockK (or a similarly Kotlin-aware mocking library) over Mockito for Kotlin code, since MockK handles Kotlin-specific constructs — `final` classes by default, extension functions, coroutines (`coEvery`/`coVerify`), and object mocking — that Mockito requires extra configuration or plugins to support.
  ```kotlin
  coEvery { repository.fetchUser(id) } returns testUser
  ```
- **DON'T:** Test `suspend` functions by wrapping them in `runBlocking` inside every test. Use `runTest` (from `kotlinx-coroutines-test`) instead — it runs on a test dispatcher that auto-advances virtual time, so tests involving `delay()` run instantly rather than actually waiting in real time, and it also surfaces uncaught coroutine exceptions as test failures.
- **DO:** Use Kotest's or JUnit 5's parameterized/table-driven test support for covering multiple input variations, favoring Kotlin-idiomatic spec styles (`StringSpec`, `BehaviorSpec`) when the team has standardized on Kotest, rather than forcing every test into a JUnit 4/5 style that fights the language's expressiveness.
- **DON'T:** Leave `!!` or force-unwrapped nullable assertions in test setup code just because "it's only a test." A `!!` that fails in test setup produces a much less informative failure than a proper assertion (`assertNotNull`, or a Kotest matcher) that reports what was actually expected.
- **DO:** Write test doubles for sealed class hierarchies as real subtype instances rather than mocking the sealed type itself — since sealed hierarchies are meant to be exhaustively matched, constructing a real `UiState.Error("boom")` in a test is both simpler and safer than mocking.
- **DO:** Assert on `Flow` emissions with the `turbine` library's `test { }` DSL for readable, sequential assertions on emitted values, rather than manually collecting a flow into a list and asserting on it after the fact for anything beyond the simplest case.

- **DO:** Use Kotest's property-based testing (`checkAll { ... }`) or `kotlin.test`'s data-driven equivalents for functions with a large, well-defined input domain (pure numeric/string transformations), catching edge cases that hand-picked example-based tests might miss.
- **DON'T:** Mark test classes/functions `open` or otherwise weaken Kotlin's default-final visibility purely to satisfy an older mocking library's requirement to subclass for mocking — prefer a mocking library (MockK) that supports mocking final classes directly via bytecode manipulation, rather than degrading production code's design for testability of a specific tool.
- **DO:** Use `mockk(relaxed = true)` deliberately and sparingly — it auto-stubs every unspecified call with a default return value, which is convenient for large interfaces but can silently hide the fact that a test never verified a call it should have; prefer explicit `every { }` stubbing for interactions the test actually cares about.
- **DO:** Structure Kotest specs (`BehaviorSpec`, `FunSpec`, `DescribeSpec`) consistently across a project — mixing several different Kotest spec styles in one codebase makes each test file's structure unpredictable to a new reader.

- **DO:** Use `assertSoftly` (Kotest) or manually collect multiple assertion failures together when a single test genuinely needs to verify several independent properties of one result, so a single run reports every failing property instead of stopping at the first.
- **DON'T:** Write test assertions using raw `==` comparison on data classes with a `Double`/`Float` property expecting exact equality — floating-point comparison needs an explicit tolerance (`shouldBe(expected plusOrMinus 0.001)` in Kotest, or a custom comparator) since exact equality is unreliable for computed floating-point values.
- **DO:** Keep test doubles (fakes implementing an interface with simple in-memory behavior) as a preferred alternative to mocks for repositories/collaborators with more than a couple of methods — a hand-written fake repository backed by a `MutableMap` is often easier to read and reuse across many tests than a long chain of `every { }` stubs.

## Common AI-Assistant Mistakes in Kotlin

- **DON'T:** Write "Java in Kotlin syntax" — manual getter/setter-style properties with explicit backing fields for simple cases, verbose `for` loops instead of collection operators, or explicit `null` checks with braces instead of safe calls and Elvis. This produces code that compiles and runs but ignores the language's actual idioms, and reads as foreign to a Kotlin-fluent reviewer.
  ```kotlin
  // DON'T — Java-style
  fun getActiveNames(users: List<User>): List<String> {
      val result = ArrayList<String>()
      for (u in users) {
          if (u.isActive()) {
              result.add(u.getName())
          }
      }
      return result
  }

  // DO — idiomatic Kotlin
  fun activeNames(users: List<User>): List<String> =
      users.filter { it.isActive }.map { it.name }
  ```
- **DON'T:** Sprinkle `!!` throughout generated code as a quick way to make nullable types compile against non-null expectations. This is one of the single most common generation mistakes — it defeats Kotlin's core null-safety guarantee and reintroduces exactly the runtime `NullPointerException`s the type system is designed to prevent at compile time.
- **DON'T:** Default to `GlobalScope.launch` for "fire and forget" async work in generated examples. This is a frequent pattern in outdated tutorials and produces leaked, uncancellable work; use a properly scoped coroutine launcher tied to the surrounding component's lifecycle instead.
- **DON'T:** Overuse scope functions (`let`, `run`, `apply`, `also`, `with`) by nesting them or chaining them where a plain statement would be clearer — e.g. wrapping a simple non-null property access in `?.let { it.foo() }` when the value is already known non-null, or chaining `.apply { }.let { }.also { }` in one expression. Each scope function should earn its use by genuinely improving readability (null-safety, fluent configuration, or scoping a temporary computation), not be applied reflexively to every expression.
  ```kotlin
  // DON'T — overused, unclear which `it`/`this` is in play
  return user.let { it.profile }.run { this?.settings }.also { println(it) }?.theme

  // DO
  val settings = user.profile?.settings
  println(settings)
  return settings?.theme
  ```
- **DON'T:** Confuse `apply` (returns the receiver, use for configuring an object) with `let` (returns the lambda result, use for transforming/null-checking a value) with `also` (returns the receiver, use for a side effect like logging) with `run`/`with` (returns the lambda result, evaluated with the receiver as `this`) — using the wrong one is a common generation error that either returns the wrong value or requires an unnecessary extra `.let` to correct it downstream.
- **DON'T:** Generate a `data class` for a type that clearly has identity-based equality needs (a database entity) or generate a plain `class` with manually written `equals`/`hashCode`/`copy`-equivalent boilerplate for a type that's obviously a value holder — pick the construct that matches the type's actual semantics rather than defaulting to whichever is more familiar from other languages.
- **DON'T:** Invent Kotlin standard library or coroutine functions that don't exist by pattern-matching on plausible-sounding names (e.g. a nonexistent `Flow.collectAsList()`, or assuming `withTimeout` returns `null` on timeout instead of throwing `TimeoutCancellationException`). Verify against the actual `kotlinx.coroutines`/stdlib API surface rather than guessing from the shape of similar functions.
- **DON'T:** Generate an `else` branch on every `when` over a sealed class/interface out of habit, even when exhaustiveness checking is exactly the reason the hierarchy was sealed in the first place. This quietly reintroduces the "forgot to handle a new case" bug class the sealed hierarchy was meant to eliminate.
- **DON'T:** Ignore existing project conventions around dependency injection (Hilt/Koin/Dagger annotations already in use) and generate manual constructor wiring or a different DI framework's annotations in isolation. Match the project's established DI approach rather than introducing a second, inconsistent one.
- **DON'T:** Generate a suspend function that internally launches a `Job` and returns immediately without awaiting it, presenting fire-and-forget behavior as if it were a completed, awaited operation. If a function is `suspend`, its caller reasonably expects it to have finished its work — including any child work it started — by the time it returns, unless explicitly documented otherwise.
- **DON'T:** Assume Java-interop nullability annotations always carry over correctly, and generate Kotlin code that treats every value from a Java library as strictly non-null without verifying. Unannotated Java APIs surface as Kotlin platform types (`String!`), and treating them as guaranteed non-null reintroduces the exact `NullPointerException` risk Kotlin's null safety is meant to eliminate.
- **DON'T:** Generate a class hierarchy using `open`/`abstract` inheritance for something that's a better fit for Kotlin's sealed classes or a simple composition-based strategy, defaulting to inheritance patterns more familiar from Java rather than the idiom Kotlin's type system specifically supports well.

## Quick Checklist
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
