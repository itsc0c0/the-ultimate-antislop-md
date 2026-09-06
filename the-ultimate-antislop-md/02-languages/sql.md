# SQL

## Query Style and Formatting

- **DO:** Use `snake_case` for table and column names as the most broadly portable convention (avoiding the need to quote identifiers, which several engines require for mixed-case or reserved-word identifiers), rather than `camelCase` or `PascalCase` names that need quoting to preserve case in case-sensitive engines like PostgreSQL.
- **DO:** Write SQL keywords in a consistent case (all uppercase — `SELECT`, `FROM`, `WHERE`, `JOIN` — is the most common convention) and identifiers (table/column names) in a consistent case matching the schema's actual convention, so keywords are visually distinct from data at a glance.
  ```sql
  -- GOOD — keywords uppercase, identifiers as the schema defines them,
  -- one clause per line, consistent indentation
  SELECT o.id, o.total, c.name
  FROM orders AS o
  JOIN customers AS c ON c.id = o.customer_id
  WHERE o.status = 'shipped'
    AND o.created_at >= '2026-01-01'
  ORDER BY o.created_at DESC;
  ```
- **DO:** Put each major clause (`SELECT`, `FROM`, `JOIN`, `WHERE`, `GROUP BY`, `ORDER BY`) on its own line for any query beyond the most trivial single-table lookup, so the query's structure is scannable and diffs in version control show which specific clause changed.
- **DO:** Alias tables with short, meaningful aliases (`orders AS o`, not `orders AS a` or `orders AS xyz123`) and use the alias consistently to qualify every column reference once more than one table is involved, removing ambiguity about which table a column comes from.
- **DON'T:** Write unqualified column names in a multi-table query once there's any chance of a name collision (`id`, `created_at`, `name` are common offenders across joined tables) — an unqualified column reference that happens to resolve today can silently break or resolve to the wrong table after an unrelated schema change adds a same-named column elsewhere.
- **DO:** Use explicit `JOIN` syntax (`FROM a JOIN b ON a.id = b.a_id`) rather than the old implicit comma-join style with join conditions folded into `WHERE` (`FROM a, b WHERE a.id = b.a_id`) — explicit joins separate "how tables relate" from "how rows are filtered," and make it structurally obvious (and analyzer-checkable) whether a join condition was accidentally omitted, which is how a comma join silently becomes a cross join.
  ```sql
  -- BAD — easy to forget the join condition and silently get a cross join
  SELECT * FROM orders, customers WHERE orders.customer_id = customers.id;

  -- GOOD — the join condition is structurally required, not optional
  SELECT * FROM orders JOIN customers ON orders.customer_id = customers.id;
  ```
- **DO:** Use `AS` explicitly when aliasing columns and tables (even though many dialects allow omitting it) for clarity, especially in a long `SELECT` list mixing raw columns and computed expressions.
- **DO:** Format multi-line `IN` lists, `CASE` expressions, and subqueries with consistent indentation reflecting their nesting depth, so a reader can tell at a glance which parenthesized block a given line belongs to.
- **DO:** Run a SQL formatter/linter (`sqlfluff`, a dialect-aware IDE formatter) with a shared, checked-in configuration in projects with a meaningful volume of hand-written SQL, so formatting consistency doesn't depend on every contributor's personal habits.
- **DON'T:** Bury business logic inside deeply nested, unformatted subqueries or a single enormous unreadable one-liner when the same result could be expressed more clearly with a Common Table Expression (`WITH` clause) that names each intermediate step.
  ```sql
  -- GOOD — named CTEs make each intermediate step legible
  WITH recent_orders AS (
      SELECT * FROM orders WHERE created_at >= '2026-01-01'
  ), customer_totals AS (
      SELECT customer_id, SUM(total) AS total_spent
      FROM recent_orders
      GROUP BY customer_id
  )
  SELECT c.name, ct.total_spent
  FROM customer_totals ct
  JOIN customers c ON c.id = ct.customer_id;
  ```
- **DO:** Comment non-obvious business logic embedded in a query (why a specific filter exists, what a magic status code or threshold value means) directly in the SQL, since a query file reviewed months later carries none of the original context otherwise.
- **DO:** Use consistent, singular-or-plural table naming (pick one convention — plural table names like `orders` are the most common default across ORMs and style guides — and apply it project-wide) rather than mixing `order` and `line_items` and `Customer` naming styles within the same schema.
- **DO:** Prefer `COALESCE(value, fallback)` over database-specific null-handling functions (`ISNULL()`, `NVL()`) when the query might need to run on more than one database engine, since `COALESCE` is standard SQL supported broadly, while the alternatives are vendor-specific.
  ```sql
  -- Portable across PostgreSQL, MySQL, SQL Server, etc.
  SELECT COALESCE(nickname, first_name, 'Anonymous') AS display_name FROM users;
  ```
- **DON'T:** Use `SELECT DISTINCT` as a default fix for a query returning unexpected duplicate rows without first understanding *why* the duplicates appear (usually a join fanning out against a one-to-many relationship). `DISTINCT` can mask a join or grouping bug rather than fixing it, and it adds a real sorting/deduplication cost to every execution.
- **DO:** Write window functions (`ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)`, running totals, ranking) instead of a correlated subquery or self-join when the goal is a per-group calculation alongside individual rows — window functions are typically both clearer to read and faster to execute than the older self-join patterns they replace.
  ```sql
  SELECT
      customer_id,
      order_date,
      total,
      SUM(total) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_total
  FROM orders;
  ```
- **DO:** Use meaningful, descriptive names for computed/aliased columns (`AS total_revenue`, not `AS col1` or the unlabeled default the engine assigns) so a query's result set is self-documenting when consumed downstream.
- **DO:** Prefer `INNER JOIN`/`LEFT JOIN` spelled out explicitly over bare `JOIN` (which defaults to `INNER JOIN` in every major engine, but reads ambiguously to someone unfamiliar with that default) when a query's correctness genuinely depends on the join type — being explicit removes any doubt about whether the join type was a deliberate choice or an accident.
- **DO:** Use recursive CTEs (`WITH RECURSIVE`) for querying hierarchical or graph-like data (an org chart, a category tree, a bill-of-materials) natively in SQL, instead of fetching all rows into application code and reconstructing the hierarchy there — the recursive CTE keeps the traversal logic in the database and avoids pulling potentially large unrelated data across the network.
  ```sql
  WITH RECURSIVE org_chart AS (
      SELECT id, name, manager_id, 1 AS depth FROM employees WHERE manager_id IS NULL
      UNION ALL
      SELECT e.id, e.name, e.manager_id, oc.depth + 1
      FROM employees e JOIN org_chart oc ON e.manager_id = oc.id
  )
  SELECT * FROM org_chart ORDER BY depth;
  ```
- **DON'T:** Nest `CASE` expressions or subqueries so deeply that the query's logic can no longer be reconstructed by reading it top to bottom — extract intermediate logic into a named CTE, even for a value that's used only once, purely for the readability benefit of giving it a name.
- **DO:** Use `GROUP BY` with every non-aggregated selected column listed explicitly (rather than relying on a database-specific extension that tolerates omitting some of them), since standard SQL requires it and relying on a vendor extension makes a query silently non-portable to another engine.
- **DO:** Format long `IN (...)` value lists and multi-condition `WHERE` clauses with one value/condition per line once they exceed a handful of items, so a reviewer can scan the list for a specific value without parsing a long unbroken line.
- **DO:** Use a database's native JSON column type (PostgreSQL's `jsonb`, MySQL's `JSON`) with its JSON-path query operators when a column's data is genuinely semi-structured and doesn't benefit from normalization — but treat this as a deliberate exception for specific semi-structured fields (settings blobs, event payloads), not a default way to avoid schema design for genuinely structured, queryable data that belongs in real columns.
  ```sql
  -- Querying inside a jsonb column directly, with an index to back it
  SELECT * FROM events WHERE payload->>'event_type' = 'signup';
  CREATE INDEX idx_events_payload_type ON events ((payload->>'event_type'));
  ```
- **DO:** Use the database's native full-text search capability (PostgreSQL's `tsvector`/`tsquery` with a GIN index, MySQL's `FULLTEXT` index) for genuine text search needs on moderate data volumes, rather than a slow, unindexed `LIKE '%term%'` scan — reach for a dedicated search engine (Elasticsearch, Meilisearch) only once the database's native full-text search genuinely can't keep up with the required scale or relevance-ranking sophistication.
- **DO:** Use a schema migration tool (the framework's own migration system, or a standalone tool like Flyway/Liquibase) to track schema changes as versioned, ordered, reviewable files, rather than applying ad hoc `ALTER TABLE` statements directly against a production database outside of any tracked history.
- **DON'T:** Write a migration that both changes a column's type/constraint *and* backfills a large volume of existing data in a single blocking statement on a large production table — separate schema changes from data backfills, and batch the backfill, so a single migration doesn't hold a long-running lock against a live table.

## Indexing Awareness

- **DO:** Index every column (or leading column of a composite index) that's regularly used in a `WHERE` clause, a `JOIN` condition, or an `ORDER BY`/`GROUP BY` on a table of meaningful size — an unindexed filter/join column forces a full table scan that gets linearly slower as the table grows.
- **DO:** Understand that a composite (multi-column) index is usable for queries filtering on a *prefix* of its columns in the order they're defined — an index on `(customer_id, status)` speeds up a query filtering on `customer_id` alone or on `customer_id AND status`, but does not help a query filtering on `status` alone.
  ```sql
  -- An index on (customer_id, status) helps this:
  SELECT * FROM orders WHERE customer_id = 42 AND status = 'shipped';
  -- ...and this (leading-column prefix):
  SELECT * FROM orders WHERE customer_id = 42;
  -- ...but NOT this (status isn't the leading column):
  SELECT * FROM orders WHERE status = 'shipped';
  ```
- **DON'T:** Add an index on every column "just in case." Each additional index adds storage overhead and slows down every `INSERT`/`UPDATE`/`DELETE` on that table (since every index must be maintained on write) — index deliberately, based on actual query patterns, not reflexively.
- **DO:** Use `EXPLAIN`/`EXPLAIN ANALYZE` (the exact syntax and detail vary by database engine) to check whether a query actually uses the index you expect, rather than assuming an index exists on a column means the planner will use it — a wrapped column (`WHERE LOWER(email) = ...`), an implicit type mismatch, or a low-selectivity column can all cause the planner to skip an available index in favor of a full scan.
- **DON'T:** Wrap an indexed column in a function or expression in a `WHERE` clause (`WHERE YEAR(created_at) = 2026`, `WHERE LOWER(email) = 'x@example.com'`) unless the database supports and has an expression/functional index specifically covering that expression — most standard indexes can't be used once the indexed column is transformed inside the predicate.
  ```sql
  -- BAD — wraps the indexed column, defeats a plain index on created_at
  SELECT * FROM orders WHERE YEAR(created_at) = 2026;

  -- GOOD — a sargable range predicate the planner can use with a plain index
  SELECT * FROM orders WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';
  ```
- **DO:** Prefer sargable predicates (ones a planner can translate directly into an index range scan) — direct comparisons, `BETWEEN`, prefix `LIKE 'abc%'` — over predicates that force a full scan, such as a leading-wildcard `LIKE '%abc'` or a function applied to the indexed column.
- **DO:** Consider a covering index (one that includes every column a query needs, via `INCLUDE` or by putting selected columns into the index itself) for a hot, frequently-run query, so the engine can answer entirely from the index without a separate lookup into the table's main storage.
- **DON'T:** Assume indexing is a "set it once" concern — as query patterns and data volume change, periodically review actual slow-query logs and `EXPLAIN` output rather than relying on indexes chosen when the table was small or the application looked different.
- **DO:** Understand the cost of index maintenance on write-heavy tables — a table receiving very high insert/update volume with many indexes will show that cost directly in write latency, so index choices on such tables should weigh read speed against write cost more carefully than on a mostly-read table.
- **DO:** Use a unique index (not just a `UNIQUE` constraint applied loosely) to both enforce a business uniqueness rule at the database level and get the query-speed benefit of an index on that column, rather than relying solely on application-level uniqueness checks that are vulnerable to race conditions.
- **DO:** Index foreign key columns explicitly — some database engines (notably MySQL/InnoDB) create an index automatically for a declared foreign key, but others (PostgreSQL) do not, so a foreign key column can silently have no supporting index unless one is added deliberately, leaving every join or cascade-delete check on that relationship doing a full scan.
- **DO:** Consider a partial/filtered index (`CREATE INDEX ... WHERE status = 'active'` where the engine supports it) when queries consistently filter on a specific subset of rows (only "active" or non-deleted rows, for instance) — a partial index is smaller and faster to maintain than a full index covering rows the application rarely queries.
- **DON'T:** Add a low-selectivity index (one on a column like a boolean flag or a status with only two or three distinct values across millions of rows) expecting a large performance win — the query planner often reasonably chooses a full scan over such an index anyway, since the index doesn't narrow the candidate rows down by much.
- **DO:** Order the columns in a composite index by selectivity and actual query patterns — generally putting the column used for equality filtering before the column used for a range filter or sort, since an index can use an equality-filtered leading column efficiently and still apply the next column's ordering/range within each equality group.
- **DO:** Re-evaluate indexes after a significant schema or query-pattern change (a new feature introducing a new common filter, an old feature being removed) rather than only setting indexes once at initial schema design and never revisiting them as the application evolves.

## Avoiding `SELECT *`

- **DON'T:** Use `SELECT *` in application code, views, or any query whose result feeds further processing. It fetches columns nobody asked for (wasting I/O and network bandwidth), it breaks silently when the schema adds a column that changes positional assumptions downstream, and it defeats covering-index optimizations that only work when the exact needed columns are known.
  ```sql
  -- BAD — fetches every column, including ones the caller doesn't use,
  -- and its result shape silently changes if the schema changes
  SELECT * FROM users WHERE id = 42;

  -- GOOD — explicit about what's needed; stable and optimizable
  SELECT id, name, email FROM users WHERE id = 42;
  ```
- **DO:** List exactly the columns a query's caller actually needs, even when that's most of the table's columns — being explicit documents intent and insulates the query from future schema changes (a new column added to the table doesn't silently appear in, or break the shape of, every downstream consumer).
- **DON'T:** Assume `SELECT *`'s cost is negligible on a "small" table — a query on a small table today runs against a growing table tomorrow, and habits formed on small tables (`SELECT *`, missing indexes, unindexed joins) are exactly the ones that cause production incidents once the table grows past the size where they stop mattering.
- **DO:** Reserve `SELECT *` for genuinely exploratory, interactive use at a database console/REPL where the full row is being inspected by a human — never for code that ships.
- **DO:** When a query genuinely needs "all current columns" for something like a generic export or audit tool, generate the explicit column list programmatically from the schema at build/deploy time rather than using a literal `SELECT *` at query time, so behavior is still explicit and reviewable even though it's automated.
- **DON'T:** Use `SELECT *` inside a subquery or CTE purely out of habit, even when the outer query only consumes one or two of its columns — it's the same problem one level removed, and the same explicit-column discipline should apply at every nesting level.
- **DO:** Watch specifically for `SELECT *` combined with a `JOIN` — this duplicates any identically-named columns across the joined tables (both tables' `id`, for instance) in the result set, which is a common and confusing source of "why do I have two `id` columns" bugs.
- **DO:** Use `EXISTS`/`NOT EXISTS` instead of `SELECT *` inside a subquery whose only purpose is a presence check (`WHERE EXISTS (SELECT 1 FROM ... WHERE ...)`) — by convention, `SELECT 1` (or any constant) signals "we only care whether a row exists, not its contents," and most query planners already optimize `EXISTS` subqueries to stop at the first match without needing that hint, but the explicit constant keeps the query's intent unambiguous to a human reader too.
  ```sql
  SELECT * FROM customers c
  WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
  ```
- **DON'T:** Fetch entire rows into application code only to discard most fields immediately after (e.g., fetching a full user record just to check one boolean flag) — request exactly the narrow projection the calling code actually uses, both for `SELECT *` and for a manually-specified but still overly broad column list.
- **DO:** Reassess a query's column list whenever the calling code that consumes it changes — a column list is only as accurate as the last time someone updated it, and a stale explicit list (still fetching a column the caller stopped using two refactors ago) carries a smaller but real version of the same "fetch more than needed" cost `SELECT *` has.

## Parameterized Queries vs. String Concatenation (Injection Prevention)

- **DO:** Use parameterized queries (prepared statements with bound placeholders) for every value that originates outside the literal SQL text — user input, values computed from other queries, configuration read at runtime — with no exceptions. This is the actual mechanism that prevents SQL injection; treat it as a non-negotiable baseline for any code that touches a database, not a "nice to have."
  ```sql
  -- BAD — string-built SQL: a textbook SQL injection vulnerability
  -- query = "SELECT * FROM users WHERE username = '" + input + "'"

  -- GOOD — parameterized/prepared, the driver handles safe binding
  -- query = "SELECT * FROM users WHERE username = ?"
  -- driver.execute(query, [input])
  ```
- **DON'T:** Build a query by string concatenation or interpolation and rely on manual escaping (a hand-rolled "replace single quotes" function, or even a library's generic string-escaping helper used outside a proper parameter-binding API) as a substitute for real parameterization. Manual escaping is context-sensitive, error-prone, and has repeatedly been shown to have bypasses across different encodings and edge cases; parameter binding removes the entire category of risk rather than trying to sanitize around it.
- **DO:** Recognize that parameterization works for *values*, not for structural SQL — table names, column names, `ORDER BY`/`ASC`/`DESC` direction, or the SQL keyword itself can't be bound as a parameter. When these must be dynamic, validate them against a strict, hardcoded allow-list of known-safe identifiers before interpolating them into the query text.
  ```sql
  -- Dynamic sort column/direction: validate against an allow-list first,
  -- since these positions can't be bound parameters.
  -- allowed_columns = {"name", "created_at", "total"}
  -- allowed_directions = {"ASC", "DESC"}
  -- if column not in allowed_columns or direction not in allowed_directions: reject
  -- query = f"SELECT * FROM orders ORDER BY {column} {direction}"
  ```
- **DO:** Apply the same parameterization discipline to every place raw SQL appears, not just obviously user-facing forms — internal admin tools, reporting scripts, data-migration code, and "trusted internal" services are common places where ad hoc concatenated queries slip in on the (false) assumption that the input is safe.
- **DON'T:** Trust that upstream validation (a form field's regex check, a type constraint in an API schema) makes concatenation safe. Validation can be bypassed, incomplete, or simply absent on some code path that reaches the query later; parameterize at the query layer regardless of what validation happened earlier, as the actual injection defense.
- **DO:** Use an ORM's or query builder's parameterized methods for the large majority of application queries, but stay alert to raw-SQL escape hatches every ORM provides (raw query fragments, "unsafe" string-interpolated helpers) — these reintroduce exactly the same injection risk if a raw fragment concatenates a variable directly instead of using the escape hatch's own parameter-binding option.
- **DO:** Apply least-privilege database credentials per application/service (a reporting service's DB user shouldn't have `DROP TABLE` rights, a read-only dashboard shouldn't use a read-write connection) as defense in depth — parameterization prevents injection, but least-privilege limits the blast radius if some other layer of defense ever fails.
- **DON'T:** Log full raw SQL with bound parameter *values* interpolated back into the query text in a way that ends up in shared logs, if any of those values are sensitive (passwords, tokens, PII) — log parameterized query text and redact or omit sensitive bound values separately.
- **DO:** Use stored procedures with parameterized inputs as an additional layer of injection defense in environments that already rely on them heavily, but don't treat "we use stored procedures" alone as sufficient — a stored procedure that itself builds and executes dynamic SQL by concatenating its own input parameters is just as vulnerable to injection as inline application code doing the same thing.
- **DO:** Validate the *type* and basic shape of input before it reaches the query layer at all (an ID parameter is actually numeric, a date parameter actually parses as a date) as a defense-in-depth measure alongside parameterization — this catches malformed input earlier, with a clearer error, even though parameterization alone is what actually prevents injection.
- **DON'T:** Assume NoSQL or document-database query builders are immune to injection-style attacks just because they aren't "SQL" — parameter-unsafe query construction in MongoDB-style query objects (building a query object from unsanitized user input that can inject operators like `$where` or `$gt`) has its own well-documented injection patterns; apply the same "never build a query from unsanitized concatenated/merged user input" discipline regardless of the specific database technology.
- **DO:** Use an automated static-analysis or SAST tool capable of flagging string-concatenated SQL (many linters and security scanners detect this pattern directly) as a CI gate, catching a slip back into unsafe query construction before it reaches code review.

## Normalization vs. Denormalization Tradeoffs

- **DO:** Default to a normalized schema (each fact stored once, related data split across tables joined by foreign keys) for transactional/OLTP systems, since normalization prevents update anomalies (the same fact edited inconsistently in multiple places) and keeps storage compact.
- **DO:** Consider deliberate, targeted denormalization (duplicating a value, pre-computing an aggregate, flattening a join into a wide table) specifically for read-heavy paths where join cost or aggregation cost is a measured, real bottleneck — not as a default starting point, and not without first confirming normalization is actually the bottleneck.
- **DON'T:** Denormalize preemptively "for performance" without first measuring that the normalized version is actually too slow for the real workload. Premature denormalization trades away consistency guarantees for a performance problem that may not exist yet, and the resulting duplicate data now needs to be kept in sync by application logic, triggers, or a scheduled job.
- **DO:** When denormalizing, be explicit and deliberate about how the duplicated data stays in sync — a database trigger, an application-level write path that updates both copies, or an accepted eventual-consistency window via an async job — rather than duplicating data and hoping it stays consistent by convention.
- **DO:** Use materialized views (where the database supports them) for a denormalized, precomputed read-optimized shape of normalized source data, since the database manages the refresh mechanism rather than requiring hand-written synchronization code.
- **DON'T:** Conflate "denormalized" with "no foreign keys" or "no constraints." A denormalized reporting/analytics schema can still enforce referential integrity where it makes sense; denormalization is about controlled redundancy for read performance, not about abandoning data-integrity guarantees altogether.
- **DO:** Recognize that OLAP/analytics and data-warehouse workloads commonly favor deliberately denormalized star/snowflake schemas (fact tables plus denormalized dimension tables) because their read patterns — large aggregations across huge row counts — genuinely benefit from avoiding join fan-out, which is a different tradeoff than a typical OLTP application database.
- **DO:** Document the intended level of normalization for a given table/schema (and any deliberate denormalization) so future contributors understand which duplicated values are intentional design versus accidental drift that needs fixing.
- **DO:** Recognize the classic normal-form progression as a design vocabulary, even if applied informally — first normal form (atomic column values, no repeating groups), second (every non-key column depends on the whole primary key, relevant for composite keys), third (no non-key column depends on another non-key column) — since most real-world application schemas naturally land around third normal form without needing to invoke the formal definitions constantly.
  ```sql
  -- Violates normalization: repeating groups packed into one column
  -- CREATE TABLE orders (id INT, item_names TEXT); -- "widget,gadget,gizmo"

  -- Normalized: a proper one-to-many relationship instead
  -- CREATE TABLE orders (id INT PRIMARY KEY);
  -- CREATE TABLE order_items (order_id INT, item_name TEXT,
  --   FOREIGN KEY (order_id) REFERENCES orders(id));
  ```
- **DON'T:** Store a comma-separated or JSON-encoded list of IDs in a single column as a substitute for a proper join table, purely to avoid writing a join — this makes the values inside that column invisible to indexing, foreign-key constraints, and ordinary `WHERE`/`JOIN` queries, trading a small amount of upfront schema design effort for much larger, compounding query and integrity problems later.
- **DO:** Use a proper join/association table for genuine many-to-many relationships, with foreign keys to both sides and, where relevant, a composite unique constraint or its own primary key, rather than approximating the relationship with denormalized array/list columns on either side.
- **DO:** Weigh normalization decisions against actual query patterns for the specific table — a table that's written once and read very frequently in one specific shape is a much stronger candidate for denormalization than a table with balanced, varied read/write access across many different query shapes.

## Transaction Discipline

- **DO:** Wrap any sequence of writes that must succeed or fail together (a multi-table update representing one logical operation, like transferring funds between two accounts) in an explicit transaction (`BEGIN`/`COMMIT`, or the equivalent transactional API in the application's database driver/ORM), so a failure partway through rolls back every change instead of leaving the database in a half-updated, inconsistent state.
  ```sql
  BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
  COMMIT;
  -- If either UPDATE fails, roll back instead of committing a half-done transfer.
  ```
- **DON'T:** Leave a transaction open for longer than necessary — holding locks across a slow network call, a call out to an external API, or user-interaction wait time inside an open transaction blocks other transactions needing the same rows/tables and can exhaust connection pools under load. Keep the transactional scope tight around just the database operations that need atomicity.
- **DO:** Understand and choose the appropriate isolation level (`READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`, etc., with behavior varying by database engine) for a given transaction's needs, rather than accepting whatever the database's default is without knowing what it actually guarantees — the default isolation level differs across popular databases and its guarantees against phenomena like non-repeatable reads and phantom reads differ accordingly.
- **DO:** Use row-level locking (`SELECT ... FOR UPDATE`) deliberately when a transaction reads a row with the intent to update it based on that read, to prevent a race where another transaction modifies the same row in between the read and the write (the classic read-then-write race condition).
  ```sql
  BEGIN;
  SELECT quantity FROM inventory WHERE product_id = 42 FOR UPDATE;
  -- application checks quantity, then:
  UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42;
  COMMIT;
  ```
- **DON'T:** Rely on application-level "check then act" logic across two separate, unlocked queries (read the current value, decide in application code, write a new value) for anything that must be correct under concurrent access — without a transaction and appropriate locking, this pattern is a race condition waiting for concurrent traffic to trigger it.
- **DO:** Design retry logic for transactions that can fail due to serialization conflicts or deadlocks (common under `SERIALIZABLE` isolation or high write contention) — detect the specific "retry me" error code the database returns and retry the whole transaction, rather than treating every transaction failure identically.
- **DO:** Batch bulk write operations into reasonably sized transactions (not one enormous transaction touching millions of rows, and not one transaction per single row) — a single giant transaction holds locks and accumulates undo/rollback overhead for its whole duration, while one-row-per-transaction pays commit overhead on every single row.
- **DON'T:** Assume every statement is implicitly transactional just because the database supports transactions — some databases/configurations default to autocommit mode per statement, meaning each statement commits independently unless a transaction is explicitly started; verify what the actual default behavior is for the specific database and driver in use.
- **DO:** Set a reasonable statement/transaction timeout so a stuck or unexpectedly slow transaction doesn't hold locks indefinitely and cascade into blocking unrelated traffic.
- **DO:** Use `SAVEPOINT` for partial rollback within a larger transaction when one sub-step of a multi-step operation can fail and be recovered from without discarding the whole transaction's other already-valid work, rather than either committing prematurely between steps or rolling back the entire transaction for a recoverable sub-failure.
  ```sql
  BEGIN;
  INSERT INTO orders (...) VALUES (...);
  SAVEPOINT before_optional_step;
  -- if this next step fails, roll back just to the savepoint, not the whole transaction:
  -- ROLLBACK TO SAVEPOINT before_optional_step;
  INSERT INTO order_promotions (...) VALUES (...);
  COMMIT;
  ```
- **DON'T:** Assume a `ROLLBACK` undoes non-transactional side effects performed inside the same logical operation — an email sent, a file written, an external API called from application code between the transaction's `BEGIN` and `COMMIT` will not be undone by a database rollback. Design the operation so external side effects happen only after the transaction has successfully committed, or use a durable outbox pattern to coordinate them.
- **DO:** Understand deadlocks as a normal, expected occurrence under concurrent write load with multiple tables/rows locked in different orders by different transactions, not as a bug in isolation — design write paths to acquire locks in a consistent order across the whole application (e.g., always lock accounts in ascending ID order for a transfer) to reduce deadlock frequency, and always have retry logic for the deadlock error that does occur.
- **DO:** Use connection pooling with a bounded pool size appropriate to the database's actual connection limits, and make sure every acquired connection eventually returns to the pool (via a `finally`/`ensure`/`using`-style guaranteed release) — a connection leak under load is a common cause of a service that works fine in testing and then exhausts the database's connection limit in production.

## Common AI-Assistant Mistakes

- **DON'T:** Generate queries that work fine on a small development/seed dataset but don't scale — a query missing an index-backed filter, an N+1 query pattern issued once per row of an outer result set, or an unbounded `SELECT` with no `LIMIT` against a table expected to grow large. Reason about the query's behavior at realistic production data volumes, not just against a handful of test rows.
- **DON'T:** Generate application code that issues one query per row of an outer loop (the N+1 pattern) — e.g., fetching a list of orders, then querying each order's line items individually inside a loop — instead of a single join or a batched `IN (...)`/eager-load query that fetches everything needed in one or two round trips.
  ```sql
  -- BAD — N+1: one query for orders, then one more per order
  -- SELECT * FROM orders;
  -- for each order: SELECT * FROM line_items WHERE order_id = ?;

  -- GOOD — one additional query total, batched by the collected IDs
  -- SELECT * FROM orders;
  -- SELECT * FROM line_items WHERE order_id IN (<collected order ids>);
  ```
- **DON'T:** Ignore an existing schema's indexes, naming conventions, and constraints when generating new queries or migrations — check what indexes already exist before assuming a query needs a new one, and match existing naming/casing conventions rather than introducing an inconsistent style.
- **DON'T:** Default to `SELECT *` in generated code as a quick way to "just get the data" — list the actual columns needed, matching the surrounding codebase's existing query style.
- **DON'T:** Build SQL via string concatenation/interpolation of any external or user-influenced value in generated code, even inside a "just prototyping" snippet — default to parameterized queries every time a query includes a variable, since prototype code has a well-documented tendency to reach production unchanged.
- **DON'T:** Generate a schema or migration without foreign key constraints, appropriate `NOT NULL` constraints, or a primary key, on the assumption that "the application will enforce it." Database-level constraints are the last line of defense against data corruption from a bug anywhere in the application layer, and they should back up, not replace, application-level validation.
- **DON'T:** Generate a data migration or bulk update without wrapping it in an explicit transaction (with a tested rollback path) or without a `LIMIT`/batching strategy for a large table — an unbounded `UPDATE`/`DELETE` against a large production table run outside a controlled, batched, monitored process is a common source of real incidents (lock contention, replication lag, accidental full-table impact).
- **DON'T:** Assume a specific SQL dialect's syntax works identically everywhere — date/time functions, string concatenation operators (`||` vs. `CONCAT()` vs. `+`), `LIMIT`/`OFFSET` vs. `TOP`/`FETCH`, and upsert syntax (`ON CONFLICT` vs. `ON DUPLICATE KEY UPDATE` vs. `MERGE`) all differ meaningfully across PostgreSQL, MySQL, SQL Server, SQLite, and others — verify which database engine the target project actually uses before generating dialect-specific syntax.
- **DON'T:** Generate a schema without considering realistic data growth — a `TINYINT`/`SMALLINT` primary key or a column sized for "current" data volume without headroom is a common oversight that turns into a painful migration once the table outgrows the assumption.
- **DON'T:** Generate a query that silently returns incorrect results under `NULL` handling — comparing `column = NULL` (which is never true; use `IS NULL`), or assuming `NOT IN (subquery)` behaves intuitively when the subquery can return `NULL` values (it doesn't — a `NULL` in the `NOT IN` list makes the whole condition evaluate to unknown/false for every row, a well-known SQL surprise). Prefer `NOT EXISTS` over `NOT IN` when the subquery's result could contain `NULL`.
- **DON'T:** Generate a bulk `INSERT` as one statement per row inside an application loop when a single multi-row `INSERT` or a bulk-load utility would do the same work in a fraction of the round trips — batch inserts explicitly rather than defaulting to row-by-row insertion.
- **DON'T:** Assume a query that returns correct results is automatically a *good* query — verify it uses appropriate indexes and has a reasonable execution plan, not just that its output looks right against the sample data at hand.

## Quick Checklist
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
