# Java

## Naming & Style Conventions

- **DO:** Use lowercase, dot-separated, reverse-domain package names (e.g. `com.acme.billing.invoicing`) with no underscores or camelCase segments. This keeps classpath layout predictable and avoids collisions across libraries.
- **DO:** Name classes and interfaces with `UpperCamelCase` nouns or noun phrases (`InvoiceService`, `PaymentGateway`), and methods/fields with `lowerCamelCase` verb phrases for methods (`calculateTotal`) and noun phrases for fields (`totalAmount`). Consistent casing lets readers tell at a glance whether an identifier is a type, a member, or a constant.
- **DO:** Name constants with `UPPER_SNAKE_CASE` and declare them `static final`. This is the one place SCREAMING_SNAKE_CASE belongs in Java; using it elsewhere (e.g. for regular fields) signals confusion about mutability.
  ```java
  // DON'T
  public static final int maxRetryCount = 5;

  // DO
  public static final int MAX_RETRY_COUNT = 5;
  ```
- **DO:** Prefix boolean accessor methods and variables with `is`, `has`, `can`, or `should` (`isValid`, `hasPermission`, `canRetry`). A boolean read like a yes/no question is self-documenting at call sites: `if (order.isShippable())`.
- **DON'T:** Use Hungarian notation, type prefixes, or scope markers (`strName`, `m_total`, `IPayable` for an interface). Java's type system and IDE tooling already surface type information; encoding it in the name only adds noise and drifts out of sync when types change.
- **DON'T:** Abbreviate identifiers beyond well-known acronyms (`URL`, `ID`, `HTTP`). `usrRepo` and `calcTtl` save a few keystrokes and cost every future reader a moment of decoding; write `userRepository` and `calculateTotal`.
- **DO:** Treat acronyms as words in identifiers — `HttpClient` and `parseXmlDocument`, not `HTTPClient` or `parseXMLDocument` — per the prevailing convention in the JDK and the Google Java Style Guide. Mixed-case acronyms inside longer identifiers are hard to scan and inconsistent with method-name camelCasing.
- **DO:** Run an automated formatter (`google-java-format`, Spotless, or the IDE's built-in formatter bound to a shared config) and a linter (Checkstyle, PMD, Error Prone) in CI. Style debates and whitespace-only diffs disappear once formatting is mechanical rather than a matter of individual taste.
- **DON'T:** Hand-roll formatting choices per file — mixed brace styles, inconsistent import ordering, or ad hoc line-wrapping. Inconsistent style across a codebase makes diffs noisier than the actual change and signals that no one owns the code's readability.
- **DO:** Group and order imports deterministically (no wildcard imports, alphabetized within groups) and let the formatter enforce it. A wildcard import (`import java.util.*;`) hides exactly which classes are in play and risks silent name collisions as the codebase grows.
- **DON'T:** Use single-letter or cryptic variable names outside of tight, obviously-scoped loops (`for (int i = 0; ...)` is fine; `int i = getInventoryCount();` is not). A name should describe what the value represents, not how it was typed.
- **DO:** Prefer package-by-feature (`com.acme.billing`, `com.acme.shipping`, each containing its own controllers/services/repositories) over package-by-layer (`com.acme.controllers`, `com.acme.services`) for anything beyond a trivial application. Feature packages keep related classes co-located and make package-private visibility actually enforce encapsulation between features.
- **DO:** Write Javadoc on public classes, interfaces, and non-trivial public methods, describing behavior, invariants, and thrown exceptions — not restating the method name in prose. `/** Returns the customer's total. */` above `getTotal()` adds nothing; documenting rounding behavior, currency handling, or what happens on a null discount does.
- **DON'T:** Use raw generic types (`List`, `Map` without type parameters) in new code. Raw types disable compile-time type checking for the entire collection and reintroduce the class of bugs generics were added to prevent; always write `List<Invoice>`, `Map<String, Customer>`.

- **DO:** Name test classes with a convention your build tool's test discovery expects and your team recognizes (`OrderServiceTest`, or `OrderServiceShould` for behavior-style naming) so test reports and IDE navigation stay predictable.
- **DON'T:** Name utility/helper classes with a bare, unqualified `Utils`, `Helper`, or `Manager` suffix and no further context (`Utils.java` holding forty unrelated static methods). Name the class for what it actually operates on — `DateFormatters`, `HttpHeaderUtils` — so its contents are predictable from the name alone.
- **DO:** Keep member ordering consistent within a class (static fields, instance fields, constructors, public methods, then private helper methods) per a documented team convention, so navigating an unfamiliar class doesn't require hunting for where a particular kind of member lives.
- **DO:** Use `var` for local variable type inference (Java 10+) only where the right-hand side already makes the type obvious — a constructor call or a clearly-named factory method — not where it would hide a non-obvious type from the reader.
  ```java
  // DO — type is obvious from the right-hand side
  var invoices = new ArrayList<Invoice>();

  // DON'T — hides a non-obvious return type
  var result = process(request);
  ```
- **DON'T:** Name a `getX()`-style method that performs expensive computation or has side effects as if it were a cheap field accessor. Callers reasonably assume `getX()` is O(1) and side-effect-free; name it `computeX()`, `fetchX()`, or `loadX()` instead when that assumption doesn't hold.
- **DO:** Enforce a consistent maximum line length (typically 100–120 characters) via the formatter so multi-line diffs and side-by-side code review stay easy to scan.

- **DO:** Prefer the Google Java Style Guide or a similarly well-established public style guide as the codebase's baseline rather than inventing bespoke conventions from scratch — adopting a widely-known standard means new hires and tooling (formatters, linters) already understand it, and disputes get resolved by pointing at documented precedent instead of individual preference.
- **DON'T:** Let generated code introduce a second naming convention alongside an existing, established one already visible in the surrounding file (e.g. writing `snake_case` locals into a file that consistently uses `camelCase` elsewhere). Match the file's existing convention even when it isn't your personal preference; consistency within a file matters more than any one rule in isolation.
- **DO:** Name overloaded methods so their distinguishing parameter is obvious from context, and avoid overloading with parameter lists that are easy to confuse (two `int` parameters in different orders across overloads) — prefer distinctly named methods when the semantic difference is significant enough to be error-prone via overloading alone.
- **DO:** Keep annotation ordering consistent across similar declarations (e.g. always `@Override` before `@Deprecated` before a custom annotation) so annotated declarations are visually scannable across the codebase.
- **DON'T:** Let a single class file grow to hundreds or thousands of lines without addressing the underlying cause — file length is a weak proxy, but a file too large to hold in your head at once is a real signal that responsibilities should be split, not just an arbitrary style nit to silence.

## OOP Discipline

- **DO:** Favor composition over inheritance as the default way to reuse behavior — inject a collaborator and delegate to it rather than extending a base class to inherit its methods. Composition keeps types decoupled from a fixed superclass and avoids fragile-base-class problems where a change to the parent silently breaks every subclass.
  ```java
  // DON'T — inheriting for code reuse
  class ReportGenerator extends PdfWriter { /* ... */ }

  // DO — composing the capability
  class ReportGenerator {
      private final PdfWriter pdfWriter;
      ReportGenerator(PdfWriter pdfWriter) { this.pdfWriter = pdfWriter; }
  }
  ```
- **DO:** Reserve inheritance for genuine "is-a" relationships where the subtype must satisfy the Liskov Substitution Principle — any code written against the base type keeps working when handed a subtype. If a subclass narrows preconditions, widens postconditions unexpectedly, or throws `UnsupportedOperationException` on inherited methods, it does not belong in that hierarchy.
- **DON'T:** Build "god classes" that own unrelated responsibilities — a `UserManager` that handles authentication, email dispatch, report generation, and database migrations. Split by responsibility so each class has one reason to change; a class whose name needs "and" or "Manager"/"Helper"/"Util" to justify its scope is usually a sign that it should be several classes.
- **DO:** Design interfaces around what a specific consumer needs (Interface Segregation Principle) rather than one fat interface every implementer must fully satisfy. A `Printable`, `Scannable`, and `Faxable` split lets a simple printer implement only `Printable` instead of stubbing `scan()` and `fax()` with `UnsupportedOperationException`.
  ```java
  // DON'T
  interface MultiFunctionDevice { void print(); void scan(); void fax(); }
  class SimplePrinter implements MultiFunctionDevice {
      public void print() { /* ... */ }
      public void scan() { throw new UnsupportedOperationException(); }
      public void fax() { throw new UnsupportedOperationException(); }
  }

  // DO
  interface Printable { void print(); }
  interface Scannable { void scan(); }
  class SimplePrinter implements Printable { public void print() { /* ... */ } }
  ```
- **DO:** Keep fields `private` and expose behavior through methods rather than getters/setters for every field. A class that is just a bag of public getters and setters ("anemic domain model") pushes its logic out into unrelated service classes and loses the encapsulation OOP is meant to provide.
- **DON'T:** Expose mutable internal collections or arrays directly from a getter. Returning the live `List` field lets any caller mutate internal state behind the object's back; return an unmodifiable view or a defensive copy instead.
  ```java
  // DON'T
  public List<Item> getItems() { return items; }

  // DO
  public List<Item> getItems() { return List.copyOf(items); }
  ```
- **DO:** Favor immutability for value-like objects — final fields set once in the constructor, no setters, defensive copies of mutable inputs. Immutable objects are inherently thread-safe, easier to reason about, and safe to share and cache without synchronization.
- **DO:** Program to interfaces at collaboration boundaries (`List<T>`, `PaymentProcessor`) rather than concrete classes (`ArrayList<T>`, `StripePaymentProcessor`), so implementations can be swapped or mocked without touching consumers.
- **DON'T:** Use inheritance purely to share static utility-style logic between unrelated classes. A shared "helper" superclass with no real is-a relationship is a smell; extract that logic into a standalone utility class or an injected collaborator instead.
- **DO:** Prefer `sealed` classes/interfaces (Java 17+) when a type hierarchy is meant to be closed and exhaustively known — e.g. a small, fixed set of payment methods. Sealed types let the compiler enforce exhaustiveness in `switch` expressions, catching a missing case at compile time instead of at runtime.
  ```java
  sealed interface PaymentMethod permits CardPayment, BankTransfer, WalletPayment {}
  ```
- **DON'T:** Let a class's public API leak implementation details (exposing an internal `HashMap` type on a method signature, or a builder that mirrors internal field layout one-to-one). Consumers should depend on behavior and contract, not on how the class happens to be implemented today.
- **DO:** Apply the Single Responsibility Principle at the method level too — a method that validates input, calls three services, formats output, and logs is asking to be decomposed. Small, named methods with a single clear purpose are individually testable and self-documenting.
- **DON'T:** Reach for the Singleton pattern as a substitute for proper dependency injection. Hand-rolled singletons (static `getInstance()`) create hidden global state, make unit testing painful, and are superseded by a DI container's own singleton-scoped beans, which stay swappable in tests.
- **DO:** Use `equals()`/`hashCode()` overrides (or records, which generate them) whenever a type has value semantics and will be used as a map key, in a set, or compared for logical equality — and always override both together, consistently, using the same fields.
- **DO:** Mark classes and methods `final` by default unless they're explicitly designed and documented for extension, following the "design for inheritance or prohibit it" principle. An open class with no documented extension contract invites subclasses that violate invariants the base class relies on internally.
- **DON'T:** Write telescoping constructor overloads (three, four, five constructors differing only in how many optional parameters they accept). This forces callers to memorize which overload maps to which parameters; use a builder, or named-parameter-style static factory methods, instead.
  ```java
  // DON'T
  public Pizza(String size) { ... }
  public Pizza(String size, boolean cheese) { ... }
  public Pizza(String size, boolean cheese, boolean pepperoni) { ... }

  // DO
  Pizza pizza = new Pizza.Builder("large").cheese(true).pepperoni(true).build();
  ```
- **DO:** Use descriptively-named static factory methods (`Order.pending(...)`, `Order.fromLegacyRecord(...)`) over multiple overloaded constructors when the different construction paths carry distinct meaning — a name communicates intent that an overload's parameter list cannot.
- **DON'T:** Expose mutable `public static` non-final fields as a way to share data across classes. This is global mutable state wearing an OOP disguise, with all the same testability and thread-safety problems as a global variable in any other language.
- **DO:** Apply the Dependency Inversion Principle at architectural boundaries — define the interface in the business/domain module and let infrastructure (database access, HTTP clients, message queues) implement it, rather than having domain logic depend directly on infrastructure types.
- **DON'T:** Let domain/business entities extend framework base classes or implement framework-specific interfaces (a domain object extending a JPA `MappedSuperclass` with framework annotations threaded through business logic) unless that coupling is a deliberate, reviewed trade-off — it otherwise ties core business rules to a specific persistence or web framework.
- **DO:** Reach for a `record` instead of a full hand-written class whenever a type's sole purpose is to carry an immutable set of related values with structural equality — it eliminates the constructor/getter/`equals`/`hashCode`/`toString` boilerplate that a hand-written value class otherwise requires.
- **DO:** Model the Template Method or Strategy pattern with composition and functional interfaces (`Comparator`, `Function`, a small custom `@FunctionalInterface`) rather than an abstract base class with a `protected abstract` hook method, when the varying behavior is a single operation. A `Strategy` object passed in is easier to test in isolation and to swap at runtime than a hierarchy of subclasses each overriding one method.
  ```java
  // DON'T — inheritance-based strategy
  abstract class DiscountPolicy { abstract BigDecimal apply(BigDecimal price); }
  class NoDiscount extends DiscountPolicy { BigDecimal apply(BigDecimal p) { return p; } }

  // DO — composition with a functional interface
  interface DiscountPolicy { BigDecimal apply(BigDecimal price); }
  DiscountPolicy noDiscount = price -> price;
  ```
- **DON'T:** Let a class silently violate its own declared contract under specific inputs (e.g. a `Comparator` implementation that isn't actually transitive, or a `compareTo` inconsistent with `equals` without documenting why). Contract violations in core interfaces cause corrupted behavior in unrelated code — a `TreeSet` built on an inconsistent `Comparator` can silently "lose" elements that `equals()` would say are distinct.
- **DO:** Use the Observer pattern (or, more commonly today, an event bus / application event publisher in a framework like Spring) for genuine one-to-many notification needs, rather than having a class hold direct references to every interested collaborator and call each one explicitly — this keeps the publisher decoupled from the specific set of subscribers.
- **DO:** Favor small, focused value objects (a `Money` type wrapping amount and currency, an `EmailAddress` type wrapping and validating a string) over passing primitive types like `BigDecimal` or raw `String` around for domain concepts that have their own validation rules and behavior — this is sometimes called avoiding "primitive obsession."

## Exception Handling

- **DO:** Reserve checked exceptions for recoverable conditions a caller is expected to catch and handle, and use unchecked (`RuntimeException`) subclasses for programming errors or conditions the caller can't reasonably recover from. Checked exceptions on things like "I/O failed while reading a required config file" make sense; checked exceptions on "invalid internal state" just force boilerplate `catch` or `throws` clauses everywhere with no recovery path.
- **DON'T:** Swallow exceptions with an empty catch block. Silently discarding an exception hides failures until they surface as a confusing, unrelated symptom far from the real cause.
  ```java
  // DON'T
  try {
      process(order);
  } catch (Exception e) {
      // ignore
  }

  // DO
  try {
      process(order);
  } catch (OrderProcessingException e) {
      log.error("Failed to process order {}", order.getId(), e);
      throw e; // or a translated exception, or a defined fallback
  }
  ```
- **DON'T:** Catch `Exception` or `Throwable` broadly just to keep code compiling or to suppress an IDE warning. A blanket catch swallows exceptions the code has no business handling — including `OutOfMemoryError`-style errors and unrelated bugs — and masks the specific failure the caller actually needs to respond to.
- **DO:** Catch the most specific exception type that the calling code can meaningfully act on, and let everything else propagate. Catching narrowly keeps error-handling logic honest about what it actually recovers from.
- **DO:** Preserve the original exception as the `cause` when wrapping or translating it (`throw new ServiceException("failed", e)`), never dropping it. Losing the cause destroys the stack trace that would otherwise pinpoint the real failure.
- **DON'T:** Use exceptions for ordinary control flow (e.g. throwing to break out of a loop, or using exceptions to signal "not found" where an `Optional` or a null-safe return would do). Exceptions are comparatively expensive (stack trace capture) and make normal-path logic harder to follow than a plain conditional.
- **DO:** Design custom exception hierarchies that carry meaningful context (order ID, correlation ID, the invalid value) rather than just a string message. Structured exception data lets logging, monitoring, and upstream handlers act on specifics instead of parsing message text.
- **DO:** Use try-with-resources for every `AutoCloseable`/`Closeable` resource (streams, connections, locks implementing `AutoCloseable`) instead of manual `finally` blocks. Try-with-resources guarantees closure even when the try block throws, and it correctly chains a close-time exception as a suppressed exception rather than masking the original.
  ```java
  // DON'T
  FileInputStream in = new FileInputStream(path);
  try {
      readData(in);
  } finally {
      in.close(); // this close() can itself throw and mask a prior exception
  }

  // DO
  try (FileInputStream in = new FileInputStream(path)) {
      readData(in);
  }
  ```
- **DON'T:** Throw generic `Exception` or `RuntimeException` directly from application code. A caller can't selectively catch a bare `Exception`; define or reuse a specific exception type so callers can distinguish failure modes.
- **DO:** Fail fast with precondition checks (`Objects.requireNonNull`, explicit validation throwing `IllegalArgumentException`/`IllegalStateException`) at the boundary of a method rather than letting invalid state propagate and fail confusingly several calls later.
- **DON'T:** Log an exception and then rethrow it at every layer of the call stack. Logging the same exception repeatedly at each level floods logs with duplicate stack traces; log it once — typically at the boundary where it's handled or turned into a user-facing response — and let intermediate layers simply propagate it.
- **DO:** Use multi-catch (`catch (IOException | SQLException e)`) when multiple exception types warrant identical handling, rather than duplicating the catch block per type.
- **DO:** Convert checked exceptions from third-party APIs into an application-specific unchecked exception at the boundary where they're not recoverable, so the checked-exception "tax" doesn't propagate deep into business logic that doesn't need to know about `SQLException` or `IOException` directly.

- **DO:** Document the exceptions a public method can throw — via `@throws` Javadoc for unchecked exceptions callers should know about, and via the `throws` clause itself for checked ones — so callers can see the failure surface without reading the implementation.
- **DON'T:** Declare a checked exception on a method meant to be used as a lambda target for a standard functional interface (`Function<T,R>`, `Supplier<T>`). Standard functional interfaces don't declare checked exceptions, so this forces every caller to wrap the call in a try/catch inline or use an awkward sneaky-throw workaround; either redesign the interface or wrap the checked exception at the boundary.
- **DO:** Use `Optional` as a return type instead of throwing a "not found"-style exception when absence is a normal, frequently expected outcome (a cache miss, an optional configuration lookup) — reserve thrown exceptions for conditions that are genuinely exceptional given the method's contract.
- **DON'T:** Catch an exception and rethrow it as a completely unrelated exception type that discards its original semantic meaning (catching a `SQLException` and rethrowing as `IllegalArgumentException`, for instance). Translate to a type that still accurately describes what actually failed.
- **DO:** Guard constructors and public setters with `Objects.requireNonNull(param, "message")` for required parameters, producing an immediate, precisely-located `NullPointerException` instead of a confusing failure several unrelated calls later.
  ```java
  public Order(Customer customer, List<Item> items) {
      this.customer = Objects.requireNonNull(customer, "customer must not be null");
      this.items = List.copyOf(Objects.requireNonNull(items, "items must not be null"));
  }
  ```
- **DO:** Consider a sealed `Result<T, E>`-style return type for expected, recoverable failure paths in widely-called internal APIs, when checked-exception ceremony would otherwise force `throws` declarations through many layers that don't actually handle the failure themselves.

- **DO:** Use a dedicated resilience library (Resilience4j, Spring Retry) for retry, circuit-breaker, and timeout policies around unreliable external calls, rather than hand-rolling a `while` loop with a manual counter and a `catch` block around it. Purpose-built resilience libraries correctly handle backoff, jitter, and circuit state that a quick hand-rolled loop typically gets wrong.
  ```java
  // DON'T — ad hoc retry with no backoff
  for (int i = 0; i < 3; i++) {
      try { return client.call(); } catch (IOException e) { /* retry */ }
  }
  throw new RuntimeException("failed after retries");

  // DO
  @Retry(name = "paymentGateway", fallbackMethod = "fallback")
  PaymentResult call() { return client.call(); }
  ```
- **DON'T:** Let a single top-level `catch (Exception e)` at the very edge of the application (e.g. a servlet filter or a `@ControllerAdvice`) be the *only* place exceptions are ever handled. A single global catch-all is appropriate as a last-resort safety net for turning unexpected failures into a generic error response, but specific, expected failure modes (validation errors, not-found, conflict) should still be handled with their own specific status codes and messages closer to where they occur.
- **DO:** Use `assert` statements only for internal invariant checks during development/testing (they're disabled by default at runtime unless `-ea` is passed), never for validating externally-supplied input where the check must always run in production.
- **DO:** Prefer a custom unchecked `ApplicationException` base class (with subtypes per failure category) for application-specific errors, so a single `catch (ApplicationException e)` at a boundary can uniformly log and translate any of them, while more specific catches upstream can still target individual subtypes.
- **DON'T:** Rely on exception messages (string matching/parsing) to distinguish failure cases in calling code. String-matching a `getMessage()` value is fragile — it breaks the moment the message wording changes — use distinct exception types or an explicit error code/field instead.
- **DO:** Log the full exception object (not just `e.getMessage()`) through the logging framework's exception-aware overload (`log.error("message", e)`), so the stack trace is preserved in the log output rather than discarded.

## Logging & Observability

- **DO:** Use a logging facade (SLF4J) backed by a configured logging implementation (Logback, Log4j2) instead of `System.out.println`/`System.err.println` for anything beyond a disposable script. `println` has no levels, no structured output, no per-package configuration, and can't be selectively enabled or routed to different destinations.
- **DON'T:** Log at `ERROR` for expected, handled outcomes (a routine "not found" result, a validation failure returned to the caller). Reserve `ERROR` for genuinely unexpected failures that warrant investigation; use `WARN` for recoverable or unusual-but-handled situations, `INFO` for significant business events, and `DEBUG`/`TRACE` for detailed diagnostic flow.
- **DO:** Use parameterized logging calls (`log.info("Order {} processed for {}", orderId, customerId)`) instead of building the message via string concatenation. Parameterized calls skip the cost of formatting the string entirely when the log level is disabled, and they integrate correctly with structured logging backends.
  ```java
  // DON'T — always builds the string, even if INFO is disabled
  log.info("Order " + orderId + " processed for " + customerId);

  // DO
  log.info("Order {} processed for {}", orderId, customerId);
  ```
- **DON'T:** Log sensitive data — passwords, full payment card numbers, auth tokens, or PII beyond what's operationally necessary. Logs are commonly retained long-term, shipped to third-party aggregation platforms, and accessible to a wider set of people than the originating application's own access controls.
- **DO:** Attach a correlation or trace ID to every log line within a request's scope (via MDC — Mapped Diagnostic Context — or a structured logging field), so related log entries across multiple services can be correlated together when investigating a single request during an incident.
- **DO:** Configure log levels per environment externally (verbose in development, `INFO`/`WARN` in production) rather than hardcoding a level in source, so verbosity can be adjusted without a redeploy and production isn't flooded with debug-level noise by default.
- **DON'T:** Log every method's entry and exit as a blanket habit. Indiscriminate logging drowns out the signal that actually matters during an incident investigation; log meaningful state transitions and decisions, not mechanical control flow.
- **DO:** Emit structured (JSON) log output in production environments that feed a log aggregation platform, so fields are queryable directly rather than requiring regex extraction from free-text messages.

## Collections API Usage

- **DO:** Choose the collection implementation based on actual access patterns — `ArrayList` for index-based random access and iteration, `LinkedList` only when frequent insertion/removal at both ends dominates (which is rare — `ArrayDeque` usually wins even there), `HashMap`/`HashSet` for unordered fast lookup, `LinkedHashMap`/`LinkedHashSet` when insertion order must be preserved, and `TreeMap`/`TreeSet` when sorted iteration is required.
- **DON'T:** Default to `LinkedList` out of habit. `ArrayDeque` outperforms `LinkedList` for stack/queue use cases with less memory overhead per element (no per-node object), and `ArrayList` outperforms it for nearly everything else; `LinkedList`'s O(n) indexed access is a common accidental performance bug.
- **DO:** Use `List.of()`, `Map.of()`, `Set.of()` (Java 9+) or `Collections.unmodifiableX` for collections that should never change after creation. Immutable collections prevent an entire class of bugs where a shared collection is mutated by code that shouldn't have write access.
  ```java
  // DON'T
  List<String> roles = new ArrayList<>();
  roles.add("ADMIN"); roles.add("USER");
  return roles; // caller can mutate this freely

  // DO
  return List.of("ADMIN", "USER");
  ```
- **DON'T:** Call `.add()`/`.put()` on a collection returned from `List.of()`/`Map.of()` or from `Collections.unmodifiableList(...)`. These throw `UnsupportedOperationException` at runtime; the compiler won't catch the mistake, so know which of your collections are immutable before mutating them.
- **DO:** Use `Map.computeIfAbsent`, `merge`, `getOrDefault`, and `putIfAbsent` for common read-modify-write map patterns instead of manual `containsKey`/`get`/`put` sequences. These are both more concise and avoid the double lookup (and potential race in concurrent contexts) of check-then-act.
  ```java
  // DON'T
  List<String> list = map.get(key);
  if (list == null) {
      list = new ArrayList<>();
      map.put(key, list);
  }
  list.add(value);

  // DO
  map.computeIfAbsent(key, k -> new ArrayList<>()).add(value);
  ```
- **DON'T:** Mutate a collection while iterating over it with a for-each loop (`ConcurrentModificationException` waiting to happen). Use an `Iterator`'s own `remove()`, `removeIf()`, or collect changes into a separate collection and apply them after iteration.
  ```java
  // DON'T
  for (String s : names) {
      if (s.isBlank()) names.remove(s); // throws ConcurrentModificationException
  }

  // DO
  names.removeIf(String::isBlank);
  ```
- **DO:** Prefer `removeIf`, `replaceAll`, and `forEach` default methods on `Collection`/`List` for simple bulk operations — they're clearer than an equivalent manual loop and less error-prone around iterator invalidation.
- **DO:** Size `ArrayList`/`HashMap` with an initial capacity when the eventual size is known or estimable (`new ArrayList<>(expectedSize)`), avoiding repeated internal resizing/copying for large collections built in a hot path.
- **DON'T:** Use `Vector`, `Hashtable`, or `Stack` in new code. These are legacy synchronized collections from Java 1.0/1.1 with poor performance characteristics for single-threaded use; prefer `ArrayList`/`HashMap`/`ArrayDeque`, and if synchronization is actually needed, use `java.util.concurrent` collections instead.
- **DO:** Use `Optional<T>` as a return type to signal "may legitimately have no value" for methods that would otherwise return null, especially at API boundaries — but don't use it for fields, method parameters, or collection elements, where it adds overhead without a corresponding benefit.
- **DON'T:** Call `Optional.get()` without first checking `isPresent()` or without preferring `orElse`/`orElseThrow`/`map`. A bare `.get()` on an empty `Optional` throws `NoSuchElementException`, defeating the entire purpose of using `Optional` in the first place.
- **DO:** Use `Collectors.toUnmodifiableList()`/`toUnmodifiableMap()` (or `.toList()` in Java 16+, which is unmodifiable) when a stream's result should not be mutated downstream.
- **DO:** Compare collections for equality with `.equals()` (which `List`, `Set`, and `Map` implement structurally) rather than manually iterating — but remember `List.equals` cares about order while `Set.equals` and `Map.equals` do not, so pick the collection type that matches the equality semantics you actually want.

- **DO:** Return an empty collection (`List.of()`, `Collections.emptyList()`) instead of `null` from a method whose declared return type is a collection type. This removes the need for every caller to null-check before iterating and matches the universally expected contract for collection-returning methods.
- **DON'T:** Return `null` where an empty `List`/`Set`/`Map` is the honest answer — "there are zero results" and "the result is absent/unknown" are different states, and conflating them into `null` forces defensive null checks throughout the calling code.
- **DO:** Accept the narrowest useful collection-family type as a parameter — `Iterable<T>` when a method only needs to iterate once, `Collection<T>` when it needs `size()`, `List<T>` only when order/indexing genuinely matters — so callers aren't forced to construct a more specific type than the method actually needs.
- **DO:** Use `EnumSet`/`EnumMap` for collections keyed by enum constants. They're backed by a bit vector/array internally and are dramatically faster and more memory-efficient than a general-purpose `HashSet`/`HashMap` for that specific, common case.
  ```java
  EnumSet<DayOfWeek> weekend = EnumSet.of(DayOfWeek.SATURDAY, DayOfWeek.SUNDAY);
  ```
- **DO:** Prefer `ArrayDeque` over `java.util.Stack` for LIFO stack behavior — `Stack` extends the legacy synchronized `Vector` and inherits its performance overhead and awkward mixed API (it also exposes `Vector`'s indexed methods, which don't belong on a stack abstraction).
- **DON'T:** Assume `HashMap`/`HashSet` iteration order is meaningful or stable across JVM versions or even across separate runs. If consistent iteration order matters, use `LinkedHashMap`/`LinkedHashSet` explicitly rather than relying on incidental `HashMap` behavior that happens to look ordered today.

- **DO:** Understand `HashMap`'s amortized O(1) get/put depends on a well-distributed `hashCode()` — a poorly-implemented `hashCode()` (e.g. one that always returns a constant) silently degrades every operation on that key type to O(n), a subtle performance bug that's invisible in small test data and only shows up at scale.
- **DO:** Use `Collections.unmodifiableX` wrapping (or the `List.of`/`Map.of` family) at API boundaries even for internally-mutable collections, so external callers get a read-only view while internal code retains a mutable reference for legitimate updates — this pattern separates "who is allowed to mutate this" from "who merely needs to read it."
  ```java
  private final List<Item> items = new ArrayList<>();
  public List<Item> getItems() { return Collections.unmodifiableList(items); }
  public void addItem(Item item) { items.add(item); } // internal mutation still allowed
  ```
- **DON'T:** Use a `List<Object[]>` or a `List<Map<String, Object>>` as a substitute for a properly typed domain class just to avoid declaring a small `record`. Untyped, loosely-structured collections push type-checking work onto every caller at runtime instead of the compiler catching mistakes at compile time.
- **DO:** Use `Collections.binarySearch` only on collections that are actually sorted according to the same ordering the search assumes — calling it on an unsorted list produces an undefined, essentially garbage result with no warning.
- **DO:** Prefer `Set.of(...)`/`new HashSet<>(List.of(...))` for small fixed membership checks (`if (VALID_STATUSES.contains(status))`) over a chain of `||` equality comparisons — it's both more readable and scales better if the set of valid values grows.
- **DON'T:** Assume `ArrayList.remove(int)` and `ArrayList.remove(Object)` behave the same way for a `List<Integer>` — `list.remove(1)` removes the element *at index* 1, while `list.remove(Integer.valueOf(1))` removes the *value* 1. This autoboxing ambiguity is a well-known trap; be explicit about which behavior is intended.

## Generics & Type Safety

- **DO:** Apply the PECS principle (Producer Extends, Consumer Super) when designing method parameters that accept generic collections — use `? extends T` for a source you only read from, and `? super T` for a destination you only write to — so APIs stay maximally flexible for callers without sacrificing compile-time type safety.
  ```java
  static <T> void copy(List<? extends T> source, List<? super T> destination) {
      for (T item : source) destination.add(item);
  }
  ```
- **DON'T:** Use raw types (`List` instead of `List<T>`) or unchecked casts as a workaround for a generics problem instead of properly bounding the type parameters. Raw types disable compile-time checking entirely for that usage, silently reintroducing the exact class of `ClassCastException` bugs generics exist to prevent.
- **DO:** Design public generic methods so their type parameters can be inferred from the arguments passed, rather than forcing every caller to write explicit type witnesses (`Collections.<String>emptyList()`) because the type only appears in the return position with nothing to infer it from.
- **DON'T:** Apply `@SuppressWarnings("unchecked")` broadly across an entire method or class when only a single line actually needs it. Scope the suppression to the smallest possible block so future, unrelated unchecked-generics issues introduced later in the same method aren't silently hidden by an overly broad existing suppression.
  ```java
  // DON'T — suppresses warnings for the whole method
  @SuppressWarnings("unchecked")
  void process(Object input) { ... many lines ... }

  // DO — suppressed only where the unchecked cast actually happens
  void process(Object input) {
      @SuppressWarnings("unchecked")
      List<String> names = (List<String>) input;
      ...
  }
  ```
- **DO:** Use `@SafeVarargs` on a varargs generic method only when it's actually verified safe (the varargs array isn't stored or exposed in a way that could cause heap pollution) — never apply it reflexively just to silence the compiler's heap-pollution warning.
- **DO:** Prefer `List<T>` over `T[]` in generic APIs where a collection type will do. Java can't create a generic array directly due to type erasure, and the resulting interactions between arrays' runtime type checks and generics' compile-time-only checks are a well-documented source of surprising `ArrayStoreException`s.
- **DON'T:** Add a generic type parameter to an internal, non-reused helper class purely because it seems more "proper." Generic complexity should earn its keep through genuine reuse across multiple concrete types — a single-use generic parameter is unnecessary ceremony.

## Streams API Idioms vs. Overuse

- **DO:** Use the Streams API for genuinely declarative transformations — filter, map, reduce, collect — where it reads more clearly than the imperative loop it replaces. A pipeline that filters active users, extracts emails, and collects them into a list is a textbook case.
  ```java
  List<String> activeEmails = users.stream()
      .filter(User::isActive)
      .map(User::getEmail)
      .collect(Collectors.toList());
  ```
- **DON'T:** Force a stream pipeline onto logic that's naturally imperative or has side effects at every step (building up several different accumulator variables, early-exiting with complex conditions, or performing I/O per element). A `for` loop is often clearer, easier to debug, and easier to step through than a five-stage stream chained across multiple lines for such cases.
- **DON'T:** Nest streams three or four levels deep, or chain fifteen intermediate operations into one unreadable pipeline, purely to prove it can be done in "one expression." Break long pipelines into named intermediate variables or extracted methods so each stage's intent is legible.
  ```java
  // DON'T — unreadable single expression
  return orders.stream().filter(o -> o.getStatus() == Status.PAID)
      .flatMap(o -> o.getItems().stream()).collect(Collectors.groupingBy(Item::getCategory,
      Collectors.summingDouble(i -> i.getPrice() * i.getQuantity()))).entrySet().stream()
      .sorted(Map.Entry.comparingByValue(Comparator.reverseOrder())).limit(5)
      .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue, (a,b)->a, LinkedHashMap::new));

  // DO — named stages
  List<Item> paidItems = paidOrderItems(orders);
  Map<String, Double> revenueByCategory = revenueByCategory(paidItems);
  Map<String, Double> top5Categories = topEntries(revenueByCategory, 5);
  ```
- **DO:** Use method references (`User::getEmail`, `String::isBlank`) over equivalent lambdas when they're a direct 1:1 call. `.map(User::getEmail)` is both shorter and communicates intent more directly than `.map(u -> u.getEmail())`.
- **DON'T:** Use `.forEach()` with a mutating side effect on an external variable as a substitute for a proper reduction or a plain loop. Streams are not designed around external mutation, and a lambda that mutates a captured variable (via an array or atomic wrapper hack) is a strong signal the code should be a loop or a `.reduce()`/`.collect()` instead.
  ```java
  // DON'T
  int[] total = {0};
  items.forEach(i -> total[0] += i.getPrice());

  // DO
  int total = items.stream().mapToInt(Item::getPrice).sum();
  ```
- **DO:** Use primitive stream specializations (`IntStream`, `LongStream`, `DoubleStream`) for numeric aggregation instead of boxing into `Stream<Integer>` and unboxing repeatedly. This avoids unnecessary autoboxing overhead and gives access to `sum()`, `average()`, `summaryStatistics()` directly.
- **DON'T:** Reuse a `Stream` instance after a terminal operation has been called on it. Streams are single-use; calling another operation on an already-consumed stream throws `IllegalStateException` at runtime, so build a fresh stream (or a `Supplier<Stream<T>>`) if the same data needs traversing twice.
- **DO:** Prefer `Collectors.groupingBy`, `partitioningBy`, and `joining` over hand-written accumulation loops for those exact shapes of aggregation — they're both standard and well-optimized.
- **DON'T:** Use parallel streams (`.parallelStream()`) by default "for performance." Parallel streams only pay off for large datasets with CPU-bound, side-effect-free work and correctly sized worksteal pools; for small collections or I/O-bound operations they add thread-pool overhead and can be slower, and side effects in the pipeline turn into hard-to-reproduce race conditions.
- **DO:** Benchmark before reaching for `.parallel()`, and never use it over a shared mutable collector target or with an operation that has ordering dependencies.
- **DO:** Prefer `Stream.toList()` (Java 16+) over `.collect(Collectors.toList())` for the common case where an unmodifiable list is acceptable — it's shorter and clearer about the immutability guarantee.

- **DO:** Use `Collectors.toMap(keyFn, valueFn, mergeFn)` with an explicit merge function whenever key collisions between elements are possible. The two-argument overload throws `IllegalStateException` at runtime on the first duplicate key, which is easy to miss until it happens in production against real data.
  ```java
  // DON'T — throws on the first duplicate SKU
  Map<String, Double> prices = items.stream()
      .collect(Collectors.toMap(Item::getSku, Item::getPrice));

  // DO — explicit conflict resolution
  Map<String, Double> prices = items.stream()
      .collect(Collectors.toMap(Item::getSku, Item::getPrice, (a, b) -> b));
  ```
- **DON'T:** Throw a checked exception directly from inside a stream lambda without a wrapper — most stream functional interfaces don't declare checked exceptions, and the resulting compile error tempts people into ugly sneaky-throw hacks. Extract the throwing call into a named method that catches and translates the checked exception, or wraps it into an unchecked one, before it's used in a lambda.
- **DO:** Use `Stream.generate`/`Stream.iterate` combined with `.limit()` for lazily-produced bounded sequences, avoiding the need to eagerly materialize a large intermediate list before the stream pipeline even starts.
- **DO:** Use `Collectors.teeing` (Java 12+) — or two dedicated reduction/collector calls sharing one upstream stream via `Stream.of(stream1, stream2)`-style patterns where teeing isn't available — when a single pass needs to compute two different aggregates, instead of iterating (or re-fetching) the same source data twice.
- **DON'T:** Ignore that streams built over ordered sources (like `List`) preserve encounter order by default even through `.map()`/`.filter()`, but that ordering can add overhead in a parallel stream; call `.unordered()` deliberately when order truly doesn't matter and the pipeline runs in parallel.

- **DO:** Prefer `Collectors.partitioningBy` over `Collectors.groupingBy` when the grouping key is genuinely a boolean condition — `partitioningBy` always returns both `true` and `false` keys (even if one bucket is empty), which is often exactly what downstream code expects and avoids a `NullPointerException`/missing-key surprise that `groupingBy` with a boolean key wouldn't guarantee.
- **DON'T:** Use `.peek()` for anything beyond debugging a pipeline during development. `.peek()` is designed for non-interfering diagnostic actions, its execution is not guaranteed by the JDK spec for pipelines whose terminal operation doesn't need to visit every element (e.g. `.findFirst()` can short-circuit before `.peek()` runs on later elements), and relying on it for actual business-logic side effects produces surprising, spec-undefined behavior.
- **DO:** Use `IntStream.range`/`IntStream.rangeClosed` instead of a `for` loop when you specifically want to produce or transform a range of integers through the Streams API (e.g. `IntStream.range(0, n).mapToObj(...)`), but recognize that for simple iteration with an index a plain `for` loop is often clearer and doesn't require boxing back out of the `IntStream`.
- **DO:** Terminate a stream pipeline with the most specific terminal operation available — `anyMatch`/`allMatch`/`noneMatch` for boolean checks, `count()` for a size, `findFirst()`/`findAny()` for a single element — rather than materializing a full list purely to call `.size()` or `.isEmpty()` on it afterward, since the specific terminal operations can short-circuit and avoid processing the whole source.
  ```java
  // DON'T
  boolean hasOverdue = invoices.stream().filter(Invoice::isOverdue).collect(Collectors.toList()).size() > 0;

  // DO
  boolean hasOverdue = invoices.stream().anyMatch(Invoice::isOverdue);
  ```

## Dependency Injection (Spring Conventions)

- **DO:** Use constructor injection as the default for required dependencies, not field injection with `@Autowired` on the field itself. Constructor injection makes dependencies explicit, allows fields to be `final`, fails fast at startup if a bean is missing, and makes the class trivially testable without a Spring context (just call `new Service(mockDep)`).
  ```java
  // DON'T
  @Service
  public class OrderService {
      @Autowired
      private PaymentGateway paymentGateway;
  }

  // DO
  @Service
  public class OrderService {
      private final PaymentGateway paymentGateway;
      public OrderService(PaymentGateway paymentGateway) {
          this.paymentGateway = paymentGateway;
      }
  }
  ```
- **DON'T:** Use field injection (`@Autowired` directly on a field) as a habit. It hides required dependencies from the constructor signature, makes the class impossible to instantiate without reflection or a full Spring context, and permits circular dependencies that constructor injection would surface immediately as a startup failure.
- **DO:** Keep beans stateless (or hold only injected, effectively-immutable collaborators) since the default Spring bean scope is singleton — a singleton bean with mutable instance state shared across every request is a concurrency bug waiting to happen.
- **DON'T:** Reach for `@Autowired` on setters "for optional dependencies" as the default pattern. Prefer `Optional<T>` constructor parameters, `@Autowired(required = false)` sparingly, or — better — separate the optional collaborator into its own explicitly-configured bean so the class's required contract stays visible in the constructor.
- **DO:** Scope component classes narrowly with stereotype annotations that match their role — `@Service` for business logic, `@Repository` for persistence (which also enables Spring's exception translation for that layer), `@Controller`/`@RestController` for web endpoints, and plain `@Component` only when none of the more specific stereotypes fit.
- **DON'T:** Inject the `ApplicationContext` directly and pull beans out of it manually (`context.getBean(...)`) as a substitute for proper injection. This is the Service Locator anti-pattern re-created inside a DI framework — it hides the class's real dependencies and defeats the compile-time and startup-time safety DI is meant to provide.
- **DO:** Favor Java/Kotlin `@Configuration` classes with `@Bean` methods for third-party types you don't own (an `ObjectMapper`, an `HttpClient`) rather than trying to retrofit component annotations onto classes you can't modify.
- **DO:** Use `@ConfigurationProperties` with a typed, validated properties class for structured configuration, rather than sprinkling `@Value("${some.key}")` across many unrelated classes. A typed properties class is one place to see the full configuration surface, and it supports `@Validated` constraints (`@NotNull`, `@Min`) that fail fast at startup on misconfiguration.
- **DON'T:** Use `@Autowired` with `List<SomeInterface>` or `Map<String, SomeInterface>` without understanding that Spring will inject *every* matching bean — a legitimate and useful pattern for strategy dispatch, but a common source of "why did an extra implementation get picked up" bugs when a new bean is added without realizing it will land in that collection.
- **DO:** Prefer profile-specific `@Configuration`/`@Bean` definitions (`@Profile("test")`) or Spring Boot's externalized `application-{profile}.yml` over `if` branches inside bean methods that check the active environment manually.
- **DON'T:** Scatter business logic inside `@Configuration` classes or `@Bean` factory methods. Configuration classes should wire dependencies together; actual behavior belongs in the beans themselves, kept out of the wiring layer so it stays trivial to reason about at startup.
- **DO:** Use `@Transactional` at the service layer (not the repository or controller layer) on methods, being explicit about propagation and read-only status (`@Transactional(readOnly = true)` for query methods) — and be aware that self-invocation within the same class bypasses the Spring AOP proxy, so a `@Transactional` method called from another method in the same class silently runs without a transaction.

- **DO:** Use `@Qualifier` or distinctly-named `@Bean` methods when multiple implementations of the same interface exist in the application context, rather than relying on Spring's implicit name-matching fallback to disambiguate. Explicit qualification is discoverable in code and doesn't silently break when an unrelated field or parameter gets renamed.
- **DON'T:** Set `@ComponentScan` (or Spring Boot's default scanning from the application's root package) so broadly that it unintentionally picks up test-only fixtures, example/demo beans, or unrelated modules on the classpath. Scope scanning to the packages that should actually be scanned.
- **DO:** Prefer an existing Spring Boot starter's auto-configuration over manually wiring the same beans by hand, unless there's a specific, documented reason to override the default — starters are maintained, tested, and consistent across the ecosystem in a way ad hoc wiring usually isn't.
- **DON'T:** Annotate a `private` method with `@Transactional` (or any other proxy-based Spring AOP annotation like `@Cacheable`, `@Async`). Spring's default proxy-based AOP can only intercept calls that go through the public proxy, so annotations on private methods — and on public methods called from within the same class (self-invocation) — are silently ignored with no compile-time or startup warning.
  ```java
  @Service
  public class OrderService {
      public void placeOrder(Order order) {
          save(order); // self-invocation — @Transactional below is silently skipped
      }

      @Transactional
      private void save(Order order) { repository.save(order); }
  }
  ```
- **DO:** Use Lombok's `@RequiredArgsConstructor` (or, in a records-friendly codebase, a compact constructor pattern) to remove constructor boilerplate for classes with several `final` injected dependencies, while keeping the underlying constructor-injection pattern itself unchanged.

- **DO:** Use `@Value` injection sparingly and only for genuinely simple, standalone configuration values — for anything with more than two or three related settings, group them into a `@ConfigurationProperties` class instead so the configuration's shape and validation live together in one type.
- **DON'T:** Wire cross-cutting concerns (logging, security checks, metrics) by hand inside every service method when Spring AOP (`@Around` advice) or a servlet/gateway filter can apply the same behavior declaratively across many methods at once, keeping business logic free of repeated boilerplate.
- **DO:** Keep `@RestController` classes thin — request/response mapping, input validation delegation, and HTTP-status decisions — with actual business logic living in `@Service` classes underneath. A controller that directly executes business rules can't be reused or tested without spinning up the web layer.
- **DON'T:** Rely on Spring's field-based `@Value("${property}")` default-value syntax (`@Value("${timeout:5000}")`) scattered through many classes as the source of truth for defaults — centralize defaults in the properties/YAML file itself (or the properties class) so there's one place to see every configurable default.
- **DO:** Use constructor injection to build small, explicit "assembled" objects for testing (`new OrderService(fakeRepo, fakeGateway)`) as the default in unit tests, reserving a full Spring test context (`@SpringBootTest`) for genuine integration tests where the wiring itself is what's under test.

## Build Tools (Maven & Gradle)

- **DO:** Pin exact dependency versions (or use a managed BOM / version catalog) rather than dynamic version ranges like `1.+` or `LATEST`. Dynamic versions make builds non-reproducible — the same commit can pull a different, potentially breaking, dependency version tomorrow.
- **DON'T:** Declare a dependency version directly in every module of a multi-module project. Centralize versions in a Maven `<dependencyManagement>` block / BOM import, or a Gradle version catalog (`libs.versions.toml`), so upgrading a library is a one-line change instead of a project-wide search-and-replace.
  ```toml
  # gradle/libs.versions.toml
  [versions]
  jackson = "2.17.1"
  [libraries]
  jackson-databind = { module = "com.fasterxml.jackson.core:jackson-databind", version.ref = "jackson" }
  ```
- **DO:** Separate `compileOnly`/`provided` scope dependencies (annotation processors like Lombok, servlet APIs supplied by a container) from `implementation`/`compile` dependencies that actually ship in the runtime artifact. Conflating them bloats the deployed artifact and can pull in classes that conflict with what the runtime container provides.
- **DON'T:** Use Gradle's `implementation` when a dependency's types genuinely leak into your module's public API (a return type from a public method comes from that library) — use `api` in that case so downstream consumers correctly get it on their compile classpath instead of hitting mysterious "cannot find symbol" errors.
- **DO:** Keep build scripts declarative and push nontrivial logic into a custom plugin, a `buildSrc`/included build, or a script that's actually tested — rather than accumulating ad hoc `doLast {}` blocks and shell-outs scattered across a growing `build.gradle`.
- **DON'T:** Check dynamic, non-reproducible SNAPSHOT dependencies into a release build. A `1.0.0-SNAPSHOT` dependency can change contents without a version bump, meaning "the same" release build can silently include different code between builds; pin exact released versions for anything shipped.
- **DO:** Run `mvn dependency:tree` / `gradle dependencies` regularly (and ideally enforce it in CI) to catch version conflicts and unexpected transitive dependencies before they cause runtime `NoSuchMethodError`s from a silently-shadowed older jar.
- **DO:** Use the Maven Wrapper (`mvnw`) or Gradle Wrapper (`gradlew`) checked into the repository rather than relying on a locally installed build tool version. The wrapper guarantees every contributor and CI runner builds with the exact same tool version, eliminating "works on my machine" build failures.
- **DON'T:** Disable or bypass the reproducible-build/lock-file mechanisms (`gradle.lockfile`, Maven's `dependency:tree` reproducibility, enforcer plugin rules) just to make a build pass faster locally. These exist specifically to prevent the exact classpath from silently drifting between environments.
- **DO:** Fail the build on compiler warnings that matter (`-Werror` for a curated warning set, or Error Prone/NullAway checks) rather than letting warnings accumulate unseen in build logs no one reads.

- **DO:** Structure multi-module builds along real architectural boundaries (`api`, `core`, `infra`) so the module dependency graph reflects and enforces the intended direction of dependency (e.g. `core` never depends on `infra`), rather than splitting modules arbitrarily by size.
- **DON'T:** Configure the build to silently ignore test failures (`ignoreFailures = true` in Gradle's test task, or a Maven Surefire configuration that swallows failures) as a way to keep CI green. A build that reports success despite failing tests defeats the entire purpose of running them in CI.
- **DO:** Cache dependency resolution and build outputs in CI (Gradle's build cache, Maven's local repository caching between runs) to keep feedback loops fast without sacrificing the reproducibility that comes from pinned versions.
- **DO:** Run a dependency vulnerability scanner (OWASP Dependency-Check, Snyk, or the equivalent your org standardizes on) as part of the build pipeline, failing on known-critical CVEs in resolved dependencies rather than relying on someone remembering to check manually.

- **DO:** Set explicit, pinned Java toolchain versions in the build (Gradle's `java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }`, Maven's `maven.compiler.release`) rather than relying on whatever JDK happens to be first on a developer's or CI runner's `PATH`. This guarantees the same bytecode target and available language features regardless of the local machine's default JDK.
- **DON'T:** Publish a library artifact whose `pom.xml`/`build.gradle` declares dependencies with `compile`/`implementation` scope for things that are actually only needed to build the library itself (test frameworks, code generators) — consumers then unnecessarily inherit those dependencies transitively, bloating their own classpath.
- **DO:** Use Gradle's version catalogs or Maven's BOM imports consistently across every module in a multi-module project so a single version bump (e.g. upgrading Jackson after a CVE) is a one-line, propagates-everywhere change rather than a multi-file hunt.
- **DO:** Keep the build's own logic (custom tasks, plugins) under the same code-quality bar as production code — tested, reviewed, and not accumulating undocumented one-off hacks — since a broken or unmaintainable build script blocks every developer on the team simultaneously.

## Testing (JUnit 5 & Mockito)

- **DO:** Structure tests with the Arrange-Act-Assert (or Given-When-Then) pattern and name test methods to describe the scenario and expected outcome (`shouldThrowWhenBalanceIsNegative`, or JUnit 5's `@DisplayName` for a readable sentence). A test's name and structure should tell you what broke without opening the test body.
  ```java
  @Test
  void shouldRejectWithdrawalWhenBalanceIsInsufficient() {
      // Arrange
      Account account = new Account(BigDecimal.TEN);
      // Act & Assert
      assertThrows(InsufficientFundsException.class,
          () -> account.withdraw(BigDecimal.valueOf(20)));
  }
  ```
- **DO:** Use JUnit 5's `@ParameterizedTest` with `@ValueSource`/`@CsvSource`/`@MethodSource` to cover multiple input variations with one test method, rather than copy-pasting near-identical test methods for each case.
  ```java
  @ParameterizedTest
  @CsvSource({"0,false", "-1,false", "1,true"})
  void isPositive_returnsExpected(int input, boolean expected) {
      assertEquals(expected, MathUtils.isPositive(input));
  }
  ```
- **DON'T:** Mock types you don't own the behavior contract for — value objects, DTOs, or simple data holders. Mocking a plain data class produces a fragile test that verifies interactions with a stub rather than actual behavior; just construct the real object.
- **DO:** Use Mockito's `@Mock`/`@InjectMocks` with `@ExtendWith(MockitoExtension.class)` for unit tests that isolate a class from its collaborators, and reserve mocking for genuine boundaries (external services, repositories, clocks) rather than mocking everything a class touches.
- **DON'T:** Over-specify mock interactions with `verify()` calls for every single method invocation, including incidental ones. Over-verification couples the test tightly to implementation details, so an unrelated refactor breaks tests that never should have cared how the internals were wired.
- **DO:** Prefer state-based assertions (checking the resulting object/return value) over interaction-based verification (`verify(mock).method()`) whenever the outcome can be observed directly — interaction testing is appropriate specifically for side-effecting calls (e.g., "the email service was told to send exactly one email") where there's no other observable state to assert on.
- **DON'T:** Let tests share mutable static state or rely on execution order. Each test should be independently runnable and produce the same result in isolation or in any order; shared static fields between tests are a classic source of flaky, order-dependent test suites.
- **DO:** Use `@BeforeEach` for genuinely shared setup and keep it minimal and visible in the test class — avoid deep inherited test base classes with hidden setup that later readers have to trace through several files to understand what state a test starts with.
- **DON'T:** Assert only that "no exception was thrown" as a test's entire assertion. A test that calls a method and asserts nothing about its result or side effects gives false confidence and won't catch a regression that silently returns the wrong value.
- **DO:** Use `assertThrows`/`assertDoesNotThrow` for exception-path tests, and AssertJ's fluent assertions (`assertThat(result).isEqualTo(...)`) where available for more readable failure messages than raw JUnit `assertEquals`.
- **DO:** Keep unit tests fast and hermetic (no real network calls, no real database, no `Thread.sleep`) and push anything requiring real infrastructure into a clearly separated integration test suite (e.g. Testcontainers-backed, run on a different Maven/Gradle profile or CI stage).
- **DON'T:** Write a test that depends on wall-clock time (`LocalDate.now()`) without injecting a fixed `Clock`. Time-dependent tests without a controllable clock are flaky by construction and eventually fail at midnight boundaries or on leap days in CI.
- **DO:** Use Mockito's `given()`/`when()` stubbing only for interactions that are actually exercised by the test; an unused stub (Mockito's strict stubbing will flag this by default) usually means dead test setup that should be deleted.

- **DO:** Use `@Nested` test classes (JUnit 5) to group related scenarios under a shared context (e.g. "when the account balance is sufficient" / "when the account balance is insufficient") — the resulting test report reads as structured, readable specification rather than a flat list of loosely related method names.
  ```java
  @Nested
  class WhenBalanceIsInsufficient {
      @Test
      void withdrawalThrows() { /* ... */ }
  }
  ```
- **DON'T:** Assert against implementation details — private field values reached via reflection, or exact internal call counts on collaborators that aren't actually part of the class's contract. Assert on the observable, public behavior a caller actually depends on, so refactoring the internals doesn't break unrelated tests.
- **DO:** Use object-mother or test-data-builder patterns for constructing complex domain objects in tests, rather than duplicating long, brittle multi-field constructor calls across dozens of test methods that all break together when a field is added.
- **DO:** Treat code coverage as a diagnostic signal pointing at untested areas, not as a target to be maximized directly — a suite with 100% line coverage and weak assertions (or none) is worse than a lower-coverage suite that actually verifies behavior.

- **DO:** Use JUnit 5's `assertAll()` to group multiple related assertions about the same result so all of them run and report together, rather than a sequence of separate `assertEquals` calls where the first failure aborts the test before the rest are even checked.
  ```java
  assertAll("order",
      () -> assertEquals(Status.PAID, order.getStatus()),
      () -> assertEquals(2, order.getItems().size())
  );
  ```
- **DON'T:** Use `Thread.sleep()` in a test to "wait for" asynchronous work to finish. This makes tests both slow (always waiting the full sleep duration) and flaky (too short under load, wastefully long otherwise) — use Awaitility, a `CountDownLatch`, or a synchronous test double instead of racing against wall-clock time.
- **DO:** Verify mock interaction order with Mockito's `InOrder` API only when the actual order of calls is part of the contract being tested (e.g. "validate before saving") — don't add ordering verification to tests where any order would be equally correct, since it needlessly couples the test to an implementation detail.
- **DO:** Use `@Captor`/`ArgumentCaptor` to assert on the actual value passed to a mocked collaborator when the test cares about that value's content, rather than a loose `any()` matcher that would pass even if the wrong data were sent.
- **DON'T:** Leave `@Disabled`/`@Ignore` tests in the suite indefinitely without a tracked follow-up (a linked ticket in the annotation's reason string, or a `TODO` with an owner). A disabled test with no plan to re-enable it quietly stops testing whatever it used to cover, and the team loses that coverage without realizing it.

## Concurrency (java.util.concurrent)

- **DO:** Use an `ExecutorService` (or, on Java 21+, structured concurrency via `StructuredTaskScope`) to manage thread pools instead of manually constructing and starting raw `Thread` instances. A managed executor gives you pool sizing, backpressure, and lifecycle control that hand-rolled thread management does not.
  ```java
  // DON'T
  new Thread(() -> processOrder(order)).start();

  // DO
  ExecutorService executor = Executors.newFixedThreadPool(poolSize);
  executor.submit(() -> processOrder(order));
  ```
- **DON'T:** Use `Executors.newCachedThreadPool()` or `Executors.newFixedThreadPool()` without bounding the underlying queue and considering rejection policy in production code. An unbounded work queue under sustained load can accumulate tasks until the process runs out of memory; prefer explicitly configuring a `ThreadPoolExecutor` with a bounded `BlockingQueue` and a defined `RejectedExecutionHandler`.
- **DO:** Prefer `CompletableFuture` for composing asynchronous operations (`thenApply`, `thenCompose`, `allOf`) over manually managing `Future.get()` calls and callback threading by hand. `CompletableFuture` chains express the dependency graph of async work declaratively and compose cleanly.
- **DON'T:** Call `Future.get()` (or block on a `CompletableFuture`) without a timeout in code that must stay responsive. An unbounded blocking call on a stuck downstream dependency can hang a request thread indefinitely; use `get(timeout, unit)` or `orTimeout(...)`.
- **DO:** Use the classes in `java.util.concurrent` (`ConcurrentHashMap`, `CopyOnWriteArrayList`, `AtomicInteger`/`AtomicReference`, `BlockingQueue` implementations) instead of manually synchronizing plain collections with `synchronized` blocks. These are purpose-built, well-tested, and typically far more scalable than a hand-rolled lock around a `HashMap`.
  ```java
  // DON'T
  private final Map<String, Integer> counts = new HashMap<>();
  synchronized void increment(String key) {
      counts.merge(key, 1, Integer::sum);
  }

  // DO
  private final ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();
  void increment(String key) {
      counts.merge(key, 1, Integer::sum);
  }
  ```
- **DON'T:** Use double-checked locking for lazy singleton initialization without the field being `volatile`, and prefer avoiding it altogether — Java's class-loading guarantees mean an initialization-on-demand holder class (a private static nested class) achieves the same lazy, thread-safe result with none of the memory-visibility subtlety.
- **DO:** Prefer higher-level concurrency utilities — `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `ReentrantLock`/`ReadWriteLock` — over hand-written `wait()`/`notify()` coordination. Manual `wait`/`notify` is notoriously easy to get wrong (missed signals, spurious wakeups) and the standard library's alternatives handle those edge cases correctly.
- **DON'T:** Assume `volatile` is sufficient for compound operations like increment-and-check. `volatile` only guarantees visibility, not atomicity — `count++` on a volatile `int` is still a read-modify-write race; use `AtomicInteger`/`AtomicLong` for atomic compound updates.
- **DO:** Always shut down an `ExecutorService` explicitly (`shutdown()`/`awaitTermination()`, or use it inside try-with-resources on Java 19+'s `AutoCloseable` executors) rather than letting its threads leak past the scope that created them, especially in short-lived contexts like per-request thread pools.
- **DO:** Design tasks submitted to a shared thread pool to be non-blocking or explicitly isolate blocking I/O work onto its own bounded pool, separate from CPU-bound work — mixing blocking calls into a pool sized for CPU-bound parallelism (e.g. `Runtime.getRuntime().availableProcessors()` threads) starves it under load.
- **DON'T:** Share mutable state across threads without a defined synchronization strategy "because it probably won't race in practice." Data races are undefined behavior in the Java Memory Model and can manifest as anything from silently wrong values to crashes, often only under production load — not reliably reproducible in testing.
- **DO:** Use `ThreadLocalRandom.current()` instead of a shared `java.util.Random` instance across threads — a shared `Random` under concurrent access has internal synchronized state that becomes a contention bottleneck.

- **DO:** Pass an explicit `Executor` to `CompletableFuture.supplyAsync`/`runAsync` for I/O-bound work rather than relying on the default `ForkJoinPool.commonPool()`. The common pool is shared process-wide (including by parallel streams), so unrelated work elsewhere in the process can starve it, and blocking I/O work on it can starve CPU-bound work that also depends on it.
- **DON'T:** Synchronize on `this` or a coarse class-level lock without deliberately reasoning about the actual critical section. Over-broad locking serializes unrelated operations and kills throughput under concurrent load; a lock's scope should match exactly the data it's protecting, no more.
- **DO:** Favor immutable objects and message-passing (bounded `BlockingQueue`s between worker threads) over shared mutable state guarded by locks wherever the design allows it — this eliminates entire classes of concurrency bugs rather than depending on every future maintainer applying correct lock discipline.
- **DO:** Use structured concurrency (`StructuredTaskScope`, stable in modern JDKs) for fork-join-style groups of concurrent subtasks whose cancellation and error propagation need to be coordinated as a unit — it ensures that if one subtask fails, siblings are cancelled automatically rather than left running orphaned.

- **DO:** Size thread pools deliberately based on the actual workload — roughly `number of cores` for CPU-bound work, and a larger, I/O-wait-aware size (or a dedicated bounded pool separate from CPU work) for blocking I/O-bound tasks — rather than an arbitrary fixed number copied from an unrelated example.
- **DON'T:** Use `Collections.synchronizedList`/`synchronizedMap` and then iterate over it without manually synchronizing on the wrapper during iteration. The synchronized wrapper only makes individual method calls atomic; iteration is a sequence of calls, and an unguarded iteration over a concurrently-modified synchronized collection can still throw `ConcurrentModificationException` or see inconsistent state — either wrap the whole iteration in a `synchronized` block on the collection, or use a proper `java.util.concurrent` collection instead.
  ```java
  // DON'T — still unsafe without an external lock around iteration
  List<String> list = Collections.synchronizedList(new ArrayList<>());
  for (String s : list) { ... } // not thread-safe

  // DO
  List<String> list = new CopyOnWriteArrayList<>(); // safe for iterate-heavy, write-light use
  ```
- **DO:** Use `CompletableFuture.exceptionally`/`handle`/`whenComplete` to define explicit error-handling behavior for each async stage, rather than letting an exception silently propagate to a `.get()` call far downstream where it surfaces as an opaque `ExecutionException` wrapping the real cause.
- **DON'T:** Assume `ExecutorService.submit()` surfaces a task's uncaught exception automatically. An exception thrown inside a `Runnable` submitted via `submit()` is captured in the returned `Future` and only surfaces when `.get()` is called on it — silently swallowed if nothing ever calls `.get()`. Either always inspect the returned `Future`, or set an explicit `Thread.UncaughtExceptionHandler` on the pool's thread factory for fire-and-forget submissions.

## Security-Conscious Coding

- **DO:** Use parameterized queries (`PreparedStatement`, or an ORM's own parameter binding) for every SQL statement built from external input, never string concatenation. Concatenated SQL built from user-controlled values is directly vulnerable to SQL injection, one of the longest-standing and still most common serious web application vulnerabilities.
  ```java
  // DON'T — SQL injection vulnerable
  String sql = "SELECT * FROM users WHERE email = '" + email + "'";

  // DO
  PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE email = ?");
  stmt.setString(1, email);
  ```
- **DON'T:** Deserialize untrusted data with Java's native `ObjectInputStream` (or any deserialization mechanism susceptible to gadget-chain attacks) without a strict allowlist of permitted classes. Unrestricted deserialization of untrusted input is a well-documented remote-code-execution vector; prefer a safer data format (JSON via a well-configured mapper) for untrusted input in the first place.
- **DO:** Validate and sanitize all externally-supplied input — web request parameters, uploaded files, message-queue payloads — at the boundary using an established validation layer (Bean Validation's `@Valid`/`@NotNull`/`@Size`, or explicit checks), rather than trusting client-supplied data implicitly deeper in the application.
- **DON'T:** Build file-system paths from user-supplied input without normalizing and verifying the resolved path stays within the intended base directory. Unvalidated path construction is vulnerable to path-traversal attacks (`../../etc/passwd`-style input escaping the intended directory).
- **DO:** Hash passwords with a purpose-built, adaptive algorithm (bcrypt, Argon2, PBKDF2) via an established library, never a fast general-purpose hash (MD5, bare SHA-256) — fast hashes are trivially brute-forced with modern hardware, defeating the point of hashing stored credentials at all.
- **DON'T:** Hardcode secrets — API keys, database passwords, signing keys — directly in source code or committed configuration files. Use a secrets manager, environment variables injected at deploy time, or an external vault, and make sure local-only secret files are excluded via `.gitignore`.
- **DO:** Rely on a templating engine's automatic output escaping (Thymeleaf, properly-used JSP tags) for any user-supplied content rendered into HTML, to prevent cross-site scripting, and set restrictive security headers (`Content-Security-Policy` and friends) on web responses.
- **DO:** Disable external entity resolution on XML parsers (`DocumentBuilderFactory`, `XMLInputFactory`) by default to prevent XXE (XML External Entity) attacks when parsing any XML that could originate from an untrusted source.

## Pattern Matching & Modern Switch (Java 16+)

- **DO:** Use pattern matching for `instanceof` (`if (obj instanceof String s) { ... }`, Java 16+) instead of a separate explicit cast after the type check. This eliminates the redundant cast entirely, along with its own theoretical `ClassCastException` risk if the check and cast ever drifted apart.
  ```java
  // DON'T
  if (obj instanceof String) {
      String s = (String) obj;
      process(s);
  }

  // DO
  if (obj instanceof String s) {
      process(s);
  }
  ```
- **DO:** Use switch expressions with `->` syntax (Java 14+) that directly yield a value, instead of the older statement-style `switch` requiring an explicit `break` on every branch — this eliminates the classic fall-through-by-forgotten-`break` bug entirely for cases that don't need it.
- **DON'T:** Rely on unmarked, implicit fall-through between `case` labels in a traditional `switch` statement. Accidental fall-through from a missing `break` is one of the most notorious classic Java bug patterns; if fall-through is genuinely intentional, use explicit multi-label cases (`case A, B ->`) or the `switch` expression form, or at minimum comment clearly why fall-through is deliberate.
- **DO:** Use record patterns (Java 21+) to destructure a record directly within a `switch`/`instanceof` pattern (`case Point(int x, int y) when x == y -> ...`), avoiding manual field-by-field extraction after a separate type check.
  ```java
  static String describe(Object shape) {
      return switch (shape) {
          case Circle(double radius) when radius > 10 -> "large circle";
          case Circle c -> "circle";
          case Rectangle(double w, double h) when w == h -> "square";
          default -> "unknown";
      };
  }
  ```
- **DO:** Use text blocks (`"""..."""`, Java 15+) for embedded multi-line string content — SQL, JSON templates, HTML fragments — instead of manually concatenated string literals full of escaped quotes and explicit `\n` characters.
- **DON'T:** Reach for a text block where a plain single-line string literal is clearer and shorter. Text blocks exist for genuinely multi-line content, not as a stylistic default applied to every string in a file.
- **DO:** Combine pattern matching with guarded conditions (`case Integer i when i > 0 -> ...`, Java 21+) for logic that goes beyond a simple type match, instead of a type-matched case followed by a separate nested `if` inside the branch body.

## API Design, Deprecation & Backward Compatibility

- **DO:** Mark obsolete public API members `@Deprecated(since = "2.3", forRemoval = true)` (Java 9+) with accompanying Javadoc explaining the replacement and migration path, rather than either silently removing them or leaving an undocumented `@Deprecated` with no guidance on what to use instead.
- **DON'T:** Break a published library's public method signatures — parameter types, return type, the set of checked exceptions thrown — in a minor or patch release. Follow semantic versioning: breaking changes belong in a major version bump, ideally preceded by a deprecation cycle that gives consumers time to migrate.
- **DO:** Add new methods to an existing public interface with third-party implementers as `default` methods, so existing implementations keep compiling without modification. Reserve non-default new interface methods for interfaces that are effectively closed to outside implementation.
  ```java
  public interface PaymentProcessor {
      PaymentResult process(Payment payment);

      // DO — default method, existing implementers unaffected
      default boolean supportsRefunds() { return false; }
  }
  ```
- **DON'T:** Change the observable behavior of an existing public method silently — for instance, changing which exception type is thrown for the same failure condition. This is a breaking change even though the method's signature is unchanged, and it can silently break callers who catch the old, specific exception type.
- **DO:** Prefer additive changes — new overloads, a new optional builder parameter — over modifying an existing public method's established contract when extending a widely-used API, minimizing the blast radius of the change for existing consumers.
- **DO:** Version published library artifacts with semantic versioning (`MAJOR.MINOR.PATCH`) consistently and document breaking changes clearly in release notes, so downstream consumers can assess upgrade risk before pulling in a new version.

## I/O, Files & Resource Handling

- **DO:** Use the NIO.2 `Path`/`Files` API (`Files.readAllLines`, `Files.newBufferedReader`, `Path.resolve`) instead of the legacy `java.io.File` API for new code. NIO.2 provides clearer exception messages, proper symbolic-link handling, and a more consistent, composable API surface.
- **DON'T:** Read an entire file into memory (`Files.readAllBytes`/`readAllLines`) when the file's size could be large or unbounded. Stream it instead — `Files.lines()` combined with incremental processing, or a `BufferedReader` read loop — so memory usage stays bounded regardless of how large the file turns out to be.
  ```java
  // DON'T — loads the whole file into memory at once
  List<String> lines = Files.readAllLines(path);
  lines.forEach(this::process);

  // DO — processes line by line with bounded memory
  try (Stream<String> lines = Files.lines(path)) {
      lines.forEach(this::process);
  }
  ```
- **DO:** Wrap raw streams with buffering (`BufferedReader`, `BufferedOutputStream`) whenever doing many small reads or writes. Unbuffered I/O that makes a system call per small chunk is dramatically slower than the buffered equivalent.
- **DO:** Always specify an explicit `Charset` (`StandardCharsets.UTF_8`) when reading or writing text, rather than relying on the JVM's platform-default charset — the default varies by operating system and locale, producing inconsistent behavior for the exact same code across different environments.
- **DON'T:** Leave a `Scanner`, `InputStream`, or `OutputStream` unclosed after use. Try-with-resources applies here just as much as to database connections — a forgotten file handle is a common resource leak that, under sustained load, can exhaust a process's available file descriptors.
- **DO:** Use `Files.createTempFile`/`createTempDirectory` for temporary file needs, and ensure temp resources are actually cleaned up (a shutdown hook for long-running processes, `deleteOnExit` for short-lived ones) rather than letting stale temporary files accumulate on disk over time.

## Annotations & Reflection Discipline

- **DO:** Design custom annotations with a clear, singular purpose and the narrowest sufficient retention policy — `RUNTIME` only when the annotation genuinely needs to be inspected at runtime, `SOURCE`/`CLASS` otherwise. Unnecessary runtime retention adds a small but real overhead every time that element is reflectively inspected.
- **DON'T:** Use reflection to bypass access modifiers (`setAccessible(true)` to read or write a private field) in regular application code as a shortcut around a poorly-designed API. Fix the API's visibility or design instead of reaching for reflection to work around encapsulation that was presumably intentional in the first place.
  ```java
  // DON'T — reaching into private state via reflection
  Field field = obj.getClass().getDeclaredField("balance");
  field.setAccessible(true);
  field.set(obj, newBalance);

  // DO — expose a proper method if the mutation is legitimate
  obj.adjustBalance(newBalance);
  ```
- **DO:** Cache the results of expensive reflective lookups (`Class.getDeclaredMethods()`, annotation scanning across a classpath) rather than repeating them on every invocation. Reflection is meaningfully slower than a direct method call, and an uncached reflective lookup sitting in a hot path is a common, easily-fixed performance issue.
- **DO:** Prefer compile-time alternatives to reflection where they exist — annotation processors that generate real code at compile time (Lombok, MapStruct, Dagger) instead of an equivalent built on runtime reflection — since compile-time generation catches errors earlier and avoids the runtime reflection cost entirely.
- **DON'T:** Use an annotation as a substitute for documenting genuinely non-obvious behavior. An annotation communicates a specific, framework-understood meaning to tooling; free-form rationale and intent still belong in a comment or Javadoc.
- **DO:** Validate that a reflectively-invoked method or constructor actually exists and is accessible before invoking it, handling `NoSuchMethodException`/`IllegalAccessException` explicitly, rather than letting a reflection-based integration fail with an opaque low-level exception far from the actual misconfiguration that caused it.

## Enums Done Right

- **DO:** Give enum constants their own behavior via constant-specific method bodies or an abstract method implemented per constant, instead of a `switch` on the enum value scattered across the codebase. This keeps behavior colocated with the constant it belongs to and makes adding a new constant a compile-time-checked, single-location change.
  ```java
  enum Operation {
      PLUS { public int apply(int a, int b) { return a + b; } },
      MINUS { public int apply(int a, int b) { return a - b; } };
      public abstract int apply(int a, int b);
  }
  ```
- **DON'T:** Use `Enum.ordinal()` for business logic — persistence, comparison, external representation. Ordinal values shift silently when constants are reordered or a new one is inserted, which can corrupt persisted data or comparisons with no compiler warning at all. Use an explicit field for any value that must stay stable across future code changes.
- **DO:** Implement a shared interface across enum constants when several unrelated enum types need to expose common behavior polymorphically, letting calling code depend on the interface rather than a specific enum type.
- **DON'T:** Add a new enum constant without auditing existing exhaustive `switch` statements over that enum for a `default` case that would silently swallow the new value without giving it real, specific handling. A modern `switch` expression (see Pattern Matching & Modern Switch) forces this to be addressed at compile time when there's no `default`; an older statement-style `switch` with a catch-all `default` does not.
- **DO:** Use `EnumMap`/`EnumSet` (see Collections API Usage) whenever a collection is specifically keyed or populated by enum constants, for both the performance benefit and the added expressiveness over a general-purpose `HashMap`/`HashSet`.
- **DO:** Consider a single-element enum as Java's standard, thread-safe, serialization-safe way to implement a singleton, since it gets serialization and reflection-attack resistance handled correctly for free — advantages a hand-rolled singleton class doesn't get without extra, easy-to-miss work.

## Common AI-Assistant Mistakes in Java

- **DON'T:** Generate verbose, pre-Java-8-style boilerplate (manual getters/setters plus manual `equals`/`hashCode`/`toString`, or an explicit builder class) for a plain immutable data carrier when a `record` (Java 16+) expresses the same thing in one line. Defaulting to old idioms out of habit produces code that's both longer and less clear about intent than the modern equivalent.
  ```java
  // DON'T — 30+ lines of boilerplate for a value holder
  public class Point {
      private final int x;
      private final int y;
      public Point(int x, int y) { this.x = x; this.y = y; }
      public int getX() { return x; }
      public int getY() { return y; }
      @Override public boolean equals(Object o) { /* ... */ }
      @Override public int hashCode() { /* ... */ }
      @Override public String toString() { /* ... */ }
  }

  // DO
  public record Point(int x, int y) {}
  ```
- **DON'T:** Generate code that opens a `FileInputStream`, `Connection`, `Scanner`, or similar `Closeable` resource with a manual `try`/`finally` (or worse, no cleanup at all) when try-with-resources is idiomatic and safer. This is one of the most common "looks fine but leaks resources under exception" patterns produced by pattern-matching on older training examples instead of current idioms.
- **DON'T:** Invent library methods, class names, or overloads that sound plausible but don't exist (`String.isNullOrEmpty()` — that's Guava's `Strings.isNullOrEmpty`, not a `java.lang.String` method; `List.getFirst()` did not exist before Java 21; a `Collectors.toImmutableList()` that doesn't exist in the standard library). Always verify a method exists in the actual JDK version and dependencies the project targets rather than assuming an API "should" have a convenience method because a similar library does.
- **DON'T:** Assume the newest language features (`record`, pattern matching for `switch`, sealed classes, virtual threads) are available without checking the project's actual configured Java version (`<maven.compiler.release>`, `sourceCompatibility`). Suggesting Java 21 syntax in a codebase pinned to Java 11 produces code that simply doesn't compile.
- **DON'T:** Default to `Vector`, `Hashtable`, raw `Thread`, `StringBuffer` (instead of `StringBuilder` for single-threaded use), or anonymous inner classes for simple functional interfaces (instead of lambdas) — these are legacy idioms that outdated training examples over-represent relative to their actual use in current, well-maintained Java code.
  ```java
  // DON'T
  Runnable r = new Runnable() {
      @Override public void run() { doWork(); }
  };

  // DO
  Runnable r = () -> doWork();
  ```
- **DON'T:** Generate a null check followed by a manual default-value assignment where `Optional.ofNullable(...).orElse(...)`, `Objects.requireNonNullElse(...)`, or a simple `??`-style pattern (via a ternary) would be clearer, or conversely, wrap every single nullable field access in `Optional` where a plain null check reads better — match the idiom to the codebase's existing null-handling convention rather than mixing styles.
- **DON'T:** Fabricate a Maven/Gradle dependency coordinate or version that "sounds right" (e.g. guessing at `com.google.guava:guava:33.5.0` without checking Maven Central for what actually exists) instead of confirming it. A hallucinated artifact coordinate breaks the build the moment someone actually runs it, often with a confusing "could not resolve" error far from the point the dependency was added.
- **DON'T:** Add a stream pipeline, a design pattern (Builder, Factory, Strategy), or an extra abstraction layer to a two-line piece of logic just because it's a "best practice" in the abstract. Match the amount of ceremony to the actual complexity of the problem — a `Builder` for a two-field constructor argument is pure overhead, not good design.
- **DON'T:** Ignore an existing project's established conventions (its logging framework, its exception hierarchy, its DI style) and introduce a different one in generated code "because it's more standard." Consistency with the surrounding codebase is worth more than a marginally different but locally unfamiliar idiom.
- **DON'T:** Generate a `catch (Exception e) { e.printStackTrace(); }` block as a stand-in for real error handling. This is a common shortcut in generated examples that looks like handling but neither logs to the application's actual logging pipeline nor gives the caller any way to react — replace it with a proper logger call and a deliberate decision about whether to rethrow, return a fallback, or surface the failure.
- **DON'T:** Hand-roll something the standard library already provides well — a manual string-joining loop instead of `String.join`/`Collectors.joining`, or a bespoke retry loop instead of the resilience library the project already depends on. Reinventing well-tested standard functionality introduces bugs the library already solved and adds unnecessary review burden.
- **DON'T:** Mix exception-handling styles within one generated change — logging in some catch blocks and not others, or mixing checked and unchecked exception idioms inconsistently across the same file. Pick one strategy and apply it uniformly throughout the change.
- **DON'T:** Silently alter a public method's signature or observable behavior while ostensibly fixing something unrelated. A generated diff should do what was asked and nothing more — an incidental behavior change to unrelated callers is a regression the requester didn't sign up for.
- **DON'T:** Generate an entire new abstraction layer (a repository interface plus implementation plus a factory plus a builder) in response to a request for one small method, when the existing codebase already has a simpler, established pattern for the same kind of change. Match the scope of the change to what was actually asked; propose the larger refactor separately if it's genuinely warranted, rather than bundling it in unasked.
- **DON'T:** Assume a third-party library's API from an older, more widely-represented major version when the project's actual dependency is a newer major version with breaking changes (e.g. suggesting Mockito 1.x's `when(...).thenReturn(...)` static-import style patterns that predate `@ExtendWith(MockitoExtension.class)`, or a Spring 5-era XML configuration style in a Spring Boot 3/Jakarta EE codebase). Check the actual dependency versions declared in the build file before assuming API shape.
- **DON'T:** Leave `TODO`/placeholder comments or stub method bodies (`throw new UnsupportedOperationException("not implemented")`) in code presented as a finished deliverable without calling out explicitly, in the response, that a piece is intentionally incomplete. Silent gaps in generated code are far more costly to discover later than an upfront, explicit callout.

## Quick Checklist
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
