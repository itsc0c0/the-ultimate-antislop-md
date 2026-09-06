# PHP

## Modern PHP Idioms (PHP 8+, Typed Properties, Avoiding Legacy Patterns)

- **DO:** Run a static analyzer (PHPStan or Psalm) at a meaningful strictness level in CI, since PHP's type system — even with native types everywhere — still leaves gaps (untyped array shapes, generics-like patterns PHP doesn't natively express) that a static analyzer's more thorough type inference catches that the runtime type checker alone won't.
- **DO:** Use PHPDoc array-shape and generic-like annotations (`@param array{id: int, name: string} $user`, `@return Collection<int, User>`) specifically to give static analyzers extra precision PHP's native type system can't express on its own, treating PHPDoc as a supplement to native types for exactly these gaps, not a replacement for them.
- **DO:** Declare types everywhere PHP 8+ allows them — parameter types, return types, and typed properties — instead of relying on PHPDoc comments or no typing at all. Native types are enforced at runtime, checked by static analyzers (PHPStan, Psalm), and understood by every modern IDE, where a PHPDoc-only annotation is just an unenforced comment.
  ```php
  // BAD — PHP 5-era: no types anywhere, easy to pass the wrong thing
  class Order {
      public $total;
      function addItem($item, $qty) { ... }
  }

  // GOOD — PHP 8+: types are enforced, self-documenting, and IDE-checkable
  class Order {
      public float $total = 0.0;
      public function addItem(Item $item, int $qty): void { ... }
  }
  ```
- **DO:** Use constructor property promotion (PHP 8.0+) to collapse a typed-property-plus-constructor-assignment pair into one parameter declaration, cutting boilerplate on every value object and DTO.
  ```php
  // BAD — repeats each property three times
  class Point {
      public float $x;
      public float $y;
      public function __construct(float $x, float $y) {
          $this->x = $x;
          $this->y = $y;
      }
  }

  // GOOD — promoted properties, one declaration each
  class Point {
      public function __construct(
          public readonly float $x,
          public readonly float $y,
      ) {}
  }
  ```
- **DO:** Mark properties `readonly` (PHP 8.1+) when a value object or DTO's state should never change after construction, letting the engine enforce immutability instead of relying on convention and code review to catch an accidental later mutation.
- **DO:** Use enums (PHP 8.1+, `enum Status: string { case Draft = 'draft'; case Published = 'published'; }`) instead of class constants or bare strings for a fixed, closed set of values — enums give exhaustiveness in `match` expressions and prevent an invalid value from ever being constructed.
- **DO:** Use `match` expressions instead of `switch` statements for value-producing branches — `match` uses strict comparison by default (no accidental type-juggling fallthrough), has no fallthrough between arms, and is itself an expression, removing the need for a temporary variable assigned inside each `case`.
  ```php
  // BAD — switch allows loose comparison and silent fallthrough bugs
  switch ($status) {
      case 'draft': $label = 'Draft'; break;
      case 'published': $label = 'Live'; break;
      default: $label = 'Unknown';
  }

  // GOOD — match is strict, exhaustive-checked, and an expression
  $label = match ($status) {
      'draft' => 'Draft',
      'published' => 'Live',
      default => 'Unknown',
  };
  ```
- **DO:** Use nullsafe method chaining (`$user?->address?->city`, PHP 8.0+) instead of nested `isset()`/`??`-guarded chains when traversing a sequence of possibly-null objects.
- **DO:** Use named arguments (PHP 8.0+) when calling a function with several optional or boolean parameters, so the call site documents what each value means instead of relying on positional order.
  ```php
  // BAD — unclear what true/false mean without checking the signature
  createUser('Alice', 'a@example.com', true, false);

  // GOOD — self-documenting at the call site
  createUser(name: 'Alice', email: 'a@example.com', isAdmin: true, sendWelcomeEmail: false);
  ```
- **DON'T:** Write PHP 5-era patterns in new code: untyped properties accessed as loose associative-array-like bags, `mysql_*` functions (removed entirely since PHP 7), string-based class instantiation without validation, or manual `array()` instead of the short `[]` literal syntax.
- **DON'T:** Use loose comparison (`==`) where strict comparison (`===`) is intended, especially when comparing against `0`, `""`, `null`, or `false` — PHP's type-juggling rules for `==` have well-documented surprises (e.g., certain numeric-looking strings compare equal to `0`), and `===` avoids the whole class of bugs.
  ```php
  // BAD — "abc" == 0 was true under PHP's old loose-comparison rules
  // for numeric-string coercion, and other == surprises still exist
  if ($userInput == 0) { ... }

  // GOOD — strict comparison means what it says
  if ($userInput === 0) { ... }
  ```
- **DO:** Use the null coalescing assignment operator (`$data['key'] ??= 'default';`) to assign a default only when a variable/array key is unset or null, instead of a verbose `isset()`-guarded if statement.
- **DO:** Prefer first-class callable syntax (`strlen(...)`, PHP 8.1+) over string-based (`'strlen'`) or array-based (`[$this, 'method']`) callables when passing an existing function/method as a callback, since it's checked at compile time and works cleanly with static analysis.
- **DON'T:** Suppress deprecation and type-related warnings with `@` or a blanket error-reporting downgrade instead of fixing the underlying legacy pattern — a warning about a deprecated function or an implicit nullable parameter is telling you exactly what to modernize.
- **DO:** Use union types (`int|string $id`) and intersection types (PHP 8.1+, `Countable&Iterator $items`) when a parameter or return value genuinely accepts more than one concrete type, instead of falling back to no type declaration at all or a vague `mixed`.
  ```php
  // BAD — no type at all, the widest possible surface for misuse
  function findUser($id) { ... }

  // GOOD — precisely states the two accepted shapes
  function findUser(int|string $id): ?User { ... }
  ```
- **DO:** Use `never` as a return type (PHP 8.1+) for a function that always throws or otherwise never returns control to its caller (a helper that always throws a specific validation exception), so static analyzers understand code after the call is genuinely unreachable.
- **DO:** Use readonly promoted properties together with a private constructor and a named static factory method for value objects that need validation at construction time, since promoted constructor parameters alone can't run arbitrary validation logic before assignment.
  ```php
  final class EmailAddress
  {
      private function __construct(public readonly string $value) {}

      public static function fromString(string $value): self
      {
          if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
              throw new InvalidArgumentException("Invalid email: {$value}");
          }
          return new self($value);
      }
  }
  ```
- **DO:** Use `#[Attribute]`-based PHP attributes (PHP 8.0+) instead of magic PHPDoc annotation comments (`@Route`, `@ORM\Column`) when the framework/library in use supports attribute-based configuration — attributes are parsed and validated by the engine itself, unlike PHPDoc annotations, which are just comments a library has to parse with its own regex-based tooling.
- **DON'T:** Use `mixed` as a type hint as a way to avoid deciding on a real type. `mixed` is occasionally correct (a function that genuinely accepts anything, like a generic cache setter), but reaching for it as a default because "figuring out the real type is annoying" defeats the entire purpose of adopting typed PHP.
- **DO:** Use the `??=`, `?->`, and spread operator (`...$args`) together where they genuinely simplify code, but don't chain so many of PHP 8's newer operators into one line that the expression becomes hard to parse visually — readability still outranks terseness.
- **DO:** Use `array_map`/`array_filter`/`array_reduce` for straightforward collection transformations instead of a manual `foreach` loop building up a result array by hand, when the transformation is simple enough that the functional form stays readable.
  ```php
  // BAD — manual accumulator loop for a simple transformation
  $activeNames = [];
  foreach ($users as $user) {
      if ($user->isActive()) {
          $activeNames[] = $user->getName();
      }
  }

  // GOOD — expresses the transformation directly
  $activeNames = array_map(
      fn(User $u) => $u->getName(),
      array_filter($users, fn(User $u) => $u->isActive())
  );
  ```
- **DO:** Use arrow functions (`fn($x) => $x * 2`, PHP 7.4+) for short, single-expression closures that only need to read (not modify) variables from the enclosing scope — they automatically capture the enclosing scope by value, removing the need for an explicit `use (...)` clause that a regular closure would require for the same variables.
- **DON'T:** Reach for `list()`/`[$a, $b] = $array` array destructuring in a way that silently discards or misassigns values when the source array's shape isn't actually guaranteed — validate the array's shape first, or use named keys in the destructuring (`['id' => $id, 'name' => $name] = $data`) to make the expected shape explicit and catch a missing key as a warning rather than silently assigning `null`.
- **DO:** Use `static` return type (PHP 8.0+, distinct from `self`) on factory/fluent-interface methods so a subclass calling the inherited method gets back an instance of the subclass, not hardcoded to the base class — the same motivation as Objective-C's `instancetype`.
  ```php
  class QueryBuilder
  {
      public function where(string $condition): static
      {
          $this->conditions[] = $condition;
          return $this;
      }
  }
  ```
- **DO:** Use the `WeakMap` class (PHP 8.0+) when associating auxiliary data with objects without preventing those objects from being garbage collected — a plain array/hash map keyed by object would keep every such object alive indefinitely, which is exactly the kind of leak `WeakMap` is designed to avoid.

## PSR Standards (PSR-12 Style, Autoloading)

- **DO:** Format code to PSR-12 (the current PHP-FIG coding style standard): 4-space indentation, opening braces for classes/methods on their own line, opening braces for control structures on the same line, one blank line after the `namespace` declaration, and a single blank line between `use` import groups.
- **DO:** Run an automated formatter (`php-cs-fixer` or `phpcs`/`phpcbf` with a PSR-12 ruleset) as part of the development workflow and CI, rather than relying on manual formatting discipline across contributors.
- **DO:** Use PSR-4 autoloading (declared in `composer.json`'s `autoload` section, mapping a namespace prefix to a source directory) so class names and file paths are structurally linked, and Composer's generated autoloader can resolve any class without a manual `require`/`include`.
  ```json
  {
      "autoload": {
          "psr-4": { "App\\": "src/" }
      }
  }
  ```
- **DON'T:** Use manual `require`/`include` statements to pull in class files in a Composer-managed project. This bypasses the autoloader, makes load order a manual concern again, and reintroduces exactly the fragility PSR-4 autoloading was designed to remove.
- **DO:** Name one class, interface, trait, or enum per file, with the file name matching the type name exactly (`UserRepository.php` contains `class UserRepository`), as PSR-4 autoloading requires this mapping to resolve classes automatically.
- **DO:** Group and order `use` import statements consistently (commonly: PHP built-ins, then framework/vendor classes, then application classes, each group alphabetized) and let a formatter enforce the ordering rather than doing it by hand.
- **DO:** Declare `declare(strict_types=1);` as the very first statement in every new PHP file, so type coercion on scalar type declarations is strict rather than PHP's default weak/coercive mode — this turns a class of "wrong type silently coerced" bugs into an immediate, loud `TypeError`.
- **DON'T:** Mix tabs and spaces, or use inconsistent indentation width, within the same file or project — PSR-12 mandates 4-space indentation with no tabs, and inconsistent whitespace produces noisy diffs that obscure real changes in code review.
- **DO:** Follow PSR-12's line-length guidance (a soft limit around 120 characters, with lines kept well under that where practical) and its rule that each statement ends on its own line — a formatter enforces both automatically, but understanding the rule helps when reviewing a formatter's diff.
  ```php
  // PSR-12: braces for control structures on the same line as the keyword,
  // one space before the opening brace, closing brace aligned with the
  // start of the control structure's line.
  if ($condition) {
      doSomething();
  } elseif ($otherCondition) {
      doSomethingElse();
  } else {
      doDefault();
  }
  ```
- **DO:** Order class members in a consistent sequence (constants, then properties, then constructor, then public methods, then protected, then private) as PSR-12 recommends, so any class in the codebase can be scanned the same way regardless of who wrote it.
- **DO:** Configure the autoloader's classmap-optimized mode (`composer dump-autoload --optimize` or `--classmap-authoritative`) for production deployments, since PSR-4's default autoloader does a small amount of filesystem-probing per class resolution that a precomputed classmap skips entirely — this is a deployment-time optimization on top of PSR-4, not a replacement for it during development.
- **DON'T:** Define more than one namespace per file, or place code outside the declared namespace's expected directory structure — this breaks the direct mapping PSR-4 autoloading depends on and can produce confusing "class not found" errors that have nothing to do with the class actually being missing.
- **DO:** Use `composer.json`'s `autoload-dev` section for test-only namespaces (mapping a `tests/` directory to a `Tests\` namespace) so test helper classes autoload correctly without being included in the production autoload map.
- **DO:** Build against PSR-7 (HTTP message interfaces) and PSR-15 (HTTP server request handlers/middleware) interfaces when writing framework-agnostic HTTP-handling code, rather than a framework's own concrete request/response classes, so the same middleware/handler code can be reused across any PSR-7/PSR-15-compliant framework instead of being locked to one.
  ```php
  // A PSR-15 middleware depends only on the interfaces, not a specific framework
  final class RequestIdMiddleware implements MiddlewareInterface
  {
      public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
      {
          return $handler->handle($request->withAttribute('request_id', bin2hex(random_bytes(8))));
      }
  }
  ```
- **DO:** Use a PSR-11-compliant dependency injection container (Symfony's DI container, PHP-DI, or a framework's built-in service container) for wiring up application services, rather than manually instantiating a deep object graph at every entry point — a container centralizes construction logic and makes swapping an implementation (a fake for testing, a different provider in production) a configuration change rather than a code change scattered across call sites.
- **DO:** Use a templating engine with automatic output escaping (Twig, Blade) for HTML rendering rather than raw PHP templates with manual `htmlspecialchars()` calls, when a project's scale justifies the dependency — automatic escaping removes an entire class of XSS bugs caused by a single forgotten manual escape call.

## Error Handling (Exceptions vs. Old-Style `@` Suppression)

- **DO:** Throw specific, custom exception classes (extending `Exception` or a more specific built-in like `InvalidArgumentException`, `RuntimeException`, `DomainException`) for recoverable domain failures, rather than returning `false`/`null`/a magic sentinel value on failure the way older PHP APIs (and PHP 5-era generated code) often did.
  ```php
  // BAD — PHP 5-era: caller must remember to check a magic return value
  function findUser(int $id) {
      $row = $db->query(...);
      return $row ?: false;
  }

  // GOOD — failure is explicit and can't be silently ignored
  function findUser(int $id): User {
      $row = $db->query(...);
      if (!$row) {
          throw new UserNotFoundException("No user with id {$id}");
      }
      return User::fromRow($row);
  }
  ```
- **DON'T:** Suppress errors with the `@` operator (`@file_get_contents($path)`, `@$array['missing_key']`) to silence a warning instead of fixing or explicitly handling the underlying condition. `@` suppresses *all* errors on that expression, including ones you didn't anticipate, and it makes debugging a production failure much harder since the warning never reaches the error log.
- **DO:** Check preconditions explicitly instead of suppressing the resulting warning — `file_exists()`/`is_readable()` before `file_get_contents()`, `array_key_exists()`/`isset()` before array access, `is_numeric()` before a numeric cast — so failure is handled deliberately rather than muted.
- **DO:** Use `try`/`catch`/`finally` for genuinely exceptional, recoverable conditions, catching the most specific exception type that's expected and letting unexpected exception types propagate rather than being caught by an overly broad `catch (\Throwable $e)` that hides bugs.
- **DON'T:** Catch `\Throwable` or the base `\Exception` broadly and continue execution silently (an empty `catch` block, or one that only logs at debug level and swallows the failure). At minimum log at an appropriate severity; for user-facing code, surface a meaningful error response.
- **DO:** Configure PHP's error reporting to convert warnings and notices into visible signals during development (`error_reporting(E_ALL)`, displaying errors in dev environments) rather than a production-style configuration that hides them — PHP 5-era code often ran with notices/warnings suppressed by default, which let real bugs go unnoticed for years.
- **DO:** Use a centralized error/exception handler (framework-provided, or a custom `set_exception_handler`) to log uncaught exceptions with full context (stack trace, request data) and return an appropriate response, rather than letting uncaught exceptions leak raw stack traces to end users in production.
- **DO:** Validate and type-check external input (request data, file contents, API responses) at the boundary where it enters the system, throwing a clear validation exception immediately, rather than letting malformed data propagate deep into business logic before it causes an unrelated-looking failure.
- **DO:** Build a small hierarchy of custom exceptions rooted in a common application-specific base exception (`abstract class AppException extends \Exception {}`), so a top-level error handler can catch every application-defined failure mode in one place while unexpected `\Error`/built-in exception types still surface distinctly.
  ```php
  abstract class AppException extends \Exception {}
  final class ValidationException extends AppException {}
  final class NotFoundException extends AppException {}
  ```
- **DO:** Distinguish PHP's `\Error` hierarchy (`TypeError`, `ArgumentCountError`, `DivisionByZeroError`) from the `\Exception` hierarchy when deciding what to catch — `\Error` generally represents a programming mistake (a type violation, a call with the wrong argument count) rather than a recoverable runtime condition, so catching it broadly to "keep the app running" often just hides a bug that should have failed loudly during development/testing.
- **DON'T:** Return a mixed-type value (sometimes an object, sometimes `false`, sometimes `null`) from a function meant to signal success/failure, especially now that PHP has union return types and exceptions available — a caller has to remember and correctly check every distinct sentinel value, which is exactly the ambiguity typed exceptions and `?Type` return types were introduced to remove.
- **DO:** Use PHP's finally block (`try { ... } catch (...) { ... } finally { ... }`) for cleanup that must run whether or not an exception occurred, mirroring `defer`/`ensure` in other languages, rather than duplicating cleanup code in both the success path and every `catch` block.

## Composer & Dependency Management

- **DO:** Declare every direct dependency explicitly in `composer.json` with a reasonable version constraint (commonly a caret constraint like `^8.2` for a library, pinning to a compatible-release range) rather than relying on a dependency that happens to be pulled in transitively by another package.
- **DO:** Commit `composer.lock` to version control for applications (not for libraries meant to be installed by others), so every environment — development, CI, production — installs the exact same resolved dependency versions instead of whatever the constraint ranges happen to resolve to at install time.
- **DON'T:** Run `composer update` casually in a way that silently bumps transitive dependencies across a wide version range in a routine change — treat dependency updates as their own deliberate, reviewed change, ideally with `composer.lock`'s diff checked for what actually moved.
- **DO:** Separate development-only tooling (PHPUnit, PHPStan, php-cs-fixer) into `require-dev` rather than `require`, so a production install (`composer install --no-dev`) doesn't pull in testing and static-analysis tooling it doesn't need.
- **DO:** Run `composer audit` (or an equivalent dependency vulnerability scanner) as part of CI to catch known-vulnerable dependency versions before they ship, rather than discovering them after the fact.
- **DON'T:** Hand-edit the autoloader's generated files under `vendor/` — they're regenerated by Composer and any manual edit is silently discarded on the next `composer install`/`dump-autoload`. Fix the underlying `composer.json` autoload configuration instead.
- **DO:** Use semantic versioning constraints deliberately — `^` to allow non-breaking updates per semver, `~` for a narrower patch-level range, an exact pin only when a specific known-good version is required — and understand which one a given `composer.json` entry actually expresses before changing it.
- **DO:** Declare a `"php"` platform requirement in `composer.json` (`"require": { "php": "^8.2" }`) matching the actual minimum supported PHP version, so Composer refuses to install the project's dependencies on an incompatible PHP version rather than failing confusingly at runtime.
- **DO:** Use `composer why <package>` / `composer why-not <package> <version>` to understand why a given dependency (often transitive) is present, or why a version bump is blocked, instead of guessing from `composer.json` alone which top-level requirement pulled it in.
- **DON'T:** Add a dependency for functionality that a few lines of standard-library code would cover just as well. Every dependency is an ongoing maintenance and security-surface cost; weigh a new package against its actual footprint in the codebase before adding it.
- **DO:** Prefer packages that follow semantic versioning and have an actively maintained release history over unmaintained or pre-1.0 packages for anything load-bearing in production, and check a package's actual maintenance status before depending on it for new code.
- **DO:** Use Composer's `scripts` section to wire up common development tasks (running tests, linting, static analysis) as `composer test`/`composer lint` entries, so contributors and CI invoke the same commands consistently rather than each remembering slightly different raw tool invocations.
  ```json
  {
      "scripts": {
          "test": "phpunit",
          "lint": "phpcs --standard=PSR12 src/"
      }
  }
  ```
- **DO:** Use `composer.json`'s `conflict` and `replace` sections deliberately when a package genuinely can't coexist with another, or replaces an older package's role, rather than leaving such incompatibilities to be discovered only when Composer's solver fails at install time with a confusing error.
- **DON'T:** Install a package globally (`composer global require`) for something a specific project actually depends on to build/run — global installs aren't tracked in the project's own `composer.json`/`composer.lock`, so a teammate or CI runner without that global package installed hits a "command not found" that has nothing to do with the project's own declared dependencies.

## SQL Injection Prevention (Prepared Statements)

- **DO:** Use prepared statements with bound parameters (via PDO or `mysqli`) for every SQL query that includes any external or user-influenced value, with no exceptions — this is not a "best practice," it is the baseline requirement for safe database access in PHP.
  ```php
  // BAD — string-concatenated SQL: a classic injection vulnerability
  $result = $pdo->query("SELECT * FROM users WHERE email = '{$email}'");

  // GOOD — bound parameter, the driver handles escaping/quoting
  $stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
  $stmt->execute(['email' => $email]);
  ```
- **DON'T:** Build SQL by concatenating or interpolating variables directly into the query string, even after "sanitizing" with `addslashes()` or a hand-rolled escaping function. `addslashes()` is not context-aware SQL escaping and has known bypasses in some encodings/configurations; prepared statements with real parameter binding are the only reliably safe approach.
- **DO:** Use PDO with `PDO::ATTR_EMULATE_PREPARES` disabled (`$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false)`) when possible, so parameter binding is handled by the actual database driver's native prepared-statement protocol rather than PDO's client-side emulation, which offers a weaker safety guarantee in edge cases.
- **DO:** Parameterize dynamic values (user input, computed data) but never parameterize structural SQL — table names, column names, `ORDER BY` direction — since prepared-statement placeholders only work for value positions. Validate structural inputs against a strict allow-list of known-safe identifiers instead.
  ```php
  // GOOD — dynamic column name validated against an allow-list,
  // because it can't be a bound parameter
  $allowedColumns = ['name', 'created_at', 'email'];
  if (!in_array($sortColumn, $allowedColumns, true)) {
      throw new InvalidArgumentException('Invalid sort column');
  }
  $stmt = $pdo->query("SELECT * FROM users ORDER BY {$sortColumn}");
  ```
- **DO:** Use an ORM or query builder (Eloquent, Doctrine, Laravel's query builder) for the majority of application queries, since they parameterize values by default — but stay aware that raw-query escape hatches in every ORM (`whereRaw`, `DB::raw`, native SQL fragments) reintroduce the same injection risk if a raw fragment interpolates a variable directly.
- **DON'T:** Trust that input has already been "sanitized" upstream (by a form validator, a WAF, or client-side JavaScript) as a reason to skip parameterized queries. Defense against injection belongs at the query layer regardless of what validation happened earlier — assume every value reaching a query could be adversarial.
- **DO:** Apply the same prepared-statement discipline to less obvious SQL-adjacent surfaces — search/autocomplete endpoints, report-generation code, admin tooling — not just the obviously user-facing CRUD forms, since these are common places where ad hoc raw queries slip in.
- **DO:** Use named placeholders (`:email`) over positional placeholders (`?`) in PDO once a query has more than two or three bound parameters, since named placeholders make the binding array self-documenting and immune to a subtle bug where reordering parameters silently binds the wrong value to the wrong position.
  ```php
  // BAD — easy to accidentally swap the order of $email and $status
  $stmt = $pdo->prepare('UPDATE users SET status = ? WHERE email = ?');
  $stmt->execute([$status, $email]);

  // GOOD — order-independent, self-documenting at the call site
  $stmt = $pdo->prepare('UPDATE users SET status = :status WHERE email = :email');
  $stmt->execute(['status' => $status, 'email' => $email]);
  ```
- **DO:** Set `PDO::ATTR_ERRMODE` to `PDO::ERRMODE_EXCEPTION` explicitly when creating a PDO connection, so a failed query raises a catchable `PDOException` instead of PDO's legacy default of silently returning `false` and requiring a manual error-code check after every call.
- **DON'T:** Build a `LIKE` pattern by directly concatenating user input with wildcard characters without escaping PHP's own string interpolation *and* the pattern's special characters (`%`, `_`) — bind the fully-constructed pattern as a single parameter, and escape literal `%`/`_` characters within the user-supplied portion if they should be matched literally rather than as wildcards.
  ```php
  $escaped = str_replace(['%', '_'], ['\%', '\_'], $searchTerm);
  $stmt = $pdo->prepare("SELECT * FROM products WHERE name LIKE :term ESCAPE '\\\\'");
  $stmt->execute(['term' => "%{$escaped}%"]);
  ```
- **DO:** Review any code that constructs a query with a variable number of bound values (an `IN (...)` clause built from an array) to bind each value as its own placeholder generated dynamically, rather than concatenating the array's values directly into the query string.
  ```php
  $placeholders = implode(',', array_fill(0, count($ids), '?'));
  $stmt = $pdo->prepare("SELECT * FROM orders WHERE id IN ({$placeholders})");
  $stmt->execute($ids);
  ```
- **DO:** Use an ORM's query builder methods for filtering by dynamic column/table names indirectly — most ORMs expose a safe way to reference a column by validated name (e.g., through defined model attributes) rather than requiring raw SQL string interpolation for anything beyond simple value filtering.
- **DON'T:** Trust a "safe" wrapper function that turns out to just concatenate its inputs internally. Verify (by reading the actual implementation, not just its name) that any in-house "query helper" or "database utility" function genuinely uses parameter binding under the hood before relying on it as the project's injection defense.
- **DO:** Apply the same prepared-statement discipline when writing raw SQL inside database migrations that accept any dynamic input (rare, but possible in a data-migration script that processes existing rows) — a migration script is still application code capable of being injection-vulnerable if it builds SQL from untrusted data.

## Testing (PHPUnit)

- **DO:** Structure PHPUnit test classes to extend `TestCase`, name test methods descriptively (`testThrowsWhenEmailIsInvalid` or, with PHPUnit's attribute-based naming, a `#[TestDox]`-annotated description), and use the `assert*` family (`assertEquals`, `assertTrue`, `assertInstanceOf`, `assertThrows` via `expectException`) so failure output clearly states expected vs. actual.
  ```php
  final class OrderTest extends TestCase
  {
      public function testTotalIncludesTax(): void
      {
          $order = new Order(subtotal: 100.0, taxRate: 0.08);
          $this->assertEqualsWithDelta(108.0, $order->total(), 0.001);
      }
  }
  ```
- **DO:** Design classes for testability with constructor-injected dependencies behind interfaces, so tests can substitute a test double for a database connection, HTTP client, or clock rather than requiring real infrastructure to run the suite.
- **DO:** Use PHPUnit's data providers (`#[DataProvider('provider')]`) to run the same test logic across multiple input/expected-output pairs, instead of copy-pasting near-identical test methods that only differ in their literal values.
  ```php
  #[DataProvider('invalidEmails')]
  public function testRejectsInvalidEmail(string $email): void
  {
      $this->expectException(InvalidArgumentException::class);
      new EmailAddress($email);
  }

  public static function invalidEmails(): array
  {
      return [['not-an-email'], ['missing-domain@'], ['@missing-local.com']];
  }
  ```
- **DON'T:** Write tests that depend on a real, shared database or external API without isolation — use an in-memory/test database with transactional rollback between tests, or mock external service calls, so tests are deterministic and don't interfere with each other when run in parallel or in any order.
- **DO:** Use PHPUnit's mock builder (`createMock`, `createStub`) to isolate the unit under test from its collaborators, and prefer stubs (canned return values) over mocks (call-count/argument verification) unless the test's actual purpose is verifying an interaction happened.
- **DO:** Keep unit tests fast and hermetic (no real filesystem/network/database access) and put anything requiring real infrastructure into a separately run integration test suite, so the fast unit suite can run on every save/commit without becoming a bottleneck.
- **DO:** Measure and track code coverage (`phpunit --coverage-html`) to spot untested branches — particularly error paths and edge cases — while treating the percentage as a diagnostic signal rather than a target that can be gamed with assertion-free tests.
- **DO:** Use `setUp()`/`tearDown()` for per-test fixture creation and cleanup, and `setUpBeforeClass()`/`tearDownAfterClass()` only for genuinely expensive, safely-shared one-time setup — mutable state shared via the class-level hooks is a common source of order-dependent test failures, the same risk as RSpec's `before(:all)`.
- **DO:** Use `expectException()`/`expectExceptionMessage()` before the code under test runs, rather than wrapping the call in a manual `try`/`catch`/`fail()` pattern — PHPUnit's built-in expectation API produces clearer failure output and is the idiomatic way to assert an exception is thrown.
  ```php
  public function testRejectsNegativeAmount(): void
  {
      $this->expectException(InvalidArgumentException::class);
      $this->expectExceptionMessage('Amount must be positive');
      new Payment(amount: -10);
  }
  ```
- **DON'T:** Assert on a mock's internal call count as the primary verification for behavior that could just as easily be verified through the observable output/return value. Over-mocking (verifying implementation-detail interactions rather than results) makes tests brittle against harmless refactors.
- **DO:** Group related test assertions with PHPUnit's `assertEquals`/`assertSame` distinction understood correctly — `assertSame` uses strict `===` comparison (checking type and, for objects, identity), while `assertEquals` uses looser `==`-style comparison; pick the one that actually matches what's being verified, since `assertEquals` can mask a type mismatch that should have failed the test.
- **DO:** Use PHPUnit's `#[Test]` attribute (PHPUnit 10+) or the `test` method-name prefix consistently across a codebase, not a mix of both conventions, so test discovery and a reader's expectations stay predictable.
- **DO:** Use `#[Group('slow')]`/`#[Group('integration')]` attributes to tag tests by category, so CI or a local developer run can selectively include/exclude categories (`phpunit --exclude-group slow` for a fast local loop) without needing separate test-suite configuration files per category.
- **DON'T:** Leave a test with no assertions at all (a "test" that just calls a method and checks nothing about its result) — an assertion-free test always passes regardless of correctness and provides false confidence; PHPUnit's `--strict-coverage`/risky-test detection flags these, and they should be fixed rather than suppressed.
- **DO:** Use dependency injection containers' test/override configuration (or a lightweight manual container) to swap real service implementations for fakes in integration-level tests, rather than relying on global mutable state (static properties, singletons re-configured mid-test-run) to redirect a dependency for testing purposes.
- **DO:** Write PHPUnit tests for exception messages, not just exception classes, when the message content itself carries meaning a caller might parse or display — `expectExceptionMessageMatches()` with a regex is often more robust than an exact string match against text that might reasonably be reworded later.

## Common AI-Assistant Mistakes

- **DON'T:** Generate PHP 5-era code — untyped properties, `array()` instead of `[]`, string-based callables, or code that ignores `strict_types` — in a project whose `composer.json` targets PHP 8+. Check the declared PHP version constraint before writing new code and use the modern feature set it actually allows (typed properties, promoted constructor properties, enums, `match`, readonly properties).
- **DON'T:** Mix raw HTML output and business/data-access logic in the same file without any templating separation (`echo`-ing HTML directly inside a controller alongside SQL queries) — even a lightweight approach (a templating engine like Twig/Blade, or at minimum separate view files with only display logic) keeps concerns separated and makes the code testable and reusable.
  ```php
  // AVOID — logic and markup tangled together
  <?php
  $users = $pdo->query("SELECT * FROM users")->fetchAll();
  foreach ($users as $u) {
      echo "<li>" . $u['name'] . "</li>";
  }
  ?>

  // PREFER — data fetched separately, view stays declarative,
  // and output is escaped
  <?php foreach ($users as $user): ?>
      <li><?= htmlspecialchars($user['name'], ENT_QUOTES) ?></li>
  <?php endforeach; ?>
  ```
- **DON'T:** Use deprecated or removed functions (`mysql_query`, `each()`, `create_function()`, `split()`) that generated code sometimes reproduces from older training-era examples. Verify a function is still current for the target PHP version — several `mysql_*` functions were removed entirely in PHP 7, and other functions have been deprecated in PHP 8.x.
- **DON'T:** Concatenate user input directly into SQL strings, file paths, or shell commands. Beyond SQL injection, watch for path traversal (unsanitized `include`/`require`/file-path input) and command injection (unescaped input passed to `exec`/`shell_exec`/`system`) — use `escapeshellarg()`, allow-listed paths, and parameterized queries respectively.
- **DON'T:** Suppress warnings with `@` in generated code as a quick way to make a snippet "work" without erroring — this is a strong signal of unhandled edge cases papered over rather than fixed, and it should almost never appear in new code.
- **DON'T:** Skip output escaping when rendering user-controlled data into HTML (a reflected/stored XSS vector). Default to `htmlspecialchars()` (or the auto-escaping a templating engine provides) for any value that originated from user input or an external source, rather than trusting it's already safe.
- **DON'T:** Assume a project uses a specific framework's conventions (Laravel's Eloquent/facades, Symfony's DI container, WordPress hooks) without checking — verify the actual framework and its version in `composer.json` before generating framework-specific code, since APIs and idioms differ significantly between them and across major versions of the same framework.
- **DON'T:** Generate code with `mixed` types or no types at all as a way to sidestep deciding on a real type signature, when the project's `composer.json` already targets PHP 8+ and existing code is typed. Match the existing level of type strictness rather than reverting to a looser style out of convenience.
- **DON'T:** Bind array values directly into an `IN (...)` clause's placeholder count without generating a matching number of placeholders — a common generated-code bug is preparing `WHERE id IN (?)`with a single placeholder and then executing it against an array of several IDs, which PDO does not automatically expand.
- **DON'T:** Forget to set `PDO::ATTR_ERRMODE` to exception mode when generating new PDO connection setup code, leaving database errors to fail silently via PDO's legacy default behavior instead of raising a catchable, debuggable exception.
- **DON'T:** Generate framework-specific ORM code (Eloquent relationships, Doctrine entity mappings) without checking the ORM's actual configured conventions in the project (naming strategy, existing relationship definitions) — a plausible-looking but mismatched relationship definition can silently query the wrong table or column.
- **DON'T:** Generate code that trusts a "safe wrapper" function's name without verifying it actually parameterizes internally — check the implementation of any in-house database helper before relying on it as injection protection.
- **DON'T:** Skip Composer script/CI wiring when generating new tooling configuration (a linter, a static analyzer) — add a corresponding `composer.json` script entry so the new tool is actually invokable the same way as the project's other tooling, rather than leaving it as a one-off command nobody remembers to run.
- **DON'T:** Generate global `composer global require` install instructions for a project-level dependency — project dependencies belong in the project's own `composer.json`, tracked and locked, not installed globally on the assumption every environment already has them.

## Quick Checklist
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
