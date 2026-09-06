# Testing & QA

## Test Philosophy

### The Testing Pyramid

- **DO:** Shape the test suite like a pyramid — a large base of fast, isolated unit tests, a smaller layer of integration tests that check components working together, and a thin top layer of end-to-end tests that exercise the full system. Unit tests are cheap to write, run in milliseconds, and pinpoint failures precisely; e2e tests are expensive, slow, and tell you something is broken without saying what, so the ratio should favor the cheap, precise layer.

- **DON'T:** Invert the pyramid into an "ice cream cone" — a handful of unit tests propping up a mountain of manual or end-to-end checks. A suite dominated by slow, flaky e2e tests turns every CI run into a ten-minute gamble and makes contributors afraid to touch code, which is the opposite of what tests are supposed to buy you.
```text
ICE CREAM CONE (bad):        PYRAMID (good):
   ███████ manual/e2e            ▲  unit (many)
   ██████ e2e                   ▲▲▲ integration (some)
   ██ integration               ▲▲▲▲▲ e2e (few)
   █ unit
```

- **DO:** Treat the pyramid as a heuristic shaped by the system, not a law with a fixed ratio. A service that is mostly orchestration and thin wrappers around other services (a "glue code" system) legitimately needs more integration tests relative to unit tests than a library full of pure computational logic does — this is sometimes called a "testing trophy" (weighting integration tests heaviest) or a "honeycomb" (for systems built from many networked services), and either can be the right shape depending on where the risk actually lives.

- **DON'T:** Chase a specific numeric split (e.g., "70% unit, 20% integration, 10% e2e") as a KPI in itself. Optimizing for a ratio encourages writing filler unit tests to keep the denominator favorable instead of asking where bugs actually occur and where a test would catch them cheaply.

- **DO:** Ask "what layer would catch this bug fastest and most precisely?" when deciding where a new test belongs. A bug in a pure calculation belongs in a unit test; a bug in how two services agree on a payload shape belongs in an integration or contract test; a bug in whether the checkout flow actually completes for a real user belongs in e2e — writing it at the wrong layer means slower feedback for no extra confidence.

### What Deserves a Test

- **DO:** Prioritize tests for business logic, calculations, conditional branches, state transitions, and anything where a bug would silently corrupt data or money. These are the places where "looks right" and "is right" diverge most easily, and where a human reviewer skimming a diff is least likely to catch a subtle off-by-one or inverted condition.

- **DON'T:** Write tests for trivial code with no branching — a getter that returns a field, a constructor that assigns parameters, a one-line pass-through wrapper around another function. A test that can only ever pass (because the code has no way to be wrong) adds maintenance cost without adding any ability to catch a real regression.
```java
// Not worth a dedicated test: no logic to get wrong
public String getName() { return this.name; }

// Worth a test: a decision the code could get wrong
public BigDecimal applyDiscount(BigDecimal price, Customer c) {
    return c.isLoyaltyMember() ? price.multiply(LOYALTY_RATE) : price;
}
```

- **DO:** Write a regression test for every bug fix, ideally one that reproduces the bug and fails before the fix is applied. This both proves the fix actually addresses the reported symptom and permanently guards against the same class of bug reappearing after a later refactor.

- **DON'T:** Skip testing code just because it "looks obviously correct." Non-trivial branching, boundary conditions, and off-by-one errors are exactly the kind of mistake that looks obviously correct to the person who just wrote it — that is precisely why an independent, automated check is worth having.

- **DO:** Weigh the cost of a bug in production against the cost of writing the test. Authentication, payment processing, data migrations, and anything touching user data deserve disproportionate test investment because the blast radius of a bug there is large; a rarely-used internal admin script formatting a debug log line does not need the same rigor.

- **DON'T:** Write tests that re-verify a well-known, well-tested third-party library's own behavior — for example, asserting that a date library correctly adds days, or that a JSON library correctly parses JSON. That library already has its own test suite; your test suite should verify your code's use of it, not the library's internals.

### Testing Behavior, Not Implementation Details

- **DO:** Write assertions against observable outputs, return values, and side effects that are part of the function or module's public contract. A caller of `calculateShipping(order)` cares that it returns the right number for a given order — not which internal helper method it delegated to, or in what order it iterated a list, unless that order is itself part of the contract.

- **DON'T:** Assert on private state, internal method call counts, or implementation-specific sequencing that a caller could never observe and that isn't part of the documented contract. A test that fails because you refactored a private helper's name — with the public behavior completely unchanged — is testing the wrong thing and actively punishes good refactoring.
```typescript
// BAD: coupled to an implementation detail (which private helper ran)
expect(cartSpy.recalculateInternalTotals).toHaveBeenCalledTimes(1);

// GOOD: coupled to the observable contract
expect(cart.getTotal()).toBe(42.50);
```

- **DO:** Ask "would this test still pass if I rewrote the implementation from scratch, keeping the same inputs/outputs?" A test that answers "no" for a reason unrelated to a real behavior change is coupled to implementation, not behavior, and should be rewritten against the public interface.

- **DON'T:** Use reflection, monkey-patching, or access to private/internal members purely to peek inside an object just to make an assertion. If a value genuinely can't be observed through the public API, that's usually a sign the public API is incomplete or the value doesn't need verifying at that layer — not a reason to break encapsulation from the test.

### Avoiding Brittle Tests Coupled to Internals

- **DO:** Prefer black-box testing through public interfaces wherever the public interface can express what you need to verify. Black-box tests survive internal refactors — renaming a private method, splitting a function into two, switching data structures — because none of that is visible from outside.

- **DON'T:** Couple a test to the exact order in which internal collaborators are invoked unless that order is a genuine, documented part of the contract (e.g., "call `validate()` before `save()`" is a real API contract; "the function happens to loop over field A before field B internally" is not).

- **DO:** Distinguish "sociable" unit tests (which exercise a unit together with its real, lightweight collaborators) from "solitary" unit tests (which isolate the unit completely with test doubles). Both are legitimate; solitary tests are useful for pinning down one component's logic in isolation, while sociable tests catch integration mistakes between closely related units that solitary tests, by design, cannot see.

- **DON'T:** Snapshot an entire internal object graph (every private field, every nested structure) when only a handful of fields are actually part of the behavior under test. A broad, implementation-shaped snapshot fails on any internal restructuring, training the team to blindly accept snapshot updates rather than reading them.

- **DO:** Remember that tests are a maintenance liability as well as a safety net — every test is code that must be read, understood, and kept passing through future changes. A test that is tightly coupled to internals costs real engineering time on every unrelated refactor without buying a corresponding increase in the ability to catch actual bugs, which is a bad trade.

## Unit Testing

### Naming Conventions

- **DO:** Name each test after the behavior it verifies, the condition under which it applies, and the expected outcome, so a failing test's name alone tells you roughly what broke without opening the file. Patterns like `returns_404_when_user_not_found` or nested `describe`/`it` blocks that read as a sentence ("UserService > when user does not exist > throws NotFoundError") both work well because they encode intent, not just a label.
```javascript
// BAD: tells you nothing when it fails in a CI log
it('test1', () => { ... });
it('works', () => { ... });

// GOOD: failure alone tells you what broke
describe('applyDiscount', () => {
  it('applies the loyalty rate when the customer is a loyalty member', () => { ... });
  it('leaves the price unchanged when the customer is not a member', () => { ... });
});
```

- **DON'T:** Name tests generically (`test1`, `testFoo`, `testEdgeCase`, `testBasic`). A generic name forces every reader — including you, six months later — to open the test body just to learn what it checks, which defeats the purpose of a name as a first line of documentation.

- **DO:** Pick one naming convention (method-under-test/state/expected-outcome, given-when-then prose, or nested BDD `describe` blocks) and apply it consistently across the codebase or at least within a module. Consistency lets readers pattern-match test names quickly instead of re-learning the convention file by file.

- **DON'T:** Mix multiple unrelated naming styles arbitrarily within the same test file or suite. Alternating between terse `testX` names and full BDD sentences in the same file signals no one owns the suite's readability, and it erodes the value of naming conventions entirely.

### Arrange-Act-Assert / Given-When-Then

- **DO:** Structure each test into three clear phases — set up the preconditions (arrange/given), perform the one action under test (act/when), and check the outcome (assert/then) — ideally separated visually with blank lines or comments so a reader can scan a test in seconds. This structure is language- and framework-agnostic and makes it immediately obvious what is being tested versus what is just scaffolding.
```python
def test_withdraw_reduces_balance_by_amount():
    # Arrange
    account = Account(balance=100)

    # Act
    account.withdraw(30)

    # Assert
    assert account.balance == 70
```

- **DON'T:** Interleave assertions with setup and action code so that arrange, act, and assert are scattered throughout the test body. A test that checks something, does more setup, performs another action, and checks something else again is really multiple tests wearing one name, and a failure in the middle leaves the reader guessing which phase actually broke.

- **DO:** Keep the "act" step to a single, clearly identifiable call or action per test wherever possible. If a test needs to perform several distinct actions to set up a realistic scenario, treat all but the last as part of arrange, and make clear which one line is the actual behavior under test.

- **DON'T:** Bury the action under test among several other calls that look equally important. If a reader can't point to the one line that represents "the thing being tested," the test's intent is unclear and future edits are likely to change that line by accident.

### One Logical Assertion Focus Per Test

- **DO:** Focus each test on verifying one logical behavior, even if that requires several physical assertion statements (for example, checking multiple fields of one resulting object, which together describe one outcome). The test should have one reason to fail, and that reason should be obvious from the test's name.
```java
@Test
void createOrder_returnsOrderWithCorrectTotals() {
    Order order = service.createOrder(items);

    // Multiple asserts, one logical concept: "the returned order is correct"
    assertEquals(3, order.getItemCount());
    assertEquals(new BigDecimal("59.97"), order.getTotal());
    assertEquals(OrderStatus.PENDING, order.getStatus());
}
```

- **DON'T:** Cram unrelated behaviors into a single test just to save typing (e.g., one giant `testEverything()` that creates a user, updates it, deletes it, and checks pagination along the way). When such a test fails, the failure message tells you almost nothing about which of the five unrelated things actually broke, and fixing one behavior risks silently breaking the assertions for another that nobody's looking at anymore.

- **DO:** Split distinct scenarios and branches into separate, clearly named tests rather than one test with several conditional paths inside it. Separate tests fail independently and their names document each scenario individually, which a single sprawling test cannot do.

- **DON'T:** Loop over several unrelated fixture inputs inside one test with assertions in the loop body. When one iteration fails, most test runners report only "assertion failed at line N" without telling you which input caused it; use your framework's parametrized/table-driven test feature instead, which reports each case as its own named result.
```python
# BAD: which input failed? you have to add a print statement to find out
def test_discount_rates():
    for amount, expected in [(100, 90), (200, 170), (0, 0)]:
        assert apply_discount(amount) == expected

# GOOD: each case reports independently, by name
@pytest.mark.parametrize("amount,expected", [(100, 90), (200, 170), (0, 0)])
def test_discount_rates(amount, expected):
    assert apply_discount(amount) == expected
```

### Test Independence and No Shared Mutable State

- **DO:** Write every test so it can run alone, in any order, and repeatedly, producing the same result each time. A suite where tests must run in a specific sequence to pass is fragile by construction — a reordering by the test runner, a new parallel-execution mode, or someone running a single test in isolation while debugging will break it.

- **DON'T:** Let one test depend on state left behind by a previous test (a record inserted by test A that test B assumes exists, a counter incremented by test A that test B reads). This kind of hidden coupling is invisible in the test code itself and only surfaces as a mysterious failure when someone reorders, filters, or parallelizes the suite.
```javascript
// BAD: test B silently depends on test A having run first
let sharedUsers = [];
test('creates a user', () => { sharedUsers.push(makeUser()); });
test('lists users', () => { expect(sharedUsers.length).toBe(1); }); // breaks if run alone

// GOOD: each test creates exactly what it needs
test('lists users', () => {
  const repo = new InMemoryUserRepo([makeUser()]);
  expect(repo.list().length).toBe(1);
});
```

- **DO:** Re-create fixtures fresh for every test — a new object instance, a fresh database transaction that rolls back, a clean temp directory — rather than reusing one shared mutable instance across the whole file. Fresh state per test is the single most reliable way to eliminate order-dependence and cross-test contamination.

- **DON'T:** Store test fixtures in module-level or class-level mutable variables and rely on `beforeEach`/`setUp` to only partially reset them. A half-reset shared object is worse than no reset at all, because it looks isolated in the test code while silently carrying state between runs.

### Avoiding Testing Framework and Third-Party Library Internals

- **DON'T:** Write tests that verify a framework or library is doing what it's documented to do — for example, testing that a web framework actually routes a request to the right handler, or that a UI framework's state hook actually triggers a re-render. That behavior is the framework's own responsibility and is already covered by the framework's own test suite; testing it again in your codebase adds cost without adding coverage of anything you control.

- **DO:** Test the custom configuration, adapters, and glue code where your code meets the framework — a custom middleware, a non-trivial validation rule, a serializer that transforms framework objects into your API's response shape. This is where bugs specific to your usage actually live.

- **DON'T:** Re-test standard library behavior that is already extensively documented and battle-tested (that sorting a list actually sorts it, that string concatenation actually concatenates). Spend that test-writing time on the parts of the system that are genuinely yours and genuinely capable of being wrong.

- **DO:** Focus unit test effort on your own business logic, decision points, and data transformations, treating well-established frameworks and libraries as trusted infrastructure. If a third-party library turns out to be unreliable enough that you need to guard against its behavior, write a narrow test around your usage of it (or wrap it behind an interface you control), not a test that duplicates the library's own suite.

## Mocking & Test Doubles

### The Taxonomy: Dummies, Stubs, Fakes, Mocks, and Spies

- **DO:** Learn and use the precise vocabulary for test doubles, because each kind implies a different testing intent. A **dummy** is a placeholder object passed only to satisfy a parameter list and never actually used; a **stub** returns pre-programmed answers to calls made during the test but isn't itself verified; a **fake** is a lightweight but genuinely working implementation (an in-memory database standing in for a real one); a **mock** is pre-programmed with expectations about which calls it should receive and is itself asserted against after the fact; a **spy** wraps a real object and records how it was called, for later inspection, without replacing its real behavior.
```text
DUMMY  — passed but never used:        processInvoice(order, /* unused */ dummyLogger)
STUB   — returns canned data:          stub.getUser(1).returns({ id: 1, name: 'Ann' })
FAKE   — lightweight real impl:        new InMemoryUserRepository()
MOCK   — expectation + verification:   expect(mockEmailer.send).toHaveBeenCalledWith(...)
SPY    — wraps the real thing:         spyOn(realLogger, 'warn') // still logs for real
```

- **DON'T:** Call every test double "a mock" indiscriminately. Blurring the distinction leads to misuse in practice — most commonly, asserting call counts and argument shapes (a mock's job) on something that was only ever meant to feed data into the test (a stub's job), which produces tests that fail on harmless implementation changes.

### Avoiding Over-Mocking

- **DON'T:** Mock so many of a unit's collaborators that the test only proves the mocks return what you told them to return. A test that replaces every dependency with a scripted double and then asserts on the scripted output is circular — it will pass even if the real implementation is completely broken, because no real code path involving actual logic ever executes.
```javascript
// BAD: mocks everything, tests nothing real
test('calculates total', () => {
  const mockPricer = { getPrice: jest.fn().mockReturnValue(10) };
  const mockTax = { calculate: jest.fn().mockReturnValue(1) };
  const mockDiscount = { apply: jest.fn().mockReturnValue(0) };
  const total = checkout(mockPricer, mockTax, mockDiscount, item);
  expect(total).toBe(11); // just re-states what the mocks were told to return
});
```

- **DO:** Reserve mocking for genuine external boundaries — the network, the filesystem, the system clock, sources of randomness, third-party APIs, email/SMS gateways, payment processors — and let a unit collaborate with its real, in-process dependencies whenever that collaboration is cheap and deterministic. The goal of a test double is to remove non-determinism and slowness from the boundary of the system, not to remove your own code from the test.

- **DON'T:** Mock the specific dependency that contains the core logic you're actually trying to verify. If the thing under test is "does `applyDiscount` correctly call the pricing engine and use its result," mocking the pricing engine so completely that it returns a fixed number for every input means the test can never catch a real bug in how the two interact.

- **DO:** Ask, for each dependency, "is this slow, non-deterministic, or does it have real-world side effects (network I/O, sending an email, charging a card)?" before reaching for a test double. If the answer is no — a pure calculation, an in-memory data structure, a small internal collaborator — let the real object run; it's usually just as fast as a mock and it actually gets exercised.

### Real Dependencies vs. Test Doubles

- **DO:** Prefer real objects or lightweight in-memory fakes over mocks for dependencies that are fast and deterministic. An in-memory implementation of a repository interface, or an embedded/in-memory database, exercises real logic (query building, mapping, edge cases in the interface) far more faithfully than a mock that simply returns whatever you told it to.

- **DON'T:** Use real, slow, or side-effecting dependencies inside unit tests — a real network call to a third-party API, a real payment gateway charge, a real outbound email. These belong behind a test double at the unit level, with the real integration verified separately, at the integration or contract-test layer, where slowness and non-determinism are expected and budgeted for.

- **DO:** Design dependencies to be injectable (constructor injection, factory functions, or a DI container) so that swapping a real implementation for a fake or a mock in a test doesn't require monkey-patching or reaching into module internals. Code that hardcodes `new ExternalPaymentClient()` deep inside business logic cannot be tested without either hitting the real service or resorting to fragile module-mocking tricks.
```python
# BAD: hardcoded dependency, forces monkeypatching to test
class OrderService:
    def charge(self, order):
        client = StripeClient()  # concrete dependency created inline
        return client.charge(order.total)

# GOOD: injected dependency, trivially swapped for a fake in tests
class OrderService:
    def __init__(self, payment_client):
        self.payment_client = payment_client

    def charge(self, order):
        return self.payment_client.charge(order.total)
```

- **DON'T:** Default to mocking every dependency "just in case" as a blanket policy. A codebase-wide habit of mocking everything produces suites that are fast but tell you almost nothing about whether components actually work together, pushing all real integration risk undetected into production.

### Avoiding Brittle Mock-Heavy Tests

- **DON'T:** Assert on exact call arguments, call order, or call counts for incidental implementation details that aren't part of the actual contract — for example, verifying a logger was called with one specific exact string, or that an internal helper ran before another internal helper, when neither ordering nor wording is something a caller actually depends on.

- **DO:** Prefer state-based assertions (checking the resulting value, the resulting object, the resulting persisted record) over interaction-based assertions (checking that a specific method was called with specific arguments) whenever both are available. State-based assertions describe *what* the code should produce and survive refactors; interaction-based assertions describe *how* the code should do it internally and break the moment that internal "how" changes even if the "what" is still correct.
```typescript
// BRITTLE: breaks if you rename or reorder an internal call, even if output is unchanged
expect(cart.addItem).toHaveBeenCalledWith(item, { recalc: true });

// RESILIENT: verifies the actual outcome, survives internal refactors
cart.addItem(item);
expect(cart.getTotal()).toBe(item.price);
```

- **DON'T:** Let a suite accumulate so many interaction-based assertions against mocks that a routine internal refactor (renaming a private method, splitting one function into two, changing the order two independent calls happen in) breaks dozens of unrelated tests. When refactoring becomes scarier than leaving bad code alone because "the tests will all turn red for no real reason," the test suite has become a liability instead of a safety net.

- **DO:** Reserve interaction-based (mock) verification for cases where the interaction itself *is* the contract being tested — for example, verifying that a failed payment triggers exactly one notification email, where "exactly one call happened" is the actual requirement, not an implementation detail.

### Mocking Time, Randomness, and Network

- **DO:** Replace the system clock, random number generators, and UUID generation with controllable test doubles whenever a test's outcome depends on them. A test that calls the real `Date.now()` or an unseeded random function will occasionally produce a different, unpredictable input and fail sometime later for no code-related reason — exactly the profile of a flaky test.
```javascript
// BAD: depends on the real clock, subtly flaky near midnight/DST/month boundaries
function isExpired(token) { return Date.now() > token.expiresAt; }
test('expired token is rejected', () => {
  expect(isExpired({ expiresAt: Date.now() - 1 })).toBe(true); // race with real clock
});

// GOOD: clock is injected/faked and fully controlled by the test
test('expired token is rejected', () => {
  const clock = new FakeClock('2026-01-01T00:00:00Z');
  expect(isExpired({ expiresAt: clock.now() - 1 }, clock)).toBe(true);
});
```

- **DON'T:** Let a unit test make a real network call, even to a "harmless" internal service or a sandboxed third-party API. A real call introduces latency, external availability as a dependency of your test suite, and non-determinism (timeouts, rate limits, transient errors) that has nothing to do with whether your code is correct.

## Integration & E2E Testing

### Test Environment Parity

- **DO:** Keep integration and e2e test environments as close to production as practically possible — same database engine and version, same OS/container base image, comparable configuration and feature flags. Bugs caused by environment drift (a query that behaves differently under a different SQL dialect, a timezone difference, a missing environment variable) are exactly the bugs that "all tests passed" gives false confidence about when the test environment doesn't match production.

- **DON'T:** Test against a lightweight substitute (e.g., an in-memory SQLite database) while production runs a different, heavier engine (e.g., PostgreSQL) without consciously accepting the risk that dialect-specific behavior (case sensitivity, date arithmetic, locking semantics, specific SQL functions) will pass in tests and fail in production. If you make this tradeoff for speed, document it and add targeted tests against the real engine for the queries most likely to diverge.

- **DO:** Use disposable, real dependencies for integration tests via containerized infrastructure (spinning up a real database, message queue, or cache in a container for the duration of the test run) rather than mocking the dependency entirely at this layer. This gets you the speed and isolation of a fresh environment per run while still exercising real behavior, including real driver code and real query execution.
```javascript
// Testcontainers-style setup: a real, disposable Postgres instance per test run
beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16').start();
  db = await connect(container.getConnectionUri());
  await runMigrations(db);
});

afterAll(async () => {
  await container.stop();
});
```

- **DON'T:** Assume "it passed in CI" is equivalent to "it will work in production" when the CI environment's resource limits, network topology, or data volume are wildly different from production's. Load-sensitive or timing-sensitive bugs routinely hide behind a CI environment that's faster, smaller, or more isolated than the real deployment target.

### Avoiding Flaky Tests

- **DON'T:** Use a hardcoded `sleep(2000)` (or equivalent fixed delay) to "wait" for an asynchronous operation to finish. A fixed sleep is either too short (intermittent failures under load, when the environment is briefly slower) or wastefully too long (needlessly slows the whole suite down) — and it's virtually never exactly right.
```python
# BAD: guesses how long async work takes; flaky under load, slow otherwise
def test_order_processed():
    submit_order(order)
    time.sleep(3)  # hope processing finished by now
    assert get_order_status(order.id) == "PROCESSED"

# GOOD: waits for the actual condition, with a sane upper bound
def test_order_processed():
    submit_order(order)
    wait_until(lambda: get_order_status(order.id) == "PROCESSED", timeout=5)
```

- **DO:** Wait for a specific, observable condition (a returned event, a callback firing, a polling check with a bounded timeout) instead of an arbitrary fixed delay. Most testing libraries provide a `waitFor`/`eventually`/polling helper specifically for this; use it instead of reinventing a worse version with `sleep`.

- **DON'T:** Let two tests — especially when run in parallel — read or write the same mutable test data (the same database row, the same file, the same shared counter). A race between two tests mutating the same resource produces failures that are non-reproducible locally and erode trust in the whole suite ("just re-run it, it's flaky").

- **DO:** Give every test its own isolated data — a uniquely-generated ID or namespace per test, a database transaction that rolls back at the end of the test, or a dedicated schema/database per parallel worker. Isolation at the data level is what makes parallel execution and reordering safe in the first place.

- **DON'T:** Leave timing-dependent assumptions baked into a test — relying on wall-clock ordering of asynchronous events, assuming a background job finishes within an implicit window, or comparing against `Date.now()` computed at two different points without controlling the clock. Any of these can flip a test from pass to fail purely based on how the machine running it happened to schedule work that run.

- **DO:** Treat a flaky test as a bug to be fixed with the same priority as a bug in production code, not as background noise to be re-run past. A test that fails intermittently for reasons unrelated to the code under test trains the team to ignore red CI runs, which is far more dangerous than not having the test at all.

### Test Data Setup and Teardown Discipline

- **DO:** Set up exactly the fixtures a test needs, explicitly, within that test (or a clearly scoped `beforeEach`), and tear them down afterward so the environment is clean for the next run. Explicit, local setup makes a test readable on its own — you can see everything it depends on without hunting through shared global fixture files.

- **DON'T:** Write a test that assumes data left over from a previous manual run, a seed script executed once, or another test suite entirely — "assume the admin user with ID 1 already exists in the database." This kind of implicit dependency is invisible in the test itself and breaks the moment the test runs against a fresh database, a different environment, or in a different order.

- **DO:** Use factory or builder functions to construct test data, with sensible defaults and the ability to override just the fields relevant to a given test. Factories keep tests readable (only the fields that matter to the scenario are visible) and DRY (a schema change updates one factory instead of dozens of inline fixture literals).
```typescript
// Factory with sensible defaults, override only what the test cares about
function makeUser(overrides = {}) {
  return { id: uuid(), email: 'test@example.com', role: 'member', ...overrides };
}

test('admin can delete a post', () => {
  const admin = makeUser({ role: 'admin' }); // only the relevant field is visible
  expect(canDelete(admin, post)).toBe(true);
});
```

- **DON'T:** Let teardown fail silently or get skipped when a test fails partway through. Orphaned test data (a row that never got cleaned up because the test crashed before its `afterEach` ran) compounds over time and causes confusing failures in unrelated later tests that stumble onto the leftover data.

### Contract Testing for Service Boundaries

- **DO:** Use contract tests — consumer-driven contracts, schema validation against an OpenAPI/protobuf definition, or dedicated compatibility test suites — at the boundary between independently deployed services, so a breaking change is caught without needing to spin up every combination of services together. Contract tests are fast, focused on exactly the interface, and pinpoint which side of a boundary broke compatibility.

- **DON'T:** Rely solely on full end-to-end tests across many microservices to catch API incompatibilities between them. E2E tests that span several services are slow, flaky (every service in the chain is a new source of instability), and when one fails, it's often unclear which service actually introduced the breaking change without significant digging.

- **DO:** Version and validate API contracts as part of CI — running the producer's implementation against the consumer's recorded expectations (or vice versa) automatically on every change, so incompatible changes fail fast at the pull-request stage rather than being discovered during an integration or, worse, a production deployment.

### Scoping What E2E Tests Should Cover

- **DO:** Reserve end-to-end tests for the small number of critical user journeys where verifying the whole system wired together (real UI, real backend, real database, ideally realistic third-party integration) genuinely matters — sign-up, checkout, login, and other flows where a break would be a headline-level incident. E2E tests are the most expensive layer to write, run, and maintain; spend that budget where it buys the most confidence.

- **DON'T:** Try to push every edge case and permutation through the e2e layer just because "it tests the real thing." Edge cases, error handling, and boundary conditions are almost always better and faster covered by unit and integration tests; reserve e2e tests for confirming that the pieces are wired together correctly end to end, not for exhaustively covering logic that a unit test already covers more cheaply.

- **DO:** Keep e2e test code maintainable by abstracting UI interaction details behind reusable helpers (a Page Object or equivalent screen/component abstraction), so that when a selector or a page's structure changes, you update it in one place instead of in every test that touches that page.
```javascript
// Page Object: UI details live in one place, tests read as user intent
class CheckoutPage {
  async submitOrder() { await this.page.click('[data-testid="submit-order"]'); }
  async getConfirmationText() { return this.page.textContent('.confirmation'); }
}

test('completes checkout', async () => {
  await checkoutPage.submitOrder();
  expect(await checkoutPage.getConfirmationText()).toContain('Order confirmed');
});
```

- **DON'T:** Let e2e tests assert on brittle, presentation-only details (exact pixel positions, incidental CSS class names, whitespace) unless that presentation detail is literally the thing under test. Couple assertions to stable, purpose-built hooks (test IDs, accessible roles/labels) instead of implementation-tied selectors that change with every unrelated styling tweak.

## Test-Driven Development

### Red-Green-Refactor Discipline

- **DO:** Follow the three-step cycle deliberately — write a failing test that specifies the desired behavior (red), write the minimal code needed to make it pass (green), then improve the implementation's structure while keeping the test passing (refactor). Each step has a distinct purpose: red proves the test can actually fail, green proves the behavior now exists, and refactor pays down any shortcuts taken to get to green, all with a passing test as a safety net.
```text
1. RED:      write `test_withdraw_rejects_amount_over_balance` — run it, watch it fail
2. GREEN:    add just enough logic to `withdraw()` to make that one test pass
3. REFACTOR: clean up the implementation; re-run the test to confirm it still passes
```

- **DON'T:** Skip the "watch it fail" step. A test that has never been observed failing might be broken in a way that makes it pass unconditionally (a typo in the assertion, a mocked dependency that's always truthy) — the only way to know a test is actually capable of catching the bug it claims to catch is to see it fail first, for the right reason.

- **DON'T:** Skip the refactor step and move straight from green to the next red test. Code written to satisfy "the minimal thing that makes this test pass" is often ugly or duplicated by design — TDD without the refactor step accumulates the same cruft as writing code without tests at all, just with green checkmarks next to it.

### When TDD Helps

- **DO:** Reach for TDD when working through complex business logic with many distinct cases and edge conditions, when fixing a bug (write the failing repro test first, then fix it), and when designing a new public API or interface, where writing the test first forces you to think through the interface from the caller's perspective before committing to an implementation.

- **DO:** Use TDD when requirements are well understood enough to state as concrete input/output examples up front. TDD works best when you can say "given this input, the correct output is X" before writing any implementation — the tighter that specification, the more useful writing the test first becomes.

### When TDD Is Overkill

- **DON'T:** Force strict TDD onto exploratory or throwaway work — a spike to see if an approach is even feasible, a quick prototype to validate a UI direction, research code figuring out an unfamiliar third-party API's actual behavior. Writing tests first for code whose shape you don't yet know slows down the exact kind of fast iteration that exploratory work needs; write the tests once the design has stabilized enough to be worth locking in.

- **DON'T:** Mandate TDD org-wide as dogma disconnected from the kind of problem being solved. Visual/UI layout tweaks, quickly-changing requirements, and genuinely novel design work often benefit more from writing code, looking at the result, and iterating — with tests following once the behavior settles — than from a rigid test-first process applied uniformly regardless of fit.

## Coverage

### Coverage as a Signal, Not a Target

- **DO:** Treat a coverage report as a diagnostic tool that highlights code paths nobody has exercised with a test — a prompt to go look, and decide deliberately whether that gap deserves a test — rather than a score to be maximized for its own sake.

- **DON'T:** Impose a rigid, org-wide coverage threshold (e.g., "100% coverage required to merge," or "no PR may lower coverage by more than 0.1%") without regard for what the uncovered code actually is. Blunt thresholds like this reliably produce the exact behavior they were meant to prevent: developers writing hollow tests purely to move the number, on trivial code that never needed a test in the first place.

- **DO:** Use coverage reports specifically to find gaps in the code that matters most — untested error-handling branches, untested edge cases in critical business logic — and then decide, case by case, whether the gap is worth closing. Coverage is most valuable as a targeted "what did we forget" check, not a suite-wide pass/fail gate.

### Avoiding Coverage-Gaming

- **DON'T:** Write a test that executes a code path without asserting anything meaningful about its result, purely to make the coverage tool mark those lines as covered. A "smoke test" that calls a function and only checks that it didn't throw, when the function actually computes a value that should be verified, provides the illusion of a safety net with none of the substance.
```javascript
// COVERAGE-GAMING: line is "covered," but nothing about correctness is verified
test('calculateTax runs', () => {
  const result = calculateTax(order);
  expect(result).toBeDefined(); // passes even if the tax math is completely wrong
});

// MEANINGFUL: covers the same line, and would actually catch a bug
test('calculateTax applies the correct rate', () => {
  const result = calculateTax({ subtotal: 100, taxRate: 0.08 });
  expect(result).toBe(8);
});
```

- **DO:** For any test you're unsure adds real value, sanity-check it by deliberately breaking the implementation (flip a comparison operator, change a constant, remove a line) and confirming the test actually fails. A test suite's real strength is measured by what it catches, not by what percentage of lines it merely touches.

### What 100% Coverage Does and Doesn't Guarantee

- **DO:** Understand precisely what a coverage percentage measures — typically, that every line (or branch) was *executed* at least once by the test suite. It says nothing about whether the test that executed a line actually checked that the line produced the correct result.

- **DON'T:** Treat 100% coverage as a proxy for "bug-free" or "well-tested." Coverage cannot catch a case that was never written at all (a missing `else` branch that should have handled a null input but doesn't — there's no line to "not cover" because the handling code simply doesn't exist), and it cannot catch a test with a wrong or missing assertion, since the line still counted as executed either way.

- **DO:** Consider mutation testing occasionally as a stronger signal than line coverage — a tool that automatically introduces small deliberate bugs ("mutants": flips a comparison, changes a constant) into the code and checks whether the test suite catches each one. A test suite that has high line coverage but lets most mutants survive is proving exactly the coverage-gaming problem described above: lines are executed, but nothing meaningful is actually being verified.

## CI Test Discipline

### Fast Test Suites

- **DO:** Keep the unit test tier fast — ideally completing in seconds, not minutes — so it can run on every save, every commit, and every pull request without becoming a bottleneck contributors start avoiding. Fast feedback is what makes tests useful as a day-to-day tool rather than a chore run once before merging.

- **DON'T:** Mix slow integration or e2e tests into the same tier that runs on every local save or every commit push. Separate the suite into fast and slow tiers (e.g., unit tests on every push, integration/e2e on a merge queue or nightly run) so slow tests gate what they need to gate without slowing down everyone's everyday feedback loop.

- **DO:** Periodically profile the test suite to find and fix its slowest individual tests. In most suites, a small number of poorly-written tests (an unnecessary real sleep, an unindexed database query, an oversized fixture) account for a disproportionate share of total run time, and fixing just those pays off every single run from then on.

### Parallelization

- **DO:** Design tests so they can safely run in parallel, across multiple processes or CI workers, each with its own isolated data and scope. Parallel execution is one of the most effective ways to cut CI wall-clock time as a suite grows, but only pays off if tests don't secretly depend on shared, mutable resources.

- **DON'T:** Write tests that read or write a shared global resource — a fixed row in a shared test database, a shared file on disk, a shared in-memory singleton — assuming they'll never run at the same time as another test. Under parallel execution, this produces intermittent, hard-to-reproduce failures that look exactly like flakiness because that's exactly what they are.

- **DO:** Shard large test suites across multiple CI workers/machines once single-worker runtime becomes a bottleneck for the team, using the test runner's or CI provider's built-in sharding support so each worker gets a balanced, independent slice of the suite.

### Deterministic Tests

- **DO:** Make every test produce the same pass/fail result on every run, given the same code — no reliance on unseeded randomness, no reliance on the real system clock, no reliance on network or external service availability inside the deterministic tiers of the suite (unit and most integration tests).

- **DON'T:** Use an unseeded random number generator inside a test in a way that can occasionally generate an input that violates an edge-case assumption the test didn't account for. If randomized inputs are useful (as in property-based/fuzz testing), fix and log the seed on failure so the exact failing case can be reproduced deterministically afterward.
```python
# BAD: different random input every run; fails unpredictably, not reproducibly
def test_sort_handles_random_input():
    data = [random.randint(-100, 100) for _ in range(50)]
    assert sorted(data) == my_sort(data)

# GOOD: seeded, so a failure is exactly reproducible
def test_sort_handles_random_input():
    rng = random.Random(20260904)  # fixed seed
    data = [rng.randint(-100, 100) for _ in range(50)]
    assert sorted(data) == my_sort(data)
```

### Avoiding Execution-Order Dependence

- **DON'T:** Write tests that assume they run in file order, declaration order, or alphabetical order. Many modern test runners randomize or parallelize execution order by default (or offer it as an option), and a suite that silently depends on a particular order will fail unpredictably the moment that assumption is violated.

- **DO:** Periodically run the suite with test order explicitly randomized (most frameworks support this: a `--randomize`/`--random` flag, or plugins like randomized-order runners) specifically to surface hidden order dependencies before they surface on their own, at an inconvenient time, in front of the whole team.

### Snapshot Testing Pitfalls

- **DON'T:** Snapshot large, broad structures — an entire rendered component tree, a full API response with dozens of fields, a whole configuration object — where the resulting diff on any change is so large and noisy that reviewers stop reading it and just accept the update. A snapshot nobody actually reviews before approving is a snapshot that will silently accept a real regression the day one occurs.

- **DO:** Scope snapshots narrowly to the specific piece of output whose exact shape is worth guarding (e.g., the generated structure of a config file, or a specifically formatted error message), and pair snapshots with explicit, targeted assertions for the individual fields that actually matter to the test's purpose.
```javascript
// BROAD (risky): any unrelated change anywhere in the tree updates this snapshot
expect(render(<Dashboard />)).toMatchSnapshot();

// NARROW (safer): the assertion is scoped to what the test is actually about
expect(screen.getByTestId('total-balance')).toHaveTextContent('$1,204.50');
```

- **DON'T:** Treat an updated snapshot as automatically correct just because it was regenerated by the tool. A real regression and a legitimate intentional change produce an identical-looking snapshot diff; the only way to tell them apart is to actually read the diff before accepting it, every time.

- **DO:** Reserve snapshot testing for outputs that are genuinely hard to assert on piece by piece (generated code, complex serialized formats, rendered markup where the overall shape is the point) and prefer explicit assertions everywhere else, where they communicate intent far more clearly than a snapshot file ever can.

### Flaky Test Quarantine and Retry Policy

- **DO:** When a flaky test is identified, either fix its root cause immediately or explicitly quarantine it (mark it skipped or moved to a separate non-blocking tier) with a tracked follow-up to fix it — never let it sit silently failing-and-passing in the main suite, where it teaches the team to distrust red CI runs in general.

- **DON'T:** Paper over flakiness with blanket automatic retries on every test failure. Auto-retrying every failed test hides real, intermittent bugs (race conditions, resource leaks) behind a "passed on retry" green checkmark and makes the underlying problem someone else's surprise in production later.

### Test Reporting and Failure Diagnostics

- **DO:** Make CI test failures easy to diagnose without re-running locally — clear failure messages that state expected vs. actual, stack traces, and, for e2e tests, artifacts like screenshots, videos, or logs captured at the point of failure. The faster a failure can be understood from the CI output alone, the faster it gets fixed instead of shrugged off.

- **DON'T:** Let assertion failures produce generic, unhelpful messages ("assertion failed", "expected true, got false") when the testing library supports rich failure output (diffs, matcher-specific messages). A test that's hard to diagnose from its own failure output effectively forces a debugging session for every red run, which is exactly the friction a good test suite is supposed to remove.

## Common AI-Assistant Mistakes in Testing

### Assertion-Free or Tautological Tests

- **DON'T:** Generate a test that runs the code under test but never asserts anything meaningful about the result — checking only that a value `isDefined`/`isNotNull`, or that a function call didn't throw, when the function actually computes something whose correctness matters. This is a specific and common failure mode for AI-generated tests: it produces a green checkmark and a plausible-looking test body while verifying nothing about whether the logic is right.
```python
# TAUTOLOGICAL: passes no matter what parseInvoice actually returns
def test_parse_invoice():
    result = parse_invoice(raw_text)
    assert result is not None

# MEANINGFUL: actually pins down the expected values
def test_parse_invoice():
    result = parse_invoice(raw_text)
    assert result.total == Decimal("129.99")
    assert result.line_items[0].description == "Widget"
```

- **DO:** For every generated test, ask "if the implementation were subtly wrong, would this assertion actually fail?" before considering the test complete. If the answer is no — the assertion would pass regardless of what the code under test returns — the test needs a real, specific expected value, not a weaker existence check.

### Mirroring the Implementation Instead of Verifying Behavior

- **DON'T:** Derive a test's "expected" value by re-implementing the same formula or logic the code under test uses, rather than an independently-known correct answer. A test that computes `price * qty * (1 - discountRate)` to check a function that internally computes `price * qty * (1 - discountRate)` will pass even if that formula itself is wrong — it only proves the code agrees with itself.
```javascript
// SELF-CONFIRMING: the test just repeats the implementation's own formula
function total(price, qty, discountRate) { return price * qty * (1 - discountRate); }
test('computes total', () => {
  expect(total(10, 3, 0.1)).toBe(10 * 3 * (1 - 0.1)); // proves nothing about correctness
});

// INDEPENDENT: expected value comes from working the math out separately
test('computes total with 10% discount', () => {
  expect(total(10, 3, 0.1)).toBe(27); // hand-computed, catches a wrong formula
});
```

- **DO:** Hardcode independently-derived expected values — worked out by hand, taken from a spec or ticket, or from a known-correct reference implementation — for at least the core test cases of any nontrivial logic. This is the difference between a test that can catch a bug in the formula itself and one that can only catch a typo in restating that formula.

### Deleting or Weakening a Failing Test Instead of Fixing the Bug

- **DON'T:** Respond to a newly-failing test by commenting it out, adding `.skip`/`xit`, loosening an assertion's tolerance, or deleting it outright, in order to get the suite back to green — without first determining whether the failure reveals a real bug. A failing test is a signal; suppressing the signal instead of investigating it is how known regressions ship to production with a clean CI history.

- **DO:** Treat every newly-red test as requiring a decision, made explicitly and visibly: either the code has a real bug and needs fixing, or the test's expectation was wrong and needs updating with a clear note explaining why the new expected behavior is correct. Both are legitimate outcomes; silently doing either without explanation is not.

- **DON'T:** Widen an assertion's tolerance, increase a timeout, or remove a specific check simply because it's currently failing, without understanding why it started failing. This kind of "fix" makes the test permanently less capable of catching the exact class of bug it just caught, while looking, on the surface, like the test still exists and still passes.

### Claiming "Tests Pass" Without Actually Running Them

- **DON'T:** State that tests pass, that a suite is green, or that coverage meets some number, based on reading the code and reasoning about what should happen, rather than actually executing the test command and observing its real output. Code that looks correct on inspection frequently isn't, which is the entire reason automated tests exist in the first place — a claim of "passing" that skips execution defeats that purpose while sounding exactly as confident as a claim backed by a real run.

- **DO:** Actually run the test suite (or the relevant subset) and report what the tool's own output says — pass/fail counts, the exit code, specific failure messages — rather than a general assurance. If tests cannot be run in the current environment for some reason, say so explicitly instead of implying they were run.

- **DON'T:** Report a specific coverage percentage, test count, or benchmark number that wasn't actually produced by running the corresponding tool. A fabricated-sounding-plausible number is worse than no number, because it's indistinguishable from a real one until someone checks.

### Overusing Brittle Snapshot Tests

- **DON'T:** Default to snapshot testing as the go-to strategy for verifying any function or component's output, rather than as a deliberate choice for the specific cases where the full output shape is genuinely what needs guarding. Reaching for a snapshot instead of thinking through what should actually be asserted produces tests that "cover" a lot of surface area while communicating almost nothing about intent to a future reader.

- **DO:** Write explicit, specific assertions for the fields and values that matter to the behavior under test, and reserve snapshots for the narrower set of cases (generated file formats, complex serialization) where comparing the whole structure at once is genuinely the right tool, keeping those snapshots as small and targeted as the case allows.

### Skipping Edge Cases and Error Paths

- **DON'T:** Generate tests that only cover the happy path — valid input, successful response — while skipping null/empty/boundary inputs, invalid input, concurrent access, and failure responses. Error-handling code is disproportionately where real bugs live, precisely because it's exercised far less often in normal operation than the happy path is, which makes it the last place a test suite should skip.

- **DO:** Explicitly work through the edge cases relevant to the function's input domain before considering the test set complete — empty collections, zero and negative numbers, values at the boundary of a valid range, null/undefined/missing fields, duplicate entries, extremely large inputs, timeouts, and permission/authorization failures — and write tests for the ones with real consequences if mishandled.
```text
For a function `applyDiscount(price, discountPercent)`, edge cases worth considering:
- discountPercent = 0 (no discount)
- discountPercent = 100 (fully discounted — is $0 valid?)
- discountPercent > 100 (invalid — should it throw or clamp?)
- price = 0 or negative (invalid input — how is it handled?)
- discountPercent as a non-numeric or null value
```

- **DON'T:** Assume that because the happy path is well tested, the function is well tested. A suite that is entirely happy-path tests can reach high coverage numbers while never once exercising the exact conditions under which the code is most likely to fail in production.

### Inflating Coverage With Low-Value Tests

- **DON'T:** Generate a large number of near-duplicate or trivial tests — one test per getter, minor rephrasings of the same scenario, tests that exercise the same branch repeatedly with cosmetically different inputs — as a way to raise a test count or coverage percentage rather than to cover genuinely distinct behavior. Volume is not the goal; each test should earn its place by covering a case the others don't.

- **DO:** Prioritize a smaller number of well-chosen tests that each cover a meaningfully distinct behavior, branch, or edge case over a larger number of tests that differ only superficially. When asked to "add more test coverage," look for untested logic paths and unhandled edge cases first, rather than padding the count with more variations on cases that are already covered.

### Fabricating or Guessing Test Data, APIs, and Assertions

- **DON'T:** Invent plausible-looking but unverified details when writing a test — a method name that "should" exist on a class, a response shape that "seems right" for an API, an expected value that "looks about right" for a calculation — without checking the actual implementation, documentation, or a known-correct source. A test built on a guessed contract can pass against a mock that shares the same guess and fail immediately against the real system, or worse, silently test the wrong thing forever.

- **DO:** Verify the real signature, return shape, and behavior of whatever is being tested (by reading the actual implementation or its documentation) before writing assertions against it, and use realistic, representative test data rather than arbitrary placeholder values that don't reflect the shapes the code will actually see in production (e.g., a name field tested only with `"test"` misses unicode, length limits, and special characters that real names contain).

### Ignoring Pre-Existing Flaky or Failing Tests

- **DON'T:** Treat an already-failing or already-flaky test encountered while working on unrelated code as someone else's problem to route around (skip it, ignore its output, or work only in a mental mode of "the rest of the suite is fine"). An existing red or flaky test is exactly as much a signal of real risk as a new one — noticing it and flagging it (or fixing it, if it's in scope) is part of doing the job responsibly, not optional cleanup.

- **DO:** Surface any pre-existing failing or flaky tests discovered incidentally while working in an area, rather than silently working around them. Even if fixing them isn't in scope for the current task, calling them out prevents them from being mistaken for newly broken tests later, and prevents a known problem from persisting simply because everyone assumed someone else was handling it.

## Quick Checklist
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
