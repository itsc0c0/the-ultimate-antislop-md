# C#/.NET

## Naming Conventions

- **DO:** Follow the .NET naming conventions consistently: `PascalCase` for types, methods, properties, and namespaces; `camelCase` for local variables and method parameters; an `I` prefix for interfaces (`IRepository`); and a leading underscore convention for private fields only if the team has standardized on it (`_logger`), applied uniformly. Consistency with the wider .NET ecosystem's conventions makes a codebase instantly navigable to anyone with .NET experience.
  ```csharp
  public interface IOrderRepository
  {
      Task<Order?> FindByIdAsync(Guid orderId);
  }
  ```
- **DON'T:** Mix naming conventions within the same codebase — some classes using `_camelCase` private fields, others using bare `camelCase`, others using `m_camelCase`. Pick one convention (an `.editorconfig` with enforced analyzers is the mechanism, not a wiki page) and let tooling enforce it rather than relying on reviewers to catch drift.
- **DO:** Name async methods with an `Async` suffix (`GetUserAsync`, `SaveChangesAsync`) so callers can tell at the call site that a method returns a `Task`/`Task<T>` and must be awaited, per the Task-based Asynchronous Pattern (TAP) convention. Omitting the suffix hides the asynchronous nature of the call and makes it easy to forget an `await`.
- **DON'T:** Suffix synchronous methods with `Async`, and don't leave asynchronous methods without the suffix — mismatched naming actively misleads callers about which methods need awaiting.
- **DO:** Name boolean properties and methods as assertions (`IsValid`, `HasItems`, `CanExecute`) matching the same convention used across the BCL (`String.IsNullOrEmpty`), so a boolean's meaning is unambiguous without checking its declaration.
- **DO:** Use `PascalCase` even for constants and `static readonly` fields (`public const int MaxRetryCount = 5;`), unlike Java's `UPPER_SNAKE_CASE` convention — following the platform's own convention rather than importing a convention from a different language ecosystem.
- **DON'T:** Use Hungarian notation or type-encoding prefixes (`strName`, `bIsActive`) — these were common in early Windows/COM-era C# code and are explicitly discouraged by Microsoft's own current guidelines; the IDE's type information makes them redundant noise today.
- **DO:** Name generic type parameters descriptively when there's more than one or when a single letter doesn't convey enough (`TKey`, `TValue`, `TEntity`) rather than defaulting to `T1`, `T2` for multi-parameter generics.
- **DO:** Enforce naming and style with `.editorconfig` plus Roslyn analyzers (including the built-in .NET analyzers or StyleCop) in the build, so violations show up as build warnings/errors rather than as review comments repeated project after project.

- **DO:** Prefix interfaces with `I` (`IRepository`, `IDisposable`) consistently — this is one of the few "type-encoding" prefixes still endorsed by current Microsoft guidance, precisely because it disambiguates an interface from its implementing class at a glance in a language where both commonly share a base name (`IOrderRepository` / `OrderRepository`).
- **DON'T:** Use all-caps acronyms longer than two letters in identifiers (`HTTPClient`, `XMLParser`); follow Pascal-casing for acronyms of three or more letters (`HttpClient`, `XmlParser`), per the .NET naming guidelines — two-letter acronyms like `IO` or `ID` stay capitalized (`FileIO`, `UserId`).
- **DO:** Name namespaces to mirror the project's folder structure and assembly name (`Acme.Billing.Invoicing`), keeping the mapping between "where a file lives" and "what namespace it's in" predictable for navigation.
- **DO:** Use `record`/`record struct` naming the same as any other type (`PascalCase`), and default to positional records for simple immutable data (`public record Money(decimal Amount, string Currency);`) rather than a verbose class with manually written boilerplate for the same shape.
- **DON'T:** Name a method that mutates the object it's called on as if it were a query returning a new value (`list.Sort()` — a real BCL example — mutates in place, while LINQ's `list.OrderBy()` returns a new sequence; when designing your own APIs, keep this distinction unambiguous rather than replicating an inconsistency).

- **DO:** Name events and their handler delegates using the standard .NET pattern — the event itself as a noun/verb-phrase without a prefix (`OrderPlaced`), the `EventHandler<TEventArgs>` delegate type suffixed `EventArgs` (`OrderPlacedEventArgs`) — matching the convention used throughout the BCL so the pattern is immediately recognizable.
- **DON'T:** Use `Hungarian`-adjacent prefixes reintroduced by habit from other ecosystems (`bIsActive`, `strName`) or a leading underscore on anything other than private fields (some teams also use it for private static fields) — keep the prefix meaning consistent and limited to what the team's `.editorconfig` actually documents.
- **DO:** Name asynchronous LINQ-style extension methods with the `Async` suffix consistently even when they return `IAsyncEnumerable<T>` rather than `Task<T>` (`GetItemsAsync()` returning `IAsyncEnumerable<Item>`), since the suffix communicates "this uses `await`/asynchronous iteration," not specifically "this returns a `Task`."
- **DO:** Favor `nameof(parameter)` over a hardcoded string literal when referencing a member/parameter name in exceptions or logging (`throw new ArgumentNullException(nameof(customer))`), so a rename via refactoring tools keeps the reference correct automatically.

## Async/Await Correctness

- **DON'T:** Write `async void` methods except for top-level event handlers that the platform itself requires to return `void` (e.g. a WinForms/WPF button-click handler). An `async void` method's exceptions cannot be caught by the caller with a normal `try`/`catch` — they escape to the synchronization context and typically crash the process — and the caller has no `Task` to await, so there's no way to know when it completes.
  ```csharp
  // DON'T
  public async void ProcessOrder(Order order) { await SaveAsync(order); }

  // DO
  public async Task ProcessOrderAsync(Order order) { await SaveAsync(order); }
  ```
- **DON'T:** Block on asynchronous code with `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` from synchronous calling code. This is one of the most common causes of deadlocks in classic ASP.NET / WPF / WinForms applications: blocking synchronously on a task that needs to resume on a captured `SynchronizationContext`, while that same context's single thread is the one doing the blocking, deadlocks the application. Make the call chain `async` all the way up instead.
  ```csharp
  // DON'T — classic deadlock risk on a UI/ASP.NET classic context
  var result = GetDataAsync().Result;

  // DO
  var result = await GetDataAsync();
  ```
- **DO:** Use `ConfigureAwait(false)` on awaited calls inside library/non-UI code that doesn't need to resume on the original synchronization context, to avoid unnecessary context-capturing overhead and reduce deadlock risk when that library is called synchronously from a context-sensitive caller. Modern ASP.NET Core has no `SynchronizationContext` by default, so this matters most for libraries that might be consumed by UI apps or classic ASP.NET, and for reusable NuGet packages.
- **DON'T:** Apply `ConfigureAwait(false)` reflexively everywhere, including in ASP.NET Core application code (as opposed to library code) where there's no synchronization context to avoid in the first place, or in code that genuinely does need to resume on the original context (e.g. to touch UI controls afterward). Understand *why* it's being added rather than pasting it on every `await` out of habit.
- **DO:** Propagate `CancellationToken` parameters through the full async call chain and honor them in long-running loops and I/O calls (`await SomeAsync(cancellationToken)`), rather than accepting a token parameter and never actually passing it further down or checking it.
  ```csharp
  public async Task<List<Item>> LoadItemsAsync(CancellationToken cancellationToken)
  {
      return await _db.Items
          .Where(i => i.IsActive)
          .ToListAsync(cancellationToken);
  }
  ```
- **DO:** Use `Task.WhenAll` to run independent asynchronous operations concurrently when their results are genuinely independent, rather than `await`ing them one after another sequentially when there's no data dependency between them.
  ```csharp
  // DON'T — sequential when independent
  var user = await GetUserAsync(id);
  var orders = await GetOrdersAsync(id);

  // DO — concurrent
  var userTask = GetUserAsync(id);
  var ordersTask = GetOrdersAsync(id);
  await Task.WhenAll(userTask, ordersTask);
  var user = userTask.Result; // safe: task is already complete here
  ```
- **DON'T:** Wrap an already-asynchronous operation in `Task.Run` inside server-side (ASP.NET) code just to "make it async." `Task.Run` schedules work onto the thread pool — useful for offloading genuinely CPU-bound work from a UI thread — but wrapping I/O-bound async work in it on the server adds an extra thread-pool hop for no benefit, since the I/O operation was already non-blocking.
- **DO:** Use `IAsyncDisposable`/`await using` for resources whose cleanup is itself asynchronous (e.g. flushing an async stream), rather than forcing synchronous `Dispose()` cleanup logic that blocks on async work internally.
- **DON'T:** Catch and discard `TaskCanceledException`/`OperationCanceledException` indiscriminately across an entire async call chain. Cancellation is a legitimate, expected outcome that callers should be able to distinguish from an actual failure — swallow it only at the boundary that's actually responsible for responding to cancellation (e.g. returning an appropriate response), not silently everywhere it might bubble up.
- **DO:** Return `Task.CompletedTask` (or a cached completed `Task<T>`) from a method that implements an async-returning interface but has no actual asynchronous work to do, rather than marking it `async` with no `await` inside, which triggers a compiler warning and adds unnecessary state-machine overhead.

- **DO:** Use `IAsyncEnumerable<T>` with `await foreach` for asynchronously streamed sequences of results (e.g. paged database results, a streaming API response) rather than buffering the entire result set into a `List<T>` before returning it, when the consumer can genuinely process items as they arrive.
- **DON'T:** Mix synchronous blocking calls with `async`/`await` within the same method (calling a synchronous, blocking overload of an operation that also has an async version available). If a method is `async`, every I/O-bound operation inside it should use its async counterpart consistently — a stray synchronous call defeats the point of making the surrounding method asynchronous at all.
- **DO:** Return `ValueTask`/`ValueTask<T>` instead of `Task`/`Task<T>` for hot-path async methods that frequently complete synchronously (e.g. a cache-hit fast path), to avoid the heap allocation a `Task` requires — but only after profiling shows it matters, since `ValueTask` has stricter usage rules (it must not be awaited twice) that make it easy to misuse.
- **DO:** Use `Task.Delay` with a `CancellationToken` for cancellable async waits, never `Thread.Sleep`, inside asynchronous code — `Thread.Sleep` blocks the calling thread synchronously regardless of the surrounding method being `async`.
- **DON'T:** Fire off multiple `async void` "handlers" from a loop expecting them to be awaited together. Since `async void` returns nothing awaitable, there's no way to know when they've all completed or whether any of them threw; use `Task`-returning methods collected into a list and awaited with `Task.WhenAll` instead.

- **DO:** Understand that `async`/`await` doesn't create new threads by itself — an `await`ed I/O operation typically frees the calling thread back to the pool while the I/O completes, whereas `Task.Run` explicitly schedules work onto a thread-pool thread; conflating "asynchronous" with "runs on another thread" leads to unnecessary `Task.Run` wrapping of work that was already non-blocking.
- **DON'T:** Await a `Task` inside a loop one at a time when the tasks could be started together and awaited as a batch, unless there's a genuine reason (e.g. rate limiting, or each iteration depending on the previous one's result) that requires strict sequencing.
- **DO:** Use `Task.WhenAny` with a manually-created timeout `Task` (or a `CancellationTokenSource` with `CancelAfter`) to implement "wait for this operation, but give up after N seconds," rather than a manual polling loop checking `Task.IsCompleted` on a timer.
- **DO:** Flow a single `CancellationTokenSource` created at the top of a logical operation (a web request, a background job run) down through every downstream async call that participates in that operation, so a single cancellation signal (client disconnect, shutdown) reliably stops the whole chain rather than just the top-level call.

## LINQ Idioms vs. Overuse

- **DO:** Use LINQ query/method syntax for genuinely declarative filtering, projection, and aggregation over in-memory collections or `IQueryable` data sources — it's usually clearer than an equivalent hand-written loop and, over `IQueryable`, translates directly into an efficient SQL query at the provider level.
  ```csharp
  var topCustomers = orders
      .Where(o => o.Status == OrderStatus.Completed)
      .GroupBy(o => o.CustomerId)
      .Select(g => new { CustomerId = g.Key, Total = g.Sum(o => o.Amount) })
      .OrderByDescending(x => x.Total)
      .Take(10);
  ```
- **DON'T:** Chain LINQ methods so deeply, or nest so many lambdas, that the query becomes harder to read than the loop it replaced. Break a long chain into intermediate named variables, or extract a well-named local function/method for a complex predicate rather than an inline multi-line lambda buried in the middle of a chain.
- **DON'T:** Call `.ToList()`/`.ToArray()` mid-chain "just in case" before the query is actually complete, especially against `IQueryable`. Materializing early forces the rest of the pipeline to run in memory instead of translating to the data source's native query, which for a database source means pulling far more rows across the wire than necessary.
- **DO:** Understand the difference between deferred execution (a LINQ query built against `IEnumerable`/`IQueryable` isn't actually run until enumerated) and materialization (`.ToList()`, `.Count()`, `.First()`) — re-enumerating a deferred query multiple times re-executes the whole pipeline (and, for `IQueryable`, re-hits the database) each time, which is a common accidental-performance and accidental-N-database-calls bug.
  ```csharp
  // DON'T — re-executes the query against the DB on every enumeration
  var query = _db.Orders.Where(o => o.CustomerId == id);
  var count = query.Count();      // hits the DB
  var list = query.ToList();      // hits the DB again

  // DO — materialize once, reuse the in-memory result
  var orders = await _db.Orders.Where(o => o.CustomerId == id).ToListAsync();
  var count = orders.Count;
  ```
- **DON'T:** Use `.Where(...).First()` (or `.Single()`) when `.First(predicate)` (or `.Single(predicate)`) says the same thing without allocating an intermediate filtered sequence. Prefer the predicate-overload form of `First`/`FirstOrDefault`/`Single`/`Any`/`Count` directly over chaining `.Where()` first.
- **DO:** Use `FirstOrDefault()`/`SingleOrDefault()` when "not found" is an expected, handled outcome, and `First()`/`Single()` when the element's presence is a precondition whose absence is genuinely exceptional — using the wrong one either masks a bug (silently returning `default` for a value that should always exist) or throws where a graceful "not found" path was actually expected.
- **DON'T:** Use `.Single()` in a hot path when only uniqueness needs verifying and the risk of multiple matches is a real bug worth surfacing loudly — but be aware `Single()` incurs the cost of scanning for a second match to confirm uniqueness, which is unnecessary overhead if the data is already guaranteed unique by a database constraint; prefer `First()` in that case.
- **DO:** Use LINQ's `.Select(...)` for projection down to only the fields actually needed, especially against `IQueryable`/Entity Framework — projecting to a DTO before materializing pulls only the needed columns instead of full entities.
- **DON'T:** Overuse LINQ for logic with significant side effects per element (writing to a database, calling an external API per item). `.Select()` with side effects inside the lambda works but reads misleadingly as a pure transformation; a plain `foreach` loop states the intent (deliberate, sequenced side effects) far more honestly.

- **DO:** Use pattern matching (`switch` expressions, `is` patterns, property patterns) for multi-branch conditional logic over a type or its shape, since C#'s pattern matching is exhaustiveness-aware for closed hierarchies (with a discard `_` arm required otherwise) and reads more directly than a chain of `if`/`else if` type checks.
  ```csharp
  decimal ApplyDiscount(Customer c) => c switch
  {
      { Tier: CustomerTier.Gold } => 0.20m,
      { Tier: CustomerTier.Silver } => 0.10m,
      _ => 0m
  };
  ```
- **DON'T:** Use `.Count()` (the LINQ extension method, which enumerates) on a source that's already a `List<T>`/array exposing a `.Count`/`.Length` property. Calling the LINQ `Count()` extension on an `IList<T>` does get optimized by the BCL to use the property internally, but on a plain `IEnumerable<T>` it forces a full enumeration — know which one you actually have and prefer the direct property when it's available and the type is statically known.
- **DO:** Use `.Zip()` for combining two sequences element-by-element into a single projection, rather than manually indexing into both with a `for` loop when both sequences are the same, known length.
- **DO:** Prefer expression-bodied LINQ queries/methods (`=>`) for simple, single-expression transformations, keeping the method-syntax vs. query-syntax choice (`.Where(...)` vs `from x in ... where ...`) consistent within a file rather than mixing both styles for equivalent logic.
- **DON'T:** Use `.OrderBy()` followed by `.Reverse()` when `.OrderByDescending()` says the same thing in one call and typically performs at least as well.

- **DO:** Use `SelectMany` for flattening/joining nested sequences (a customer's orders, each order's line items, into a single flat sequence of line items) instead of a nested `foreach` loop with a manually-populated accumulator list.
  ```csharp
  var allLineItems = customers
      .SelectMany(c => c.Orders)
      .SelectMany(o => o.LineItems);
  ```
- **DON'T:** Use `.GroupBy()` against `IQueryable`/Entity Framework expecting it to always translate efficiently to SQL `GROUP BY` — some grouping/aggregation shapes translate well, others force client-side evaluation depending on the provider and EF Core version; check the generated SQL for anything beyond a simple `GroupBy().Select(g => new { g.Key, Count = g.Count() })` shape.
- **DO:** Use LINQ's `Aggregate()` sparingly, and only when the built-in named aggregations (`Sum`, `Average`, `Max`, `Count`) genuinely don't cover the calculation — a hand-rolled `Aggregate()` is usually less readable than the equivalent named method when one exists, and less readable than a plain loop when none does.
- **DON'T:** Forget that LINQ's `OrderBy`/`ThenBy` produce a *stable* sort per the documented contract, but that relying on "whatever order a `GroupBy` happens to enumerate its groups in" is not itself guaranteed — add an explicit `OrderBy` after grouping if a specific group order actually matters to the consumer.

## Nullable Reference Types

- **DO:** Enable nullable reference types (`<Nullable>enable</Nullable>` in the project file, or `#nullable enable`) on any project targeting C# 8+, and treat every compiler nullability warning as something to actually resolve, not suppress. This turns an entire class of `NullReferenceException`s into compile-time warnings, which is a substantial reliability win essentially for free once the codebase is annotated.
- **DON'T:** Suppress nullable warnings with the null-forgiving operator (`!`) as a default way to make the compiler stop complaining. Like Kotlin's `!!`, the `!` operator tells the compiler "trust me, this isn't null" without any runtime check — if the assumption is wrong, it's a runtime `NullReferenceException` in exactly the place the nullable-reference-types feature was designed to prevent.
  ```csharp
  // DON'T
  string name = user.Profile!.Name!;

  // DO
  string name = user.Profile?.Name ?? "Unknown";
  ```
- **DO:** Annotate method signatures accurately with `?` on parameters and return types that can genuinely be null, rather than declaring everything non-nullable and then routinely passing/returning null anyway (which the compiler will flag, and which then gets silenced with `!`). Accurate annotations are what make the whole feature worth enabling.
- **DON'T:** Enable nullable reference types at the project level and then leave broad swaths of legacy code with unresolved warnings treated as noise. Either migrate incrementally with `#nullable enable` scoped to specific files/namespaces until the whole project is clean, or accept the feature provides little value while most of the codebase is still warning-suppressed.
- **DO:** Use `ArgumentNullException.ThrowIfNull(value)` (C# 11+/.NET 7+) or an explicit `??  throw new ArgumentNullException(nameof(value))` guard at public API boundaries, even with nullable reference types enabled — NRT annotations are a compile-time-only convention, not enforced at runtime, so a caller from unannotated code (or one that uses `!`) can still pass an actual null at runtime.
- **DO:** Model "value or absence" for value types with `Nullable<T>`/`T?` (e.g. `DateTime?`, `int?`), and use pattern matching (`if (value is { } v)`, or `value.HasValue`) rather than checking `!= null` and then accessing `.Value` separately in a way that can throw if the check and the access ever drift apart.
- **DON'T:** Overuse the null-conditional operator (`?.`) to silently no-op through a chain where a missing value partway through actually indicates a bug that should be surfaced, not quietly ignored. `?.` is for legitimate optionality, not a blanket way to avoid ever thinking about whether null is actually expected at each link in the chain.
- **DO:** Annotate generic type constraints with `where T : notnull` when a generic type parameter should never be null, letting the compiler enforce that constraint at every call site rather than discovering a null generic argument at runtime.
- **DON'T:** Disable nullable warnings for an entire file with a blanket `#nullable disable` as a permanent fix rather than a temporary, tracked step in an incremental migration. A permanently-disabled file silently opts out of every future nullability benefit for all code added to it going forward, not just the code that existed when it was disabled.
- **DO:** Use the null-coalescing assignment operator (`??=`) for lazy-initialization-style "set only if null" logic (`_cache ??= BuildCache();`) instead of a longer explicit `if (_cache == null) { _cache = ...; }` block.
- **DO:** Treat a nullable warning surfaced on a third-party library call as real signal to investigate — either the library's own nullable annotations (if present) are telling you something legitimate about that call, or the library predates nullable annotations and the boundary needs an explicit, deliberate decision rather than a reflexive `!`.
- **DO:** Use required members (`required` modifier, C# 11+) for properties that must be set at object-initialization time but can't be constructor parameters (e.g. on a type that must stay usable with object-initializer syntax) — the compiler then enforces that every construction site actually sets them, closing a gap that nullable reference types alone don't cover for mutable object-initializer-based construction.
- **DON'T:** Mark a property `required` and then also give it a non-null default value that would satisfy it anyway — this defeats the purpose of `required`, which is specifically to force the caller to supply a real value rather than silently accept a default.
- **DO:** Use nullable value-type pattern matching (`if (maybeDate is { } date)`) as the idiomatic, expression-friendly way to both null-check and unwrap a `Nullable<T>` in one step, in preference to the older `.HasValue`/`.Value` two-step pattern.

## Exception Handling

- **DO:** Throw specific, meaningful exception types (`ArgumentException`, `InvalidOperationException`, or a custom domain exception) rather than the generic `Exception` base class, so callers can catch precisely what they know how to handle.
- **DON'T:** Use an empty `catch { }` block or `catch (Exception) { }` with no logging and no rethrow. Silently swallowing an exception hides the failure until it manifests later as a confusing, disconnected symptom — always at minimum log it with context, and reconsider whether the catch should exist at that layer at all.
  ```csharp
  // DON'T
  try { ProcessPayment(order); }
  catch { }

  // DO
  try { ProcessPayment(order); }
  catch (PaymentDeclinedException ex)
  {
      _logger.LogWarning(ex, "Payment declined for order {OrderId}", order.Id);
      throw;
  }
  ```
- **DO:** Use `throw;` (bare, with no expression) to rethrow an exception from within a catch block, never `throw ex;`. `throw ex;` resets the exception's stack trace to the rethrow point, destroying the information about where it actually originated; bare `throw;` preserves the full original stack trace.
- **DON'T:** Use exceptions for expected, common control flow (e.g. throwing to signal "record not found" on every lookup miss in a hot path). Exceptions in .NET carry real performance cost (stack unwinding) and make normal-path logic harder to follow; return `null`/a result type/a `TryGetX` pattern for expected "not found" cases instead, reserving exceptions for truly exceptional conditions.
- **DO:** Use exception filters (`catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)`) to handle specific sub-conditions of an exception type without catching and immediately re-throwing (or nesting an `if` inside a broader catch) to distinguish cases.
- **DO:** Centralize cross-cutting exception handling for web APIs in middleware (a custom exception-handling middleware, or the built-in `UseExceptionHandler`/`IExceptionHandler` in modern ASP.NET Core) rather than repeating the same try/catch-and-format-response logic in every controller action.
- **DON'T:** Let an unhandled exception cross an `async void` boundary or a background service's main loop unguarded — either terminates the process in ways that are hard to diagnose after the fact. Wrap background/hosted service loops with their own top-level exception handling and logging, and use `AppDomain.UnhandledException`/`TaskScheduler.UnobservedTaskException` as a last-resort safety net for diagnostics, not as the primary handling strategy.
- **DO:** Include actionable context in custom exceptions (relevant IDs, the invalid value, correlation/trace IDs) via constructor parameters or the `Data` dictionary, so logs and error-tracking tools can act on structured detail instead of parsing a message string.

- **DO:** Use exception filters and pattern matching together (`catch (ApiException ex) when (ex.StatusCode is HttpStatusCode.TooManyRequests)`) to keep specific-case handling readable without a chain of nested conditionals inside a single broad catch block.
- **DON'T:** Design custom exception types with only a parameterless constructor. Follow the standard exception constructor pattern (parameterless, message-only, message-plus-inner-exception, and serialization constructor where applicable) so the type composes correctly with wrapping/rethrowing and with tooling that expects the conventional shape.
  ```csharp
  public class OrderNotFoundException : Exception
  {
      public OrderNotFoundException() { }
      public OrderNotFoundException(string message) : base(message) { }
      public OrderNotFoundException(string message, Exception inner) : base(message, inner) { }
  }
  ```
- **DO:** Validate arguments at the top of public methods with `ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrEmpty` (.NET 7/8+ guard helpers), or explicit checks throwing `ArgumentOutOfRangeException` for invalid ranges — fail immediately and specifically rather than letting a bad argument propagate into a confusing failure deeper in the call stack.
- **DON'T:** Catch `Exception` broadly in a `Main`/hosted-service entry point purely to log and swallow it without exiting or signaling failure. An unhandled startup or fatal-loop exception should generally terminate the process (or restart the specific failed component) with a clear exit code/log entry, not leave the application limping along in an unknown, possibly-corrupted state.
- **DO:** Prefer returning a typed `Result`/`OneOf<TSuccess, TError>`-style object (via a small library, or a simple discriminated-union-style record hierarchy) for expected, frequent failure paths in application/domain services, reserving thrown exceptions for truly exceptional, unrecoverable conditions — this keeps expected error handling explicit in the method signature rather than hidden in a `throws`-equivalent that C# doesn't even have as a compiler-checked construct.

- **DO:** Use `System.Diagnostics.Activity`/`ILogger`'s structured logging (`_logger.LogWarning("Payment declined for order {OrderId}", order.Id)`) with named placeholders rather than string-interpolating values directly into the log message (`$"Payment declined for order {order.Id}"`) — structured placeholders let log aggregation tools query/filter on the actual field values instead of parsing free text.
- **DON'T:** Let an exception's `Data` dictionary or custom properties go unused when they'd carry genuinely useful diagnostic context — a caught exception rethrown with only a generic message and no added context wastes an opportunity to make the eventual triage faster.
- **DO:** Use `AggregateException.Flatten()`/`InnerExceptions` explicitly when handling exceptions from `Task.WhenAll`, since a faulted `Task.WhenAll` surfaces only the *first* faulted task's exception when awaited directly — inspect the `Task`s themselves (or the `AggregateException` from `.Exception`) to see every failure, not just one.
  ```csharp
  var tasks = items.Select(ProcessAsync).ToList();
  try
  {
      await Task.WhenAll(tasks);
  }
  catch
  {
      var failures = tasks.Where(t => t.IsFaulted).Select(t => t.Exception);
      // inspect every failure, not just the one `await` rethrew
  }
  ```
- **DO:** Reserve `finally` blocks for cleanup that must run regardless of success or failure (releasing a lock, resetting state) rather than as a place to put logic that could just run after the `try` block normally when no exception path is actually involved.

## Dependency Injection (Built-in DI Container)

- **DO:** Register services with the lifetime that matches their actual state and thread-safety characteristics: `Transient` for lightweight, stateless services created fresh each time; `Scoped` for per-request state (most commonly, anything wrapping a `DbContext`); `Singleton` for genuinely shared, thread-safe, long-lived state. Picking the wrong lifetime is one of the most common .NET DI bugs.
  ```csharp
  builder.Services.AddScoped<IOrderRepository, OrderRepository>();
  builder.Services.AddSingleton<IClock, SystemClock>();
  builder.Services.AddTransient<IEmailFormatter, EmailFormatter>();
  ```
- **DON'T:** Inject a `Scoped` service (most importantly `DbContext`) into a `Singleton` service's constructor. The container captures the scoped instance at singleton-construction time (or throws a validated-scopes error if scope validation is enabled), meaning every subsequent "request" through that singleton reuses the same, likely-disposed, `DbContext` instance — a frequent source of `ObjectDisposedException` and cross-request data corruption in ASP.NET Core apps.
- **DO:** Enable scope validation in development (`ValidateScopes = true`, on by default for the ASP.NET Core `WebApplicationBuilder` in Development) so a captive-dependency mismatch (injecting scoped into singleton) throws immediately at startup/first-resolve instead of silently misbehaving in production.
- **DON'T:** Resolve services manually via `IServiceProvider.GetService<T>()` scattered through business logic (the Service Locator anti-pattern) instead of using constructor injection. This hides a class's real dependencies from its constructor signature and makes the class harder to test and reason about; reserve manual resolution for framework integration points (factories, middleware) where constructor injection genuinely isn't available.
- **DO:** Use constructor injection with `readonly` fields for required dependencies, making the class's dependency graph fully visible and immutable after construction.
- **DON'T:** Register a service against a concrete class when it should be registered against the abstraction it implements (`AddScoped<OrderService>()` instead of `AddScoped<IOrderService, OrderService>()`), unless the class genuinely has no meaningful interface — registering against the interface is what allows swapping implementations (real vs. test double) without touching consumers.
- **DO:** Use `IOptions<T>`/`IOptionsSnapshot<T>`/`IOptionsMonitor<T>` with strongly-typed configuration classes bound via `services.Configure<T>(configuration.GetSection(...))`, rather than injecting `IConfiguration` directly and pulling string keys out of it throughout business logic — this centralizes the configuration shape in one type and supports validation (`ValidateDataAnnotations()`/`ValidateOnStart()`).
- **DON'T:** Use property injection or a public settable dependency property as the default injection style — it makes a required dependency look optional, and the object can exist in a partially-constructed, invalid state if the setter is never called.
- **DO:** Register `HttpClient` usages through `IHttpClientFactory` (`AddHttpClient<T>()`) rather than constructing `new HttpClient()` directly or holding one as a long-lived field manually. Manually managed `HttpClient` instances are a well-known source of either socket exhaustion (a new instance per request) or stale DNS (one instance held forever); the factory manages the underlying `HttpMessageHandler` pooling correctly.

- **DO:** Use `TryAddXxx` (`TryAddScoped`, `TryAddSingleton`) inside library/extension-method registration code (`AddMyLibraryServices(this IServiceCollection services)`) so consuming applications can override a default registration without a duplicate-registration conflict.
- **DON'T:** Register the same interface multiple times expecting the container to merge or override the earlier registration in a specific, predictable way without checking — the built-in container's resolution behavior for multiple registrations of the same service type (last-registration-wins for a single instance, or "give me all of them" for `IEnumerable<T>`) is documented but easy to get backwards; verify rather than assume.
- **DO:** Use keyed services (`AddKeyedScoped<T>`, .NET 8+) when multiple named implementations of the same interface need to be resolved by an explicit key, rather than working around the lack of keying with a factory-of-factories or a dictionary of delegates wired up manually.
- **DO:** Dispose of `IDisposable` dependencies registered as `Transient` — the container tracks and disposes transient/scoped services it creates at the end of their owning scope, but a service resolved directly from the root provider without an explicit scope can accumulate undisposed instances for the lifetime of the application; always resolve scoped work through a proper `IServiceScope`.
- **DON'T:** Register a service as `Singleton` just because its constructor is expensive to run. Cache the expensive computation inside a properly `Scoped`/`Transient` service (e.g. via `Lazy<T>` or a memoized field) instead of forcing the whole service's lifetime — and its captured dependencies — to singleton scope for an unrelated reason.

- **DO:** Use `services.AddOptions<T>().Bind(...).ValidateDataAnnotations().ValidateOnStart()` so misconfigured options fail the application at startup rather than the first time that specific option is actually read, which could be minutes or hours into runtime.
- **DON'T:** Resolve a `Scoped` service from `IServiceProvider` outside of any request/job scope (e.g. in a singleton's constructor via direct `provider.GetService<T>()`) — this is both the service-locator anti-pattern and, worse, a captive-dependency risk if that scoped service is disposed-sensitive; create an explicit `IServiceScope` via `IServiceScopeFactory` when a singleton genuinely needs scoped work done on demand.
- **DO:** Use the generic `AddHttpClient<TClient, TImplementation>()` overload to bind a typed client class directly to its `HttpClient` configuration (base address, default headers, resilience handlers), keeping HTTP-specific configuration colocated with the client that uses it instead of configured ad hoc at each call site.
- **DO:** Register decorators (cross-cutting wrappers around an existing service, like a caching layer in front of a repository) explicitly via a factory-based registration or a dedicated decoration helper, since the built-in container doesn't provide first-class decorator support the way some third-party containers (Autofac, Scrutor as an add-on) do.

## Entity Framework Pitfalls

- **DON'T:** Access a related entity's collection property inside a loop over the parent entities without eager-loading it first — this is the classic N+1 query problem, where what looks like one query fires one additional round-trip per parent row.
  ```csharp
  // DON'T — N+1: one query per customer to load Orders
  var customers = await _db.Customers.ToListAsync();
  foreach (var customer in customers)
  {
      Console.WriteLine(customer.Orders.Count); // lazy-loads per iteration
  }

  // DO — eager-load in the original query
  var customers = await _db.Customers.Include(c => c.Orders).ToListAsync();
  ```
- **DO:** Use `.Include()`/`.ThenInclude()` deliberately for the specific navigation properties a query actually needs, or project directly with `.Select()` into a DTO that pulls only the required related fields — over-fetching via a blanket `.Include()` on every navigation property pulls unnecessary data across the wire just as under-fetching causes N+1 round-trips.
- **DO:** Use `.AsNoTracking()` for read-only queries (the majority of query traffic in most applications — displaying data, generating reports, read-only API responses). Untracked queries skip Entity Framework's change-tracking overhead entirely, which is measurably faster and lower-memory for data that will never be updated back to the database in this operation.
  ```csharp
  var products = await _db.Products
      .AsNoTracking()
      .Where(p => p.IsActive)
      .ToListAsync();
  ```
- **DON'T:** Use `.AsNoTracking()` on an entity you then intend to modify and call `SaveChangesAsync()` on. An untracked entity's changes won't be picked up automatically by `SaveChangesAsync()` — either keep tracking enabled for entities headed for a write, or explicitly re-attach and mark the entity/entry as modified before saving.
- **DO:** Wrap multi-step writes that must succeed or fail together in an explicit transaction (`_db.Database.BeginTransactionAsync()`, or rely on `SaveChangesAsync()`'s implicit single-call transaction when all changes are tracked on one `DbContext` and one `SaveChangesAsync()` call) rather than assuming multiple separate `SaveChangesAsync()` calls are atomic together — they are not.
- **DON'T:** Call `SaveChangesAsync()` inside a tight loop, once per entity, when the entities could instead all be added/updated on the same `DbContext` and saved with a single `SaveChangesAsync()` call at the end. Batching writes into one `SaveChangesAsync()` call is both faster (fewer round trips within one transaction) and gives you atomicity across the batch.
- **DO:** Be deliberate about `DbContext` lifetime — scoped per request/unit-of-work in most web applications (the default DI registration), and explicitly short-lived in background/batch processing (a fresh `DbContext` per unit of work, not one held for the life of a long-running process), since a `DbContext`'s change tracker grows unbounded and its identity-map/tracking state is not designed for long-lived reuse.
- **DON'T:** Call `.ToList()`/`.ToListAsync()` prematurely and then continue filtering/paging in memory with LINQ-to-Objects — this pulls the entire table (or an unfiltered subset) into application memory before the actual filter/`Skip`/`Take` is applied, instead of letting EF translate the full query (including pagination) into SQL that only returns the needed rows.
- **DO:** Use compiled queries (`EF.CompileAsyncQuery`) or split queries (`.AsSplitQuery()`) for genuinely hot, complex-join, or performance-sensitive query paths after profiling shows they matter — not as a default optimization applied everywhere before there's evidence of a real bottleneck.
- **DON'T:** Ignore the generated SQL entirely. Log EF Core's generated SQL in development (via logging configuration or `.ToQueryString()` on an `IQueryable`) periodically to catch accidental cartesian explosions from multiple `.Include()`s, unexpected client-side evaluation, or a query that doesn't translate the way the LINQ suggests it should.

- **DO:** Use `.ExecuteUpdate()`/`.ExecuteDelete()` (EF Core 7+) for bulk update/delete operations that don't need to load entities into memory at all, instead of loading a full set of tracked entities just to modify or remove them — this both avoids the memory/tracking overhead and executes as a single database statement.
  ```csharp
  // DON'T — loads every matching row into memory first
  var stale = await _db.Sessions.Where(s => s.ExpiresAt < now).ToListAsync();
  _db.Sessions.RemoveRange(stale);
  await _db.SaveChangesAsync();

  // DO — single set-based statement, nothing materialized
  await _db.Sessions.Where(s => s.ExpiresAt < now).ExecuteDeleteAsync();
  ```
- **DON'T:** Rely on client-side evaluation happening silently for a LINQ expression EF Core can't translate to SQL — recent EF Core versions throw at runtime instead of silently falling back to in-memory evaluation (older versions logged a warning and pulled data client-side without telling you clearly). Either way, check that predicates and projections you write are actually translatable rather than assuming any C# expression works inside a `.Where()`.
- **DO:** Use `.Find()`/`FindAsync()` for a simple primary-key lookup on a tracked context — it checks the local change tracker first before hitting the database, which `.Where(x => x.Id == id).FirstOrDefaultAsync()` does not.
- **DO:** Configure explicit indexes via Fluent API/migrations (`.HasIndex(...)`) for columns used in frequent `Where`/`OrderBy`/join predicates — EF Core does not infer indexes from LINQ query patterns, and a missing index on a large table is a common, easily overlooked source of slow queries that only shows up once the table has real production volume.
- **DON'T:** Let migrations accumulate unreviewed, especially ones autogenerated without checking the resulting SQL — always review a generated migration's `Up`/`Down` methods before applying it, since EF's migration generator can produce a column rename as a drop-and-recreate (losing data) if it can't infer the rename was intended.

- **DO:** Use owned entity types/value conversions (`OwnsOne`, `HasConversion`) to map value objects (a `Money` type, an `EmailAddress` type) cleanly into EF Core's model rather than flattening the domain model back into primitive columns just because the ORM historically preferred primitives — modern EF Core has good first-class support for this.
- **DON'T:** Assume optimistic concurrency is automatically handled — without a configured concurrency token (a `RowVersion`/`[Timestamp]` column, or `IsConcurrencyToken()` on a specific property), two overlapping updates to the same row will silently overwrite each other (last-write-wins) rather than raising a `DbUpdateConcurrencyException`, if that's not the intended behavior for the entity in question.
- **DO:** Use `.Where()` predicates that reference only mapped, translatable members and simple expressions — calling an arbitrary local C# method inside a `.Where()` clause against `IQueryable` either fails to translate or forces the whole thing to evaluate client-side; use `EF.Functions` for provider-specific translatable operations (`EF.Functions.Like(...)`) instead.
- **DO:** Set `QuerySplittingBehavior` deliberately for queries with multiple `.Include()`s across collection navigations — the default single-query behavior can produce a cartesian-product explosion in the generated SQL as included collections multiply against each other; `.AsSplitQuery()` issues separate SQL queries per included collection instead, which is often faster for that shape.

## Records & Pattern Matching

- **DO:** Use `record`/`record struct` as the default for immutable data-carrying types, taking advantage of the generated value-based `Equals`/`GetHashCode`/`ToString` and the `with` expression for non-destructive mutation — the same motivation as Java's `record` or Kotlin's `data class`.
  ```csharp
  public record Money(decimal Amount, string Currency);
  var discounted = money with { Amount = money.Amount * 0.9m };
  ```
- **DON'T:** Use a `record` for a type with genuine identity semantics (an EF-tracked entity, anything compared by a stable ID rather than by its full set of field values) — the same caution that applies to data classes in Kotlin and records in Java applies here: value equality and identity equality serve different purposes and shouldn't be conflated.
- **DO:** Use `record struct` (readonly by default when combined with `readonly record struct`) instead of a reference-type `record` for small, frequently-allocated value types (a `Point`, a `Range`) where avoiding heap allocation and enabling copy-by-value semantics genuinely matters for the hot path in question.
- **DO:** Use switch expressions with pattern matching (`type patterns`, `property patterns`, `relational patterns`, `and`/`or` combinators) for exhaustive-feeling multi-branch logic over a closed set of shapes, rather than a chain of `if`/`else if` with manual type casts.
  ```csharp
  string Describe(Shape shape) => shape switch
  {
      Circle { Radius: > 10 } => "large circle",
      Circle => "circle",
      Rectangle { Width: var w, Height: var h } when w == h => "square",
      Rectangle => "rectangle",
      _ => "unknown"
  };
  ```
- **DON'T:** Rely on a `switch` expression's default `_` arm to silently absorb genuinely new cases that should be handled explicitly, when the underlying type set is meant to be closed and exhaustively handled (analogous to avoiding a stray `else` in a sealed-type `when` in Kotlin, or an `else` in a sealed-interface `switch` in Java) — for closed hierarchies, prefer designs where the compiler can actually flag missing cases rather than silently falling through.
- **DO:** Use `is not null`/`is null` pattern checks in preference to `!= null`/`== null` where the codebase has standardized on pattern-matching style, since it composes naturally with other pattern combinators (`is not null and { Count: > 0 }`).

## Span<T>, Memory & Performance-Sensitive Code

- **DO:** Use `Span<T>`/`ReadOnlySpan<T>` for high-throughput, allocation-sensitive code that processes contiguous memory (parsing, string manipulation, buffer processing) since they provide array-like access without the heap allocation of creating new arrays/substrings for every intermediate slice.
  ```csharp
  // DON'T — allocates a new substring
  string trimmed = input.Substring(start, length).Trim();

  // DO — no allocation for the slice itself
  ReadOnlySpan<char> slice = input.AsSpan(start, length).Trim();
  ```
- **DON'T:** Reach for `Span<T>`/`Memory<T>` by default across ordinary application code where allocation overhead isn't actually a measured problem — they come with real constraints (a `Span<T>` can't be stored on the heap, used across `await` boundaries, or captured in a closure), and applying them everywhere adds complexity disproportionate to the benefit for non-hot-path code.
- **DO:** Use `StringBuilder` for building strings through many incremental append operations in a loop, rather than repeated `string` concatenation (`result += chunk`) which allocates a new string object on every single concatenation.
- **DO:** Profile before optimizing for allocation/GC pressure — use a profiler or benchmarking tool (BenchmarkDotNet) to confirm an actual hot path before introducing `Span<T>`, object pooling (`ArrayPool<T>`), or other lower-level performance techniques that trade code simplicity for speed.
- **DON'T:** Assume `struct` is always faster than `class` — a large `struct` copied by value repeatedly (through method calls, into collections) can be slower than a `class` reference due to the copying cost; reserve `struct`/`record struct` for genuinely small, short-lived value types.

## Security & Secrets Management

- **DO:** Use parameterized queries — ADO.NET command parameters, or an ORM's own parameter binding — for every SQL statement built from external input. String interpolation or concatenation into raw SQL text is directly vulnerable to SQL injection, even when a convenient-looking interpolated string (`$"..."`) is used to build the query text.
  ```csharp
  // DON'T — SQL injection vulnerable, even with a "clean-looking" interpolated string
  var sql = $"SELECT * FROM Users WHERE Email = '{email}'";

  // DO
  var users = await _db.Users
      .Where(u => u.Email == email) // EF parameterizes automatically
      .ToListAsync();
  ```
- **DON'T:** Hardcode connection strings, API keys, or signing secrets in `appsettings.json` committed to source control. Use user secrets (`dotnet user-secrets`) for local development, and a proper secret store (Azure Key Vault, AWS Secrets Manager, or environment variables injected at deploy time) for every other environment.
- **DO:** Use the ASP.NET Core Data Protection APIs (`IDataProtector`) for encrypting sensitive data at rest (tokens, cookies) rather than hand-rolling encryption with a manually chosen algorithm and key-management scheme.
- **DON'T:** Disable TLS certificate validation (`ServerCertificateCustomValidationCallback` unconditionally returning `true`) anywhere outside a clearly-scoped, clearly-labeled local development or test configuration. If this reaches production, it silently defeats TLS's protection against man-in-the-middle attacks.
- **DO:** Rely on Razor's automatic HTML output encoding and `[ValidateAntiForgeryToken]` for browser-facing, state-changing endpoints, rather than manually concatenating user input into rendered HTML or manually re-implementing CSRF protection.
- **DO:** Store password hashes using ASP.NET Core Identity's built-in password hasher (or another vetted library implementing bcrypt/Argon2/PBKDF2), never a custom-built hashing scheme.

## Delegates, Events & Functional-Style APIs

- **DO:** Use the built-in generic delegate types `Func<T,TResult>`/`Action<T>`/`Predicate<T>` for simple callback parameters instead of declaring a custom single-method delegate type, reserving a custom delegate name for cases where it genuinely improves readability at call sites for a widely-used, domain-specific callback shape.
- **DON'T:** Use C# events (`event EventHandler<T>`) as a general-purpose callback mechanism for simple one-to-one method invocation. Events are specifically designed for multicast publish/subscribe notification with the standard subscribe/unsubscribe pattern; a plain `Func`/`Action` parameter or property is simpler and more direct for a single callback with one owner.
- **DO:** Check an event for null subscribers before invoking it, using the null-conditional operator (`SomeEvent?.Invoke(this, args)`), to avoid a `NullReferenceException` when no subscribers happen to be attached at invocation time.
  ```csharp
  // DON'T — throws if no one has subscribed
  SomeEvent(this, EventArgs.Empty);

  // DO
  SomeEvent?.Invoke(this, EventArgs.Empty);
  ```
- **DO:** Unsubscribe from an event explicitly (`obj.SomeEvent -= Handler`) once a subscriber's lifetime is shorter than the publisher's. A publisher holding a live reference to a "finished" subscriber through the event delegate chain is a classic, easy-to-miss memory leak in long-lived applications.
- **DO:** Prefer expression-bodied members and local functions for small, single-purpose logic embedded within a method, rather than declaring an unnecessary separate private method that's used exactly once and adds no real reuse value.
- **DON'T:** Assume closures over loop variables behave identically across all C# language versions and loop constructs. Modern C# (5+) captures a `foreach` iteration variable per-iteration correctly, but a `for` loop's index variable is still one shared variable across iterations unless deliberately copied into a fresh per-iteration local — a subtle trap when a closure is created inside a `for` loop and expected to capture "this iteration's" value.

## Generics & Type Constraints

- **DO:** Apply generic constraints (`where T : IComparable<T>`, `where T : class, new()`) to express exactly what a generic type/method actually requires, letting the compiler enforce it at every call site instead of discovering an unsupported type at runtime through a failed cast or a reflection error.
  ```csharp
  public static T Max<T>(T a, T b) where T : IComparable<T> =>
      a.CompareTo(b) >= 0 ? a : b;
  ```
- **DON'T:** Use `object`-typed parameters with runtime type checks and casts as a substitute for genuine generics. This pushes type errors from compile time to runtime, defeating exactly the safety generics were introduced to provide.
- **DO:** Use covariant (`out T`) and contravariant (`in T`) generic interface type parameters where the relationship genuinely fits (`IEnumerable<out T>`, `IComparer<in T>`), enabling more flexible, intuitive assignments between related generic interface instantiations (e.g. assigning an `IEnumerable<Derived>` to an `IEnumerable<Base>` variable).
- **DON'T:** Leave a public generic API parameter unconstrained when the method body immediately casts or type-checks it against a specific requirement anyway. If a constraint is always needed internally, express it explicitly in the type parameter's `where` clause rather than leaving it implicit and undocumented.
- **DO:** Handle `default(T)`/`default` carefully with generic type parameters — it yields `null` for reference types but a genuine zero-valued instance for value types. Code that assumes `default(T) == null` unconditionally breaks silently the moment `T` is instantiated with a value type like `int` or a `struct`.
- **DO:** Prefer generic methods over `object`-based, reflection-heavy solutions for type-safe collection and utility operations, reserving reflection for genuine cross-cutting concerns (serialization internals, a DI container's own implementation) where generics alone can't express the requirement.

## String Handling & Culture-Aware Formatting

- **DO:** Specify an explicit `StringComparison` (`StringComparison.Ordinal`, `.OrdinalIgnoreCase`) for comparisons that aren't meant to be culture-sensitive — identifiers, file paths, protocol strings. Methods like `IndexOf`/`Contains`/`ToUpper` are culture-sensitive by default and can behave unexpectedly across locales (the well-known Turkish-I problem, where `"i".ToUpper()` doesn't produce `"I"` under a Turkish culture).
  ```csharp
  // DON'T — culture-sensitive comparison for what should be an exact identifier match
  if (input.ToUpper() == "ADMIN") { ... }

  // DO
  if (string.Equals(input, "ADMIN", StringComparison.OrdinalIgnoreCase)) { ... }
  ```
- **DON'T:** Call `ToUpper()`/`ToLower()` without a specified culture purely for comparison purposes. Use `ToUpperInvariant()`/`ToLowerInvariant()` if a case-folded string is genuinely needed, or better, skip the allocation entirely with a direct `StringComparison`-based comparison.
- **DO:** Use `CultureInfo.InvariantCulture` explicitly when formatting or parsing values that must round-trip consistently regardless of the running machine's locale — serialized data, log timestamps, configuration values — reserving the current/user culture specifically for user-facing display formatting.
- **DO:** Use raw string literals (`"""..."""`, C# 11+) for readable multi-line or heavily-quote-embedded string content, similar in spirit to Java's text blocks, instead of manually escaped multi-line concatenations full of `\"` and `\n`.
- **DON'T:** Build large strings through repeated `+`/`+=` concatenation inside a loop. Each concatenation allocates an entirely new string; use `StringBuilder` for incremental accumulation instead.

## Collections & Immutability

- **DO:** Expose `IReadOnlyList<T>`/`IReadOnlyCollection<T>`/`IReadOnlyDictionary<TKey,TValue>` from public API return types when callers shouldn't mutate what's returned, rather than a concrete `List<T>`/`Dictionary<TKey,TValue>` that visibly invites mutation via its full public API.
  ```csharp
  // DON'T — invites callers to mutate internal state
  public List<Item> GetItems() => _items;

  // DO
  public IReadOnlyList<Item> GetItems() => _items.AsReadOnly();
  ```
- **DON'T:** Assume `IReadOnlyList<T>` guarantees true structural immutability. It only hides mutation methods from that specific reference — if the underlying object is actually a `List<T>`, code holding a reference to the concrete type can still mutate it, and that change is visible through the "read-only" view too. Use `ImmutableList<T>`/`ImmutableArray<T>` (`System.Collections.Immutable`) when genuine, structural immutability is required.
- **DO:** Use `System.Collections.Immutable` types for shared state that must be safely readable from multiple threads without locking — their structural-sharing implementation means "modifying" one always produces a new instance rather than mutating the original, which is what makes them safe to share without synchronization.
- **DON'T:** Return `null` from a method whose declared return type is a collection type. Return `Array.Empty<T>()`/`Enumerable.Empty<T>()`/an empty collection instance instead, so callers don't need a defensive null check before iterating.
- **DO:** Use collection expressions (`[1, 2, 3]`, C# 12+) for concise collection-literal initialization where the target type supports them, in preference to more verbose `new List<int> { 1, 2, 3 }` construction, once the codebase has consistently adopted the newer syntax.

## Enums & Flags

- **DO:** Use `[Flags]` enums with explicit power-of-two values for bitwise-combinable option sets, combining and checking them with bitwise operators (`|`, `&`) or `HasFlag`, rather than modeling a set of combinable options as a series of separate boolean parameters.
  ```csharp
  [Flags]
  public enum FilePermissions { None = 0, Read = 1, Write = 2, Execute = 4 }

  var perms = FilePermissions.Read | FilePermissions.Write;
  bool canWrite = perms.HasFlag(FilePermissions.Write);
  ```
- **DON'T:** Depend on a non-`[Flags]` enum's implicit underlying integer value for anything persisted or serialized externally without pinning explicit values. Like Java's `ordinal()` trap, an enum member's implicit numeric value shifts if members are reordered or a new one is inserted between existing ones, silently corrupting anything that depended on the old numbering.
- **DO:** Assign explicit integer values to every enum member whose numeric representation is persisted or serialized externally, rather than relying on the compiler's implicit `0, 1, 2...` assignment, so reordering the enum's declaration in source can never silently change a stored meaning.
- **DON'T:** Use `Enum.Parse`/`Enum.TryParse` on user-supplied strings without validating that the resulting value is actually a defined member. C# enums aren't a fully closed type at the storage level — any value of the underlying integer type is technically assignable to the enum type — so downstream code that assumes every enum value it sees is one of the "known" members can be caught off guard by an unvalidated, out-of-range value.
- **DO:** Use pattern-matching `switch` expressions over enum values for exhaustive-feeling handling, and give the `default`/discard arm behavior that throws (`ArgumentOutOfRangeException`, via `_ => throw new ArgumentOutOfRangeException(nameof(status))`) rather than silently no-op-ing, so an unexpected or invalid enum value surfaces loudly instead of being quietly ignored.

## Testing (xUnit & NUnit)

- **DO:** Use `[Theory]`/`[InlineData]` (xUnit) or `[TestCase]` (NUnit) for parameterized tests covering multiple input variations with one test method, instead of copy-pasted near-identical `[Fact]`/`[Test]` methods that differ only in their literal values.
  ```csharp
  [Theory]
  [InlineData(0, false)]
  [InlineData(-1, false)]
  [InlineData(1, true)]
  public void IsPositive_ReturnsExpected(int input, bool expected)
  {
      Assert.Equal(expected, MathUtils.IsPositive(input));
  }
  ```
- **DO:** Name test methods to describe the scenario and expected outcome (`Withdraw_WhenBalanceInsufficient_ThrowsInsufficientFundsException`), following a consistent team convention (Given-When-Then, or Method_Scenario_Expected), so a failing test's name alone communicates what broke.
- **DON'T:** Share mutable state across test methods via a shared instance field without understanding the test framework's instantiation model — xUnit creates a new test class instance per test method by default (so instance fields are naturally isolated), while NUnit's default behavior differs by attribute configuration; know the framework's isolation model rather than assuming state resets or persists.
- **DO:** Use `IClassFixture<T>`/`ICollectionFixture<T>` (xUnit) or `[OneTimeSetUp]` (NUnit) for genuinely expensive shared setup (e.g. a Testcontainers-backed database) that should be created once per test class/collection rather than per test, while keeping per-test state properly isolated.
- **DON'T:** Mock `DbContext` directly for testing query logic. Mocking `DbSet<T>`/`DbContext` is brittle and doesn't exercise real LINQ-to-Entities translation; use the EF Core In-Memory provider for simple cases, or better, a real database via Testcontainers for anything where SQL translation behavior actually matters (since the in-memory provider doesn't enforce relational constraints or translate LINQ the same way a real provider does).
- **DO:** Use a mocking library (Moq, NSubstitute) to isolate the unit under test from its actual collaborators (external services, repositories) at genuine architectural boundaries, and prefer constructing real, simple objects over mocking value types or DTOs.
- **DON'T:** Assert only that an async method "completed without throwing." Await the result and assert on the actual returned value/state, and use `Assert.ThrowsAsync<T>` (not synchronous `Assert.Throws` wrapping a `.Result` call) for testing that an async method throws the expected exception.
  ```csharp
  // DON'T
  await Assert.ThrowsAsync<Exception>(() => Task.FromResult(service.DoWork()));

  // DO
  await Assert.ThrowsAsync<InvalidOperationException>(() => service.DoWorkAsync());
  ```
- **DO:** Use FluentAssertions (or the built-in assertion library's fluent equivalents) for readable, descriptive failure messages (`result.Should().BeEquivalentTo(expected)`) over terse `Assert.Equal` calls, especially when asserting on complex object graphs.
- **DON'T:** Let integration tests depend on shared, mutable external state (a shared dev database, a shared test tenant) that other tests or CI runs might concurrently modify. Use isolated, disposable infrastructure per test run (Testcontainers, a per-test-run database schema, `WebApplicationFactory` with an isolated in-memory/test configuration) so tests are reliably repeatable.
- **DO:** Test cancellation behavior explicitly for methods that accept a `CancellationToken` — pass an already-cancelled token and assert the method throws/handles `OperationCanceledException` correctly, since this path is easy to implement incorrectly and easy to forget to test.

- **DO:** Use `WebApplicationFactory<TEntryPoint>` for integration-testing ASP.NET Core applications end-to-end through the real HTTP pipeline (middleware, routing, model binding included), overriding only the specific services (like the database) that need to be swapped for test doubles via `WithWebHostBuilder`.
- **DON'T:** Test private methods directly via reflection as a substitute for testing the public behavior that exercises them. If a private method's logic is complex enough to need its own dedicated tests, that's usually a sign it should be extracted into its own separately-testable, public class rather than reached into via reflection.
- **DO:** Use `[SetUp]`/`[TearDown]` (NUnit) or constructor/`IDisposable.Dispose` (xUnit, which uses the class constructor and `Dispose` as its per-test setup/teardown) consistently for per-test resource management, understanding that xUnit deliberately has no separate `[SetUp]` attribute — the constructor is the setup method by design.
- **DO:** Assert exception messages/properties, not just the exception type, when the message or a custom exception's properties carry information the test is specifically meant to verify (e.g. which field failed validation) — asserting only the type can let a test pass even when the wrong validation failure is reported.

- **DO:** Use `[Theory]`/`[TestCase]` with `MemberData`/`TestCaseSource` for parameterized cases whose inputs are complex objects that can't be expressed as attribute-literal values, rather than forcing complex test data through `[InlineData]`'s limited literal-value constraints.
- **DON'T:** Let test project references leak into the production/shipped project (a testing library accidentally referenced from a non-test `.csproj`) — keep test-only dependencies scoped to test projects so they never ship as part of the deployed application.
- **DO:** Use `AutoFixture`/`Bogus` (or an equivalent test-data-generation library already adopted by the project) for generating realistic-but-arbitrary test data for properties the test doesn't specifically care about, keeping the test's Arrange section focused on the values that actually matter to the assertion.
- **DON'T:** Write a test whose name and Arrange section don't actually match what the Assert verifies — a stale test that was copy-pasted and modified but never fully updated is worse than no test, because it gives false confidence that a scenario is covered when it isn't.

## Common AI-Assistant Mistakes in C#/.NET

- **DON'T:** Generate code that blocks on async methods with `.Result` or `.Wait()` from within otherwise-synchronous-looking sample code, especially inside ASP.NET controllers or other contexts with a synchronization context. This is one of the most common and most damaging generated-code mistakes in C# — it compiles fine and often "works" in quick manual testing, then deadlocks under real request load.
- **DON'T:** Generate `async void` methods for anything other than a genuine UI event handler. This routinely shows up in generated examples for what should be `async Task` methods, silently discarding the ability to await or catch exceptions from that method.
- **DON'T:** Invent NuGet package names, namespaces, or API members that sound plausible but don't exist (a hallucinated overload of `HttpClient.GetAsync` that takes options that were never added, a nonexistent `Microsoft.Extensions.SomePlausibleSoundingPackage`, or an EF Core method that doesn't exist in the version actually referenced by the project). Confirm the package and its exact API surface — including the target framework/EF Core version's actual capabilities — before generating code that depends on it.
- **DON'T:** Ignore the project's actual target framework and language version when suggesting newer syntax (primary constructors, collection expressions `[1, 2, 3]`, required members, raw string literals) — verify the `.csproj`'s `<TargetFramework>`/`<LangVersion>` rather than assuming the latest C# features are available.
- **DON'T:** Generate a class that implements `IDisposable` (or wraps something that does — a `Stream`, `SqlConnection`, `HttpClient` held as a field) without actually implementing the dispose pattern correctly, or without wrapping its usage in `using`/`await using`. A generated example that new`s up a disposable resource with no `using` statement and no call to `Dispose()` is a resource leak the moment it's copied into real, longer-lived code.
  ```csharp
  // DON'T
  var connection = new SqlConnection(connectionString);
  connection.Open();
  // ... use connection, never disposed

  // DO
  await using var connection = new SqlConnection(connectionString);
  await connection.OpenAsync();
  ```
- **DON'T:** Suppress every nullable-reference-type warning with `!` as a blanket fix instead of addressing the actual nullability the compiler is flagging. This defeats a feature that's specifically meant to be enabled and heeded, not enabled and immediately worked around.
- **DON'T:** Register every service as `Singleton` in generated DI setup code "for performance" without considering the lifetime semantics of what's being registered — this is a common shortcut that creates captive-dependency bugs the moment a singleton-registered service depends on something scoped, like a `DbContext`.
- **DON'T:** Generate LINQ-to-Entities queries and then apply `.ToList()` immediately, followed by further `.Where()`/`.OrderBy()`/pagination on the in-memory list — a subtle but common mistake that silently changes a query from "translated to efficient SQL with only needed rows returned" to "entire table pulled into memory, then filtered in .NET."
- **DON'T:** Default to `Newtonsoft.Json` (`Json.NET`) attributes and calls in a project that has standardized on `System.Text.Json`, or vice versa, without checking which serializer the project actually references — mixing the two introduces inconsistent attribute behavior and, in the worst case, two different serialization results for the same type depending on which path handles it.
- **DON'T:** Generate exception handling that catches `Exception` broadly and returns a generic 500-style response for every case, discarding the distinction between validation failures, not-found conditions, and genuine server errors. Match HTTP status codes and error shapes to the specific exception/failure type rather than collapsing everything into one generic catch-all handler.
- **DON'T:** Generate a fire-and-forget `Task.Run(() => DoWorkAsync())` inside a request-handling method without awaiting or otherwise tracking it, presenting it as if the work completes within the request. In ASP.NET Core, the request can complete (and its `DI` scope get disposed, including any `DbContext`) while the detached task is still running, leading to `ObjectDisposedException`s or work that silently never finishes; use a proper background task queue (`IHostedService`, a channel-backed background worker) for genuine fire-and-forget work.
- **DON'T:** Assume a generic type or LINQ method exists on `IEnumerable<T>` when it's actually specific to a different interface (e.g., assuming `IQueryable`-only methods like certain provider-specific translation helpers work identically against plain in-memory `IEnumerable<T>`, or vice versa) — verify which interface a given LINQ operator or provider extension actually targets before relying on it.
- **DON'T:** Generate C# that ignores an already-established project pattern for cross-cutting concerns — for instance, writing manual try/catch response-formatting in a new controller action when the project already has centralized exception-handling middleware, or manually validating input when the project already uses `FluentValidation`/data annotations consistently elsewhere. Match established patterns rather than introducing a one-off alternative.
- **DON'T:** Generate a synchronous wrapper method around an async API purely to make it callable from older, synchronous-looking calling code (`public T DoWork() => DoWorkAsync().Result;`). This just relocates the blocking-on-async deadlock risk into a helper method instead of removing it — propagate `async` upward through the call chain instead of manufacturing a sync-over-async shim.
- **DON'T:** Assume a record type's positional equality/`ToString()` behavior applies to a `class` that merely looks similar, or vice versa — generate the specific construct (`record`, `record struct`, `class`, `struct`) that matches the value/identity semantics actually needed, rather than picking whichever is most familiar from recent context.
- **DON'T:** Skip disposing of a `CancellationTokenSource` created locally within a method once its token is no longer needed — like any other `IDisposable`, an undisposed `CancellationTokenSource` leaks the underlying `Timer`/registration resources if `CancelAfter` or a linked-token pattern was used.

## Quick Checklist
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
