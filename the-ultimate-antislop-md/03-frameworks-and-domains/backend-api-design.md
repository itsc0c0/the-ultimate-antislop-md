# Backend Frameworks & API Design

## General Backend Architecture

- **DO:** Separate the codebase into distinct layers — routing/controllers, business logic/services, and data access/repositories. Mixing SQL queries, validation, and HTTP response formatting in one function makes the code impossible to unit test in isolation and forces every change to be re-verified end-to-end.
```text
BAD:  route handler builds SQL, applies business rules, formats JSON
GOOD: controller -> service (business rules) -> repository (SQL) -> controller formats response
```

- **DON'T:** Put business logic directly in route handlers or controller methods. A controller stuffed with validation, branching business rules, and persistence calls ("fat controller") cannot be reused from a background job, a CLI script, or a second API version without copy-pasting the logic.
```javascript
// BAD: fat controller
app.post('/orders', async (req, res) => {
  if (!req.body.items?.length) return res.status(400).send('no items');
  const total = req.body.items.reduce((s, i) => s + i.price * i.qty, 0);
  if (total > 10000) { /* fraud check inline */ }
  const order = await db.query('INSERT INTO orders ...');
  await emailClient.send(...);
  res.json(order);
});

// GOOD: thin controller delegates to a service
app.post('/orders', async (req, res, next) => {
  try {
    const order = await orderService.create(req.body, req.user);
    res.status(201).json(order);
  } catch (err) { next(err); }
});
```

- **DO:** Keep services focused on one bounded area of responsibility rather than growing a single `UserService` or `AppService` that touches billing, notifications, auth, and reporting. A god service becomes a merge-conflict magnet and a dependency everything else is coupled to, which defeats the purpose of having services at all.

- **DON'T:** Let the data access layer leak framework-specific ORM objects all the way up to the HTTP response. Returning a raw ActiveRecord/Sequelize/Entity object lets internal columns (password hashes, internal flags, soft-delete markers) leak into API responses whenever a new column is added; map to an explicit DTO/serializer instead.

- **DO:** Use dependency injection (constructor injection, a DI container, or simple factory functions) instead of importing concrete singletons deep inside business logic. Code that does `import { db } from '../../db'` inside a service cannot be tested without a real database connection and cannot be swapped for a fake in tests.
```typescript
// BAD: hard dependency, cannot be mocked without a live DB
class OrderService {
  async create(data: OrderInput) {
    return db.orders.insert(data); // `db` imported as a global singleton
  }
}

// GOOD: dependency is injected, trivially mocked in tests
class OrderService {
  constructor(private readonly orders: OrdersRepository) {}
  async create(data: OrderInput) {
    return this.orders.insert(data);
  }
}
```

- **DON'T:** Reach for a full dependency-injection framework (e.g., a heavyweight IoC container) in a small service where three constructor parameters would do. Over-engineered DI adds indirection, XML/decorator configuration, and a learning curve that isn't repaid by a codebase with a handful of services.

- **DO:** Draw clear boundaries between layers with one-directional dependencies: controllers depend on services, services depend on repositories/gateways, and nothing depends back "up" the stack. A repository that calls back into a controller, or a model that sends HTTP responses, inverts the architecture and makes the flow of control unpredictable.

- **DON'T:** Let framework request/response objects (`req`, `res`, `HttpServletRequest`, Django's `request`) travel past the controller layer. Passing the raw request object into a service ties that service to the web framework, making it unusable from a queue consumer, a scheduled job, or a test that has no HTTP context.

- **DO:** Push cross-cutting concerns (authentication, request logging, rate limiting, CORS, compression) into middleware/interceptors rather than repeating them at the top of every handler. Centralizing them means a security fix or a new logging field is applied everywhere at once instead of requiring an audit of every route file.

- **DON'T:** Handle authorization checks ad hoc inside each handler with copy-pasted `if (user.role !== 'admin')` blocks. Inconsistent authorization logic scattered across dozens of handlers is exactly how one endpoint gets forgotten and becomes an access-control vulnerability; centralize authorization in a policy/guard layer that is unit-testable on its own.

- **DO:** Validate input at the boundary (the edge of the system) and treat everything past that boundary as trusted, well-typed data. Re-validating the same field's format in the controller, the service, and the repository is redundant defensive programming that hides where the real contract lives.

- **DON'T:** Let the same model class serve as the database entity, the request-input schema, and the response payload all at once. A single overloaded model forces awkward compromises — nullable fields that are only nullable during creation, `@JsonIgnore` scattered everywhere — where three purpose-built types (entity, input DTO, output DTO) would each stay simple.

- **DO:** Structure a backend project around features/domains (e.g., `orders/`, `billing/`, `users/`) rather than purely around technical layers (`controllers/`, `services/`, `models/`) once the codebase grows past a handful of resources. Feature-based structure keeps everything related to one capability in one place, so a change to "orders" touches one folder instead of five parallel folders.

- **DON'T:** Mix synchronous request-handling code with long-running or blocking operations (large file processing, sending bulk emails, calling a slow third-party API) inside the same request/response cycle. A request thread blocked on a 30-second external call ties up a worker/connection slot and degrades the whole service's throughput; offload it to a background job/queue and return an immediate acknowledgment (202 Accepted) or a webhook-driven completion.

- **DO:** Keep configuration (database URLs, API keys, feature flags, timeouts) out of code and out of version control, loaded from environment variables or a secrets manager, with sane defaults only for genuinely non-secret local-dev values. Hardcoded connection strings or credentials in source code become a security incident the moment the repository is made public or forked.

- **DON'T:** Let a single "utils" or "helpers" module become an unstructured dumping ground for unrelated business logic. A `utils.js` that contains date formatting, tax calculation, and email templating next to each other has no discoverable organization and nobody knows where new logic belongs; name modules for what they actually do (`taxCalculator`, `dateFormat`).

- **DO:** Design for statelessness in the request-handling layer — do not store per-request or per-user mutable state in module-level or class-level variables on the server process. State stored in server memory breaks the moment there is more than one server instance (horizontal scaling) or the process restarts, and requests from the same user can be load-balanced to a different instance.
```python
# BAD: global mutable state, breaks under multiple workers/instances
current_user_carts = {}  # keyed by user_id, lives in process memory

def add_to_cart(user_id, item):
    current_user_carts.setdefault(user_id, []).append(item)

# GOOD: state lives in a shared store (DB, Redis) reachable from any instance
def add_to_cart(user_id, item):
    redis_client.rpush(f"cart:{user_id}", json.dumps(item))
```

- **DON'T:** Assume the framework's default project layout (a fresh `rails new`, `django-admin startproject`, or `nest new`) needs no further structuring as the app grows. Frameworks bootstrap a starting point, not a scalable architecture; revisit module boundaries deliberately once a folder like `controllers/` or `views/` passes a few dozen files.

- **DO:** Make each layer fail loudly and explicitly rather than silently swallowing errors and returning defaults. A repository method that catches a database error and returns `null` (indistinguishable from "not found") hides a production outage as a routine empty result.

- **DON'T:** Let one microservice or module directly query another module's database tables "for convenience." Reaching across a module boundary into someone else's tables couples the two modules at the schema level — a column rename in one silently breaks the other with no compiler or test to catch it — defeating the entire point of the boundary; go through that module's public API/service interface instead.

- **DO:** Version internal contracts (service interfaces, event schemas, shared library APIs) just as deliberately as external ones once more than one team or process depends on them. Backend architecture problems are rarely about a single service; they're about how services stay compatible with each other as they change independently.

- **DON'T:** Treat "backend" as synonymous with "the database layer." Business rules, validation, orchestration, and policy decisions belong in the application/domain layer, not encoded as database triggers, stored procedures, or check constraints that are invisible to anyone reading the application code and hard to unit test or version alongside the code that depends on them (a light validation constraint for data integrity is fine; encoding business workflow there is not).

- **DO:** Keep controllers thin enough that a reviewer can read one and understand the request/response contract without reading the service it calls. A controller's job is translating between the transport protocol (HTTP, gRPC) and the application's internal calls — not deciding business rules.

- **DON'T:** Reinvent a pattern per feature. If the project already has an established service/repository pattern, a new "quick" endpoint that queries the database directly from the controller "just this once" erodes the architecture and gives the next contributor two conflicting examples to copy from.

- **DO:** Expose separate liveness and readiness health-check endpoints when running under an orchestrator (Kubernetes or similar): liveness answers "is this process healthy enough to keep running, or should it be restarted," while readiness answers "is this instance ready to receive traffic right now" (e.g., has it finished its startup warmup, can it currently reach its database). Conflating the two into one `/health` endpoint causes bad outcomes in both directions — a transient dependency blip can trigger unnecessary process restarts (if readiness failures are treated as liveness failures), or an instance that can't actually serve requests keeps receiving traffic (if a genuine liveness problem is masked as merely "not ready").

- **DON'T:** Let a service start accepting traffic before it has finished initializing its critical dependencies (database connections, cache warm-up, configuration loading). A readiness check that returns healthy immediately on process start, before the service can actually serve a request correctly, causes a wave of failed requests during every deploy or restart — gate readiness on the dependencies the service actually needs before declaring itself ready.

- **DO:** Handle shutdown signals (`SIGTERM`) gracefully — stop accepting new requests, finish in-flight requests, close database/queue connections cleanly, and only then exit — instead of letting the process be killed abruptly mid-request. An orchestrator routinely sends `SIGTERM` during routine deploys and autoscaling events, not just failures; a service that doesn't drain in-flight work on shutdown drops real requests on every ordinary deploy.
```javascript
// GOOD: drain in-flight requests before exiting on SIGTERM
process.on('SIGTERM', async () => {
  server.close(() => process.exit(0));  // stop accepting new connections, finish existing ones
  await db.closeGracefully();
});
```

- **DON'T:** Read configuration values ad hoc, scattered across the codebase, with no startup-time validation of required values. A missing or malformed required environment variable should fail the service at startup with a clear error, not surface hours later as a confusing runtime `undefined`/`null` deep inside a request handler that happened to be the first code path to touch that config value.

- **DO:** Validate configuration against an explicit schema at startup (required keys present, correct types, values within expected ranges) and fail fast with a clear error message naming exactly which configuration is missing or invalid, rather than starting up in a partially-configured state that fails unpredictably later.

- **DON'T:** Let feature flags accumulate indefinitely without an owner or a removal plan. A flag added for a specific rollout that's still being checked in a dozen places a year later, long after the rollout finished, is dead complexity that makes the actual code path harder to reason about — treat a feature flag's removal (once the rollout is complete and stable) as part of the same task that introduced it, not a someday cleanup.

- **DO:** Keep the core business/domain logic free of framework- and infrastructure-specific code (no ORM decorators, no HTTP-framework imports, no direct SQL) where the codebase's scale and longevity justify the investment — this is the essence of ports-and-adapters/hexagonal architecture: the domain layer defines what it needs (a `PaymentGateway` interface, an `OrderRepository` interface) and the framework-specific code plugs into those interfaces from the outside, rather than the domain layer depending directly on any particular framework or database. This isn't a default for every small service, but it pays off directly whenever the domain logic genuinely needs to be tested without a database, reused across different entry points (HTTP API, a CLI, a queue consumer), or ported to a different framework or datastore.

- **DON'T:** Let entity/model classes become pure data bags with all their behavior implemented externally in service classes that operate on their public fields (an "anemic domain model"), when the domain actually has meaningful behavior/invariants that belong with the data itself. A `User` model that's just getters and setters, with a `UserService.deactivate(user)` method that reaches in and flips a flag directly, misses the chance to have `user.deactivate()` enforce its own invariants (e.g., "can't deactivate a user with a pending order") in one place — this isn't a hard rule for every model, but a domain with real business rules benefits from encapsulating them near the data they constrain, rather than scattering "if user.status == X, then Y is allowed" checks across every service that touches that model.

## Transactions and Data Consistency at the API Layer

- **DO:** Scope a database transaction to exactly the operations that need atomicity together, and open it as late and close it as early as possible. A transaction that starts at the top of a request handler and stays open through unrelated reads, external API calls, or template rendering holds database locks far longer than necessary, directly reducing the throughput the database can sustain under concurrent load.

- **DON'T:** Make an external network call (a third-party API, another microservice) while holding a database transaction open. If that external call is slow or hangs, the transaction's locks stay held for the duration, potentially blocking every other request that touches the same rows — treat "call an external system" and "commit local database changes" as separate steps, using the outbox pattern (see Microservices Patterns) when both need to happen reliably.
```text
BAD:
  BEGIN TRANSACTION
    update order status to 'paid'
    call external shipping API to schedule delivery   <- slow/unreliable, holds the transaction open
  COMMIT

GOOD:
  BEGIN TRANSACTION
    update order status to 'paid'
    insert 'SchedulePickup' row into outbox table
  COMMIT
  -- a separate worker reads the outbox and calls the shipping API afterward, with its own retries
```

- **DO:** Understand the isolation level the application actually needs for a given operation, rather than accepting whatever the database driver's default happens to be without considering it. Read Committed (a common default) is sufficient for most CRUD operations; operations with real correctness requirements around concurrent modification (checking a balance and then decrementing it, reserving limited inventory) need either a stricter isolation level, explicit row locking (`SELECT ... FOR UPDATE`), or an application-level optimistic-concurrency check (see the `ETag`/`If-Match` pattern above) to prevent a race condition between the check and the write.

- **DON'T:** Assume "the code runs one request at a time in my testing" generalizes to "there's no race condition in production." Two nearly-simultaneous requests reading the same row, both deciding independently that an action is safe (e.g., both see 1 item left in stock and both proceed to sell it), is a routine occurrence under real concurrent traffic — write-then-read-then-decide logic on shared data needs an explicit concurrency-safety strategy, not an assumption that requests are serialized.

- **DO:** Make write endpoints that must succeed together (e.g., "debit account A and credit account B") use a single local database transaction when both rows live in the same database, and design deliberately (saga/outbox, as covered above) when they don't. Never rely on "the application usually completes both steps" as a substitute for an actual atomicity guarantee.

- **DON'T:** Let a single API request perform an unbounded number of database writes in a loop with no batching, especially inside one long-held transaction. A request that loops over 10,000 items and issues one `INSERT` per item, all inside one transaction, both holds locks disproportionately long and risks hitting transaction size/log limits — batch the writes (`INSERT ... VALUES (...), (...), (...)` or a bulk-insert API) instead.

## REST API Design

### Resource Naming and URL Structure

- **DO:** Name resource URLs with plural nouns that represent collections, not verbs. `GET /orders/42` reads as "get the order with id 42 from the orders collection"; `GET /getOrder?id=42` reinvents RPC-over-HTTP and abandons the uniform interface that makes REST predictable.
```text
BAD:  GET  /getUser?id=5
      POST /createUser
      POST /deleteUser?id=5

GOOD: GET    /users/5
      POST   /users
      DELETE /users/5
```

- **DON'T:** Mix verbs into resource paths for standard CRUD actions. If an operation doesn't fit CRUD (e.g., "activate this account"), model it as a sub-resource or state transition (`POST /accounts/5/activation`) rather than a verb hanging off the collection (`POST /accounts/activate/5`).
```text
BAD:  POST /orders/5/cancel        (a verb bolted onto the resource path)

GOOD: POST /orders/5/cancellation  (creating a "cancellation" sub-resource/event —
                                     reads as a noun, fits the resource-oriented model,
                                     and the cancellation record itself can carry
                                     its own fields: reason, cancelled_by, cancelled_at)
```

- **DO:** Nest resources to express genuine ownership or containment, and keep nesting shallow (one or two levels). `GET /authors/3/books` reads naturally because books genuinely belong to an author; nesting five levels deep (`/orgs/1/teams/2/projects/3/tasks/4/comments/5`) makes URLs brittle and forces clients to know the entire ancestor chain just to fetch a comment.
```text
BAD:  GET /orgs/1/teams/2/projects/3/tasks/4/comments/5
GOOD: GET /comments/5              (comment is globally addressable by id)
      GET /tasks/4/comments        (list comments scoped to a task)
```

- **DON'T:** Expose database primary keys as the only way to address a resource when a more stable or human-meaningful identifier exists (slug, UUID intended for public use). Sequential integer IDs in URLs leak the approximate row count and creation order of a table and make ID enumeration trivial (`/orders/1001`, `/orders/1002`, ...).

- **DO:** Use lowercase, hyphen-separated path segments consistently (`/order-items`, not `/orderItems` or `/Order_Items`). URL casing conventions are arbitrary, but consistency lets developers guess a URL correctly instead of checking documentation for every endpoint.

- **DO:** Model a genuine singleton resource (one that has no plural — the current user's profile, an account's single billing configuration) as a singular path with no ID (`GET /account/billing-config`), rather than forcing it into the plural-collection convention with a synthetic ID (`GET /billing-configs/1`). Naming a resource that will only ever have exactly one instance as if it were a member of a collection is a common, avoidable inconsistency that leaks an implementation detail (there happens to be a row with ID 1) into the URL design.
```text
BAD:  GET /users/5/profile-settings/1     (there is only ever one; "1" is meaningless)
GOOD: GET /users/5/profile-settings       (singular, no synthetic id)
```

- **DON'T:** Encode an action or a return format into the path when an HTTP method or header already exists for that purpose (`/users/5/xml`, `/getUsersAsJson`). Content negotiation (`Accept` header) and HTTP verbs already solve this; duplicating it in the URL creates two sources of truth that can disagree.

### HTTP Methods and Status Codes

- **DO:** Match HTTP methods to their defined semantics: `GET` for safe, cacheable reads with no side effects; `POST` for creating a resource or triggering a non-idempotent action; `PUT` for a full replace of a resource; `PATCH` for a partial update; `DELETE` for removal. A client, proxy, or browser is allowed to assume `GET` is safe to retry, cache, and prefetch — violating that (e.g., a `GET /logout` that ends a session) causes real, hard-to-diagnose bugs when a crawler or prefetcher hits it.
```text
BAD:  GET /users/5/delete       (a GET that mutates state)
GOOD: DELETE /users/5
```

- **DON'T:** Use `POST` for every operation regardless of semantics because it's "simpler." An API where every single endpoint is `POST /doSomething` throws away caching, idempotency guarantees, and the ability for tooling (browsers, API gateways, HTTP libraries) to reason about the request — it becomes RPC wearing an HTTP costume.

- **DO:** Return status codes that accurately describe the outcome, not just `200` for success and `500` for anything else. `201 Created` for a successful creation (with a `Location` header pointing at the new resource), `204 No Content` for a successful action with no body, `400` for a malformed request, `401` for missing/invalid authentication, `403` for authenticated-but-forbidden, `404` for a resource that doesn't exist, `409` for a conflict (e.g., duplicate unique key), `422` for semantically invalid input that parsed fine syntactically.
```text
BAD:  200 OK  { "error": "email already taken" }
GOOD: 409 Conflict  { "error": "email already taken" }
```

- **DON'T:** Return `200 OK` with an error message in the response body. This forces every client to parse the body just to know whether the request succeeded, defeats HTTP-level error handling (retries, circuit breakers, monitoring dashboards that key off status codes), and is one of the most common "it works but it's slop" mistakes in hand-rolled and AI-generated APIs alike.

- **DO:** Distinguish `401 Unauthorized` (you are not authenticated, or your credentials are invalid/expired) from `403 Forbidden` (you are authenticated, but you don't have permission for this action). Conflating them either leaks information about which resources exist to unauthenticated users, or confuses legitimate users about whether they need to log in again versus request access.

- **DON'T:** Return `404 Not Found` for a resource that exists but the caller isn't authorized to see, when the API's security model calls for hiding existence (e.g., a private repository). Conversely, don't blanket-return `403` for genuinely nonexistent resources when there's no reason to hide their nonexistence — pick one policy per resource type and apply it consistently, and document which one you chose.

- **DO:** Use `422 Unprocessable Entity` (or `400`, per the project's established convention) for validation failures, and make the choice consistently across the whole API rather than picking whichever status code a given endpoint's author preferred that day.

- **DON'T:** Invent nonstandard status codes or repurpose codes to mean something other than their defined meaning (e.g., using `418` as a generic "business rule failed" code, or `450` for "quota exceeded"). Clients, proxies, CDNs, and monitoring tools all have baked-in expectations for the standard code ranges; a custom code either gets silently mis-handled or requires everyone downstream to special-case your API.

- **DO:** Use `429 Too Many Requests` for rate limiting, with a `Retry-After` header telling the client when it can try again. This is the one standardized signal well-behaved clients and libraries already know how to back off on.

- **DON'T:** Return `500 Internal Server Error` for client mistakes (bad input, missing auth, resource not found). A `500` should mean "we have a bug or an outage," and mixing client errors into that bucket makes server-side alerting noisy and useless — every bad request from a misbehaving client pages someone for nothing.

### Idempotency

- **DO:** Make `PUT`, `DELETE`, `GET`, `HEAD`, and `OPTIONS` genuinely idempotent — calling them N times has the same effect on server state as calling them once. A client (or a retry layer, or a flaky mobile network) that resends a `PUT` after a timeout should not corrupt data by applying the change twice.

- **DON'T:** Implement `PUT` as "increment a counter" or any other operation whose result changes with each call. If an operation isn't naturally idempotent, it doesn't belong behind `PUT`; model it as a `POST` action instead and don't claim idempotency the method contract promises but the implementation doesn't honor.

- **DO:** Support client-supplied idempotency keys (an `Idempotency-Key` header) for non-idempotent `POST` operations that have real-world consequences if duplicated — payments, order creation, sending a notification. The server stores the key with the result of the first request and returns the same result for a retried request with the same key, instead of double-charging a card because a client retried a request that actually succeeded but timed out on the way back.
```text
POST /payments
Idempotency-Key: 7b1a1c1e-2f3a-4e0d-9b7a-1e6f2c9d0a11

-> first call: charges the card, stores the result under this key
-> retried call with same key: returns the stored result, does NOT charge again
```

- **DO:** Scope stored idempotency keys to the combination of the client/API key and the key value itself (not the key value alone), store the request's fingerprint (e.g., a hash of the request body) alongside the result, and reject a retried request whose key matches but whose body doesn't — with a `409 Conflict` or `422` — rather than silently returning a stale result for what is actually a different request. Otherwise, two unrelated requests (from different clients, or the same client reusing a key by mistake) that happen to share an idempotency key would incorrectly get treated as the same operation.

- **DON'T:** Store idempotency keys and their results forever with no expiration. Cap the retention window (commonly 24 hours to a few days — long enough to cover realistic retry scenarios, short enough to bound storage growth) and document it, since an idempotency key store with unbounded retention grows without limit and a key reused far outside any realistic retry window is more likely a client bug (or a client reusing a UUID generator incorrectly) than a legitimate retry.

- **DON'T:** Assume a client will only ever call an endpoint once. Networks drop responses after the server has already committed a write; design write endpoints for "at least once delivery" as the norm, not the edge case.

### Versioning

- **DO:** Version the API deliberately from the start, using whichever strategy the project has standardized on — a URL prefix (`/v1/orders`), a header (`Accept: application/vnd.myapi.v1+json`), or a query parameter — and apply it consistently across every endpoint. Inconsistent versioning (some endpoints under `/v1`, others not) forces every client integration to special-case your API's history.

- **DON'T:** Introduce a breaking change (removing a field, changing a field's type, changing an endpoint's required parameters, changing status-code semantics) inside an existing API version. Breaking changes belong in a new version; consumers who pinned to `v1` should be able to keep working indefinitely (or until a documented, communicated deprecation date) without their code breaking overnight.

- **DO:** Prefer additive, backward-compatible changes within a version — new optional fields, new endpoints, new optional query parameters. A well-designed schema lets you add a field to a response without bumping the version, because existing clients simply ignore fields they don't recognize.

- **DON'T:** Bump the major API version for every small change. Constant version churn forces clients to constantly re-integrate and signals instability; reserve version bumps for genuine breaking changes and communicate a deprecation timeline for the old version rather than dropping it abruptly.

- **DO:** Publish a deprecation policy and a sunset header (`Sunset`, `Deprecation`) when retiring an old API version, giving consumers a concrete timeline rather than silently degrading or removing the old version.

- **DO:** Roll out a genuinely risky API change gradually — behind a feature flag scoped to a percentage of traffic or a specific set of consenting clients, or by mirroring ("shadowing") production traffic to the new implementation and comparing its responses against the old one without actually serving the new responses yet — before cutting every client over at once. This catches a behavioral regression against real production traffic patterns while the blast radius is still small, rather than discovering it only after every consumer is already depending on the new behavior.

- **DON'T:** Treat "it passed in staging" as equivalent to "it's safe for every production consumer." Staging traffic rarely reproduces the full diversity of real client versions, edge-case inputs, and request volume a production API actually receives; a gradual, monitored rollout is what catches the gap between "works in staging" and "works for every real caller," especially for a change that's easy to get subtly wrong (a changed default, a reordered validation rule).

### Pagination

- **DO:** Paginate every list endpoint from day one, even if the dataset is small today. A `GET /users` that returns every row works fine in development with 50 test rows and falls over in production with 500,000, and retrofitting pagination onto an API clients already depend on is a breaking change.
```text
BAD:  GET /orders  ->  returns all 400,000 rows in one response
GOOD: GET /orders?page=2&limit=50
      GOOD: GET /orders?cursor=eyJpZCI6MTIzfQ&limit=50
```

- **DON'T:** Default `limit`/`page_size` to "unlimited" when the client omits it. Always apply a sane default (e.g., 20 or 50) and a hard maximum the client cannot override past, so a forgotten query parameter can't accidentally trigger a full-table scan and response payload.

- **DO:** Prefer cursor-based (keyset) pagination over offset-based pagination for large or frequently-changing datasets. Offset pagination (`?page=400&limit=50`) gets slower as the offset grows (the database still has to scan and discard all preceding rows) and produces skipped or duplicated items when rows are inserted or deleted between page requests; cursor pagination stays O(1)-ish per page and is stable under concurrent writes.

- **DON'T:** Return pagination metadata inconsistently across endpoints — one endpoint wrapping results in `{ data, meta: { page, total } }`, another returning a bare array with `X-Total-Count` in a header, a third using `{ items, next_cursor }`. Pick one envelope shape for the whole API and use it everywhere; a client integrating against ten endpoints shouldn't have to special-case pagination parsing for each one.

- **DO:** Include enough metadata for the client to know whether more pages exist (`has_more`, `next_cursor`, or `total_count` plus `page`/`limit`) without requiring an extra request just to find out. A response the client can't tell is complete forces defensive "keep paginating until empty" loops that are wasteful and error-prone.

- **DO:** Return `200 OK` with an empty array (or an empty `data` array inside the pagination envelope) for a list endpoint that legitimately matches zero results, and reserve `404 Not Found` for when the parent resource in the path itself doesn't exist (`GET /users/999/orders` where user 999 doesn't exist at all) — an empty result set is a completely valid, successful response to a filter/list query, not an error condition.

- **DON'T:** Return `404` for an empty list result. Conflating "the query matched nothing" with "the resource doesn't exist" forces client code to special-case what should be the simplest possible response to handle, and is a frequent, easy-to-overlook inconsistency between similar endpoints in the same API.

### Filtering, Sorting, and Field Selection

- **DO:** Support filtering and sorting through well-documented query parameters (`?status=active&sort=-created_at`) rather than requiring the client to fetch everything and filter client-side. Client-side filtering of a large list is exactly the kind of unnecessary payload bloat and wasted database work that pagination is meant to avoid in the first place.

- **DON'T:** Let filter query parameters map directly, unvalidated, onto raw database column names or raw SQL fragments. Accepting an arbitrary `sort` value and interpolating it straight into an `ORDER BY` clause is a SQL-injection vector and also an accidental way to expose internal column names; validate against an explicit allow-list of sortable/filterable fields.
```javascript
// BAD: unvalidated sort parameter interpolated into SQL
const sql = `SELECT * FROM orders ORDER BY ${req.query.sort}`;

// GOOD: allow-list validated, parameterized
const ALLOWED_SORT_FIELDS = new Set(['created_at', 'total', 'status']);
if (!ALLOWED_SORT_FIELDS.has(sortField)) throw new BadRequestError('invalid sort field');
```

- **DO:** Offer sparse fieldsets (`?fields=id,name,email`) for endpoints whose full representation is large, so clients that only need a few fields aren't forced to pay the bandwidth and serialization cost of the whole object — but only if the project's scale genuinely warrants the added complexity.

- **DON'T:** Design filter syntax that's ambiguous or overloaded (e.g., `?status=active,pending` sometimes meaning OR and sometimes meaning "the literal string") without documenting the exact semantics. Ambiguous query syntax leads every client to guess differently and produces bug reports that are really documentation gaps.

### Request/Response Design and HATEOAS

- **DO:** Keep response payloads consistent in shape across similar endpoints — the same field naming convention (`snake_case` or `camelCase`, pick one), the same date format (ISO 8601), the same null-handling (omit vs. explicit `null`) throughout the whole API. Consistency lets client code use one deserialization strategy across the whole surface instead of writing per-endpoint parsing quirks.

- **DON'T:** Mix naming conventions within the same API (`created_at` next to `updatedAt` in the same object, or `userId` in one endpoint and `user_id` in another). This is one of the fastest ways an API reads as slop — it signals no one is enforcing a shared style, and it will keep multiplying as more endpoints get added by different contributors (or different AI sessions with no shared memory of the convention).

- **DO:** Be aware of HATEOAS (Hypermedia as the Engine of Application State) as one of REST's original constraints — responses can include links to related actions/resources (`_links: { self, next, cancel }`) so a client discovers available transitions instead of hardcoding URL templates. Most production APIs today are pragmatically "REST-ish" rather than fully hypermedia-driven, and that's a legitimate choice — but know the tradeoff you're making rather than assuming plain JSON-over-HTTP is "REST" by definition.

- **DON'T:** Claim an API is "RESTful" as a badge without understanding what constraint that name actually implies (statelessness, uniform interface, cacheability, layered system, and optionally HATEOAS). Calling a collection of `POST`-only RPC-style JSON endpoints "REST" isn't wrong in casual usage, but conflating it with the real constraints in design discussions leads to design decisions (e.g., ignoring cacheability) that a genuinely REST-constrained design would have caught.

- **DO:** Return the created or updated resource's representation in the response body for `POST`/`PUT`/`PATCH` (not just a success flag), including its assigned ID and server-generated fields (timestamps, computed fields). This saves the client an immediate follow-up `GET` and is what most client SDKs and codegen tools expect.

- **DON'T:** Return wildly different response shapes for the "list" and "detail" views of the same resource type without a documented reason (e.g., the list omits large fields for performance — that's fine, but document it). A client that fetches `/orders` and then `/orders/5` expecting the same shape, minus what's obviously list-specific, will break in confusing ways if fields are renamed or restructured between the two.

- **DO:** Design error responses with a consistent, structured shape across the whole API — a machine-readable error code, a human-readable message, and (for validation errors) a list of field-level problems. See the Validation & Error Responses section below for the full pattern.

- **DON'T:** Leak stack traces, SQL error text, internal file paths, or framework version banners in API error responses, even in a "just for now" debug build that ships to production. This is both an information-disclosure security issue and a reliability signal to attackers about which framework/version/vulnerabilities to target.

### CORS, Multi-Tenancy, and Soft Deletes

- **DO:** Configure CORS with an explicit allow-list of trusted origins for any endpoint that accepts credentials (cookies, `Authorization` headers sent by a browser-based client), and understand that `Access-Control-Allow-Origin: *` combined with `Access-Control-Allow-Credentials: true` is invalid and rejected by browsers for good reason — a wildcard origin accepting credentials would let any website read another site's authenticated API responses on a user's behalf.
```text
BAD:  Access-Control-Allow-Origin: *
      Access-Control-Allow-Credentials: true      (browsers reject this combination)

GOOD: Access-Control-Allow-Origin: https://app.example.com   (explicit, echoed from an allow-list)
      Access-Control-Allow-Credentials: true
      Vary: Origin
```

- **DON'T:** Reflect the request's `Origin` header back verbatim as `Access-Control-Allow-Origin` for every request without checking it against an allow-list first. Naively echoing whatever origin the browser sent effectively disables the same-origin protection CORS exists to provide — validate the origin against a known, trusted list before echoing it.

- **DO:** Pick one consistent strategy for identifying the tenant in a multi-tenant API (a subdomain, a path prefix, a header, or a claim inside the auth token) and enforce tenant isolation at the data-access layer, not just by trusting whichever tenant identifier the request happens to carry. Every query, cache key, and background job triggered by a request needs to be scoped to the authenticated tenant, derived server-side — never solely from a client-supplied tenant ID.

- **DON'T:** Let a multi-tenant API accept a tenant ID as a plain request parameter and trust it without verifying the authenticated identity actually belongs to that tenant. This is a specific, common instance of the broken-object-level-authorization problem (see Common AI-Assistant Mistakes below) and, in a multi-tenant system, one of the most severe possible bugs — it can leak one customer's entire dataset to another.

- **DO:** Model deletion deliberately: decide per resource whether a `DELETE` performs a genuine hard delete, a soft delete (a `deleted_at` timestamp, excluded from normal queries but recoverable), or is disallowed in favor of an explicit archival/state-transition action, and document which. A soft-deleted resource that a naive `GET /orders/{id}` still returns (because the query forgot to filter `deleted_at IS NULL`) is a common and confusing bug class.

- **DON'T:** Return `200 OK`/`204 No Content` from a `DELETE` on a soft-delete resource and then keep serving that same resource from other endpoints (list views, search, related-resource lookups) as if nothing happened. Every read path needs to consistently respect the soft-delete flag, not just the one endpoint that happened to be tested.

### Response Envelope Consistency and Search Endpoints

- **DO:** Decide once, for the whole API, whether list responses are a bare JSON array or wrapped in an envelope object (`{ "data": [...], "meta": {...} }`), and apply that decision uniformly. An envelope has real advantages (room for pagination metadata, room to add top-level fields later without a breaking change) but either choice is defensible — what's not defensible is a mix, forcing every client integration to special-case each endpoint's response shape.
```json
// Pick ONE convention and use it everywhere:
{ "data": [ { "id": 1, "name": "..." } ], "meta": { "total": 240, "page": 1 } }
// -- not this shape on one endpoint and a bare array on another --
[ { "id": 1, "name": "..." } ]
```

- **DON'T:** Add a new top-level field to a bare-array list response later by wrapping some-but-not-all list endpoints in an envelope retroactively. That's a breaking change for exactly the endpoints that get wrapped (clients expecting an array now receive an object) — if there's a real chance list responses will need metadata later, choose the envelope shape from the start.

- **DO:** Model complex search (multi-field, free-text, or requiring a large/complex query payload that doesn't fit comfortably in a query string) as `POST /resource/search` (or `POST /resource/_search`) accepting a structured JSON body, when GET-with-query-params genuinely becomes unwieldy — this is a well-understood, common exception to "GET for reads," and worth documenting explicitly as intentional so it doesn't read as an accidental inconsistency.

- **DON'T:** Cram an arbitrarily complex, deeply nested filter/query language into GET query-string parameters just to preserve "GET for search" purity once the filter structure genuinely no longer fits that shape. A query string with URL-encoded nested JSON crammed into a single parameter value is harder to read, harder to validate, and hits URL-length limits sooner than a well-designed `POST` search body — recognize when the resource-and-verbs model has been outgrown for a specific case and use the documented escape hatch instead of forcing it.

### Backward Compatibility and Change Management

- **DO:** Treat a field's removal, a field's type change, a required-field addition to an existing request schema, and a change in a status code's meaning as breaking changes requiring the same care as removing an endpoint entirely. It's easy to underestimate how "small" a change like narrowing a string field to an enum, or making an optional field required, actually is from a consumer's point of view — any of these can break a client that was working correctly against the previous contract.

- **DON'T:** Ship a breaking change behind a vague, undocumented assumption that "no one is using that field/endpoint anyway." Verify with real usage data (access logs, field-usage metrics on requests/responses) before removing or changing anything a client might depend on — for a public or widely-integrated API, assume it's in use unless proven otherwise.

- **DO:** Monitor actual field and endpoint usage (which clients call which endpoints, which optional fields are actually populated in requests) before deprecating or removing anything, so the decision to deprecate is based on real evidence rather than a guess about what clients need.

- **DON'T:** Deprecate a field or endpoint without a clear migration path to its replacement. A deprecation notice that says "this will be removed" with no equivalent alternative documented forces every consumer to reverse-engineer what they're supposed to switch to — pair every deprecation with the specific replacement and, ideally, a concrete timeline.

- **DO:** Run old and new versions of a changed contract in parallel during a migration window when feasible (an old field alongside its replacement, both populated, for a transition period) rather than cutting over instantly, giving consumers time to migrate on their own schedule instead of being forced to update in lockstep with the API's release schedule.
```text
Migration window example:

v1 (current):   { "status": "A" }              // opaque single-letter code
v1 (transition): { "status": "A", "status_label": "active" }   // both present
v2 (future):    { "status": "active" }          // old field finally removed,
                                                  // once usage monitoring shows
                                                  // "status" (the old field) is no
                                                  // longer being read by any client
```

- **DON'T:** Assume semantic versioning of the API surface itself works exactly like semantic versioning of a code library. An API's "breaking change" surface includes things a library's semver often doesn't have to consider — response timing/latency characteristics some clients may implicitly depend on, rate-limit behavior, even error message wording if clients pattern-match on it — so a "safe" minor bump for a library isn't automatically safe for an API with unknown, unaudited consumers.

- **DO:** Use the expand-and-contract pattern for a database schema change that an API response is derived from, when the API and the database need to change together without a synchronized downtime deploy: first expand (add the new column/table alongside the old, backfill it, deploy code that writes to both old and new), then migrate every reader over to the new shape, and only then contract (remove the old column, once nothing reads it anymore). This lets each step deploy independently and be rolled back independently, instead of requiring one atomic cutover where the API, the application code, and the database schema all have to change in perfect lockstep.
```text
Expand-and-contract for renaming a column (e.g. "full_name" -> "display_name"):

1. EXPAND:    add display_name column; write to both full_name and display_name
2. BACKFILL:  populate display_name for existing rows
3. MIGRATE:   switch reads (API responses, internal queries) over to display_name
4. VERIFY:    confirm nothing still reads full_name (usage monitoring, as covered
              under Backward Compatibility above)
5. CONTRACT:  stop writing to full_name, then drop the column in a later deploy
```

- **DON'T:** Combine a database schema change and the API-facing contract change that depends on it into one single deploy when the two could instead be decoupled with expand-and-contract. A combined deploy means any problem discovered in either half forces rolling back both together, and it usually requires a moment where old application code and new schema (or new application code and old schema) briefly coexist during a rolling deployment — a coexistence window that expand-and-contract handles safely by design, and an all-at-once cutover often doesn't.

### Miscellaneous HTTP Semantics

- **DO:** Support `HEAD` on any resource that supports `GET`, returning the same headers a `GET` would (including `Content-Length`) with no response body — this lets a client cheaply check whether a resource exists or has changed without transferring its full payload, and comes for free from most frameworks when `GET` is implemented correctly.

- **DON'T:** Skip implementing a correct `OPTIONS` response (used by browsers for CORS preflight requests and by API tooling to discover allowed methods) or hardcode an `Allow` header that doesn't match what the route actually supports. A preflight `OPTIONS` request that fails or returns a stale set of allowed methods breaks legitimate cross-origin requests in ways that are confusing to debug from the client side, since the actual `POST`/`PUT`/etc. request never even gets sent.

- **DO:** Support `Range` requests (returning `206 Partial Content`) for endpoints serving large payloads — file downloads, video/audio streams, large export files — so clients can resume an interrupted download or request only the bytes they need, rather than forcing a full re-download on any interruption.

- **DON'T:** Ignore response compression for text-based payloads (JSON, especially large list responses) when the deployment stack supports it. Enabling gzip/brotli compression at the server or reverse-proxy layer for API responses is close to free performance for any client that sends `Accept-Encoding` — the bandwidth savings on a large JSON payload are often substantial with no code change to the application logic itself.

- **DO:** Keep trailing-slash behavior consistent and decide deliberately whether `/orders` and `/orders/` are treated as the same route or whether one redirects to the other — an inconsistent framework default (some routers treat them as distinct routes, others normalize automatically) is a small but real source of confusing 404s if the convention isn't applied uniformly across the whole API.

- **DON'T:** Use inconsistent syntax for array-valued query parameters across different endpoints of the same API — `?tags=a&tags=b` on one endpoint, `?tags=a,b` on another, `?tags[]=a&tags[]=b` on a third. Pick one array-parameter convention for the whole API (most frameworks have an idiomatic default) and document it, rather than letting each endpoint's author pick whichever their framework happened to parse correctly during testing.

### Data Representation Conventions

- **DO:** Represent monetary amounts as integers in the smallest currency unit (cents, not dollars) or as an exact decimal type, and always pair the amount with an explicit currency code (ISO 4217, e.g. `"USD"`). Floating-point representations of money accumulate rounding errors across arithmetic operations, and an amount with no currency field is ambiguous the moment the system supports more than one currency.
```json
// BAD: floating point dollars, no currency
{ "price": 19.99 }

// GOOD: integer minor units, explicit currency
{ "price_cents": 1999, "currency": "USD" }
```

- **DON'T:** Mix currency/unit representations across endpoints — one endpoint returning `price` in dollars as a float, another returning `amount_cents` as an integer, a third returning a formatted string like `"$19.99"`. Every client integrating against more than one endpoint has to special-case each representation; standardize one convention API-wide.

- **DO:** Represent all timestamps in UTC, formatted as ISO 8601 (`2026-09-04T18:22:10Z`), and document explicitly whether a field is a point-in-time instant or a timezone-naive local date (e.g., a birthdate, which should generally NOT carry a timezone/time-of-day component at all). Sending timestamps in an unspecified or server-local timezone forces every client to guess the offset, producing off-by-hours bugs that only show up for users in certain timezones or during DST transitions.

- **DON'T:** Send or accept Unix epoch timestamps as bare numbers without documenting the unit (seconds vs. milliseconds) — an undocumented epoch number is a frequent source of "the date is showing as the year 48000" bugs when a millisecond value is parsed as seconds or vice versa. Prefer ISO 8601 strings for API payloads; if a numeric epoch is used, document the unit explicitly and apply it consistently.

- **DO:** Use string enums (`"status": "pending"`) rather than bare integers (`"status": 2`) for any field with a fixed set of meaningful values in a JSON API. A string enum is self-documenting in every request/response log and debugging session; an integer code requires the reader to already know (or go look up) what `2` means, and silently shifts meaning if the underlying list of values is ever reordered.

- **DON'T:** Reorder or renumber the underlying values of an integer-backed enum once any client depends on the numeric values (common in database-backed status columns exposed directly through the API). Reordering silently changes the meaning of every previously-stored value from every client's perspective; if integers must be used at the storage layer, keep the API-facing representation as a stable string and translate at the boundary.

- **DO:** Choose identifier formats deliberately — auto-incrementing integers are simple and compact but leak ordering/volume information and enable enumeration (see Resource Naming above); UUIDs avoid that but are larger and non-sequential (which can hurt database index locality at very large scale); prefixed, sortable identifiers (e.g., `ord_01H8X...`) are a common middle ground that stays human-scannable while remaining non-guessable and self-describing about which resource type an ID belongs to.

- **DON'T:** Mix identifier formats for the same resource type across different parts of the API (a numeric `id` field and a separate `uuid` field maintained in parallel, both accepted interchangeably at different endpoints) without a clear, documented reason and migration plan. Two valid identifiers for the same resource, with inconsistent support across endpoints, is a frequent source of confusing "why does this ID work here but not there" integration bugs.

- **DO:** Serialize 64-bit integers as JSON strings, not bare JSON numbers, when a value can exceed roughly 2^53 (a database bigint/snowflake ID, a large counter). JSON's number type is defined in terms of IEEE 754 double-precision floats, and JavaScript (along with any JSON parser that maps numbers onto a native double) silently loses precision above `Number.MAX_SAFE_INTEGER` — a large ID sent as a bare number can come back from a JavaScript client subtly altered, with no error raised anywhere in the pipeline.
```json
// BAD: a 64-bit ID as a bare JSON number — a JS client may receive
// 9007199254740993 as 9007199254740992 with no error or warning
{ "transaction_id": 9007199254740993 }

// GOOD: serialized as a string, precision preserved through every JSON parser
{ "transaction_id": "9007199254740993" }
```

- **DON'T:** Assume every consumer of the API is aware of, or immune to, the JSON large-integer precision problem just because your own primary client (e.g., a backend-to-backend Java or Python integration) happens to handle 64-bit integers correctly. Any browser-based or JavaScript-based consumer of the same API is exposed to the same silent truncation risk — treat this as a serialization contract issue for the API as a whole, not something to evaluate against only one known client.

- **DO:** Represent boolean fields with an unambiguous, consistently-applied naming convention (a clear `is_`/`has_` prefix, e.g. `is_active`, `has_verified_email`) and use actual JSON booleans (`true`/`false`), never `"1"`/`"0"` strings or bare `1`/`0` integers standing in for a boolean. A field simply named `active` without a `is_`/`has_` convention is ambiguous about whether it's a boolean, a status enum, or something else entirely without checking the documentation.

- **DO:** Distinguish an explicitly-set `null` from an omitted field in `PATCH` payloads and JSON responses when the difference matters (e.g., "clear this field" vs. "don't touch this field"), and document which convention the API uses — some APIs use a sentinel or a dedicated "unset" wrapper type specifically because plain JSON's `null` is ambiguous between "the value is null" and "this key wasn't provided" once you're inside a partial-update semantics.

- **DON'T:** Return inconsistent null-handling across endpoints — one endpoint omitting a field entirely when it has no value, another returning it as explicit `null`, a third returning an empty string. Pick one convention per field type and apply it uniformly, since inconsistent null-handling is one of the more tedious classes of bugs for API consumers to work around.

### Designing for Bandwidth-Constrained and Offline-Capable Clients

- **DO:** Offer a delta/sync endpoint (`GET /orders?updated_since=<cursor>`) for clients that need to keep a local copy of a dataset reasonably fresh (a mobile app with offline support, a desktop sync client) rather than forcing them to re-fetch the entire collection on every sync. A full-collection re-fetch on every sync wastes bandwidth proportional to total dataset size instead of proportional to what actually changed, which becomes painful quickly as the dataset grows.

- **DON'T:** Assume every client has the same bandwidth and latency characteristics as the team's own development environment. A response shape and payload size that feels fine over a fast office connection can be a genuinely poor experience on a slow or intermittent mobile connection — this is exactly the kind of divergence a BFF (see Microservices Patterns above) or sparse fieldsets (see Filtering, Sorting, and Field Selection above) exist to address.

- **DO:** Design delta/sync responses to include enough information for the client to reconcile deletions, not just creates and updates — a naive "give me everything changed since X" endpoint that only reports upserts leaves a client with no way to learn that a record was removed, and its local copy silently drifts from the server's truth over time.

- **DON'T:** Let a sync/delta endpoint's cursor be based on wall-clock time alone with no tie-breaking for records that changed within the same timestamp granularity. Two records updated in the same millisecond (or the same second, depending on the timestamp precision used) can be ambiguous about which side of the "since" boundary they fall on; use a monotonically increasing cursor (a sequence number, a combination of timestamp and ID) that's guaranteed not to have ties.

## GraphQL

- **DO:** Design the GraphQL schema around the domain's actual entities and relationships, not as a thin passthrough of database tables. A schema that mirrors table names and foreign-key columns 1:1 exposes internal storage decisions and makes future schema refactors (splitting a table, renaming a column) into breaking GraphQL changes.

- **DON'T:** Model every operation as a query with side effects, or bury mutations inside query resolvers "because it was convenient." Reads belong in `Query`, writes belong in `Mutation` — GraphQL clients, caching layers, and tooling (like persisted queries and automatic retries) rely on that separation to know which operations are safe to retry or cache.

- **DO:** Design mutations to accept a single structured `input` type and return a payload object that includes both the changed data and any errors, following the common `xInput` / `xPayload` convention. This keeps mutation signatures stable as fields are added and gives clients one predictable place to check for partial failures.
```graphql
# BAD: loose scalar arguments, boolean success flag
type Mutation {
  updateOrder(id: ID!, status: String!): Boolean
}

# GOOD: structured input/payload, room to grow
input UpdateOrderInput {
  orderId: ID!
  status: OrderStatus!
}
type UpdateOrderPayload {
  order: Order
  errors: [UserError!]!
}
type Mutation {
  updateOrder(input: UpdateOrderInput!): UpdateOrderPayload!
}
```

- **DO:** Name mutations with a consistent `verbNoun` convention (`createOrder`, `updateOrder`, `cancelOrder`) applied uniformly across the whole schema, rather than mixing conventions (`orderCreate`, `updateOrder`, `order_cancel`) depending on who wrote which resolver. Since GraphQL exposes one flat namespace of mutations with no per-resource grouping the way REST's path structure provides for free, a consistent naming convention is the only thing keeping a large schema's mutation list scannable and predictable.

- **DON'T:** Design the N+1 query problem into resolvers by naively fetching related data inside a per-item resolver with no batching. A query that asks for 100 posts and each post's author will, without batching, issue 1 query for the posts and 100 separate queries for authors — a query planner cost that scales linearly with result size and can silently take down a database under real traffic.
```javascript
// BAD: N+1 — one DB call per post inside the field resolver
const resolvers = {
  Post: {
    author: (post) => db.users.findOne({ id: post.authorId }), // called once per post
  },
};

// GOOD: batched with a DataLoader, one query for all authors in the request
const userLoader = new DataLoader(async (ids) => {
  const users = await db.users.findMany({ id: { in: ids } });
  return ids.map((id) => users.find((u) => u.id === id));
});
const resolvers = {
  Post: {
    author: (post) => userLoader.load(post.authorId),
  },
};
```

- **DO:** Use a per-request DataLoader (or equivalent batching/caching utility) for every relation a resolver fetches, keyed and scoped to the lifetime of a single request so batched results from one request never leak into another user's request.

- **DON'T:** Assume DataLoader alone solves performance — it batches and dedupes within a single request but does nothing to stop a client from requesting deeply nested, exponentially expanding data (e.g., `users { posts { comments { author { posts { ... } } } } }`). Combine batching with query complexity/depth limits (below).

- **DO:** Enforce query complexity limits, depth limits, or both, and reject queries that exceed them before executing any resolver. An unauthenticated or poorly-behaved client can otherwise construct a single GraphQL query that fans out into thousands of database calls — the GraphQL equivalent of an unbounded `JOIN` — and there's no URL-based rate limit that catches it, because it's all one HTTP request.

- **DON'T:** Expose introspection in production without deciding that deliberately. Introspection is genuinely useful for internal tooling and public developer-facing APIs, but for an internal-only API it hands an attacker the entire schema, including field names for not-yet-released features; disable it in production or gate it behind authentication if the API isn't meant to be publicly explorable.

- **DO:** Use GraphQL's non-null (`!`) markers deliberately and conservatively on output types. A field marked non-null that later needs to become nullable (because the underlying data source can now return null) is a breaking schema change; when in doubt, especially for fields sourced from another service or an external API, leave them nullable.

- **DON'T:** Make input arguments non-null when they represent optional filters or updates. A `updateUser(name: String!, email: String!)` mutation forces every client to resend fields it isn't changing, and makes "clear this field" impossible to express; use nullable inputs with an explicit "was this field provided" mechanism where partial updates matter.

- **DO:** Paginate list fields using cursor-based connections (the Relay-style `edges`/`node`/`pageInfo` pattern, or a project-consistent equivalent) rather than returning a bare unbounded array. The same reasons pagination matters for REST list endpoints apply here — an unpaginated `allOrders` field is an unbounded query waiting to happen.

- **DON'T:** Let error handling inside GraphQL degrade to "everything returns 200 with an `errors` array the client may or may not check." That's the standard GraphQL transport behavior and is fine at the transport level, but the schema and resolvers should still distinguish user-facing validation errors (returned as typed payload fields, e.g. `UserError`) from unexpected server errors (surfaced via the top-level `errors` array) so clients can render field-level messages instead of a generic failure banner.

- **DO:** Version the schema by additive evolution (new fields, new types, deprecating old fields with `@deprecated(reason: "...")`) rather than a URL-based version bump. GraphQL's single-endpoint model is designed around this; clients request only the fields they declare, so adding fields is inherently non-breaking, and `@deprecated` gives a migration path without breaking existing queries.

- **DON'T:** Remove or rename a field that's still marked as in-use without a deprecation window. Even though GraphQL clients only request the fields they use, you generally cannot see every client's exact query set (especially for a public API), so a field removed without warning breaks whichever clients happened to request it.

- **DO:** Authorize at the field level, not just at the query root. A `User` type's `email` or `ssn` field may need different visibility rules than the `name` field on the same type; enforce that inside the field resolver (or a directive like `@auth(role: ADMIN)`) rather than assuming that authorizing the top-level query covers every nested field it can reach.

- **DON'T:** Trust client-supplied `first`/`last`/`limit` arguments on connection fields without a maximum cap. Just like REST pagination, an unbounded `first: 999999999` argument is a resource-exhaustion vector if the resolver doesn't clamp it server-side.

- **DO:** Use GraphQL subscriptions for genuinely real-time, server-push use cases (live order status, chat messages) over a persistent transport (WebSocket, Server-Sent Events), and treat them as a distinct operation type from queries and mutations with their own resolver and their own authorization checks — a subscription resolver needs to re-verify the subscriber's permission on every event it pushes, not just at subscribe time, since a user's access can change while the subscription is still open.

- **DON'T:** Use subscriptions as a substitute for a well-designed mutation/query pair when a simple request/response would do. Subscriptions carry real infrastructure cost (persistent connections, connection-state management, harder horizontal scaling than stateless HTTP) — reach for them when the use case genuinely needs server-initiated push, not as a default "more modern" choice for ordinary data fetching.

- **DO:** Consider schema federation/composition (combining multiple services' GraphQL schemas into one unified graph, via Apollo Federation or an equivalent) when multiple teams/services each own a slice of the overall graph, so each team can evolve their subgraph independently while clients still see one coherent schema. This is the GraphQL-world analog of the API-composition/BFF patterns described in Microservices Patterns below — solving the same "one client-facing surface, many owning services" problem.

- **DON'T:** Let a single monolithic GraphQL schema become the de facto integration point for every service in the system with no ownership boundaries, where any resolver can query any other domain's data directly with no clear line of accountability for a given type or field. As the schema grows across teams, treat each type/field's ownership as deliberately as you'd treat a microservice boundary — otherwise the schema itself becomes a distributed monolith's shared brain.

- **DO:** Mask or generalize internal error details before they reach the `errors` array in a production GraphQL response, the same way a REST API shouldn't leak stack traces — a resolver that lets a raw database exception's message propagate into the client-facing `errors[].message` field discloses the same kind of internal detail a REST 500 handler should already be scrubbing.

- **DON'T:** Return partial data silently alongside a swallowed error with no clear signal to the client about which fields actually failed to resolve. GraphQL's partial-response model (data for the fields that succeeded, plus an `errors` array for the ones that didn't) is a deliberate, useful feature — but only if resolvers correctly populate `errors` with a `path` pointing at the failed field, so clients can tell "this field is legitimately null" from "this field failed to resolve."

- **DO:** Model genuinely polymorphic data with GraphQL's union or interface types (a `SearchResult` that can be a `Product`, an `Article`, or a `Category`) rather than flattening every possible field from every possible type onto one giant type with most fields nullable depending on which "kind" the object actually is. A sprawling nullable-everything type forces every client to guess which field combination is valid for a given response instead of the schema expressing that directly.
```graphql
# BAD: one flattened type, most fields meaningless depending on "kind"
type SearchResult {
  kind: String!
  title: String
  price: Float        # only meaningful if kind == "product"
  author: String       # only meaningful if kind == "article"
}

# GOOD: a union expresses the real shape of the data
union SearchResult = Product | Article | Category
type Product { id: ID!, title: String!, price: Float! }
type Article { id: ID!, title: String!, author: String! }
```

- **DON'T:** Overuse custom scalars for types a standard scalar already models correctly (defining a custom `Int32` scalar when `Int` already does the job). Reserve custom scalars for genuinely distinct semantics the built-in scalars can't express and validate correctly (`DateTime`, `EmailAddress`, `URL`, `JSON`) — each custom scalar needs its own serialize/parse/validate logic maintained correctly on both ends, so don't introduce one without a real reason.

- **DO:** Write clear `description` strings on every type and field in the schema definition (`"""The customer's billing address"""`), since — unlike REST, where documentation is a separate artifact that can drift from the code — GraphQL's introspection makes the schema's own descriptions the documentation every client-facing tool (GraphiQL, Apollo Studio, generated docs) will actually display. An undocumented schema is effectively an undocumented API even though the type information itself is technically self-describing.

- **DO:** Use the built-in `@skip`/`@include` directives (or an equivalent client-controlled directive) when a client genuinely needs to conditionally request a field based on a runtime variable, rather than maintaining two near-duplicate query documents client-side that differ only in which fields they request.

- **DON'T:** Expose a schema field that's actually still under active development or behind a feature flag with no signal to consumers that it's unstable. If a field needs to be gated (visible only to specific clients, or clearly marked experimental), use a naming convention or a directive (`@experimental`) that's visible in introspection, rather than quietly shipping a half-finished field into the public schema with no indication it might still change shape.

- **DO:** Keep a REST API and a GraphQL API exposed over the same underlying domain consistent in the business rules and authorization logic they enforce, even though their request/response shapes are necessarily different. Two API surfaces over the same data are a common source of drift when a validation rule or a permission check gets added to one and forgotten in the other; centralize that logic in the shared service/domain layer both API surfaces call into (per General Backend Architecture above), so a rule enforced once is enforced consistently everywhere.

## gRPC & Protobuf

- **DO:** Define service contracts in `.proto` files as the single source of truth, and generate client/server stubs from them rather than hand-writing serialization code. Protobuf's whole value proposition is a strongly-typed, versioned, language-agnostic contract; hand-rolled equivalents lose that guarantee the first time client and server drift out of sync.

- **DON'T:** Reuse the same field number for a different field after removing an old one. Protobuf wire format identifies fields by number, not name; reassigning a retired field number to a new field can cause an old client that still sends the retired field to have its data silently misinterpreted as the new field's type. Reserve retired numbers explicitly.
```protobuf
// BAD: field 3 removed and its number reused immediately
message Order {
  string id = 1;
  int32 total_cents = 2;
  string new_field = 3; // previously "legacy_discount_code" = 3, now repurposed — dangerous
}

// GOOD: retired field number is reserved, never reused
message Order {
  string id = 1;
  int32 total_cents = 2;
  reserved 3;
  reserved "legacy_discount_code";
  string new_field = 4;
}
```

- **DO:** Only add new fields as optional (or, in proto3, rely on default-presence semantics deliberately) and never change an existing field's number or type once a message is in production use. Protobuf's backward/forward compatibility guarantees depend entirely on following these rules; violating them breaks compatibility silently at runtime rather than at compile time.

- **DON'T:** Design RPC methods that take or return giant, deeply nested messages when the operation genuinely only needs a few fields. Protobuf's efficiency comes partly from being purpose-built and lean; a `GetUser` RPC that returns the entire `User` graph (orders, addresses, payment methods, audit log) when the caller wanted a name and an email wastes bandwidth and couples every caller to the whole object's evolution.

- **DO:** Choose the right RPC style for the use case: unary (single request/response) for typical request/response operations, server streaming for a client requesting a large or ongoing result set (e.g., live updates, log tailing), client streaming for uploading a large or incremental payload, and bidirectional streaming for genuinely interactive protocols (chat, real-time collaboration). Defaulting everything to unary RPCs that poll in a loop reimplements streaming badly on top of a system that already supports it natively.

- **DON'T:** Ignore gRPC's status codes (`OK`, `NOT_FOUND`, `INVALID_ARGUMENT`, `PERMISSION_DENIED`, `UNAVAILABLE`, `DEADLINE_EXCEEDED`, etc.) in favor of a generic error wrapped in the response body. gRPC has its own status-code model that maps cleanly onto retry policies (`UNAVAILABLE` is generally safe to retry, `INVALID_ARGUMENT` is not) — use it the way you'd use HTTP status codes correctly in a REST API, not as a single opaque failure.

- **DO:** Set deadlines/timeouts on every outgoing gRPC call from the client side, and propagate deadlines through a call chain (service A calling service B calling service C) so a slow downstream doesn't cause an unbounded wait upstream. gRPC supports deadline propagation natively; not using it means one hung backend can cascade into every caller hanging too.

- **DON'T:** Put a REST/JSON gateway in front of gRPC services and then let the two contracts drift independently (adding a field to the `.proto` but forgetting to update the gateway's OpenAPI spec, or vice versa). If a gRPC-JSON transcoding layer (like grpc-gateway) is used to expose REST endpoints, generate that mapping from the same `.proto` source instead of maintaining it by hand.

- **DO:** Use protobuf's built-in well-known types (`google.protobuf.Timestamp`, `Duration`, `Empty`, `FieldMask`) instead of inventing ad hoc representations (a `string` for a timestamp, an `int64` of unclear units). Reusing the standard types means every generated client library already knows how to work with them correctly, including timezone and unit-precision handling.

- **DON'T:** Skip API versioning strategy for gRPC services just because protobuf handles field-level compatibility. Field-level compatibility protects individual message evolution, but a genuinely breaking change to a service's behavior or contract still needs a versioning strategy (a new package name like `orderservice.v2`, or a new service) the same way a REST API does.

- **DO:** Keep `.proto` files organized by service/domain and package them with clear, stable package names (`company.orders.v1`) since the package name becomes part of the generated code's namespace across every consuming language, and renaming it later is a breaking change for every client.

- **DON'T:** Assume gRPC is a drop-in replacement for REST in every context. It excels for internal service-to-service communication with high throughput and strict contracts, but browser clients need grpc-web or a gateway (no native browser gRPC), and human-readable debugging (curl, browser devtools) is harder with a binary wire format — pick REST/JSON at the edges where broad client compatibility and easy debugging matter more than raw efficiency.

- **DO:** Implement the standard gRPC health-checking protocol (`grpc.health.v1.Health`) so load balancers, orchestrators, and service meshes have a uniform way to ask any gRPC service "are you healthy" without each service inventing its own bespoke health-check RPC. Consistency here is what lets generic infrastructure (Kubernetes readiness probes via grpc-health-probe, a mesh's health-aware load balancing) work across every service without custom per-service configuration.

- **DON'T:** Disable server reflection in every environment by default without considering the tradeoff. Reflection (`grpc.reflection.v1`) lets tools like `grpcurl` introspect a running service's available methods and message shapes for debugging, similar in spirit to REST's OpenAPI docs — genuinely useful in development and staging, but consider whether it should stay enabled in production the same way you'd think about exposing OpenAPI docs or GraphQL introspection publicly.

- **DO:** Use gRPC interceptors (the gRPC equivalent of HTTP middleware) for cross-cutting concerns — auth-token validation, request logging, metrics, distributed tracing propagation — applied consistently across every RPC method, rather than duplicating that logic inside each service method's implementation.

- **DON'T:** Ignore client-side load-balancing configuration and assume a single long-lived connection to one backend instance is sufficient once a gRPC service is deployed with multiple replicas. Because gRPC multiplexes many calls over one persistent HTTP/2 connection, a naive client can end up pinned to a single backend instance indefinitely unless client-side load balancing (or a proxy/mesh that handles it) is explicitly configured — leaving other healthy replicas idle while one instance takes all the traffic.

- **DO:** Attach structured error details (`google.rpc.Status` with typed `google.rpc.ErrorInfo`/`BadRequest`/`RetryInfo` details, or an equivalent project convention) to a failed RPC rather than encoding structured error information into a plain error message string. A plain string forces every caller to parse text to extract machine-readable detail (which specific field was invalid, how long to wait before retrying); typed error details let generated client code access that information directly, the gRPC equivalent of a structured REST error body.

- **DON'T:** Model a genuinely one-shot request/response operation as a client-streaming or bidirectional-streaming RPC "for future flexibility." Streaming RPCs carry real complexity (backpressure, partial-failure mid-stream, connection lifecycle) that unary RPCs don't — reserve streaming for operations that actually need it now, and add a new RPC method later if a genuine streaming need emerges, rather than paying that complexity cost upfront on a speculative "might need it later."
```protobuf
service OrderService {
  // unary: the common case, a single request, a single response
  rpc GetOrder (GetOrderRequest) returns (Order);

  // server streaming: client asks once, server pushes many updates
  rpc WatchOrderStatus (WatchOrderStatusRequest) returns (stream OrderStatusUpdate);
}
```

- **DO:** Follow an established API design style guide (such as Google's AIP — API Improvement Proposals) for naming and structuring proto services and messages consistently (`ListOrders`/`GetOrder`/`CreateOrder`/`UpdateOrder`/`DeleteOrder` method naming, standard pagination fields `page_size`/`page_token`) rather than each service inventing its own method-naming convention. Consistency here has the same payoff REST's uniform interface has — a developer who's used one well-structured gRPC service can predict the shape of another.

## Framework Conventions

Each framework below has its own idioms for routing, middleware, validation, and error handling. Using a pattern from one framework inside another (e.g., writing Express-style middleware chains inside a NestJS controller, or hand-rolling Django's ORM query patterns in Flask) is a common way AI-generated code reads as subtly wrong to anyone who knows the framework — follow the idiom the project has already established, and check the current stable version's documentation before inventing a method name that "sounds right."

### Express (Node.js)

- **DO:** Use the built-in `Router()` to group related routes into separate modules instead of registering every route on the top-level `app` object. A single `app.js` with 80 `app.get`/`app.post` calls is unnavigable; `express.Router()` per resource, mounted with `app.use('/orders', ordersRouter)`, keeps route files scoped and independently testable.
```javascript
// BAD: every route flat on the app object
app.get('/orders', ...);
app.get('/orders/:id', ...);
app.post('/orders', ...);
app.get('/users', ...);
// ...80 more lines

// GOOD: routes grouped by resource
const ordersRouter = express.Router();
ordersRouter.get('/', listOrders);
ordersRouter.get('/:id', getOrder);
ordersRouter.post('/', createOrder);
app.use('/orders', ordersRouter);
```

- **DON'T:** Forget the `next(err)` call in async route handlers, or wrap every handler in a manual `try/catch` that duplicates the same error-forwarding logic. Unhandled promise rejections inside an Express route handler (pre-Express-5, without an async wrapper) crash the process or silently hang the request; use a small `asyncHandler` wrapper or upgrade to Express 5's native async error handling, applied consistently across all routes.

- **DO:** Centralize error handling in a single error-handling middleware (a 4-argument `(err, req, res, next)` function registered last) rather than formatting error responses inline in every handler. This is where consistent error-response shape and status-code mapping for known error types (validation error -> 422, not-found error -> 404) belongs.
```javascript
// GOOD: one place error responses are shaped
app.use((err, req, res, next) => {
  const status = err.statusCode || 500;
  res.status(status).json({
    error: { code: err.code || 'internal_error', message: err.message },
  });
});
```

- **DON'T:** Validate request bodies by hand with scattered `if (!req.body.email) return res.status(400)...` checks repeated per route. Use a schema validation library (Zod, Joi, express-validator) applied as middleware so the validation rule lives in one declarative place per route and the handler only runs with already-valid data.

- **DO:** Apply security middleware (`helmet` for headers, a CORS middleware configured with an explicit allow-list rather than `origin: '*'` for anything handling credentials, a rate limiter) globally near the top of the middleware stack, before routes are registered. Middleware order in Express is execution order — security middleware registered after routes doesn't protect those routes.

- **DON'T:** Put unbounded `body-parser`/`express.json()` limits on request bodies. A default with no `limit` option accepts arbitrarily large request bodies and is a denial-of-service vector; set an explicit reasonable limit (`express.json({ limit: '1mb' })`) sized to what the endpoint actually needs.

- **DO:** Keep controller functions thin — parse/validate input (or trust upstream validation middleware), call a service function, translate the result to an HTTP response. Business logic belongs in a service module that has no dependency on `req`/`res`, as covered in General Backend Architecture above.

- **DON'T:** Assume `req.body` is defined without first confirming a body-parsing middleware is registered for that content type, and don't assume it matches the expected shape without validation — a client can send anything (or nothing) as the body.

- **DO:** Register middleware in the order the request should actually flow through it (body parsing, then security headers, then auth, then route-specific logic), and keep that order intentional and documented — Express middleware order is execution order, and a subtle reordering (e.g., a rate limiter registered after the routes it's meant to protect) silently defeats the middleware's purpose without any error being raised.

- **DON'T:** Attach business-logic-bearing middleware (e.g., "look up and attach the user's subscription tier") globally to every route when it's only relevant to a subset of routes. Global middleware that does unnecessary work on every request (including routes that don't need it) adds latency and coupling across the whole app; scope middleware to the specific router/route that actually needs it.

### NestJS (Node.js)

- **DO:** Use NestJS's built-in decorators (`@Controller`, `@Injectable`, `@Module`) and its dependency injection container as designed, rather than hand-instantiating services inside controllers. NestJS's whole value is a structured, testable DI architecture built on top of Express/Fastify; bypassing it with `new OrderService()` inside a controller throws away that structure and breaks the ability to swap implementations for tests.
```typescript
// BAD: manual instantiation bypasses Nest's DI container
@Controller('orders')
export class OrdersController {
  private ordersService = new OrdersService(new OrdersRepository());
}

// GOOD: constructor injection, Nest resolves and manages the dependency
@Controller('orders')
export class OrdersController {
  constructor(private readonly ordersService: OrdersService) {}
}
```

- **DON'T:** Put validation logic inline in the controller when NestJS's `class-validator`/`class-transformer` integration with DTOs and a global `ValidationPipe` already handles it declaratively. Re-implementing field-presence and type checks by hand in the controller when a `@IsEmail()`/`@IsNotEmpty()` decorator on the DTO would do it automatically is redundant and easy to get inconsistent across endpoints.
```typescript
// GOOD: DTO declares its own validation rules
export class CreateOrderDto {
  @IsNotEmpty()
  @IsArray()
  items: OrderItemDto[];

  @IsOptional()
  @IsString()
  couponCode?: string;
}
// main.ts: app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
```

- **DO:** Use Guards for authentication/authorization, Pipes for validation/transformation, Interceptors for cross-cutting concerns (logging, response mapping, caching), and Exception Filters for error formatting — each in its designated place, rather than cramming all of that into the controller method body. This is the framework's whole architectural point: it gives every cross-cutting concern a designated extension point.

- **DON'T:** Let modules import each other's internal providers directly instead of going through each module's exported public interface. NestJS modules are meant to encapsulate their internals; importing a provider that isn't exported from another module's `exports` array (via manual `forwardRef` hacks, deep import paths, etc.) defeats the module boundary the framework was designed to enforce.

- **DO:** Return errors by throwing NestJS's built-in HTTP exceptions (`NotFoundException`, `BadRequestException`, `ForbiddenException`, or a custom exception caught by an Exception Filter) rather than manually setting `res.status()` inside a controller. NestJS's exception-handling pipeline is designed to catch these and format a consistent error response automatically.

- **DON'T:** Invent NestJS decorators, pipes, or module methods that don't exist in the installed version, especially when generating code from memory. NestJS's API has changed across major versions (e.g., microservices transport options, the `ConfigModule` API); verify against the actual installed version's documentation or the project's existing usage instead of guessing a plausible-sounding decorator name.

- **DO:** Use NestJS's built-in `ConfigModule` (or an equivalent validated config approach) for environment configuration, with a validation schema, rather than reading `process.env` directly scattered across services. Centralized, validated config catches a missing required environment variable at startup instead of as a runtime `undefined` deep inside a service.

- **DO:** Scope providers deliberately (`DEFAULT`, `REQUEST`, or `TRANSIENT` scope) and understand the performance implication of request-scoped providers — a request-scoped provider is re-instantiated for every incoming request, which is sometimes exactly what's needed (per-request context) but is real overhead if applied broadly by default rather than only where request-scoping is actually required.

- **DON'T:** Let a NestJS module's `imports` array grow into a tangle of circular module dependencies patched over with `forwardRef()` scattered throughout the codebase. Frequent `forwardRef()` usage is usually a sign the module boundaries themselves need rethinking, not that `forwardRef()` is the correct long-term fix for the circular dependency it's papering over.

### Django (Python)

- **DO:** Use Django REST Framework serializers (or Django's own form/model validation) to validate and shape input rather than reading `request.POST`/`request.body` and validating fields by hand. Serializers give you declarative field validation, nested object handling, and a consistent way to turn validation errors into a structured error response.
```python
# BAD: manual dict access and validation
def create_order(request):
    data = json.loads(request.body)
    if 'items' not in data or not data['items']:
        return JsonResponse({'error': 'items required'}, status=400)
    # ...

# GOOD: DRF serializer owns validation
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'items', 'status', 'created_at']

def create_order(request):
    serializer = OrderSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    order = serializer.save()
    return Response(OrderSerializer(order).data, status=201)
```

- **DON'T:** Put business logic inside Django views or, worse, inside `save()` overrides and model methods that silently trigger side effects (sending emails, calling external APIs) whenever a model instance is saved anywhere in the codebase, including from the admin panel or a data migration. A `save()` override with side effects fires from contexts the author never anticipated; keep side effects in explicit service functions called from the view/task that intends them.

- **DO:** Use Django's ORM query methods (`select_related`, `prefetch_related`) deliberately to avoid N+1 queries when a view serializes related objects. Django's lazy relationship loading means naive template/serializer access to `order.customer.name` inside a loop over orders issues one query per order unless the relation was prefetched.
```python
# BAD: N+1 — one query per order to fetch its customer
orders = Order.objects.filter(status='pending')
for order in orders:
    print(order.customer.name)  # extra query each iteration

# GOOD: related rows fetched in a single additional query
orders = Order.objects.filter(status='pending').select_related('customer')
```

- **DON'T:** Bypass Django's migration system by editing the database schema directly or hand-writing raw SQL DDL outside of a migration file. Migrations are the project's source of truth for schema history and are what keeps every environment (dev, CI, staging, prod) in sync; a manual schema change invisible to `makemigrations`/`migrate` will diverge and eventually cause a migration conflict or data loss.

- **DO:** Use DRF's `ViewSet`/`Router` combination for standard CRUD resources instead of hand-writing near-identical function-based views for list, retrieve, create, update, and delete. This is the framework-idiomatic way to get consistent REST conventions (pagination, filtering, permission checks) applied uniformly without repeating boilerplate per resource.

- **DON'T:** Leave Django's `DEBUG = True` or an overly permissive `ALLOWED_HOSTS`/`CORS` setting active in a production configuration. `DEBUG = True` in production renders full stack traces (including settings values) to any client that triggers a 500 error — a severe information-disclosure risk.

- **DO:** Use Django's permission classes (`IsAuthenticated`, `IsAdminUser`, custom `BasePermission` subclasses) on DRF views/viewsets rather than checking `request.user.is_staff` inline in every view method. Declarative, reusable permission classes are testable in isolation and consistently applied via `permission_classes`.

- **DON'T:** Rely on Django signals (`post_save`, `pre_delete`) as the primary mechanism for triggering important cross-cutting side effects for the same reason Rails' `after_save` callbacks are discouraged above — signal receivers fire from any code path that saves the model, including the admin panel, shell sessions, and data migrations, making the actual flow of a request hard to trace by reading the view alone; use signals sparingly, for genuinely decoupled, optional side effects, not for a primary business workflow step.

- **DO:** Use Django REST Framework's pagination classes (`PageNumberPagination`, `CursorPagination`, `LimitOffsetPagination`) configured globally or per-viewset, rather than hand-implementing pagination logic per view. This keeps the pagination envelope shape consistent across every DRF-backed endpoint automatically.

### Flask (Python)

- **DO:** Organize routes with Blueprints once the app grows past a handful of endpoints, instead of registering every route on a single global `app` object in one file. Blueprints let related routes, templates, and static assets for one feature area be grouped and registered independently — the Flask equivalent of Express's `Router()`.
```python
# GOOD: blueprint groups a resource's routes
orders_bp = Blueprint('orders', __name__, url_prefix='/orders')

@orders_bp.route('/<int:order_id>', methods=['GET'])
def get_order(order_id):
    ...

app.register_blueprint(orders_bp)
```

- **DON'T:** Rely on Flask's minimalism as an excuse to skip request validation. Flask does not ship an opinionated validation layer the way DRF or FastAPI do, which means it's tempting to read `request.json['field']` directly and let a `KeyError` become an unhandled 500 for any malformed request; use a schema library (Pydantic, Marshmallow, or Flask-specific validation extensions) explicitly, since the framework won't do it for you.
```python
# BAD: unguarded dict access turns a bad request into a 500
@app.route('/orders', methods=['POST'])
def create_order():
    items = request.json['items']  # KeyError -> unhandled 500 if missing

# GOOD: explicit schema validation with a proper 400 on failure
@app.route('/orders', methods=['POST'])
def create_order():
    data = OrderSchema().load(request.get_json(silent=True) or {})
    ...
```

- **DO:** Use Flask's `errorhandler` decorators to centralize error-to-response mapping (`@app.errorhandler(404)`, `@app.errorhandler(ValidationError)`) instead of formatting error JSON inline at every route that might fail.

- **DON'T:** Use Flask's global request context objects (`g`, `request`, `session`) as an implicit way to pass data between unrelated functions across a request's lifecycle without clear ownership. Overusing `g` as a grab-bag turns the data flow invisible — a function reading `g.some_value` has no explicit signature telling the reader where that value came from.

- **DO:** Keep the application factory pattern (`create_app()`) for anything beyond a toy script, so configuration (testing vs. production), extensions, and blueprints are wired up in one explicit place and multiple app instances (for testing) can be created cleanly.

- **DON'T:** Leave the Flask development server (`app.run(debug=True)`) as the production entry point. The built-in server is single-threaded and not hardened for production traffic; use a WSGI server (Gunicorn, uWSGI) behind a reverse proxy, and never run with `debug=True` in production — the Werkzeug debugger's interactive console is a remote-code-execution risk if reachable.

- **DO:** Use Flask extensions (Flask-SQLAlchemy, Flask-Migrate, Flask-CORS) for concerns those extensions already solve well, rather than hand-rolling database session management or CORS handling. Flask's minimalism means it deliberately doesn't ship these opinions itself, but reinventing well-solved problems (connection lifecycle per request, safe CORS header handling) from scratch introduces avoidable bugs that the maintained extensions have already worked through.

- **DON'T:** Return a Python dict or object directly from a route handler without confirming Flask actually knows how to serialize it as intended (Flask auto-converts plain dicts to JSON in modern versions, but a custom object, a Decimal, or a set will not serialize the way you expect by default). Be explicit about response serialization for anything beyond plain JSON-compatible types.

### FastAPI (Python)

- **DO:** Define request and response schemas as Pydantic models and let FastAPI's dependency-injected validation do the work, rather than manually validating a raw JSON body. This is FastAPI's core value proposition: automatic request validation, response serialization, and OpenAPI schema generation all driven from the same type-annotated models.
```python
# GOOD: Pydantic model defines and validates the request shape,
# and FastAPI derives the OpenAPI schema from it automatically
class OrderCreate(BaseModel):
    items: list[OrderItem]
    coupon_code: str | None = None

@app.post("/orders", response_model=OrderRead, status_code=201)
async def create_order(order: OrderCreate, service: OrderService = Depends(get_order_service)):
    return await service.create(order)
```

- **DON'T:** Use `def` route handlers for I/O-bound work that should be `async def`, or block the event loop inside an `async def` handler with a synchronous call (a blocking DB driver, `time.sleep`, a synchronous `requests.get`). FastAPI is built on an async event loop; a blocking call inside an `async def` handler stalls every other concurrent request being served by that worker, not just the one that made the blocking call.

- **DO:** Use FastAPI's `Depends()` system for shared logic — database sessions, current-user resolution, pagination parameters — instead of duplicating that setup at the top of every route function. Dependencies are composable, cached per-request, and show up correctly in the generated OpenAPI docs' parameter list.

- **DON'T:** Return raw ORM model instances directly from a route without a `response_model` (or equivalent explicit output schema). Without an explicit response model, whatever fields the ORM object happens to carry — including ones never meant to be public — get serialized straight into the response the moment someone adds a column.

- **DO:** Separate Pydantic schemas for input (`OrderCreate`) from output (`OrderRead`) from the internal ORM/domain model, even though it's tempting to reuse one model for all three in a small FastAPI app. Reusing one model forces awkward compromises (fields optional only during creation, `exclude` lists to hide sensitive fields on output) that separate schemas avoid entirely.

- **DON'T:** Assume FastAPI's automatic docs (`/docs`, `/redoc`) are a substitute for deliberate API documentation discipline — they reflect exactly what the Pydantic models and route decorators say, so a route with vague field names, no `description`, and no `examples` still produces vague, unhelpful auto-generated docs. Add `Field(description=...)`, `examples`, and docstrings deliberately; don't rely on the tool alone to produce good documentation.

- **DO:** Use FastAPI's `BackgroundTasks` for lightweight post-response work (sending a notification, writing an audit log entry) that doesn't need the full durability guarantees of a proper task queue, but reach for a real task queue (Celery, an async job runner) once the work needs retries, persistence across a process restart, or scheduling — `BackgroundTasks` runs in-process and is lost if the process crashes before it completes.

- **DON'T:** Override a route's dependencies only in production code paths as an informal way to "test" different behavior; use FastAPI's explicit `app.dependency_overrides` mechanism, scoped to the test suite, to substitute fakes/mocks for dependencies like the current-user resolver or the database session cleanly, rather than adding test-only branching logic (`if TESTING: ...`) directly into application code.

### Spring Boot (Java)

- **DO:** Use Spring's layered stereotypes (`@RestController`, `@Service`, `@Repository`) to keep the same controller/service/repository separation Spring is built around, and let Spring's dependency injection wire them together via constructor injection rather than field injection (`@Autowired` on a field).
```java
// BAD: field injection — hides dependencies, harder to unit test, allows partial construction
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
}

// GOOD: constructor injection — dependencies are explicit and required, easy to mock in tests
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

- **DON'T:** Return JPA entities directly from `@RestController` methods. Serializing an entity straight to JSON risks exposing lazy-loaded relationships that trigger extra queries mid-serialization (or throw `LazyInitializationException` outside a transaction), and leaks persistence-layer fields (`@Version`, internal foreign keys) into the API contract; map entities to DTOs explicitly.

- **DO:** Use Bean Validation annotations (`@Valid`, `@NotNull`, `@Size`, `@Email`) on request DTOs combined with a global `@ControllerAdvice`/`@ExceptionHandler` that catches `MethodArgumentNotValidException` and formats a consistent validation error response. This is Spring's idiomatic validation pipeline; hand-rolling `if (dto.getEmail() == null)` checks in every controller method duplicates what the annotations already express declaratively.

- **DON'T:** Let unhandled exceptions surface as Spring Boot's default whitelabel error page (or a raw stack trace) in an API that's meant to return JSON. Register a `@ControllerAdvice` with `@ExceptionHandler` methods that translate known exception types to the API's structured error format and appropriate status codes.

- **DO:** Scope `@Transactional` boundaries at the service layer, around a single logical unit of work, not at the controller or repository layer. A transaction that's too broad (wrapping unrelated operations, or spanning an external HTTP call) holds database locks longer than necessary and can turn a slow third-party API call into a database outage; a transaction that's too narrow (per-repository-call) loses atomicity across multi-step operations that need it.

- **DON'T:** Use field-level `@Autowired` for required dependencies, or component-scan magic that makes it unclear which beans are actually wired into a given class. Preferring constructor injection (see above) also lets Spring catch missing dependencies at application-context startup rather than at first use in production.

- **DO:** Use Spring profiles (`application-dev.yml`, `application-prod.yml`) and externalized configuration (`@ConfigurationProperties`, environment variables) instead of hardcoding environment-specific values (database URLs, feature flags) in code or in a single shared properties file with commented-out alternatives.

- **DO:** Secure and scope Spring Boot Actuator endpoints (`/actuator/health`, `/actuator/env`, `/actuator/heapdump`) deliberately before deploying to production — several actuator endpoints expose sensitive internal details (environment variables, full heap dumps, bean configuration) by design for operational use, and leaving them open on the public internet with default settings is a well-known, frequently-exploited misconfiguration.

- **DON'T:** Let a `@RestControllerAdvice` exception handler catch and reformat exceptions so broadly that it masks the real error type from logs and monitoring — a catch-all `@ExceptionHandler(Exception.class)` that returns a generic `500` is fine as a last-resort safety net, but pair it with specific handlers for known exception types (validation, not-found, business-rule violations) mapped to their correct status codes, rather than routing everything through one generic handler.

### Ruby on Rails

- **DO:** Follow Rails' "fat model, skinny controller" guidance loosely — but when a model itself grows to hold every piece of business logic for a domain (validations, callbacks, class methods for reporting, PDF generation, email triggering), extract that into service objects, form objects, or query objects rather than letting `ActiveRecord::Base` subclasses become god objects. The convention evolved from "avoid fat controllers," not into "put everything in the model no matter how large it gets."

- **DON'T:** Overuse ActiveRecord callbacks (`after_save`, `before_create`) for actions with external side effects (sending an email, calling a payment gateway, enqueuing unrelated background jobs). A callback chain that fires "invisibly" whenever a record is saved from any code path — including console sessions, seed scripts, and Rails admin — is one of the most notorious sources of hard-to-trace Rails bugs; trigger those side effects explicitly from the controller/service that intends them.
```ruby
# BAD: side effect hidden inside a callback, fires on every save from anywhere
class Order < ApplicationRecord
  after_save :send_confirmation_email
end

# GOOD: explicit, only happens where the intent is clear
class Orders::CreateService
  def call(params)
    order = Order.create!(params)
    OrderMailer.confirmation(order).deliver_later
    order
  end
end
```

- **DO:** Use strong parameters (`params.require(:order).permit(:status, :coupon_code)`) on every controller action that accepts input, rather than mass-assigning `params` directly to a model. Without strong parameters, a client can set any model attribute by including it in the request body — the classic Rails mass-assignment vulnerability.

- **DON'T:** Write N+1 queries by looping over an association without eager-loading it. Rails' ActiveRecord makes lazy loading the default, so `@orders.each { |o| o.customer.name }` in a view or serializer issues one query per order unless `includes(:customer)` was used on the original query — use the bullet gem or equivalent N+1 detection in development to catch these before they hit production.

- **DO:** Keep routes RESTful using `resources :orders` and its generated named routes, rather than hand-declaring custom `get`/`post` routes for standard CRUD actions. Rails' router is built around the seven conventional actions (`index`, `show`, `create`, `update`, `destroy`, `new`, `edit`); deviating from them for ordinary CRUD throws away route helpers, predictable URLs, and what every other Rails developer expects to find.
```ruby
# BAD: hand-rolled routes for standard CRUD
get '/orders', to: 'orders#list'
post '/orders/new', to: 'orders#make'

# GOOD: RESTful resource routing
resources :orders
```

- **DON'T:** Skip database-level constraints (`NOT NULL`, foreign keys, unique indexes) because a model-level `validates` rule already checks it. Model validations are bypassed by `update_column`, raw SQL, direct database access, and race conditions between concurrent requests; a uniqueness `validates` without a matching unique database index will let two simultaneous requests both pass validation and insert duplicate rows.

- **DO:** Use Rails' built-in `rescue_from` in `ApplicationController` to map exception types to consistent JSON error responses for an API-only Rails app, rather than rescuing exceptions inline in each controller action.

- **DO:** Use `Rails.application.config.api_only = true` (or generate the app with `--api`) for a Rails app that's purely a JSON API, which skips the middleware and view-layer machinery (cookie-based sessions by default, view rendering, CSRF forms) that a browser-serving Rails app needs but a pure API doesn't — running the full stack unnecessarily adds overhead and surface area with no corresponding benefit for an API-only service.

- **DON'T:** Serialize ActiveRecord objects directly to JSON via `to_json`/`as_json` on the model with no explicit serializer for anything beyond a trivial internal tool. Use a dedicated serialization layer (ActiveModel::Serializer, Jbuilder, a plain PORO serializer) so the exposed JSON shape is an explicit, reviewable contract rather than "whatever columns the model happens to have," for the same reason covered in the general architecture and Spring/Laravel sections above.

### Laravel (PHP)

- **DO:** Use Form Request classes (`php artisan make:request`) for validation instead of validating inline inside the controller method with `$request->validate([...])` repeated across similar endpoints. A dedicated Form Request class keeps validation rules, authorization checks, and custom error messages for a given action in one reusable, testable place.
```php
// BAD: validation inline, duplicated across similar endpoints
public function store(Request $request) {
    $request->validate(['email' => 'required|email', 'name' => 'required|string']);
}

// GOOD: dedicated Form Request encapsulates validation + authorization
class StoreUserRequest extends FormRequest {
    public function authorize(): bool { return $this->user()->can('create', User::class); }
    public function rules(): array {
        return ['email' => 'required|email|unique:users', 'name' => 'required|string|max:255'];
    }
}
public function store(StoreUserRequest $request) { /* $request is already validated */ }
```

- **DON'T:** Use Eloquent models directly as API response payloads without an API Resource (`php artisan make:resource`) to control exactly which fields are exposed. An Eloquent model's default `toArray()`/JSON serialization includes every column unless explicitly hidden with `$hidden`, and it's easy to forget to hide a new sensitive column added later; API Resources make the exposed shape an explicit, reviewable contract.

- **DO:** Use route model binding (`Route::get('/orders/{order}', ...)` resolving directly to an `Order` model instance) instead of manually looking up the model by ID inside every controller method. This is idiomatic Laravel and automatically returns a 404 for a missing ID without extra code.

- **DON'T:** Trigger N+1 queries by lazy-loading Eloquent relationships inside a loop (`foreach ($orders as $order) { $order->customer->name }`) without eager loading. Use `Order::with('customer')->get()` (or enable `Model::preventLazyLoading()` in non-production environments to catch these during development) the same way Rails and Django ORMs require explicit eager loading to avoid the same pitfall.

- **DO:** Use Laravel's API Resource collections and built-in pagination (`->paginate()`) for list endpoints, which automatically produces a consistent `data`/`links`/`meta` envelope, rather than hand-building pagination metadata per endpoint.

- **DON'T:** Put business logic inside route closures or directly inside controllers for anything beyond trivial CRUD. Extract to Action classes, Service classes, or (for complex domains) domain-specific classes so the controller stays a thin adapter between HTTP and the application layer, consistent with the general architecture principles above.

- **DO:** Use Laravel's exception handler (`bootstrap/app.php`'s `withExceptions` in Laravel 11+, or `app/Exceptions/Handler.php` in earlier versions) to render consistent JSON error responses for API routes, distinguishing validation errors (422 with field errors from Laravel's built-in `ValidationException` rendering), not-found (404), and unhandled exceptions (500 without leaking a stack trace in production, controlled by `APP_DEBUG=false`).

- **DON'T:** Leave `APP_DEBUG=true` in a production `.env`. Laravel's debug mode renders detailed error pages (including environment variables and stack traces) to any client that triggers an exception — a severe information-disclosure risk identical in spirit to Django's `DEBUG = True` and Flask's debugger risk above.

- **DO:** Use Laravel policies and gates for authorization logic (`$this->authorize('update', $order)` backed by an `OrderPolicy` class) rather than scattering `if ($user->id !== $order->user_id)` ownership checks inline across controller methods. Centralized, named policies are testable in isolation and give a single place to audit every authorization rule for a given model.

- **DON'T:** Dispatch time-consuming work synchronously inside a Laravel controller action when Laravel's queue system (`ShouldQueue` jobs, `Bus::dispatch`) exists specifically to defer exactly this kind of work — sending emails, processing uploads, calling slow external APIs — off the request/response cycle, consistent with the async-operation guidance covered earlier in this document.

## Microservices Patterns

- **DO:** Draw service boundaries around business capabilities/bounded contexts (orders, billing, inventory) with each service owning its own data, rather than splitting services along technical layers (a "database service," a "validation service") that all need to be called together for any single business operation. Boundaries drawn around a capability let one team change that capability's internals without coordinating a synchronized deploy with three other teams.

- **DO:** Let each service use its own internal vocabulary/model for a concept even when a neighboring service models a similar-sounding concept differently — a "customer" in the billing service and a "customer" in the support-ticketing service legitimately don't need identical fields, and forcing them into one shared canonical model couples two teams' independent evolution unnecessarily. This is the core idea behind bounded contexts in domain-driven design: the same real-world word can mean different things, with different relevant attributes, in different parts of the system, and that's fine as long as the boundary is deliberate rather than accidental drift.

- **DON'T:** Force-fit every service's data model into one shared "canonical" schema maintained centrally, treated as the single source of truth every service must directly reuse. A shared canonical model sounds appealing for consistency but in practice becomes a bottleneck — every service that needs a field the canonical model doesn't have has to negotiate a change with whichever team owns the canonical schema, recreating central-coordination overhead that splitting into services was meant to avoid.

- **DON'T:** Build a "distributed monolith" — many separately-deployed services that must all be deployed together, share a single database, and call each other synchronously in long chains for nearly every request. This gets all of microservices' operational complexity (network calls, partial failure, deployment coordination) with none of the benefit (independent deployability, isolated failure, team autonomy) — if every change requires touching four services in lockstep, they're really one system that's just harder to deploy and debug than a monolith would be.

- **DO:** Give each microservice its own private datastore (or private schema/tables within a shared instance, with no other service allowed to touch them directly) and expose data to other services only through its API or published events. This is what actually prevents the "distributed monolith" failure mode — the data-ownership boundary is what makes services independently deployable.

- **DON'T:** Let two services share a database and query each other's tables directly "temporarily," planning to fix it later. This shortcut removes the actual boundary between the services (a schema change in one breaks the other with no compile-time or API-contract signal) and in practice is rarely revisited once it's shipped and working.

- **DO:** Choose synchronous request/response (REST, gRPC) for operations where the caller genuinely needs an immediate answer to proceed (e.g., "is this payment authorized"), and asynchronous, event-driven communication (a message queue, an event bus/stream) for operations where the caller doesn't need to block on the result, or where multiple downstream services need to react independently to the same fact (e.g., "an order was placed"). Defaulting everything to synchronous calls creates request chains where every service in the chain has to be up and fast for any request to succeed.
```text
SYNC (needs an immediate answer):
  Checkout service -> calls Payment service -> waits for authorization result

ASYNC (fire-and-forget, multiple independent reactors):
  Order service publishes "OrderPlaced" event
    -> Inventory service reserves stock
    -> Notification service sends confirmation email
    -> Analytics service records the sale
  (none of these need to complete before the order-placement response returns)
```

- **DON'T:** Chain synchronous calls across many services for a single user-facing request (service A calls B calls C calls D, all waiting on each other). The end-to-end latency is the sum of every hop, and the end-to-end availability is the product of every service's uptime — four services at 99.9% uptime each combine to noticeably worse than 99.9% overall, and a single slow hop drags down the whole chain.

- **DO:** Implement circuit breakers on outbound calls to other services, so that once a downstream service starts failing or timing out repeatedly, the caller stops sending it new requests for a cooldown period instead of piling up more requests against an already-struggling service. This protects both the caller (which stops wasting resources on calls likely to fail) and the failing downstream service (which gets breathing room to recover instead of being hit with retry storms).
```text
Circuit breaker states:
  CLOSED   -> calls flow normally, failures are counted
  OPEN     -> failure threshold exceeded; calls fail fast without hitting the network
  HALF-OPEN -> after a cooldown, a trial request checks if the downstream has recovered
```

- **DON'T:** Retry a failed call to another service an unbounded number of times, or immediately in a tight loop. Combine retries with exponential backoff and a jitter, a maximum retry count, and (per above) a circuit breaker — an unbounded synchronous retry loop against a struggling downstream service is one of the most common ways one service's slowdown turns into a cascading outage across the whole system ("retry storm").

- **DO:** Design write operations that can be retried safely by using idempotency keys, the same pattern described in REST API Design above, for any inter-service call that creates or mutates state. In a distributed system, "did that call actually succeed on the other side, or did only the response get lost" is a routine occurrence, not an edge case — the receiving service needs to be able to tell "this is a retry of a request I already processed" from "this is a new request."

- **DON'T:** Assume a message will be delivered and processed exactly once by a message queue/event bus unless the specific technology and configuration genuinely guarantees that (and even then, verify — most guarantee "at least once" by default). Design consumers to be idempotent (safe to process the same message twice) rather than assuming exactly-once delivery, since most real-world messaging systems trade off toward at-least-once delivery to avoid the harder problem of guaranteeing no message is ever lost.

- **DO:** Use the Saga pattern (a sequence of local transactions coordinated via events or an orchestrator, each with a compensating action to undo it) for multi-service operations that need transactional consistency, since distributed transactions (two-phase commit across services) don't scale well and aren't supported by most messaging/queue systems. Model failure as "undo what already happened" (e.g., release the reserved inventory) rather than assuming a single atomic rollback is available across service boundaries.

- **DON'T:** Assume a partial failure in a multi-step, multi-service workflow just "won't happen" and skip writing the compensating/rollback logic. In a distributed system, "step 3 of 5 failed after steps 1 and 2 already committed" is a routine occurrence (network partition, one service being redeployed, a timeout) — not an exotic edge case — and needs an explicit answer, not a hope that it stays rare.

- **DO:** Version events published on a message bus/event stream the same deliberately as an API, since consumers may be deployed independently and on a different schedule than the publisher. Add fields additively, and include a schema version identifier in the event envelope so old and new consumers can coexist during a rollout.

- **DON'T:** Let a producer change an event's shape (renaming/removing a field, changing a type) without considering every currently-deployed consumer. Unlike a synchronous API where a client fails immediately and visibly, a broken event schema can silently corrupt or drop data in a consumer that isn't being actively watched, and the failure may not surface until much later.

- **DO:** Centralize service-to-service authentication (mutual TLS, service-to-service tokens, an API gateway/service mesh handling it uniformly) rather than each service inventing its own internal auth scheme. Inconsistent internal auth is both a security gap (some services trust any caller "because it's internal") and an operational burden (every new service has to reimplement it).

- **DON'T:** Trust "it's internal traffic" as a substitute for authentication and authorization between services. An internal network boundary is not a security boundary on its own — once any part of the internal network is compromised (a misconfigured service, a leaked credential, a compromised dependency), unauthenticated internal APIs become the easiest lateral-movement path.

- **DO:** Instrument distributed traces (a trace ID propagated across every service hop for a given request, using something like OpenTelemetry) from day one in a microservices architecture. Without distributed tracing, debugging "why was this request slow" or "where did this request fail" across a dozen services means manually correlating separate logs by timestamp — a process that doesn't scale past a couple of services.

- **DON'T:** Design a microservices split before the team, traffic, or domain complexity actually justifies the operational overhead (separate deployments, service discovery, distributed debugging, network latency and partial-failure handling). Splitting a small, single-team application into a dozen services prematurely produces the "distributed monolith" failure mode described above without ever getting the organizational-scaling benefit microservices are meant to provide — a well-modularized monolith is usually the right choice until team or scale boundaries actually demand splitting it.

## API Documentation

- **DO:** Describe REST APIs using the OpenAPI Specification (or GraphQL's introspective schema, which is self-documenting by design) as the source of truth, kept in the same repository as the code and updated in the same pull request as the code change it documents. Documentation that lives in a separate wiki, updated separately (or not at all) from the code, drifts out of sync within weeks and becomes actively misleading rather than merely incomplete.

- **DON'T:** Hand-write an OpenAPI spec that's disconnected from the actual route definitions, validation schemas, and response types, when the framework can generate it directly from those (FastAPI, NestJS with `@nestjs/swagger`, Spring with springdoc-openapi). A hand-maintained spec that isn't generated from (or validated against) the real code is a second copy of the truth that will inevitably drift from what the API actually does.

- **DO:** Document every field's type, whether it's required or optional, its format/constraints (e.g., "ISO 8601 datetime," "max 255 characters," an enum's valid values), and include a realistic example value. A field documented only as `"email": string` leaves the client guessing at validation rules the server will actually enforce, producing avoidable trial-and-error integration failures.

- **DON'T:** Document only the happy path. Document every meaningful error response (400 with its validation error shape, 401, 403, 404, 409, 429 with its headers) for each endpoint, since a client integration needs to handle failure paths just as much as success paths, and undocumented error shapes get discovered painfully, one production incident at a time.

- **DO:** Keep example requests and responses in the documentation runnable/valid against the current schema — broken examples (a field that no longer exists, a response shape that changed) are worse than no examples, because they actively mislead an integrator who copies them.

- **DON'T:** Ship an OpenAPI spec with a `servers` list pointing at a stale environment, or a spec that hasn't been regenerated/validated in a CI check after schema-relevant changes. Add a CI step that fails the build if the generated spec doesn't match what's committed (or regenerate it as part of the build), so spec drift is caught mechanically instead of relying on a human to remember.

- **DO:** Provide a changelog (or clearly labeled "added in v1.2," "deprecated, removed in v2.0" annotations) for API documentation, since consumers integrating against a long-lived API need to know what changed between versions without diffing the entire spec themselves.

- **DO:** Structure the OpenAPI spec with reusable, named components (`components/schemas`, `components/parameters`, `components/responses`) rather than repeating the same object shape inline at every endpoint that uses it. A shared `Order` schema referenced everywhere means a field added to the order response updates every endpoint's documentation consistently, instead of requiring the same edit copy-pasted across a dozen inline definitions.
```yaml
components:
  schemas:
    Order:
      type: object
      required: [id, status, total_cents]
      properties:
        id: { type: string }
        status: { type: string, enum: [pending, paid, shipped, cancelled] }
        total_cents: { type: integer }
paths:
  /orders/{id}:
    get:
      responses:
        '200':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
```

- **DON'T:** Let generated SDK/client libraries drift from the actual OpenAPI spec they're supposed to be generated from — if client SDKs are hand-maintained rather than generated, they're a second copy of the API contract with the same drift risk as a hand-written spec; prefer codegen from the spec (or contract testing that verifies the SDK against the live API) over hand-synchronization.

- **DO:** Run contract tests (verifying the running API's actual responses match its published OpenAPI/GraphQL schema) as part of CI, catching the case where an endpoint's real behavior has silently diverged from what's documented — a schema field marked required that the implementation sometimes omits, or a status code the docs don't mention that the code actually returns under some condition.

- **DON'T:** Write endpoint descriptions that just restate the method and path in prose ("GET /orders/{id}: gets an order by id"). That's not documentation, it's noise — document the things a reader can't already infer from the signature: side effects, rate limits specific to this endpoint, authorization requirements, and any non-obvious behavior (e.g., "soft-deleted orders are excluded unless `include_deleted=true` is passed").

- **DO:** Document authentication requirements per endpoint (which scheme, which scopes/roles are required) directly in the OpenAPI security schemes, not just in a general prose paragraph at the top of the docs that becomes stale as endpoints' requirements diverge.

- **DON'T:** Leave internal-only or deprecated endpoints undocumented as "security through obscurity," assuming that not documenting them makes them safe. An endpoint that's reachable is discoverable regardless of whether it's documented; either secure it properly (auth, network restriction) or remove it — don't rely on obscurity as the control.

- **DO:** Provide a ready-to-import request collection (a Postman/Insomnia collection, or an OpenAPI file that tooling can import directly) alongside prose documentation, so a new integrator can start making real requests against the API within minutes instead of hand-assembling their first request from scratch by reading examples.

- **DON'T:** Let a published Postman/Insomnia collection go stale relative to the actual API — an out-of-date collection with removed fields or old auth flows is often worse than no collection at all, since a new integrator trusts it as current and wastes time debugging a mismatch that's actually just stale documentation. Regenerate or version the collection alongside the OpenAPI spec, not as a manually-maintained, easily-forgotten side artifact.

- **DO:** Offer a mock server generated directly from the OpenAPI/GraphQL schema for frontend and integration teams to develop against before the real backend implementation is complete. This decouples frontend and backend work schedules and, because the mock is generated from the same schema that's supposed to describe the real API, it can't drift from the documented contract the way a hand-written mock or a stale fixture file eventually would.

## Rate Limiting & Throttling

- **DO:** Apply rate limits to every publicly reachable endpoint, not just authentication endpoints. An unauthenticated or authenticated-but-unlimited endpoint that does meaningful work (search, report generation, an expensive aggregation query) is a resource-exhaustion vector regardless of whether it's a "login" endpoint specifically.

- **DON'T:** Apply a single global rate limit uniformly to every endpoint regardless of cost. A cheap `GET /health` check and an expensive `POST /reports/generate` that triggers a multi-table aggregation shouldn't share the same limit — tier limits by the actual cost/sensitivity of the operation, and consider a separate stricter limit for authentication/password-reset endpoints specifically, since those are common brute-force targets.

- **DO:** Return `429 Too Many Requests` with a `Retry-After` header (or `X-RateLimit-Reset`) when a client is throttled, so well-behaved clients and libraries know exactly when to retry instead of guessing or hammering the endpoint immediately again.
```text
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1717430400
```

- **DON'T:** Silently drop or slow down requests when a client exceeds a rate limit without any signal in the response. A client that gets a mysteriously hanging or dropped connection instead of an explicit `429` has no way to distinguish "I'm being throttled" from "the service is broken," and will likely retry aggressively — the opposite of what throttling is meant to achieve.

- **DO:** Choose a rate-limiting algorithm deliberately for the traffic pattern being protected: a fixed window is simple but allows bursts at window boundaries, a sliding window smooths that out, and a token bucket naturally allows short bursts while enforcing a steady average rate — pick based on whether bursts are acceptable for that endpoint.
```text
Fixed window:   count resets to 0 at each window boundary
                (a client can send 2x the limit by timing requests around the boundary)
Sliding window: counts requests in a rolling window ending "now"
                (smooths out the boundary-burst problem, costs a bit more to compute)
Token bucket:   bucket refills at a steady rate, each request consumes a token
                (naturally allows short bursts up to the bucket size, then throttles
                 to the steady refill rate — a common default for public APIs)
```

- **DON'T:** Implement rate limiting with per-process, in-memory counters when the API runs as multiple horizontally-scaled instances behind a load balancer. Each instance would enforce the limit independently, so a client hitting N instances round-robin effectively gets N times the intended limit; use a shared store (Redis, or an API gateway's built-in distributed rate limiter) that all instances read and write against.

- **DON'T:** Key rate limits solely by IP address for authenticated APIs. Many legitimate users share an IP (corporate NAT, mobile carrier CGNAT), which causes false-positive throttling of unrelated users, while a single malicious actor can rotate IPs to bypass an IP-only limit; key by API key/user ID/client ID for authenticated traffic, falling back to IP only for unauthenticated requests.

- **DO:** Document rate limits explicitly in the API documentation, including the numeric limit, the window, and how limit-tier differences work (e.g., different limits for free vs. paid API tiers) so integrators can design their client's retry/backoff behavior around known limits instead of discovering them empirically in production.

- **DON'T:** Implement rate limiting only at the application layer with no protection at the edge (a CDN, API gateway, or load balancer). Traffic that never reaches the application because it was throttled at the edge doesn't consume application server resources or database connections; layering edge-level and application-level limits gives defense in depth against both broad traffic floods and targeted per-user abuse.

- **DO:** Apply separate, stricter throttling to expensive or sensitive operations (password reset requests, OTP/verification code sends, bulk export/report generation) than to ordinary read endpoints, since these are exactly the operations attackers target for brute-forcing, enumeration, or cost-based denial-of-service.

- **DON'T:** Let a single tenant/client in a multi-tenant system consume unbounded shared resources with no per-tenant quota, even if their traffic technically respects the global rate limit. A "noisy neighbor" tenant issuing many expensive-but-individually-legal requests can still degrade service for every other tenant sharing the same backend; enforce per-tenant quotas independent of the general rate limit.

- **DO:** Distinguish a rate limit (a short-window cap meant to prevent bursts and abuse, e.g. "100 requests per minute") from a quota (a longer-window cap meant to enforce a billing/usage tier, e.g. "10,000 requests per month"), and communicate each with its own distinct headers and error messaging. Conflating the two confuses integrators about whether they've hit a transient, retry-soon-able limit or a hard monthly cap that won't reset until the next billing cycle.

- **DON'T:** Respond to overload with a hard, uniform block for every client once a global capacity threshold is reached. Where feasible, prefer graceful degradation for lower-priority or lower-tier traffic (serving cached/slightly-stale data, disabling a non-critical feature) over an outright rejection of every request equally — treating all load as equally important during an overload event wastes the opportunity to protect the requests that matter most.

## Auth at the API Layer

- **DO:** Distinguish authentication (who is making this request) from authorization (what is this identity allowed to do) as two separate concerns, each enforced explicitly, rather than assuming a valid token alone implies permission for every action it's used with. A request with a perfectly valid JWT should still be rejected with `403` if that identity doesn't have permission for the specific resource/action being requested.

- **DON'T:** Trust a client-supplied user ID, role, or tenant ID passed as a request parameter or body field to determine authorization ("`?user_id=42&role=admin`"). Authorization decisions must be derived from the authenticated identity's server-side record (the token's verified claims, or a database lookup keyed by the verified identity) — never from a value the client itself is free to set on the request.
```text
BAD:  DELETE /orders/99?user_id=42          (trusts the client's claim about who they are)
GOOD: DELETE /orders/99  (Authorization: Bearer <verified token>)
      -> server derives the caller's identity from the verified token,
         then checks THAT identity's permission on order 99
```

- **DO:** Verify a JWT's signature, expiration (`exp`), and issuer/audience (`iss`/`aud`) claims on every request, using a well-maintained library rather than hand-parsing the token. A JWT is a signed claim, not an encrypted secret — anyone can decode and read its payload, so treat its contents as visible-but-tamper-evident, and never skip signature verification "for now" during development in a way that risks shipping unverified.

- **DON'T:** Store sensitive data (passwords, full credit card numbers, long-lived secrets) inside a JWT payload just because it's convenient to carry along. JWT payloads are base64-encoded, not encrypted, and are frequently logged, cached, or stored in browser storage — anything inside one should be treated as visible to the end user and to anyone who intercepts the token.

- **DO:** Set short expiration times on access tokens and use a separate, more tightly controlled refresh token (rotated on use, revocable server-side) for obtaining new access tokens. A long-lived access token that leaks (via logs, a compromised client, a browser extension) stays valid and exploitable for as long as its expiration window allows — short-lived tokens bound that exposure window.

- **DON'T:** Design a JWT-based auth system with no way to revoke a token before its natural expiration. Because JWT validation is typically stateless (no database lookup), a compromised or logged-out token stays valid until it expires unless the system maintains a revocation mechanism (a denylist, a short expiration plus refresh-token rotation, or a version/epoch claim checked against a stored value) — decide on a revocation strategy deliberately rather than discovering the gap during an incident.

- **DO:** Understand the session-vs-token tradeoff before defaulting to either: server-side sessions (a session ID in a cookie, backed by server-side storage) are trivially revocable and keep no sensitive claims on the client, but require a shared session store and add a lookup per request; stateless tokens (JWTs) scale horizontally with no shared session store and no per-request database lookup, but are hard to revoke early and put the burden of expiration/rotation discipline on the implementation. Pick deliberately based on the system's actual revocation and scaling needs, not by default or by whichever tutorial was most recent.

- **DON'T:** Assume "stateless" JWT auth means "no server-side state at all" is achievable or even desirable for every use case. Real systems using JWTs still typically need *some* server-side state for revocation lists, refresh-token tracking, and rate limiting per identity — the statelessness benefit is about not needing a session lookup on every single request, not about needing zero server-side auth infrastructure.

- **DO:** Use scoped, least-privilege permissions (OAuth2 scopes, role-based or attribute-based access control) rather than a single binary "is authenticated" check for APIs with more than one class of caller. A third-party integration granted a token for "read order status" should not be able to use that same token to issue refunds, even if both actions are behind "authenticated" in a simpler model.

- **DON'T:** Roll a custom authentication scheme (custom token format, custom password hashing, a home-grown "encrypted cookie" auth) when a well-reviewed standard (OAuth2/OIDC, a maintained session library, a maintained JWT library with correct defaults) covers the need. Authentication is one of the highest-consequence places to have a subtle bug, and "we wrote a custom scheme" is disproportionately likely to be the finding in a security review.

- **DO:** Validate the `Authorization` header's presence and format explicitly and return `401` with a `WWW-Authenticate` header (for schemes that call for it) on missing or malformed credentials, rather than letting a missing header propagate into a confusing downstream error (e.g., a null-pointer exception deep in a service that assumed a user was always present).

- **DON'T:** Apply authorization checks only in the UI/frontend and assume the API is implicitly protected because "the button is hidden." Every authorization rule enforced only in a client the API doesn't control (a web frontend, a mobile app) is trivially bypassed by calling the API directly; the API itself must independently enforce every authorization rule regardless of what any client does or doesn't render.

- **DO:** Use API keys for service-to-service or third-party integration authentication where a full OAuth2 flow is unnecessary overhead, but treat API keys as secrets: support rotation without downtime (allow two active keys during a rotation window), scope them to specific permissions, and never accept one in a URL query string where it ends up in server logs, browser history, and `Referer` headers.

- **DON'T:** Conflate "requires an API key" with "is authorized for everything." Even service-to-service API keys should carry (or be looked up against) a defined scope of what that specific key/integration is permitted to do, following the same least-privilege principle as user-facing auth.

- **DO:** Store API keys hashed at rest (the same principle as password storage — a fast cryptographic hash is sufficient here since a good API key already carries enough entropy that it doesn't need a slow, memory-hard hash the way a human-chosen password does), and show the raw key value to the client exactly once, at creation time. If the raw key can be retrieved again later from the dashboard/API, the storage isn't actually protecting it — a database compromise then hands over every active key directly.

- **DON'T:** Store or log API keys, access tokens, or refresh tokens in plaintext anywhere they don't strictly need to be — application logs, error-tracking tool payloads, analytics events. A credential that leaks through a debugging log is just as compromised as one that leaks through the primary datastore, and logs are frequently retained longer and accessed by more people than the credential store itself.

- **DO:** Understand the standard OAuth2 grant types at a high level and pick the one that matches the actual client: the authorization code flow (with PKCE) for user-facing apps (web and mobile) authenticating a human, the client credentials flow for pure machine-to-machine service authentication with no human user involved, and treat the legacy resource owner password credentials grant as something to avoid in new designs since it requires the client to handle the user's raw password directly.
```text
Authorization Code + PKCE  -> a human logs in via a browser/app; the client never sees the password
Client Credentials         -> service A authenticates to service B with no human involved
(avoid) Password Grant     -> client app collects username+password directly and exchanges them
```

- **DON'T:** Implement an OAuth2-*looking* flow that skips the parts that make it secure — accepting a redirect URI with no allow-list validation (an open-redirect risk), or issuing tokens without PKCE for a public client (a mobile or single-page app that cannot keep a client secret confidential). Follow the current OAuth2/OIDC specification's guidance rather than a partial, ad hoc reimplementation of "something like OAuth."

- **DO:** Use mutual TLS (mTLS) or a service mesh's built-in identity layer for service-to-service authentication in internal microservices traffic when the deployment platform supports it, since it provides strong cryptographic identity for both ends of a connection without each service needing to manage its own token-issuing logic.

- **DON'T:** Assume "we're behind a VPN/private network" is an acceptable substitute for authenticating internal service-to-service calls, as already covered under Microservices Patterns above — network location is not identity, and the two should not be conflated when deciding whether a call is trustworthy.

- **DO:** Rotate refresh tokens on use (issue a brand-new refresh token every time one is redeemed for a new access token, and invalidate the one just used) so a stolen refresh token that gets used by both the legitimate client and an attacker produces a detectable signal — the moment either party tries to reuse an already-rotated token, the server can recognize the reuse and revoke the entire token family as a precaution.
```text
1. client redeems refresh_token_A -> server issues access_token + refresh_token_B,
   marks refresh_token_A as used/invalid
2. if refresh_token_A is ever presented again (e.g. by an attacker who stole it
   earlier), the server detects reuse of an already-rotated token and revokes
   the whole token family, forcing a fresh login
```

- **DON'T:** Let a stolen refresh token remain valid indefinitely with no reuse detection, and don't set a refresh token's lifetime so long that a compromise stays exploitable for months. Pair rotation with a bounded absolute lifetime (a maximum session age even across refreshes) so a compromised token's blast radius is bounded in time even if reuse detection somehow fails to catch it.

- **DO:** Set the `HttpOnly`, `Secure`, and an appropriate `SameSite` attribute on any authentication cookie, and apply CSRF protection (a synchronizer token, or `SameSite=Strict`/`Lax` as the primary defense) for any state-changing endpoint reachable via a cookie-authenticated browser session. Cookie-based session auth is convenient (the browser handles attaching it automatically) but that same automatic attachment is exactly what makes CSRF possible if the state-changing endpoints aren't separately protected — a pure bearer-token API called only from a JavaScript client with an explicit `Authorization` header doesn't have this specific exposure, but a cookie-authenticated one does.

## Validation & Error Responses

- **DO:** Validate every field of every incoming request against an explicit schema — type, required/optional, format, length/range constraints, and allowed values for enums — before any business logic runs. Treat all external input (request bodies, query parameters, path parameters, headers) as untrusted until validated, regardless of which client is expected to be calling.

- **DON'T:** Rely solely on client-side validation (a frontend form's `required` attribute, a mobile app's input mask) as the only validation layer. Client-side validation is a UX convenience, trivially bypassed by calling the API directly (curl, a modified client, a malicious actor); the server must independently enforce every validation rule regardless of what any client already checked.

- **DO:** Return a consistent, structured error response shape across the entire API — including a machine-readable error code/type, a human-readable message, and (for validation failures) a list identifying which field(s) failed and why. A consistent shape lets client code handle errors generically instead of writing bespoke parsing per endpoint.
```json
{
  "error": {
    "code": "validation_error",
    "message": "The request could not be validated.",
    "details": [
      { "field": "email", "message": "must be a valid email address" },
      { "field": "items", "message": "must contain at least one item" }
    ]
  }
}
```

- **DON'T:** Return a single generic error message for a multi-field validation failure ("Invalid input") that forces the client to guess which field was wrong. This is especially punishing on forms with many fields — the user (or the calling client) has no way to know what to fix without trial and error.

- **DO:** Sanitize and normalize input where appropriate (trimming whitespace, normalizing unicode, rejecting null bytes) as part of the validation layer, in addition to type/format checks — validation confirms input is well-formed; sanitization ensures it's safe and consistent to store and display.

- **DON'T:** Conflate input sanitization with security controls against injection attacks (SQL injection, XSS, command injection). Sanitizing input for consistency is not a substitute for parameterized queries, output encoding, and framework-provided escaping — those are the actual defenses against injection, and validation/sanitization is a complementary, not equivalent, layer. (Covered in depth in the security-focused part of this document; here the point is not to assume "we validate input" means "we're safe from injection.")

- **DO:** Use distinct, specific error codes/types for distinct failure conditions (`resource_not_found`, `validation_error`, `duplicate_resource`, `insufficient_permissions`, `rate_limited`) rather than one generic `error: true` flag. Specific error codes let client code branch on the failure type programmatically (e.g., "if `duplicate_resource`, show a specific inline message") instead of string-matching a human-readable message that might change wording.

- **DON'T:** Change error message text or error code values between API versions (or even between deploys of the same version) without treating it as a compatibility concern. Client code that pattern-matches error messages (because no structured error code was provided) breaks the moment the wording changes — another reason structured, stable error codes matter more than the prose message.

- **DO:** Include a request/trace ID in every error response so a user can report an error and an operator can find the exact server-side log entry (and its full stack trace, request context, and any downstream calls) without ambiguity. Without a correlation ID, "it gave me an error" from a user report is nearly impossible to pin down among the logs of a busy production service.
```json
{
  "error": {
    "code": "internal_error",
    "message": "Something went wrong. Please try again.",
    "request_id": "req_9f8c2a1b3d4e"
  }
}
```

- **DON'T:** Show the same generic message for both "this failed because of something the client did wrong" and "this failed because of a server bug." Distinguish them by status code range (4xx vs. 5xx) at minimum, and ideally by error code, since the correct client behavior differs completely — a 4xx tells the client to fix the request and not blindly retry; a 5xx (especially with a `Retry-After` where applicable) tells it retrying might help.

- **DO:** Validate at the API boundary using the same schema/types that generate the API documentation (OpenAPI schema, GraphQL schema, Protobuf message definitions) so validation rules and documented contract can never drift apart — a field the docs say is required but the code doesn't actually enforce (or vice versa) is a contract violation waiting to confuse an integrator.

- **DON'T:** Let a validation error for a nested object collapse into a vague top-level message that doesn't identify which nested field failed (`"items[2].price": "must be a positive number"` is useful; `"invalid items"` is not). For deeply nested payloads, use a path-based field identifier so the client can map the error back to the exact form field or array index that failed.

- **DO:** Consider adopting a standardized error-response format like RFC 7807 (`application/problem+json`) when the API is public-facing or consumed by many independent clients, since a widely recognized shape means client libraries and tooling may already know how to parse it, rather than every API inventing its own bespoke error envelope from scratch.
```json
{
  "type": "https://example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 402,
  "detail": "Account balance ($12.40) is less than the requested amount ($50.00).",
  "instance": "/accounts/9f2a/withdrawals/771"
}
```

- **DON'T:** Treat `PATCH` and `PUT` as interchangeable when validating a request body — a `PUT` represents a full replacement and should generally require every field the resource model needs (missing required fields are a validation error), while a `PATCH` represents a partial update and should validate only the fields actually present in the payload, leaving every omitted field untouched. Applying `PUT`-style "everything required" validation to a `PATCH` request breaks the entire point of partial updates.

- **DO:** Validate path and query parameters with the same rigor as the request body — a numeric path parameter that isn't actually numeric, an enum query parameter with a value outside the accepted set, or a date parameter in the wrong format should all produce a clear `400`, not an unhandled exception or a silently-wrong query.

- **DON'T:** Let type coercion at the framework/routing layer mask invalid input silently — many routers will happily coerce a non-numeric path segment into `NaN`/`0`/`null` rather than rejecting the request outright, which can turn "the client sent garbage" into "the server queried for a nonsensical value and returned a confusing empty result" instead of a clear `400 Bad Request`.

- **DO:** Validate file uploads explicitly at the API boundary — enforce a maximum file size before reading the whole upload into memory, check the declared content type against an allow-list, and verify the actual file content matches its claimed type (a "magic bytes" check) rather than trusting the client-supplied `Content-Type` header or file extension alone, since both are trivially spoofable and a mismatched file type can be an attack vector (e.g., a script disguised as an image).

- **DON'T:** Stream an entire uploaded file into application memory before checking its declared size against the allowed maximum. Validate the size limit as early as possible (ideally enforced by the web server/reverse proxy before the request body is even fully received) so an oversized upload is rejected without ever letting an attacker exhaust server memory by uploading many large files concurrently.

- **DO:** Apply an explicit allow-list validation strategy (only permit known-good values/patterns) rather than a deny-list strategy (try to block known-bad values/patterns) wherever the set of valid input can be enumerated — an enum field, a fixed set of supported file extensions, an allowed set of sort/filter fields (as covered under REST API Design above). Deny-lists are inherently incomplete; there's always another way to express the "bad" input a deny-list author didn't anticipate, while an allow-list fails closed by construction.

- **DO:** Maintain a small, stable taxonomy of machine-readable error codes reused consistently across the whole API, rather than letting each endpoint's author invent a fresh, similarly-worded code for what's really the same underlying condition (`user_not_found` on one endpoint, `no_such_user` on another, `USER_404` on a third). A short, shared vocabulary is both easier for client code to branch on and easier for a new contributor to reuse correctly instead of coining a new variant.
```text
A small, consistent taxonomy, reused everywhere it applies:
  validation_error          -> 400/422, request failed schema validation
  authentication_required   -> 401, missing or invalid credentials
  permission_denied         -> 403, authenticated but not authorized for this action
  resource_not_found        -> 404, the requested resource does not exist
  conflict                  -> 409, e.g. a duplicate unique value, a version mismatch
  rate_limited               -> 429, too many requests
  internal_error             -> 500, unexpected server-side failure
```

- **DON'T:** Let error codes proliferate ad hoc without a shared registry or convention document that lists every valid code and what it means. Without a single reference, two different contributors (or two different AI-assisted sessions with no shared memory of prior decisions) will independently invent overlapping-but-differently-spelled codes for the same condition, which is exactly the kind of inconsistency-by-accretion this whole document exists to prevent.

- **DO:** Keep the machine-readable error `code` stable and locale-independent, and put any localization only in the human-readable `message` (driven by an `Accept-Language` header or an equivalent client-supplied locale), when the API serves clients across multiple languages. Client code should always branch on the stable `code`, never on the localized message text — a `message` that changes wording per-locale must never be the only signal a client has for identifying which error occurred.

- **DON'T:** Localize error messages by translating them at the point they're written into a raw string in application code, with no central catalog of translatable strings. Scattered inline translated strings are impossible to keep consistent (the same logical error worded three different ways in the same language across different endpoints) and impossible to hand to a translator systematically; route user-facing error text through a centralized message catalog/i18n system keyed by the same stable error code.

## Observability

- **DO:** Log every request with a structured format (JSON, not free-form text) including at minimum a timestamp, request ID, method, path, status code, and duration. Structured logs are queryable and aggregable by log-analysis tooling; free-form printf-style logs require fragile regex parsing to extract the same information later.
```json
{"timestamp":"2026-09-04T18:22:10Z","level":"info","request_id":"req_9f8c2a1b3d4e","method":"POST","path":"/orders","status":201,"duration_ms":142,"user_id":"usr_882"}
```

- **DON'T:** Log sensitive data — passwords, full credit card numbers, authentication tokens, API keys, government ID numbers — in plaintext, even at debug level, even temporarily "just for this investigation." Logs are typically retained far longer and read by far more people (support staff, log aggregation vendors, anyone with log access) than the original request; redact or omit sensitive fields at the logging layer itself so no code path can accidentally log them.

- **DO:** Propagate a single request/correlation ID across every service and log line involved in handling one request, generated at the edge (or accepted from an upstream caller and forwarded) and included in every downstream call's headers and every log statement. This is what makes it possible to reconstruct the full path of one request across a distributed system instead of manually correlating timestamps across unrelated log streams.

- **DON'T:** Rely on log statements scattered ad hoc through the codebase as the only signal for whether the API is healthy. Emit metrics (request rate, error rate, latency percentiles per endpoint — the request-oriented subset of the "four golden signals": latency, traffic, errors, saturation) to a metrics system, since logs are good for investigating a specific known incident but poor for noticing an emerging one without someone actively watching them.

- **DO:** Track latency as percentiles (p50, p95, p99), not just an average. An average can look perfectly healthy while a meaningful fraction of requests are timing out or taking seconds — the p99 is what your slowest, most frustrated real users are actually experiencing, and it's the number that catches a regression an average would smooth over.

- **DON'T:** Treat "the server didn't crash" as equivalent to "the API is healthy." A `/health` check that only verifies the process is running (and not, say, that it can reach its database and required downstream dependencies) will report healthy right up until every request is actually failing with a 500 — build health checks that verify the dependencies the service actually needs to function.

- **DO:** Instrument distributed tracing (spans for each significant operation — the inbound request, each downstream call, each database query) using a standard like OpenTelemetry, so a single slow or failing request can be visualized end-to-end across every service and dependency it touched, not just reconstructed manually from logs.

- **DON'T:** Alert on symptoms so noisy or so disconnected from actual user impact that the team learns to ignore them ("alert fatigue"). An alert that fires for every transient blip with no correlation to real degradation trains responders to dismiss alerts, which is exactly the failure mode that lets a genuine incident go unnoticed; tune alert thresholds to real user-facing impact (elevated error rate sustained over a window, not a single failed request).

- **DO:** Emit business-relevant metrics at the API layer, not just infrastructure metrics — request counts per endpoint, error rates by error code, counts of specific business events (orders created, payments failed) — since these often surface a problem (a broken integration silently failing validation on every request) faster than infrastructure metrics alone would.

- **DON'T:** Log the full request/response body indiscriminately for every request in production, especially for high-volume endpoints. Full-body logging at scale is expensive (storage, ingestion cost), often captures sensitive data inadvertently, and buries the signal that actually matters in noise; log structured metadata by default and reserve full-body capture for explicit debug-level tracing or sampled requests.

- **DO:** Correlate logs, metrics, and traces using the same request/trace ID scheme across all three, so an alert on an elevated error-rate metric can be traced directly to the specific request traces and log lines that explain it, without a manual, time-consuming cross-referencing step during an incident.

- **DO:** Define explicit SLIs (Service Level Indicators — the actual measured metric, e.g. "p99 latency of `POST /orders`") and SLOs (Service Level Objectives — the target for that metric, e.g. "99.9% of requests under 300ms over a rolling 30 days") for an API's most important endpoints, and alert based on SLO burn rate rather than on arbitrary raw thresholds picked without reference to what users actually need. An SLO gives the team an explicit, defensible answer to "is this API healthy enough" instead of a vague sense that things feel fine or feel bad.

- **DON'T:** Track only technical infrastructure health (CPU, memory, disk) as the sole observability signal for an API and skip endpoint-level, request-oriented metrics entirely. Infrastructure can look perfectly healthy (low CPU, plenty of memory) while the API itself is returning errors to real clients because of a downstream dependency, a bad deploy, or a logic bug — request-level metrics are what actually reflect the user-facing contract's health.

- **DO:** Keep audit logging (a durable, tamper-evident record of who did what to which resource, for compliance and security review) conceptually separate from operational application logging (debug traces, request timing, error diagnostics), even if they're written through the same logging pipeline. Audit logs typically need longer retention, stricter access control, and different guarantees (must not be silently dropped under load) than routine debug logs, and treating them identically risks either bloating the debug logs' retention costs or losing audit-critical events to routine log sampling/rotation.

- **DO:** Use log levels deliberately and consistently — `error` for something that needs attention, `warn` for a recoverable but noteworthy condition, `info` for normal significant events (a request completed, a job finished), `debug` for detail useful only when actively investigating something — and keep the production default at `info` or above, with `debug` enabled selectively (per-request, per-service, or temporarily) rather than left on globally, since `debug`-level logging in a busy production service at scale is often expensive enough on its own to affect performance and log-storage cost.

- **DON'T:** Log routine, expected outcomes at `error` level "to make sure it's noticed" — a validation failure from a malformed client request is not a server error and does not belong at `error` severity; misusing severity levels this way is exactly what causes alert fatigue and makes genuine `error`-level events harder to distinguish from routine noise in a dashboard or on-call rotation.

- **DO:** Sample traces (and, for very high-volume endpoints, sample debug-level logs) rather than capturing 100% of them once volume makes full capture prohibitively expensive, but bias the sampling toward keeping traces for errors and slow requests at a higher rate than routine fast successful ones — a flat random sample risks throwing away exactly the interesting traces (the failures and the outliers) that are most useful for debugging.

## Batch Operations & Partial Failures

- **DO:** Design batch/bulk endpoints (`POST /orders/bulk`, a GraphQL mutation accepting a list) to report per-item success/failure explicitly, not a single all-or-nothing status for the whole batch, unless the operation is genuinely and deliberately transactional (all-or-nothing by requirement). A batch of 100 items where item 47 fails validation shouldn't silently fail the other 99 unless that's the explicit, documented contract.
```json
{
  "results": [
    { "id": "itm_1", "status": "created" },
    { "id": "itm_2", "status": "failed", "error": { "code": "validation_error", "message": "price must be positive" } },
    { "id": "itm_3", "status": "created" }
  ]
}
```

- **DON'T:** Return a single `200 OK` for a batch operation where some items succeeded and others failed, with no indication of which. A caller that only checks the top-level status code will believe every item succeeded and never know to retry or alert on the failed ones — use `207 Multi-Status` (where the protocol supports it) or an explicit per-item results array, and document the convention clearly.

- **DO:** Make each item in a batch operation independently idempotent (via a per-item idempotency key, or a natural unique key on the item itself) so a retried batch — after a partial failure or a timeout — doesn't re-create the items that already succeeded the first time.

- **DON'T:** Process a batch operation as a single database transaction spanning every item when partial success is the intended semantic. Wrapping 500 independent inserts in one transaction means one bad row rolls back all 499 good ones — the opposite of the intended "best effort, report what failed" behavior; process items independently (or in smaller sub-transactions) when partial success is acceptable, and only use one all-encompassing transaction when the operation is genuinely required to be atomic.

- **DO:** Cap the maximum batch size accepted per request, and document the cap. An unbounded batch size is both a resource-exhaustion vector (a request that tries to process 500,000 items in one call) and a poor experience for the caller, who gets no incremental feedback until the entire oversized batch finishes or times out.

- **DON'T:** Let a batch endpoint silently drop items that fail some precondition (e.g., silently skipping items referencing a nonexistent related resource) without reporting them as failed in the response. Silent drops make bugs in the calling client invisible — the client believes its request succeeded completely and only discovers missing data much later, disconnected from the original request.

- **DO:** Consider whether a batch operation should short-circuit (stop on first failure) or continue processing remaining items (best-effort), and make that choice an explicit, documented parameter or behavior rather than an implementation accident. Different callers legitimately want different semantics (a financial batch job may want short-circuit; a bulk import may want best-effort with a failure report).

- **DO:** Treat bulk delete with the same per-item reporting discipline as bulk create/update — a `DELETE` against a filtered set (`DELETE /orders?status=cancelled`) or a bulk-delete-by-id-list endpoint should report how many were actually deleted and flag any IDs that didn't exist or couldn't be deleted (e.g., due to a foreign-key constraint), rather than a bare success/failure with no count or detail.

- **DON'T:** Let a bulk delete endpoint accept an unbounded or unfiltered scope with no confirmation mechanism. An endpoint that can delete "everything matching this filter" with a single request and no dry-run/preview option, no required confirmation token, and no audit trail is a severe blast-radius risk from a single mistaken or malicious request — for genuinely dangerous bulk operations, consider requiring an explicit count confirmation or a two-step confirm flow.

- **DO:** For a batch operation large enough to itself need the async job pattern (thousands of items, processed over minutes), expose a status endpoint reporting overall progress (`{ "processed": 4200, "total": 10000, "failed": 12 }`) rather than making the caller wait for one final response with no visibility into progress, consistent with the long-running-operations pattern covered under REST API Design above.

## Webhooks and Outbound Event Delivery

- **DO:** Sign outbound webhook payloads (an HMAC signature in a header, computed from a shared secret) so the receiving endpoint can verify the request genuinely came from your service and wasn't forged or replayed by an attacker who guessed the URL. This is the webhook equivalent of authenticating an inbound API request — the roles are just reversed.
```text
sending side:
  signature = HMAC-SHA256(secret, timestamp + "." + rawRequestBody)
  headers:  X-Webhook-Timestamp: 1717430400
            X-Webhook-Signature: sha256=4a1f...

receiving side:
  1. recompute the HMAC using the shared secret and the raw body actually received
  2. compare using a constant-time comparison (never `==` on the raw strings)
  3. reject if the timestamp is too far in the past (defends against replay of an
     old, previously-valid, intercepted request)
```

- **DON'T:** Assume a webhook delivery succeeded just because the HTTP call was made. Track delivery status explicitly (the receiver's response status code), retry failed deliveries with exponential backoff, and give the receiving party a way to see delivery history and manually redeliver — treat webhook delivery with the same at-least-once, retry-aware mindset as any other inter-service async communication.

- **DO:** Include a unique event ID in every webhook payload and document that receivers should treat delivery as at-least-once (deduplicate on that ID), for the same reason internal message-queue consumers need to be idempotent — network retries mean the same event may be delivered more than once.

- **DON'T:** Send webhook payloads containing the receiver's full internal representation of a resource (leaking fields never intended for external consumption) just because it was convenient to serialize the same internal object used elsewhere. Design a dedicated, versioned event payload shape for webhooks the same way you'd design any other public-facing API contract.

- **DO:** Set a reasonable timeout on webhook delivery attempts and don't let a slow or unresponsive receiver block the sender's own request-processing capacity. Deliver webhooks asynchronously (via a background job/queue), not inline within the request that triggered the event.

- **DON'T:** Retry webhook deliveries indefinitely with no backoff or cap. Combine retry backoff with a maximum attempt count (or a maximum retry window), and eventually mark a delivery as permanently failed and surface that to the receiving party (a dashboard, a status endpoint) rather than retrying forever against a receiver that's gone away.

- **DO:** Provide an explicit subscription-management API or dashboard (which event types a given endpoint receives, the ability to pause/resume/delete a subscription, a way to rotate the signing secret) rather than requiring a support ticket to change what a webhook receiver is subscribed to. This matters more as the number of event types grows — a receiver that only cares about `order.shipped` shouldn't be forced to filter out every other event type client-side because subscriptions aren't scoped at the API level.

- **DON'T:** Make it hard for an integrator to test or debug their webhook receiver against realistic payloads. Provide a way to send a test event on demand, and keep a delivery log (payload, response status, timestamp, retry attempts) visible to the receiving party so they can diagnose their own endpoint's failures without needing access to your internal logs.

- **DO:** Document explicitly whether webhook delivery order is guaranteed relative to the order events occurred, and if it isn't (which is the common case once retries are involved — a retried earlier event can be delivered after a later event that succeeded on its first attempt), design the payload so the receiver can determine relative ordering itself (a monotonic sequence number or the resource's own version/updated-at field) rather than relying on arrival order.

- **DON'T:** Let a receiver's naive "just apply whatever webhook arrives" logic silently apply an out-of-order update and overwrite newer state with older state. If delivery order isn't guaranteed, the payload needs to carry enough information (a version number, a timestamp) for the receiver to detect and discard a stale, out-of-order delivery — document this requirement clearly, since it's easy for an integrator to miss until it causes a real data-correctness bug.

## Caching and Performance at the API Layer

- **DO:** Use HTTP caching headers (`Cache-Control`, `ETag`, `Last-Modified`) correctly on cacheable `GET` endpoints, so clients, CDNs, and intermediate proxies can avoid re-fetching unchanged data. This is one of REST's original constraints (cacheability) and is nearly free performance once implemented correctly — a resource that rarely changes shouldn't be re-fetched and re-serialized on every single request.
```text
GET /products/42
->  200 OK
    Cache-Control: public, max-age=300
    ETag: "a1b2c3d4"

# a subsequent request with If-None-Match: "a1b2c3d4" gets a cheap 304 Not Modified
# if the resource hasn't changed, skipping the full response body entirely
```

- **DON'T:** Mark responses containing per-user or sensitive data as publicly cacheable (`Cache-Control: public`) by default, or omit cache headers entirely on endpoints that must never be cached (leaving the client/CDN to apply its own possibly-wrong default). An authenticated user's private data cached by a shared proxy and served to a different user is a serious data-leak class of bug; use `private` or `no-store` explicitly for anything user-specific.

- **DO:** Cache expensive, frequently-requested, infrequently-changing data at the application layer (Redis, an in-memory cache) with an explicit invalidation strategy tied to the write path that changes that data — invalidate or update the cache entry when the underlying data changes, rather than relying purely on a time-based expiration for data where staleness has a real user-facing cost.

- **DON'T:** Cache data with no invalidation strategy at all beyond "it'll expire eventually." A cache with only a TTL and no invalidation-on-write can serve visibly stale data for the entire TTL window after a change — acceptable for some data (a homepage banner), unacceptable for others (an account balance, an inventory count at checkout) — match the strategy to how stale that specific data is allowed to be.

- **DO:** Be deliberate about N+1 query prevention as a first-class API performance concern, not just a GraphQL-specific issue — the same problem occurs in REST endpoints that serialize a list of resources and lazily load a relation per item (see the Django, Rails, and Laravel framework sections above for ORM-specific eager-loading solutions), and in aggregating data from multiple internal services per item in a list.

- **DON'T:** Fetch more data than an endpoint's response actually needs "just in case," especially across a network boundary (a database query that selects every column, an inter-service call that fetches an entire related-object graph when only an ID is needed). Over-fetching wastes bandwidth and serialization cost on every single request, and compounds badly at scale even when each individual instance seems harmless in local testing with a handful of rows.

- **DO:** Use database connection pooling with limits tuned to the actual number of application server processes/threads and the database's connection capacity, rather than defaulting to an arbitrarily large pool size "to be safe." An oversized pool across many application instances can exhaust the database's total connection limit faster than an undersized one causes queueing — size deliberately based on real numbers, not intuition.

- **DON'T:** Perform expensive, synchronous work inline in the request/response cycle when it could be precomputed, cached, or deferred to a background job. A `GET /dashboard` endpoint that recomputes a heavy aggregation from scratch on every request, for every user, is a common self-inflicted performance problem that a materialized view, a cached aggregate, or a scheduled recomputation job solves far better than raw database indexing alone.

- **DO:** Stream a very large response (a bulk export, a large report) incrementally — newline-delimited JSON (NDJSON), chunked transfer encoding, or a similar streaming format — rather than building the entire payload in memory and sending it as one giant JSON array. Building a multi-hundred-megabyte array in memory before sending anything both risks the process running out of memory under concurrent large requests and delays time-to-first-byte until the entire dataset has been assembled, when the client could otherwise start processing records as they arrive.
```text
BAD:  build entire array in memory, then send one giant response:
      [ {...}, {...}, {...}, ... 500,000 more records ... ]

GOOD: stream newline-delimited JSON, one record at a time, constant memory:
      {"id": 1, "name": "..."}
      {"id": 2, "name": "..."}
      {"id": 3, "name": "..."}
      ...
```

- **DON'T:** Assume a streaming response format is a drop-in replacement with no client-side implications. NDJSON and chunked responses require a client that actually consumes the stream incrementally to get the memory/latency benefit — document the format clearly, since a client that just buffers the whole streamed response before parsing it gets none of the advantage and needs to know to parse line-by-line instead of as one JSON document.

- **DO:** Guard against cache stampede ("thundering herd") on a hot cache key's expiration — when a popular key expires, many concurrent requests can simultaneously discover the cache miss and all hit the origin/database at once. Use a mitigation such as a short lock around recomputation (so only one request recomputes while others wait or serve stale), staggered/jittered TTLs so many keys don't expire at the exact same instant, or serving stale-while-revalidate (return the slightly-stale cached value immediately while one request refreshes it in the background).
```text
BAD:  a single hot cache key expires -> 500 concurrent requests all miss simultaneously
      -> all 500 hit the database for the same expensive query at once -> DB overload

GOOD (stale-while-revalidate): serve the stale cached value immediately to all 500,
      while exactly one background refresh recomputes and updates the cache
```

- **DON'T:** Skip caching a "not found" result just because there's no positive data to cache. Without negative caching, a client (or a misbehaving script) repeatedly requesting a resource that doesn't exist forces a full database lookup on every single request; cache negative results too, typically with a shorter TTL than positive results, to absorb repeated lookups for nonexistent resources.

## Common AI-Assistant Mistakes in Backend/API Code

- **DON'T:** Invent framework methods, decorators, or configuration options that sound plausible but don't exist in the installed version — a `@nestjs/common` decorator that was never added, a Django ORM method name borrowed from a different ORM's API, a Spring annotation from a different Spring module that isn't on the classpath. This is one of the most common and most damaging AI-generated backend bugs because it often looks completely correct until it's actually run — verify against the project's actual installed dependency versions (check `package.json`/`requirements.txt`/`pom.xml`/`Gemfile`/`composer.json`) rather than generating from a generic training-data memory of "a framework like this one."
```text
BAD:  calling `Model.findOrFailByAsync()` on a Sequelize model — this method
      does not exist in Sequelize; it's a plausible-sounding blend of methods
      from Rails and Laravel APIs the model was trained on.
GOOD: check the installed ORM's actual documented API (e.g. Sequelize's
      real `findOne` + explicit null check, or `findByPk` with an explicit
      not-found handler) before using it.
```

- **DON'T:** Fabricate a REST endpoint, its path, or its request/response shape instead of checking the project's actual route definitions or OpenAPI spec. Generating client code against `/api/v2/users/{id}/preferences` because it "seems like the kind of endpoint this API would have" produces code that compiles/runs syntactically but fails at the first real request — always verify against the actual router/controller files, the OpenAPI spec, or by asking, rather than inferring an endpoint's existence from convention alone.

- **DON'T:** Return a `200 OK` (or arbitrarily pick `500`) for error conditions instead of a semantically correct status code, out of not tracking which status code fits which failure mode. This was covered at length above (see REST API Design → HTTP Methods and Status Codes) precisely because it's a frequent, easy-to-overlook mistake: a not-found should be `404`, a validation failure `400`/`422`, an auth failure `401`/`403` — get in the habit of picking the status code deliberately for every branch of an error-handling path, not defaulting to whatever the framework's error handler happens to produce.

- **DON'T:** Skip input validation on a newly generated endpoint because the "happy path" implementation looks complete and the task described only the success case. A generated handler that parses `req.body.email` and uses it immediately, with no format check, no length check, and no handling for a missing field, will pass a quick manual test with well-formed input and then break (or become a vulnerability) on the first malformed request in production — treat "what happens with missing, wrong-typed, or malicious input" as part of the actual task, not an optional follow-up.

- **DON'T:** Design a response schema that mirrors the database table's columns 1:1, exposing internal fields like `password_hash`, `internal_notes`, `deleted_at`, `stripe_customer_id`, or a foreign-key ID for a table the client has no business knowing exists. This happens naturally when an ORM object is serialized directly (see the framework-specific "don't return raw ORM entities" rules above) — always map to an explicit output DTO/serializer/response model, and treat "does this field belong in a public response" as a deliberate per-field decision, not a default of "include everything the query returned."
```python
# BAD: returns whatever columns the ORM query happened to select,
# including internal fields never meant to be public
@app.get("/users/{id}")
def get_user(id: int):
    return db.query(User).get(id)  # serializes every column, including password_hash

# GOOD: explicit response schema, only intentionally public fields
class UserPublic(BaseModel):
    id: int
    name: str
    created_at: datetime

@app.get("/users/{id}", response_model=UserPublic)
def get_user(id: int):
    return db.query(User).get(id)
```

- **DON'T:** Invent a new pattern for a single endpoint that ignores the conventions the rest of the codebase already uses — a different error-response shape, a different pagination style, a different validation library, a different naming convention — just because that pattern is what came to mind for this one generated snippet. An API assembled from many locally-plausible-but-mutually-inconsistent patterns is a hallmark of AI-generated slop: read a few neighboring files or existing endpoints first and match their established convention, even when a "better" pattern exists in the abstract — raise the inconsistency to a human rather than silently introducing a competing convention.

- **DON'T:** Generate a list endpoint (`GET /orders`, a GraphQL connection field) with no pagination, especially when the surrounding codebase's other list endpoints are paginated. This is an easy detail to drop when generating a single endpoint in isolation without looking at sibling endpoints for the established pattern (limit/offset params, cursor params, a default page size, a max page size) — treat pagination as a required part of any list endpoint's contract, not an enhancement to add later.

- **DON'T:** Generate a batch/bulk operation endpoint that either fails the entire batch on one bad item with no per-item detail, or silently drops failed items with a bare `200 OK`, without considering partial-failure semantics at all. As covered in the Batch Operations section above, a generated batch endpoint needs an explicit decision about all-or-nothing vs. best-effort behavior and a way to report per-item results — defaulting to whichever behavior the simplest code path happens to produce (usually: the whole batch throws on the first bad item, silently) is not a decision, it's an accident that becomes a production bug report.

- **DON'T:** Assume a generated authentication/authorization check is complete because the code compiles and a token is checked for validity, without separately verifying that the *authenticated* identity is *authorized* for the specific resource/action (see Auth at the API Layer above). A generated handler that checks `if (!user) return 401` and then proceeds to let any authenticated user modify any resource by ID, with no ownership or role check, is a textbook broken-object-level-authorization bug and one of the most common real-world API vulnerabilities — authentication alone is never sufficient justification to skip an explicit authorization check.
```javascript
// BAD: checks that *a* user is logged in, never checks it's *their* order
app.patch('/orders/:id', requireAuth, async (req, res) => {
  const order = await Order.findById(req.params.id);
  await order.update(req.body);
  res.json(order);
});

// GOOD: authenticated identity is checked against the resource's ownership
app.patch('/orders/:id', requireAuth, async (req, res) => {
  const order = await Order.findById(req.params.id);
  if (!order || order.userId !== req.user.id) return res.status(404).end();
  await order.update(req.body);
  res.json(order);
});
```

- **DON'T:** Produce a response that leaks a raw database error message, stack trace, or ORM-generated SQL string into the client-facing error body, especially when an error handler was generated quickly to "just make the endpoint return something on failure." A generic `catch (err) { res.status(500).json({ error: err.message }) }` is a common shortcut that inadvertently surfaces internal schema details, library versions, and file paths to any client that manages to trigger an unhandled error — route unexpected errors through a generic, non-leaking error handler and log the real detail server-side instead.

- **DON'T:** Generate rate limiting, pagination limits, or batch-size caps with an arbitrary hardcoded number chosen with no stated reasoning ("limit = 1000" with no comment on why), especially when a project convention or a business requirement already implies a different number. Cross-check against any existing limits used elsewhere in the codebase, and when no convention exists, state the reasoning (expected data volume, acceptable response size/latency) rather than picking a round number that seems generically safe.

- **DON'T:** Silently assume synchronous behavior is fine for a slow operation (sending an email, calling a third-party payment API, generating a report) just because it's the simplest code to write inline in a request handler. This produces API endpoints with wildly unpredictable latency and a real risk of request timeouts under load; flag operations that depend on external, slow, or unreliable calls and use a background job/queue plus a `202 Accepted` or a webhook/polling pattern instead of blocking the request thread on them, consistent with the async guidance in General Backend Architecture and Microservices Patterns above.

- **DON'T:** Generate GraphQL resolvers with an obvious N+1 pattern (fetching a related entity individually inside a per-item field resolver) because the naive version is syntactically the simplest thing to write, without adding a DataLoader or equivalent batching. This mistake is easy to make because the naive resolver works correctly in a small manual test (one or two items) and only reveals its cost at real list sizes — treat "does this resolver batch its data fetching" as a required check for any relation field, not an optimization to revisit "if it turns out to be slow."

- **DON'T:** Add a new database migration, config key, or environment variable as part of a generated change without surfacing it clearly to the human reviewer. A silently-added migration file, or code that reads a new `process.env.SOME_NEW_FLAG` with no corresponding entry in the project's env template/documentation, is easy to merge without anyone noticing it needs a deployment-time action (running the migration, setting the variable) — call out every new external dependency a generated change introduces.

- **DON'T:** Introduce a new dependency, package, or library version in generated code without checking it's actually declared in the project's manifest (`package.json`, `requirements.txt`, `pom.xml`, `Gemfile`, `composer.json`) and lockfile. Generated import statements for a package that isn't installed, or a version-specific API from a version newer than what's pinned, produce code that fails immediately at import/build time — always cross-check against what the project actually has available rather than what training data suggests "should" be available.

- **DON'T:** Over-engineer a simple CRUD endpoint with speculative abstraction layers — an interface with a single implementation, a generic repository-of-repositories, a plugin/strategy pattern for a decision that has exactly one option today — added "for future flexibility" the task never asked for. This is the opposite failure mode from the fat-controller/god-service anti-patterns covered under General Backend Architecture, but it's just as much of an antislop violation: unnecessary indirection makes the code harder to read and navigate for no present benefit, on the unverified assumption that a speculative future need will materialize in exactly the shape anticipated.

- **DON'T:** Generate an update endpoint (`PUT`/`PATCH`) that silently accepts and overwrites fields the client should never be able to set directly — `role`, `is_admin`, `account_balance`, `verified`, server-managed timestamps — just because the request body happened to include them and a generic "assign all provided fields onto the model" pattern was the easiest code to produce. This is the mass-assignment vulnerability class covered in the Rails and Laravel framework sections above, and it shows up just as easily in a hand-written or generated Express/FastAPI/Spring handler that naively spreads `req.body` onto a model — always work from an explicit allow-list of client-settable fields, never an implicit "everything the model has."
```javascript
// BAD: whatever fields the client sends get applied directly to the model
app.patch('/users/:id', requireAuth, async (req, res) => {
  const user = await User.findById(req.params.id);
  Object.assign(user, req.body);   // a client could send { "role": "admin" }
  await user.save();
  res.json(user);
});

// GOOD: only an explicit allow-list of fields can be set this way
const ALLOWED_FIELDS = ['name', 'bio', 'avatarUrl'];
app.patch('/users/:id', requireAuth, async (req, res) => {
  const user = await User.findById(req.params.id);
  for (const field of ALLOWED_FIELDS) {
    if (field in req.body) user[field] = req.body[field];
  }
  await user.save();
  res.json(user);
});
```

- **DON'T:** Generate a `DELETE` or destructive-action endpoint with no confirmation, no idempotency consideration, and no distinction between "resource already gone" and "resource successfully removed just now" in its response — a naive generated implementation frequently returns a `500` (or an unhandled exception) on a repeated `DELETE` of an already-deleted resource, when a `404` (or a `204` treated as idempotent-success) is what the method's semantics actually call for.

- **DON'T:** Assume the first plausible-looking library or package name from training data is actually the one installed in this project, or that its remembered API surface matches the version actually pinned in the lockfile. Two libraries in the same ecosystem solving similar problems (e.g., several different validation libraries, several different ORMs) have different method names and different behaviors for edge cases; check the project's actual dependency manifest and existing import statements before assuming which one is in use and what its API looks like.

- **DON'T:** Generate error handling that catches an exception only to immediately re-throw a less specific one, discarding the original error's type and message ("swallow and repackage"). This pattern often appears when a generated `try/catch` block is added defensively without a clear plan for what should happen on failure — it satisfies "there's a catch block" superficially while destroying the diagnostic information a real error handler (and the Observability practices covered above) depends on.
```python
# BAD: original exception type and detail are discarded
try:
    charge_result = payment_gateway.charge(amount, token)
except Exception:
    raise ValueError("something went wrong")   # original error info is now gone

# GOOD: preserve the original error as context, or handle it specifically
try:
    charge_result = payment_gateway.charge(amount, token)
except PaymentGatewayError as e:
    logger.error("payment charge failed", exc_info=e, extra={"amount": amount})
    raise PaymentFailedError("Unable to process payment") from e
```

## Content Negotiation, Conditional Requests, and Long-Running Operations

- **DO:** Honor the `Accept` header for content negotiation when an API genuinely supports more than one representation (JSON, XML, CSV export), and default sensibly (typically JSON) when the header is absent or unparseable, returning `406 Not Acceptable` only when the client explicitly requested a format the server truly cannot produce.

- **DON'T:** Encode the response format in the URL path (`/orders.json`, `/orders.xml`) as the primary mechanism when the `Accept` header already exists for exactly this purpose. A path-based format suffix is a legitimate legacy pattern in some ecosystems, but pick one mechanism per API and apply it consistently rather than supporting both inconsistently across endpoints.

- **DO:** Support conditional requests (`If-Match`, `If-None-Match`, `If-Modified-Since`) for both reads and writes where they matter. For reads, they enable cheap `304 Not Modified` responses (see Caching above); for writes, `If-Match` with an `ETag` implements optimistic concurrency control — the server rejects an update with `412 Precondition Failed` if the resource has changed since the client last read it, preventing a "lost update" where two concurrent editors silently overwrite each other's changes.
```text
GET /documents/9
->  200 OK
    ETag: "v7"

PATCH /documents/9
If-Match: "v7"
{ "title": "new title" }

->  412 Precondition Failed   (someone else already saved v8 in the meantime;
                                the client should re-fetch and retry, not blindly overwrite)
```

- **DON'T:** Implement "last write wins" silently for concurrent edits to the same resource without at least considering whether optimistic concurrency control is warranted for that resource. For low-stakes data (a user's UI preference) silent overwrite is often fine; for anything where a lost update has a real cost (collaborative documents, inventory counts, financial records), design deliberately for conflict detection rather than by accident.

- **DO:** Model operations that genuinely take longer than a typical request timeout (report generation, large exports, video transcoding, ML inference) as asynchronous jobs: the initial request returns `202 Accepted` immediately with a job/task resource the client can poll (`GET /jobs/{id}`) or a webhook the server calls on completion, rather than holding the HTTP connection open until the work finishes.
```text
POST /reports
->  202 Accepted
    Location: /jobs/job_8f2a
    { "job_id": "job_8f2a", "status": "pending" }

GET /jobs/job_8f2a
->  200 OK
    { "job_id": "job_8f2a", "status": "completed", "result_url": "/reports/rpt_991" }
```

- **DON'T:** Hold an HTTP request open for a long-running operation and rely on the client, load balancer, and every proxy in between all being configured with a generous enough timeout to not kill the connection first. Timeouts are configured independently at multiple layers (client library, reverse proxy, load balancer, gateway) that the API author doesn't fully control; a request design that depends on all of them cooperating for an operation that takes minutes is fragile by construction — use the async job pattern instead.

- **DO:** Give long-running job resources an explicit, documented set of states (`pending`, `running`, `completed`, `failed`, and ideally `cancelled`) and support querying and, where meaningful, cancelling a job in progress (`DELETE /jobs/{id}` or a dedicated cancel action), rather than leaving the client to infer job status only from whether a result eventually appears.

- **DON'T:** Design "create" semantics ambiguously between `POST` (server assigns the ID, not idempotent without a key) and `PUT` (client assigns the ID, naturally idempotent — an upsert) without picking one deliberately per resource. Using `PUT /orders/{client-generated-uuid}` to create a resource is a legitimate and idiomatic pattern when the client is the natural source of the identifier (e.g., an offline-first client generating UUIDs); using it inconsistently with `POST`-based creation elsewhere in the same API just because "PUT was already imported" for another endpoint creates confusion about which semantic a given endpoint follows.

## API Gateway, BFF, and Service Composition Patterns

- **DO:** Use an API gateway as the single entry point for cross-cutting concerns at the edge of a microservices system — TLS termination, authentication, rate limiting, request routing to the correct backend service, and response caching — so individual services don't each reimplement the same edge concerns independently and inconsistently.

- **DON'T:** Let the API gateway itself accumulate business logic (data transformation beyond simple routing, orchestration of multiple backend calls with business rules baked in). A gateway that grows business logic becomes a second, harder-to-test, harder-to-deploy monolith sitting in front of the "microservices" it's supposedly just routing to — keep it focused on cross-cutting, protocol-level concerns.

- **DO:** Consider a Backend-for-Frontend (BFF) — a thin, client-specific aggregation layer (one for the web app, a different one for the mobile app) — when different client types need meaningfully different data shapes or aggregation from the same underlying services. A BFF lets a mobile client get a lean, purpose-built response (fewer fields, pre-aggregated data to save round trips on a slow connection) without forcing the general-purpose backend API to grow client-specific special cases.

- **DON'T:** Build a generic "one API to rule every client" response shape that tries to satisfy a web dashboard's rich-data needs and a mobile app's bandwidth-constrained needs simultaneously by cramming both into every response. This tends to produce APIs with an ever-growing pile of optional query parameters (`?include=this&exclude=that&fields=a,b,c`) trying to let every client opt in/out of what it needs — past a certain complexity, a dedicated BFF per client type is simpler than one endpoint trying to be everything to everyone.

- **DO:** Use API composition (a gateway or BFF making several parallel backend calls and merging the results) for read-heavy aggregation across services, and consider CQRS-style read models (below) when composition itself becomes a performance bottleneck for a frequently-hit aggregate view.

- **DON'T:** Assume service discovery is unnecessary "because we know the IP addresses" in any environment where services are deployed on dynamic infrastructure (containers, autoscaling groups). Hardcoded service addresses break the moment infrastructure reschedules a service instance; use a service registry/discovery mechanism (DNS-based, a dedicated registry, or what the container orchestration platform provides) instead of hardcoded endpoints.

- **DO:** Consider the strangler fig pattern when migrating a monolith to services incrementally — route an increasing slice of traffic for one capability at a time to a new service behind a facade/gateway, while the monolith continues serving everything not yet migrated, until the monolith's corresponding code can be retired. This avoids the high-risk, high-cost "big bang rewrite," which has a long, well-documented track record of running over budget, over schedule, or failing outright before ever shipping.

- **DON'T:** Attempt a full rewrite-and-cutover migration from a monolith to microservices in one release. A big-bang migration means the old and new systems can't be run side by side to de-risk the transition, there's no way to migrate one capability at a time and validate it under real traffic before moving the next, and any single serious bug discovered late blocks the entire cutover.

- **DO:** Apply the bulkhead pattern — isolating resource pools (connection pools, thread pools, queues) per downstream dependency — so that one failing or slow dependency can't exhaust resources shared with calls to unrelated, healthy dependencies. Without isolation, a single slow downstream service can consume an entire shared thread/connection pool with requests waiting on it, starving unrelated requests that don't even depend on that slow service.
```text
BAD:  one shared HTTP connection pool for calls to Payments, Inventory, and Notifications
      -> Payments service degrades and hangs -> pool exhausted by pending Payments calls
      -> Inventory and Notifications calls now ALSO fail, despite being perfectly healthy

GOOD: separate connection pools (or a bulkhead per downstream) for each dependency
      -> Payments degrading only exhausts the Payments pool; Inventory and
         Notifications keep working normally
```

- **DON'T:** Let a service accept unbounded work from upstream callers or queues with no backpressure mechanism. A consumer that keeps pulling messages off a queue faster than it can process them, or a service that accepts every inbound request regardless of its current load, degrades in an uncontrolled way (unbounded memory growth, cascading timeouts) instead of failing predictably; apply backpressure (bounded queues, load shedding once past a concurrency limit, returning `503 Service Unavailable` deliberately when overloaded) so the system degrades gracefully rather than falling over entirely.

- **DO:** Treat a `503 Service Unavailable` (with a `Retry-After` header) as the correct, deliberate response when a service is intentionally shedding load because it's past capacity, distinct from a `500` (an unexpected bug) — a `503` tells the caller and any monitoring "this is a known, handled overload condition," which is a very different signal than an unhandled exception.

- **DON'T:** Assume a service mesh (Istio, Linkerd, or similar) is required for every microservices deployment. A mesh adds real capability (mTLS, fine-grained traffic policies, uniform observability, retries/circuit-breaking pushed to the infrastructure layer instead of reimplemented per service) but also real operational complexity — it's a deliberate tradeoff appropriate once the number of services and the cross-cutting policy needs justify it, not a default for a handful of services.

- **DO:** Use consumer-driven contract testing (a consumer service publishes its expectations of a provider's API, and the provider's CI runs those expectations against its own build) when two independently-deployed services need confidence that a change to the provider hasn't broken the consumer, without standing up a full end-to-end integration environment for every CI run. This catches breaking changes at the provider's build time, before a bad deploy ever reaches an environment where the consumer would actually fail.

- **DON'T:** Rely solely on end-to-end tests running against a fully deployed multi-service environment as the only safety net for cross-service compatibility. End-to-end environments are slow, flaky, and expensive to run on every commit across every service; use them for genuine end-to-end validation, but catch most contract-breaking changes earlier and cheaper with consumer-driven contract tests or schema-compatibility checks (protobuf's `buf breaking`, GraphQL schema diff tools) run per-service in CI.

- **DO:** Introduce an anti-corruption layer (a translation layer at the boundary) when integrating with a legacy system, a third-party API, or another team's service whose data model doesn't cleanly match your own domain's model. Translating at the boundary keeps that external system's quirks, inconsistencies, and legacy naming from leaking into and polluting your own domain model — without one, a legacy system's awkward data shapes tend to spread throughout the consuming codebase over time.

- **DON'T:** Let a single external dependency's API shape dictate your own service's internal domain model just because "that's the shape the data already comes in." Modeling your domain directly around an external system's representation makes your service brittle to that external system's changes and conflates "how we receive this data" with "how we think about this data" — two concerns that deserve to be decoupled by an explicit translation/mapping layer.

## CQRS, Event Sourcing, and the Outbox Pattern

- **DO:** Consider CQRS (Command Query Responsibility Segregation) — separate models/paths for writes (commands, validated against business rules) and reads (queries, optimized for the shapes callers actually need) — when a domain's read patterns and write patterns have genuinely diverged needs (e.g., writes are normalized and transactional, but reads need a heavily denormalized, pre-joined view for performance). Applying CQRS to a simple CRUD resource with no such divergence just adds architectural overhead with no compensating benefit — it is a targeted pattern, not a default.

- **DON'T:** Treat CQRS as requiring event sourcing or a separate datastore for reads and writes by definition — the core idea (separate models for reading vs. writing) is valid even within a single database and a single service; the more elaborate versions (separate datastores, eventual consistency between them) are an escalation to reach for only when the simpler version's limits are actually being hit.

- **DO:** Use the transactional outbox pattern when a service needs to atomically update its own database and publish an event about that change — write the event to an "outbox" table in the same local transaction as the business data change, then have a separate process (a poller or change-data-capture stream) publish it to the message broker. This avoids the classic dual-write problem, where writing to the database and publishing to a message queue as two separate, non-atomic operations can leave the two permanently out of sync if the process crashes between them.
```text
BAD (dual write, not atomic):
  1. save order to database
  2. [process crashes here]
  3. publish "OrderCreated" event  <- never happens; DB and event stream now disagree

GOOD (transactional outbox):
  1. in ONE local transaction: save order + insert an "OrderCreated" row into outbox table
  2. a separate relay process reads the outbox table and publishes to the message broker,
     marking each row published once confirmed
```

- **DON'T:** Publish an event to a message broker and then separately try to write to the local database as two independent steps in either order, assuming both will "usually" succeed together. This is exactly the dual-write problem the outbox pattern exists to solve; treat any service that needs both a local state change and a published event as needing an explicit strategy for keeping the two consistent, not an assumption that failures are rare enough to ignore.

- **DO:** Reserve event sourcing (storing an append-only log of domain events as the system of record, deriving current state by replaying them) for domains where the audit trail of what happened, and when, is itself a first-class requirement (financial ledgers, inventory movement history) — it brings real benefits there (a complete history, the ability to rebuild any past state, natural support for temporal queries) but at the cost of significant complexity (event schema evolution, snapshotting for performance, a genuinely different mental model for both reads and writes).

- **DON'T:** Adopt event sourcing for an ordinary CRUD-shaped domain just because it's an interesting pattern. The complexity cost (every query needs either a projection/read model or an expensive replay, every event schema change needs careful versioning, debugging "what is the current state" requires understanding the whole event history) is rarely worth paying unless the domain's actual requirements (audit, replay, temporal queries) call for it specifically.

- **DO:** Route a message that repeatedly fails processing (a "poison message" — malformed, referencing data that no longer exists, or triggering a bug in the consumer) to a dead-letter queue after a bounded number of retry attempts, rather than letting the consumer retry it forever and block every message behind it in the same queue partition/ordering group. A single poison message with no dead-letter path can stall an entire queue's throughput indefinitely, turning one bad event into a full processing outage for every other, perfectly valid message queued behind it.
```text
BAD:  consumer retries the same failing message forever
      -> every message behind it in the same ordered queue is stuck waiting

GOOD: after N failed attempts, move the message to a dead-letter queue
      -> processing continues for subsequent messages
      -> the dead-lettered message is available for manual inspection/replay
```

- **DON'T:** Treat a dead-letter queue as a place messages go to be forgotten. Alert on messages arriving in the dead-letter queue, and provide a way to inspect and manually replay them once the underlying issue (a bug, bad data) is fixed — a dead-letter queue with no monitoring or replay tooling just quietly accumulates data-loss incidents that no one notices until much later.
## Multi-Region and Data Residency Considerations

- **DO:** Design write paths deliberately around the consistency model a multi-region deployment actually provides — a single-writer-region-with-read-replicas setup gives strong consistency for writes but adds latency for writes originating far from the primary region, while a multi-writer/active-active setup removes that latency cost but requires an explicit conflict-resolution strategy for concurrent writes to the same record from different regions. Neither is free; pick deliberately based on which cost (write latency vs. conflict-resolution complexity) the system can better absorb.

- **DON'T:** Assume a globally-distributed API can offer the same read-your-own-write consistency guarantee everywhere with no added latency or design cost. A user who writes data in one region and immediately reads it back through a different region's replica can see stale data if replication lag hasn't caught up — either accept and document that possibility, route a given user's requests consistently to one region, or pay the cost of synchronous cross-region replication for the specific data where staleness isn't acceptable.

- **DO:** Treat data residency and cross-border data transfer requirements (where regulation constrains which region a given user's data may be stored or processed in) as an explicit input to the service/data architecture from the start, not a retrofit. Once a system assumes one global datastore, splitting specific users' or tenants' data out to a region-pinned datastore later is a substantial migration, not a configuration change.

- **DON'T:** Route every request to a single global backend region regardless of the requester's location when latency to that region is a real, avoidable cost for a meaningful fraction of users. Use geo-routing (DNS-based, an anycast edge network, or a gateway that picks the nearest healthy region) for latency-sensitive APIs — but make sure the routing decision is consistent with the actual data-residency and consistency model chosen above, rather than routing purely on latency and creating cross-region consistency problems as a side effect.

## Quick Checklist
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
