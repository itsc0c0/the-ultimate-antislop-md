# Databases & Data Engineering

## Relational Schema Design

### Normalization (1NF–3NF)

- **DO:** Ensure every table satisfies First Normal Form: each column holds a single atomic value, and there are no repeating groups of columns like `phone1`, `phone2`, `phone3`. Atomic columns let you filter, index, and constrain individual values instead of parsing delimited strings at query time.
```sql
-- BAD: repeating groups, unbounded and unqueryable
CREATE TABLE contacts (id INT PRIMARY KEY, phone1 TEXT, phone2 TEXT, phone3 TEXT);

-- GOOD: one row per fact, arbitrarily many phones per contact
CREATE TABLE contact_phones (
  id INT PRIMARY KEY,
  contact_id INT NOT NULL REFERENCES contacts(id),
  phone TEXT NOT NULL,
  label TEXT
);
```

- **DON'T:** Store comma-separated or JSON-encoded lists of IDs in a plain text column to avoid creating a join table (e.g. `tag_ids TEXT` holding `"3,17,42"`). This breaks foreign key integrity, makes `WHERE tag_id = 17` require a full-table scan with string matching, and silently allows malformed values; use a proper many-to-many junction table instead.

- **DO:** Bring tables to Second Normal Form by ensuring every non-key column depends on the *whole* primary key, not just part of it, whenever the table has a composite primary key. A column that only depends on one part of a composite key belongs in a separate table keyed by that part alone.
```sql
-- BAD: 'product_name' depends only on product_id, not on (order_id, product_id)
CREATE TABLE order_items (
  order_id INT, product_id INT, product_name TEXT, quantity INT,
  PRIMARY KEY (order_id, product_id)
);

-- GOOD: product_name lives with product_id where it actually depends
CREATE TABLE products (product_id INT PRIMARY KEY, product_name TEXT);
CREATE TABLE order_items (
  order_id INT, product_id INT REFERENCES products(product_id), quantity INT,
  PRIMARY KEY (order_id, product_id)
);
```

- **DO:** Bring tables to Third Normal Form by removing transitive dependencies — non-key columns that depend on other non-key columns rather than directly on the primary key. A `zip_code` that determines `city` and `state` means `city`/`state` should not be duplicated on every row that also stores `zip_code`.
```sql
-- BAD: city/state are transitively dependent on zip, not on employee id
CREATE TABLE employees (id INT PRIMARY KEY, zip TEXT, city TEXT, state TEXT);

-- GOOD: the zip -> city/state fact lives once, in its own table
CREATE TABLE zip_codes (zip TEXT PRIMARY KEY, city TEXT, state TEXT);
CREATE TABLE employees (id INT PRIMARY KEY, zip TEXT REFERENCES zip_codes(zip));
```

- **DON'T:** Treat normalization as an end in itself and normalize past the point where it serves a real correctness goal. Splitting a table into a dozen 1-to-1 sub-tables "for purity" when the extra tables have no independent lifecycle or access pattern just multiplies joins without eliminating any duplication risk.

- **DO:** Use normalization as the default starting point for transactional (OLTP) schemas, then denormalize deliberately and locally where profiling shows a real need. Starting normalized keeps each fact stored once, so updates can never leave two copies of the same value out of sync.

- **DON'T:** Confuse "normalized" with "many small tables of unrelated data." Normalization is about eliminating redundancy and functional-dependency violations, not about table count; a wide table with no duplicated facts and no partial/transitive dependencies is already normalized even if it has thirty columns.

- **DO:** Recognize Boyce-Codd Normal Form (BCNF) violations in tables with multiple overlapping candidate keys, where a non-candidate-key column still determines part of a candidate key. This is rare in typical business schemas but worth checking in tables that model many-to-many relationships with extra attributes (e.g., a scheduling table where `(room, time_slot)` and `(teacher, time_slot)` both partially determine other columns).

### Denormalization Tradeoffs

- **DO:** Denormalize deliberately, in writing, when a specific read pattern is measured to be too slow under the normalized form — and document which column is now a cached copy of which source of truth. An undocumented denormalized column looks like a bug to the next engineer who finds it out of sync.
```sql
-- Deliberate denormalization: order_total is a cached sum of order_items,
-- kept in sync by the application (or a trigger) on every item change.
ALTER TABLE orders ADD COLUMN cached_total NUMERIC(12,2);
-- Comment in migration: "cached_total mirrors SUM(order_items.line_total);
-- recomputed on item insert/update/delete. Source of truth: order_items."
```

- **DON'T:** Duplicate a value across tables "for convenience" without a plan for keeping the copies consistent. A `user_email` column copied onto every `orders` row will drift the moment a user changes their email and nothing updates historical orders — decide explicitly whether that's the desired snapshot behavior or a bug waiting to happen.

- **DO:** Prefer computed/generated columns or materialized views over manually-synchronized denormalized columns when the database supports them. A generated column is guaranteed consistent by the engine; an application-maintained duplicate is only as consistent as every code path that touches it.
```sql
-- PostgreSQL / MySQL generated column: always consistent, never drifts
ALTER TABLE order_items
  ADD COLUMN line_total NUMERIC(12,2)
  GENERATED ALWAYS AS (quantity * unit_price) STORED;
```

- **DON'T:** Denormalize an entire schema upfront "for performance" before any query pattern is known. Premature denormalization locks in assumptions about access patterns that are often wrong, and now every write path has to maintain redundant copies that normalized form would never have required.

- **DO:** Use denormalization for read-heavy reporting and analytics schemas (star/snowball schemas, wide fact tables) where write consistency is handled by a controlled ETL process rather than ad hoc application code. Batch-controlled denormalization is far safer than denormalization maintained by dozens of scattered application code paths.

- **DON'T:** Denormalize by copying a mutable entity's full set of attributes into a child record when only an immutable snapshot is needed, and call it "done" without an update path. If the business genuinely wants a snapshot (e.g., the shipping address at time of order), name it that way and never expect it to reflect later edits to the customer's address book.

- **DO:** Reassess denormalization decisions once traffic patterns change or an index/materialized view could achieve the same speedup without redundant storage. Denormalization is a tool for a measured problem, not a permanent architectural default that outlives the reason it was introduced.

### Primary Key Design

- **DO:** Give every table a single, stable, immutable primary key that is never derived from mutable business data (names, emails, SSNs, order numbers users can request changed). A key that can change forces every foreign key reference and every external system that stored the old key to be updated in lockstep.

- **DON'T:** Use a "natural key" made of business data as a primary key unless that data is truly immutable and guaranteed unique for the life of the system. An email address looks unique until two users need to swap emails, or a company merges two customer records — at which point every foreign key referencing it becomes a migration nightmare.

- **DO:** Choose between auto-incrementing integers/bigints and UUIDs deliberately, based on real tradeoffs: integers are compact, sequential (cache- and index-friendly), and easy to reason about, while UUIDs avoid exposing row counts, support ID generation in distributed/offline clients before an insert, and merge safely across shards.
```sql
-- Sequential surrogate key: compact, index-friendly, but guessable/enumerable
CREATE TABLE orders (id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, ...);

-- UUID surrogate key: unguessable, generable client-side, but larger and
-- (for random v4) can fragment a B-tree index under high insert volume
CREATE TABLE orders (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), ...);
```

- **DON'T:** Default to random (v4) UUIDs as primary keys on high-write tables without considering the index fragmentation cost. Random UUIDs scatter inserts across the entire B-tree instead of appending at the end, which increases page splits and can measurably hurt insert throughput and cache locality on large tables; consider sequential/time-ordered UUIDs (UUIDv7, ULID, or a Snowflake-style ID) when both global uniqueness and insert locality matter.

- **DO:** Use surrogate keys (auto-increment or UUID) as the primary key even when a natural unique identifier exists, and enforce the natural identifier's uniqueness with a separate `UNIQUE` constraint. This gives you a stable join key that never has to change even if the business rules around the natural identifier do.
```sql
CREATE TABLE users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- stable surrogate key
  email TEXT NOT NULL UNIQUE                            -- enforced natural identifier
);
```

- **DON'T:** Expose raw sequential primary keys directly in public URLs or APIs when enumeration is a concern (`/orders/1042` lets anyone guess `/orders/1043`). Either use UUIDs for externally-facing identifiers, or keep the internal sequential ID private and expose a separate opaque public identifier/slug.

- **DO:** Use composite primary keys for genuine associative/junction tables where the combination of foreign keys *is* the identity of the row (e.g., `(user_id, role_id)` for a membership table), rather than adding a meaningless synthetic `id` column on top. The natural composite key already enforces "no duplicate membership" for free via the primary key constraint.

- **DON'T:** Assume every table needs a synthetic auto-increment `id` regardless of what the table represents. A pure junction table with a synthetic `id` plus a separate `UNIQUE(user_id, role_id)` constraint duplicates work the composite primary key would have done alone.

### Foreign Keys & Referential Integrity

- **DO:** Declare foreign key constraints for every relationship the schema is supposed to enforce, even if the application layer also validates them. The database is the last line of defense against orphaned rows caused by a bug, a manual data fix, a race condition, or a script run against production without the app layer involved.
```sql
ALTER TABLE order_items
  ADD CONSTRAINT fk_order_items_order
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE;
```

- **DON'T:** Rely solely on application-level "referential integrity" (checking in code that a referenced row exists before inserting) without a database-level foreign key. Application checks can be bypassed by a different service, a background job, a console/REPL session, or a concurrent request racing past the check — the database constraint is the only guarantee that holds under all of those.

- **DO:** Choose an explicit `ON DELETE` behavior (`CASCADE`, `RESTRICT`, `SET NULL`, `SET DEFAULT`, or `NO ACTION`) for every foreign key rather than accepting the database default without thinking about it. The right choice depends on whether child rows should die with the parent, block the parent's deletion, or survive as orphans with a nulled reference — picking the default by accident often produces the wrong one of these.
```sql
-- Comments belong to a post: delete them when the post goes away
comments.post_id REFERENCES posts(id) ON DELETE CASCADE

-- Orders reference a customer: don't allow deleting a customer with orders
orders.customer_id REFERENCES customers(id) ON DELETE RESTRICT

-- Audit log references a user, but should survive user deletion
audit_log.user_id REFERENCES users(id) ON DELETE SET NULL
```

- **DON'T:** Use `ON DELETE CASCADE` reflexively on every foreign key without considering blast radius. A cascade chain three or four tables deep means one `DELETE FROM customers WHERE id = 1` can silently wipe out orders, invoices, and payment records with no confirmation step and no way to know in advance how many rows were affected.

- **DO:** Index every foreign key column explicitly on databases that do not do this automatically (MySQL/InnoDB creates one automatically; PostgreSQL and SQL Server do not). An unindexed foreign key makes every child-lookup by parent a sequential scan, and it makes deleting a parent row slow because the database must scan the child table to check the constraint.
```sql
-- PostgreSQL does NOT auto-index foreign keys — this must be created explicitly
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

- **DON'T:** Add foreign keys pointing across bounded contexts or microservice ownership boundaries just because the data happens to live in the same physical database. A foreign key from `service_a`'s tables into `service_b`'s tables creates an undeclared coupling that breaks the moment those tables are split into separate databases; use an application-level reference (store the ID, validate via API/event) instead.

- **DO:** Use `DEFERRABLE INITIALLY DEFERRED` constraints (where supported) for the rare cases where a transaction must temporarily violate a foreign key or uniqueness constraint mid-transaction, such as reordering a set of rows that share a unique `position` column. Deferring the check to commit time avoids working around the constraint with fragile temporary values.

- **DON'T:** Leave foreign key columns nullable by default without deciding whether the relationship is genuinely optional. A nullable foreign key that should always be populated (e.g., every `order` must have a `customer_id`) hides missing-data bugs behind rows that simply have no parent instead of failing loudly at insert time.

### Choosing Appropriate Data Types

- **DO:** Use the narrowest, most specific data type that correctly represents the domain of a value — `DATE` for calendar dates with no time component, `BOOLEAN` for two-state flags, a proper `ENUM`/check-constrained type for a closed set of values. Specific types let the database validate, index, and store the data more efficiently than a generic type ever could.

- **DON'T:** Store dates, numbers, or booleans as `VARCHAR`/`TEXT` "to keep things flexible." A date stored as text can hold `"2024-13-45"` or `"next tuesday"` with no complaint from the database, sorts lexicographically instead of chronologically, and every query that filters or compares it needs an explicit cast.
```sql
-- BAD: no validation, wrong sort order, awkward range queries
CREATE TABLE events (id INT PRIMARY KEY, event_date TEXT);

-- GOOD: validated format, correct sort order, native range/interval math
CREATE TABLE events (id INT PRIMARY KEY, event_date DATE NOT NULL);
```

- **DO:** Store monetary values as a fixed-point decimal type (`NUMERIC`/`DECIMAL` with explicit precision and scale) or as integer minor units (cents), never as `FLOAT`/`DOUBLE`. Binary floating-point cannot represent most decimal fractions exactly, so repeated arithmetic on money stored as float accumulates rounding errors that show up as off-by-a-cent discrepancies in financial reports.
```sql
-- BAD: binary float cannot exactly represent 19.99, errors compound
CREATE TABLE prices (id INT PRIMARY KEY, amount FLOAT);

-- GOOD: exact decimal arithmetic
CREATE TABLE prices (id INT PRIMARY KEY, amount NUMERIC(12,2) NOT NULL);
```

- **DON'T:** Use `TIMESTAMP WITHOUT TIME ZONE` (or a naive datetime type) for any value that crosses machine or region boundaries, such as event timestamps, scheduling data, or audit logs. A naive timestamp has no way to record which time zone it was captured in, so every downstream consumer has to guess, and the ambiguity produces off-by-hours bugs the moment the app or its users span more than one time zone; store `TIMESTAMPTZ`/UTC-normalized timestamps instead and convert to local time only for display.

- **DO:** Pick an explicit, sufficient size for numeric and string types instead of accepting an arbitrary default. Choosing `SMALLINT` for a value that will never exceed a few thousand, or `VARCHAR(20)` for a code that is always exactly 6 characters, documents the domain and catches out-of-range or malformed data at write time.

- **DON'T:** Use `VARCHAR` with no length limit (or `TEXT` everywhere by default) for fields that have a genuine, known maximum length, such as country codes, postal codes, or phone numbers. An unbounded text column accepts garbage of any size, defeats simple sanity-check constraints, and forces the application to be the only place length is ever validated.

- **DO:** Use a native `ENUM` type or a `CHECK` constraint against a fixed list for columns with a small, well-known set of valid values (e.g., `status IN ('pending', 'shipped', 'delivered', 'cancelled')`). This rejects invalid values at the database level instead of relying on every application code path to validate them consistently.
```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  status TEXT NOT NULL CHECK (status IN ('pending','shipped','delivered','cancelled'))
);
```

- **DON'T:** Reach for a native `ENUM` type in databases where altering it later is expensive or awkward (e.g., adding a value to a PostgreSQL enum type used to require special handling inside transactions in older versions, and removing a value is still not directly supported). A `CHECK` constraint against a lookup table or a well-tested application-level enum is often more maintainable for values that are expected to grow.

- **DO:** Use a dedicated `UUID` type (or the database's closest native equivalent) instead of storing UUIDs as `VARCHAR(36)` when the database supports one. The native type stores the value in 16 bytes instead of 36 ASCII characters, compares and indexes faster, and rejects malformed UUID strings at write time.

- **DON'T:** Store structured, frequently-queried data as JSON/JSONB "because it's flexible" when the fields are known in advance and every query needs to filter or join on them. JSON columns skip type-checking, referential integrity, and (without specialized functional indexes) efficient filtering on individual fields — reserve JSON columns for genuinely variable or sparse attributes, not for data that is really a set of well-known columns in disguise.

### Indexing Strategy: When to Index

- **DO:** Index columns that are frequently used in `WHERE` clauses, `JOIN` conditions, `ORDER BY` clauses, and as foreign keys, based on the queries the application actually runs — not on guesses about what "might" be queried someday. An index that no query plan ever uses is pure overhead: it still has to be updated on every write.

- **DON'T:** Add an index to every column "just in case" or because a column is present in the `SELECT` list. Indexing is a targeted tool driven by actual read patterns and measured query plans, not a blanket policy — over-indexing slows every write and bloats storage without a matching read-side benefit.

- **DO:** Verify that an index is actually being used by the query planner (via `EXPLAIN`/`EXPLAIN ANALYZE`) before assuming it helps, and re-check after data volume or query patterns change. A planner can ignore an index entirely if the table is small enough that a sequential scan is cheaper, or if statistics are stale enough that its cost estimate is wrong.

- **DON'T:** Add an index on a low-cardinality column (a boolean flag, a column with only 2–3 distinct values across millions of rows) expecting it to speed up lookups. A B-tree index on a column where half the table shares the same value gives the planner little to narrow down and is usually skipped in favor of a sequential scan; a partial index on the rare value, or a composite index that leads with a more selective column, is usually the better tool.
```sql
-- Low-value index: 'is_deleted' is true/false for millions of rows
CREATE INDEX idx_orders_is_deleted ON orders(is_deleted);  -- rarely helps

-- Better: partial index targeting the rare, actually-queried case
CREATE INDEX idx_orders_active ON orders(id) WHERE is_deleted = FALSE;
```

- **DO:** Index columns used in `ORDER BY`/pagination cursors, not just columns used in `WHERE` filters. A query that filters efficiently via an index but then sorts a large intermediate result set with no supporting index still pays for an expensive sort step.

- **DON'T:** Assume a `UNIQUE` constraint and an index are the same recommendation for every case — a `UNIQUE` constraint always creates an index as a side effect (in most databases), but plenty of columns need an index without needing uniqueness, and vice versa a column can need uniqueness enforced without being a hot lookup path.

- **DO:** Use partial/filtered indexes to index only the subset of rows that queries actually target, when the database supports them (PostgreSQL, SQL Server). A partial index on `WHERE status = 'pending'` is far smaller and faster to maintain than a full index when 95% of rows are `'completed'` and queries only ever look at the pending ones.

- **DON'T:** Forget that indexes speed up reads at the direct cost of writes — every `INSERT`, `UPDATE`, or `DELETE` has to update every index on the affected columns. A table with fifteen indexes to support every conceivable report query will have measurably slower write throughput than one with three well-chosen indexes; be honest about which indexes are earning their keep.

- **DO:** Use covering indexes (an index that includes every column a query needs, via `INCLUDE` or by ordering key columns appropriately) for hot, narrow, frequently-run queries where an index-only scan would avoid touching the table's heap/data pages entirely.
```sql
-- Query only needs order_id, status, and total for a given customer:
CREATE INDEX idx_orders_customer_covering
  ON orders (customer_id)
  INCLUDE (status, total);   -- PostgreSQL syntax; enables index-only scan
```

### Composite Index Column Order

- **DO:** Order composite index columns with equality-filtered columns first, then range-filtered columns, then columns only used for sorting — matching how the query actually narrows down rows. A composite index is only useful as a left-to-right prefix match; putting the wrong column first can make the whole index unusable for a given query.
```sql
-- Query: WHERE customer_id = ? AND status = 'pending' ORDER BY created_at
CREATE INDEX idx_orders_customer_status_created
  ON orders (customer_id, status, created_at);
-- customer_id (equality, most selective) -> status (equality) -> created_at (sort)
```

- **DON'T:** Put the least selective column first in a composite index just because it appears first in the `WHERE` clause as written. `CREATE INDEX ON orders (status, customer_id)` when `status` has 4 distinct values and `customer_id` has millions is far less useful than the reverse order, because the leading column determines how much of the index the planner can prune before it even looks at the second column.

- **DO:** Recognize that a composite index on `(a, b, c)` can serve queries filtering on `a` alone, or `a` and `b`, or `a`, `b`, and `c` — but generally cannot efficiently serve a query that filters on `b` or `c` alone without `a`. Plan composite indexes around the actual left-anchored query shapes in the codebase rather than creating a separate single-column index per column out of habit.

- **DON'T:** Create both a composite index `(a, b)` and a redundant single-column index on `a` alone — the composite index already serves any query that only filters on `a`, since `a` is its leading column. The redundant single-column index just duplicates maintenance cost for no additional query coverage.

- **DO:** Put the column used for equality filtering before the column used for a range filter (`>`, `<`, `BETWEEN`) when both appear in the same query, since a B-tree index can only use one range condition to narrow the scan before falling back to filtering the remaining rows one by one.
```sql
-- Query: WHERE tenant_id = ? AND created_at > ?
CREATE INDEX idx_events_tenant_created ON events (tenant_id, created_at);
-- tenant_id equality narrows first; created_at range then scans a tight slice
```

- **DON'T:** Assume column order in a composite index is irrelevant because "the database will figure it out." The physical B-tree structure is sorted by the first column, then the second within each value of the first, and so on — reordering the same columns produces a structurally different index with different query coverage, not an equivalent one.

- **DO:** Place a trailing sort column at the end of a composite index whose leading columns are all equality filters, so the database can return already-sorted results directly from the index without a separate sort step.

### Avoiding Over-Indexing

- **DO:** Periodically audit indexes for ones that are never used by the query planner (most databases expose usage statistics — e.g., `pg_stat_user_indexes` in PostgreSQL) and drop them. An unused index is pure cost: it consumes storage, slows every write to that table, and adds noise when a human is trying to understand the schema's real access patterns.
```sql
-- PostgreSQL: find indexes that have never been scanned
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%_pkey';
```

- **DON'T:** Keep every index that was ever added "just in case it's needed again." Indexes are cheap to recreate later from a migration file if the access pattern actually returns; keeping a dead index around indefinitely for a hypothetical future need is a permanent, ongoing write-performance tax paid for a benefit that may never materialize.

- **DO:** Consolidate near-duplicate indexes on the same table into a single well-chosen composite index rather than keeping several overlapping single-column indexes. Three separate indexes on `customer_id`, `status`, and `created_at` each serve narrower cases than one well-ordered composite index on `(customer_id, status, created_at)`, at three times the write-time maintenance cost.

- **DON'T:** Add a new index to fix a slow query without first checking whether an existing index already covers it, or nearly does. Two indexes that differ only in a trailing included column, or in ascending vs. descending order where the database can already scan either direction, are usually redundant.

- **DO:** Weigh the cost of an index against the write volume of its table. A reporting table that receives a nightly batch load can tolerate many indexes since writes are rare and reads are the priority; a high-throughput transactional table where rows are inserted or updated many times per second needs a much more conservative indexing budget.

- **DON'T:** Add indexes to support a single ad hoc/one-time report or admin query that runs once a quarter. A permanent index that exists to serve an infrequent query is rarely worth its ongoing write cost — run that query with a temporary index, an `EXPLAIN`-guided one-off table scan, or against a read replica/analytics copy instead.

### Constraints as Correctness Tools

- **DO:** Add a `NOT NULL` constraint to every column that the business logic actually requires to always have a value, rather than leaving columns nullable "by default" and relying on the application to remember to check. A `NOT NULL` constraint turns a missing-value bug into an immediate, loud insert failure instead of a `null` silently propagating through the system until something downstream crashes.

- **DON'T:** Leave a column nullable just because the ORM's default migration generator makes columns nullable unless told otherwise. Accepting framework defaults without reviewing them means the schema documents "we didn't think about this" rather than an actual business rule about optionality.

- **DO:** Use `UNIQUE` constraints to enforce business-level uniqueness rules (one active email per user, one SKU per product) at the database level, not only via an application-layer "check first, then insert" pattern. A check-then-insert done in application code is a race condition under concurrent requests; only a database-level uniqueness constraint (backed by an index) is airtight.
```sql
-- BAD (race condition): two concurrent signups can both pass this check
-- SELECT COUNT(*) FROM users WHERE email = ?  -- app checks, then inserts

-- GOOD: the constraint is the source of truth, concurrency-safe
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);
```

- **DON'T:** Treat `CHECK` constraints as unnecessary because "the application already validates this." Validation logic gets duplicated, forgotten, or bypassed by scripts/migrations/other services touching the same table; a `CHECK` constraint is a single, unavoidable point of enforcement regardless of which code path writes the row.
```sql
ALTER TABLE products ADD CONSTRAINT chk_price_non_negative CHECK (price >= 0);
ALTER TABLE bookings ADD CONSTRAINT chk_dates_ordered CHECK (end_date > start_date);
```

- **DO:** Use `CHECK` constraints for simple, single-row, always-true invariants (ranges, non-negativity, valid enums, cross-column ordering within the same row) — the kind of rule that should never be false regardless of which application wrote the row. These are cheap to evaluate and catch entire classes of bugs before they ever reach application code.

- **DON'T:** Try to encode multi-row or cross-table business rules as `CHECK` constraints. Standard `CHECK` constraints can only see the row being written, not other rows or other tables — rules like "a customer can have at most 3 active subscriptions" belong in application logic, a trigger, or a serializable transaction, not in a `CHECK` clause that cannot express them correctly (and will be silently ignored on some databases if it references another table).

- **DO:** Give every constraint an explicit, meaningful name (`chk_price_non_negative`, `fk_orders_customer`) instead of letting the database auto-generate an opaque one. A named constraint produces a readable, immediately actionable error message ("violates check constraint chk_price_non_negative") instead of a cryptic autogenerated identifier that requires digging into the schema to interpret.

- **DO:** Use a database-level default value (`DEFAULT`) for columns with a well-known, unconditional default, rather than relying on every application code path to remember to set it. A `created_at TIMESTAMPTZ NOT NULL DEFAULT now()` is populated correctly by every insert path automatically, including manual `INSERT` statements and other services writing to the same table.

- **DON'T:** Confuse a `DEFAULT` value with a substitute for a `NOT NULL` constraint when the column's correctness genuinely requires an explicit value from the caller every time. A default silently fills in a value that may be wrong for a specific row (e.g., defaulting `country` to `'US'` for every user regardless of where they actually signed up) — use a default only when the fallback is truly always correct, not merely convenient.

- **DO:** Add exclusion constraints (PostgreSQL `EXCLUDE`) or equivalent application-enforced rules for genuinely non-overlapping-range invariants, such as preventing two bookings for the same room from overlapping in time. This is a case standard `UNIQUE`/`CHECK` constraints cannot express but which the database can still enforce directly.
```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE room_bookings
  ADD CONSTRAINT excl_room_no_overlap
  EXCLUDE USING gist (room_id WITH =, during WITH &&);
```

### Soft Deletes & Audit Columns

- **DO:** Add `created_at` and `updated_at` timestamp columns (with database-managed defaults/triggers, not application-set values) to every table where history and debugging matter. Database-managed timestamps are consistent regardless of which code path performs the write and cannot be forgotten by a new contributor.

- **DON'T:** Rely on the application layer to set `updated_at` on every single `UPDATE` statement. It is easy to add a new update path (a bulk update, a migration, a direct SQL fix) that forgets to touch `updated_at`, silently breaking anything that depends on it (cache invalidation, sync jobs, "last modified" displays); use a database trigger or the ORM's built-in timestamp hooks consistently instead.

- **DO:** Decide deliberately between hard deletes and soft deletes (an `deleted_at`/`is_deleted` flag) per table, based on whether the business needs to recover or audit deleted data, rather than defaulting to soft deletes everywhere out of caution. Soft deletes are not free: every single query on that table must now remember to filter out deleted rows, forever.
```sql
-- Soft delete pattern: every query must add this filter, or a default
-- scope/view must enforce it, or deleted rows silently reappear.
SELECT * FROM users WHERE deleted_at IS NULL AND email = ?;
```

- **DON'T:** Apply soft deletes to every table as a blanket policy and then forget to filter `deleted_at IS NULL` in a new query, a report, or a uniqueness constraint. A forgotten filter on a soft-deleted table is one of the most common sources of "deleted" data reappearing in the UI, and a unique constraint that doesn't account for soft-deleted rows can permanently block recreating a record with the same natural key (e.g., re-signing up with an email that belongs to a "deleted" account).

- **DO:** Scope unique constraints on soft-deletable tables to exclude soft-deleted rows (a partial unique index), so a "deleted" email address can be reused by a new signup. Without this, soft delete silently and permanently reserves every unique value it ever touched.
```sql
CREATE UNIQUE INDEX uq_users_email_active
  ON users (email) WHERE deleted_at IS NULL;
```

- **DON'T:** Use soft deletes on tables with strict compliance/legal deletion requirements (e.g., GDPR "right to erasure" data) as if flagging a row `deleted_at` satisfies the requirement. Soft-deleted data is still present in the database, in backups, and in replicas — regulatory erasure typically requires actual removal or irreversible anonymization, on a defined schedule, not a flag.

- **DO:** Track who made a change (`created_by`, `updated_by`) on tables where accountability matters, populated from the authenticated actor rather than trusted from client input. Audit columns that can be set arbitrarily by the client are worthless for accountability — they need to be derived server-side from the authenticated session.

### Multi-Tenancy Schema Patterns

- **DO:** Choose a multi-tenancy isolation strategy deliberately — shared tables with a `tenant_id` column, schema-per-tenant, or database-per-tenant — based on tenant count, isolation/compliance requirements, and operational overhead, rather than defaulting to whichever is easiest to build first. Each has a real tradeoff: shared tables scale operationally but risk cross-tenant data leaks from a missed filter; database-per-tenant isolates blast radius but multiplies migration and connection-pooling overhead by tenant count.

- **DON'T:** Build a shared-table multi-tenant schema and rely on every single query remembering to add `WHERE tenant_id = ?`. A single forgotten filter in one endpoint is a cross-tenant data leak; enforce tenant scoping structurally — row-level security policies, a query-building layer that injects the filter automatically, or a middleware that cannot be bypassed — rather than trusting every hand-written query.
```sql
-- PostgreSQL row-level security: tenant filter enforced by the database
-- itself, not by hoping every query remembers WHERE tenant_id = ?
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON invoices
  USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

- **DO:** Include `tenant_id` as the leading column of composite indexes and, where applicable, of composite primary/foreign keys in a shared-table multi-tenant schema. Since virtually every query is scoped to one tenant, `tenant_id` is the most consistently useful leading filter column across the whole schema.

- **DON'T:** Let a unique constraint on a shared multi-tenant table apply globally when it should apply per-tenant (e.g., a `UNIQUE(sku)` that should really be `UNIQUE(tenant_id, sku)`). A global constraint means Tenant A's use of SKU `"WIDGET-1"` blocks Tenant B from ever using the same SKU, which is very rarely the intended business rule.

- **DO:** Plan an explicit migration path from shared-table multi-tenancy to isolated tenancy (or vice versa) for the tenants that eventually need it (a large enterprise customer demanding a dedicated database, or a compliance regime requiring physical isolation), rather than assuming the initial choice is permanent. Multi-tenancy needs often diverge across a customer base as it grows.

### Partitioning & Sharding Basics

- **DO:** Consider table partitioning (splitting one logical table into multiple physical segments by range, list, or hash of a column) once a single table grows large enough that maintenance operations (index rebuilds, vacuum, backups) and query performance start to suffer, and there is a natural partition key such as date or tenant. Partitioning lets the database prune entire partitions from a query plan instead of scanning the whole table, and lets old partitions be dropped or archived in one cheap operation instead of a slow row-by-row delete.
```sql
-- PostgreSQL range partitioning by month: old data can be dropped
-- by detaching a partition instead of a slow DELETE over billions of rows
CREATE TABLE events (
  id BIGINT, occurred_at TIMESTAMPTZ NOT NULL, payload JSONB
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2026_01 PARTITION OF events
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

- **DON'T:** Reach for partitioning as a first response to "the table is slow" before ruling out simpler fixes — missing indexes, a bad query plan, stale statistics, or a query that fetches far more data than it needs. Partitioning adds real operational complexity (every query ideally includes the partition key, constraints and foreign keys have restrictions across partitions) and is not a free performance upgrade.

- **DO:** Choose a sharding key (for horizontal sharding across separate database instances, as distinct from partitioning within one instance) based on the access pattern that dominates the workload — typically the tenant ID or user ID — so that the overwhelming majority of queries hit exactly one shard. A poorly chosen shard key turns routine queries into fan-out queries across every shard, which defeats the purpose of sharding.

- **DON'T:** Shard a database before proving, with real numbers, that a single well-tuned instance (with read replicas, adequate indexing, and vertical scaling) cannot handle the load. Sharding multiplies operational complexity — cross-shard joins, cross-shard transactions, rebalancing — and should be adopted only once genuinely necessary, not preemptively "for scale" the application may never reach.

- **DO:** Keep related data that is frequently queried or transacted together on the same shard/partition (e.g., all of one tenant's rows across every table) so that common operations stay single-shard. Splitting a tenant's data across shards forces distributed transactions or joins for routine operations that should be trivially local.

### Naming Conventions

- **DO:** Adopt one consistent naming convention for tables, columns, indexes, and constraints across the entire schema (e.g., `snake_case`, singular or plural table names — pick one and stay consistent) and document it. Inconsistent naming (`user_id` here, `userId` there, `Users` and `orders` mixed case) forces every query author to guess or look it up, and breaks case-sensitive database configurations.

- **DON'T:** Mix naming conventions within the same schema, such as some tables plural (`users`) and others singular (`order`), or some foreign keys named `user_id` and others just `user`. Inconsistency compounds: an ORM's naming-convention auto-mapping breaks, generated migrations look wrong, and new contributors copy whichever inconsistent example they saw first.

- **DO:** Name foreign key columns after the referenced table in the singular plus `_id` (`customer_id` referencing `customers.id`) so the relationship is obvious from the column name alone, without needing to check the schema. A predictable naming pattern lets a reader infer joins correctly on sight.

- **DON'T:** Use reserved words or ambiguous generic names (`user`, `order`, `group`, `table`, `data`, `value`) as table or column names without quoting concerns in mind. Reserved words force every query to use dialect-specific quoting/escaping, which is a constant, avoidable source of syntax errors and dialect-portability problems.

- **DO:** Prefix or namespace tables belonging to a specific module or bounded context when a database is shared by multiple applications or teams (`billing_invoices`, `billing_payments`) so ownership and grouping are clear from the name alone in a database browser or a raw `\dt` listing.

### Views & Materialized Views

- **DO:** Use a regular (non-materialized) view to encapsulate a complex or frequently-repeated query — a multi-table join, a commonly-applied filter, a set of columns intentionally exposed to a reporting tool — so the logic lives in one named, reviewable place instead of being copy-pasted across the codebase. A view is just a stored query; it adds no storage or staleness risk, only a layer of naming and reuse.
```sql
CREATE VIEW active_customers AS
SELECT id, name, email FROM customers WHERE deleted_at IS NULL AND status = 'active';
```

- **DON'T:** Stack views on top of views several layers deep without checking the combined query plan. Each layer of view-on-view nesting adds planning complexity, and on some databases can prevent the planner from pushing filters down efficiently through all the layers — occasionally verify with `EXPLAIN` that a deeply nested view still produces a sane plan rather than assuming composability is free.

- **DO:** Use a materialized view (a view whose result is computed once and stored, then refreshed on a schedule or on demand) for expensive aggregations or joins that are read far more often than the underlying data changes — a daily sales dashboard, a search index derived from several tables. This trades staleness (bounded by the refresh interval) for read speed, the same tradeoff as any other cache.
```sql
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT date_trunc('day', created_at) AS day, SUM(total) AS revenue
FROM orders GROUP BY 1;

-- Refresh on a schedule (e.g., via a cron job or scheduler)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;
```

- **DON'T:** Treat a materialized view as automatically up to date. Unlike a regular view, a materialized view's contents are frozen at the last refresh — querying it without understanding the refresh schedule can silently return stale data to a caller who assumed it was live; document the refresh cadence prominently wherever the materialized view is used.

- **DO:** Use the non-blocking refresh variant (`REFRESH MATERIALIZED VIEW CONCURRENTLY` in PostgreSQL, which requires a unique index on the view) for materialized views queried by live traffic, rather than the default blocking refresh, which locks the view against reads for the refresh's full duration.

- **DON'T:** Use a materialized view as a substitute for proper indexing on the base tables. A materialized view that's really just `SELECT * FROM orders` with no aggregation adds a stale, redundant copy of the data with none of the benefit — reach for a materialized view specifically when there's real computation (aggregation, multi-table join) worth avoiding on every read.

### Full-Text Search & Specialized Column Types

- **DO:** Use the database's native full-text search capabilities (PostgreSQL's `tsvector`/`tsquery` with a GIN index, MySQL's `FULLTEXT` index, SQL Server's Full-Text Search) for genuine text-search requirements — searching articles, product descriptions, support tickets by keyword — instead of `LIKE '%term%'` pattern matching, which cannot use a standard B-tree index and forces a full scan on every search.
```sql
-- LIKE with a leading wildcard cannot use a standard index — full scan
SELECT * FROM articles WHERE body LIKE '%database%';

-- Native full-text search: relevance-ranked, index-backed
ALTER TABLE articles ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', body)) STORED;
CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);
SELECT * FROM articles WHERE search_vector @@ to_tsquery('english', 'database');
```

- **DON'T:** Reach for a dedicated external search engine (Elasticsearch/OpenSearch) as a default before checking whether the primary database's native full-text search already meets the actual requirement. Running and keeping a second search-specific system in sync is real, ongoing operational overhead that isn't justified until the native option's relevance ranking, language support, or scale genuinely falls short.

- **DO:** Use native geospatial types and indexes (PostGIS's `geography`/`geometry` types with a GiST index, MySQL's spatial types with a spatial index) for location-based queries (nearest N points, points within a radius, polygon containment) rather than computing distance with raw latitude/longitude math in application code on every row, which cannot use a spatial index at all.
```sql
CREATE INDEX idx_stores_location ON stores USING GIST (location);
SELECT name FROM stores
ORDER BY location <-> ST_MakePoint(:lng, :lat)::geography
LIMIT 10;   -- nearest 10 stores, index-backed
```

- **DON'T:** Store latitude and longitude as two independent plain numeric columns and expect efficient "nearby" queries. Without a spatial index, a "find stores within 5km" query becomes a full-table scan computing distance for every row — a real geospatial type with a spatial index turns the same query into an index-backed search.

- **DO:** Query JSON/JSONB columns using the database's native JSON operators and functional/expression indexes when a specific nested field is queried often, rather than always fetching the whole JSON blob and filtering in application code.
```sql
-- Index a specific frequently-queried JSON field directly
CREATE INDEX idx_orders_metadata_source ON orders ((metadata->>'source'));
SELECT * FROM orders WHERE metadata->>'source' = 'mobile_app';
```

### Access Control & Least Privilege

- **DO:** Grant database accounts/roles only the specific privileges the application or person actually needs (least privilege) — a typical application connection needs `SELECT`/`INSERT`/`UPDATE`/`DELETE` on its own tables, not schema-altering `CREATE`/`DROP`/`ALTER` privileges, and almost never superuser/admin rights. A compromised application credential with minimal privileges limits the blast radius of that compromise; a compromised superuser credential does not.
```sql
CREATE ROLE app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
-- Deliberately no CREATE, DROP, ALTER, or superuser privileges
```

- **DON'T:** Use a single shared, highly-privileged database credential (often the default superuser/root/admin account) for every application, migration tool, and human who ever needs database access. A shared over-privileged credential makes it impossible to know who or what actually performed a given action, and a leak of that one credential compromises everything at once instead of one narrow slice.

- **DO:** Use a separate, more restricted credential for read-only workloads (reporting tools, analysts, dashboards, read replicas) than for the primary read-write application connection. A `SELECT`-only role cannot accidentally or maliciously modify data, which is a meaningful safety property for any tool or person whose actual job never requires writing.

- **DON'T:** Grant broad access to production data to every developer or every service "for convenience" during development. Production data routinely contains real customer PII; broad, unaudited access to it is both a security risk and, in many jurisdictions, a compliance violation — use masked/synthetic data for development wherever the real thing isn't specifically required, and gate real production access behind an audited, justified process.

- **DO:** Use row-level security (PostgreSQL `RLS`, or application-enforced equivalents) or column-level privilege restrictions when different callers of the same database need to see different subsets of the same table's rows or columns — a support agent role that can see order status but not full payment details, for instance — rather than relying entirely on application-layer filtering that could be bypassed by a direct connection.

- **DON'T:** Leave default database accounts, default passwords, or overly permissive default network access (a database listening on a public IP with no firewall/VPC restriction) in place past initial setup. Default credentials and open network access are among the most common causes of database breaches, and they are found by automated scanners within minutes of exposure, not eventually.

### Database Documentation & Schema Comments

- **DO:** Attach human-readable comments directly to tables and columns using the database's native comment mechanism (`COMMENT ON TABLE`/`COMMENT ON COLUMN` in PostgreSQL, `COMMENT` clause in MySQL's `CREATE TABLE`) for any table or column whose purpose, units, or valid range isn't obvious from its name alone. A comment that travels with the schema itself (visible via `\d+` or `SHOW CREATE TABLE`) survives far better than the same information living only in a wiki page nobody finds.
```sql
COMMENT ON COLUMN orders.total IS 'Order total in minor currency units (cents), always non-negative';
COMMENT ON TABLE orders_archive IS 'Orders older than 7 years, moved here by the nightly archival job; see retention policy doc';
```

- **DON'T:** Leave ambiguous columns (a generic `type INT`, a `flags` bitmask, a `value` column whose unit isn't clear) entirely undocumented and expect every future reader to reverse-engineer their meaning from application code. An undocumented ambiguous column is a recurring tax paid by every developer (and every AI assistant) who touches that table afterward, repeated indefinitely until someone finally writes the explanation down.

- **DO:** Maintain an entity-relationship diagram (ERD) or equivalent schema visualization for any database with more than a handful of tables, regenerated automatically from the live schema where possible rather than hand-maintained and prone to drifting out of date. A current ERD is often the fastest way for a new contributor (human or AI) to understand how the tables relate before writing a single query.

- **DON'T:** Let schema documentation live only in an out-of-band document (a wiki page, a design doc) with no automated check that it still matches the actual schema. Documentation that isn't verified against reality degrades silently — treat auto-generated documentation (schema comments, generated ERDs from the live database) as more trustworthy by default than hand-maintained descriptions that can quietly go stale.

- **DO:** Document non-obvious business rules encoded as constraints (why a `CHECK` constraint enforces a particular range, why a partial unique index excludes certain rows) directly in the constraint's name or an adjacent comment, so the *reason* for the rule survives alongside the rule itself, not only its mechanical effect.

### Reference & Lookup Tables vs Hardcoded Application Constants

- **DO:** Store a genuinely data-driven set of values (countries, currencies, tax categories, status codes that might grow or whose display text might change) in a proper lookup/reference table rather than hardcoding them as constants scattered across application code. A reference table can be queried, joined against, updated without a deploy, and validated by a foreign key — none of which is true of a hardcoded list duplicated in several places in application code.
```sql
CREATE TABLE order_statuses (
  code TEXT PRIMARY KEY,
  display_name TEXT NOT NULL,
  sort_order INT NOT NULL
);
-- orders.status now has a real FK target instead of an unenforced hardcoded list
ALTER TABLE orders ADD CONSTRAINT fk_orders_status FOREIGN KEY (status) REFERENCES order_statuses(code);
```

- **DON'T:** Build a reference table for a truly fixed, small, rarely-changing set of values where a simple `CHECK` constraint or application-level enum would be simpler and sufficient (the seven days of the week, for instance). A reference table adds a join and a maintenance surface that isn't worth it when the set of values is genuinely closed and effectively never changes.

- **DO:** Keep a reference table's values and an application-level enum in sync deliberately when both exist (the application enum for compile-time type safety, the database table for referential integrity and queryability) — via a generated/derived source, a migration that updates both together, or a startup check that verifies they match — rather than letting the two drift independently.

- **DON'T:** Duplicate the same "list of valid statuses" logic independently in multiple services or multiple layers of the same application (a hardcoded list in the frontend, a different hardcoded list in the backend, and a `CHECK` constraint in the database, each maintained separately). When one drifts out of sync with the others — a new status added to the database but not the frontend's dropdown, for instance — the result is a confusing, hard-to-trace bug that only shows up for the specific new value.

### Sequence & Auto-Increment Gotchas

- **DO:** Expect gaps in auto-increment/sequence values as normal, not a bug — a rolled-back transaction, a failed insert, or (in PostgreSQL) simply the way sequences are pre-allocated in batches by some drivers all consume a sequence value that is never reused. Code or reports that assume IDs are perfectly contiguous will be wrong; never rely on ID gaps (or their absence) to infer anything about row counts or deletions.

- **DON'T:** Use `MAX(id) + 1` computed in application code as a substitute for the database's native auto-increment/sequence mechanism to generate a new primary key. This is a textbook race condition — two concurrent inserts can both read the same `MAX(id)`, both compute the same "next" value, and one insert either fails on a uniqueness violation or, worse, silently overwrites data if uniqueness isn't enforced.

- **DO:** Understand the specific auto-increment behavior of the database in use when planning for high availability or read replicas — for example, be aware that some replication topologies or multi-primary setups need explicit sequence configuration (interleaved ranges, or a distributed ID generator) to avoid two primaries generating the same ID concurrently.

- **DON'T:** Assume a sequence's current value survives a database restart or failover with byte-perfect precision in every configuration. Depending on caching settings (e.g., a sequence `CACHE` value greater than 1) some already-allocated values can be skipped after a restart — expected, harmless behavior for a surrogate key, but worth knowing before treating sequence gaps as suspicious.

- **DO:** Reset or explicitly set a sequence's starting value after a bulk data import or a table restore from backup that inserted rows with explicit IDs, so the next auto-generated ID doesn't collide with an already-imported row.
```sql
-- After bulk-loading orders with explicit IDs, resync the sequence
SELECT setval('orders_id_seq', (SELECT MAX(id) FROM orders));
```

### Encryption At Rest, In Transit & At the Column Level

- **DO:** Enable encryption at rest for the database's underlying storage (most managed database services enable this by default; verify it explicitly for self-managed instances) and require encrypted connections (TLS) between the application and the database, especially over any network path that isn't a fully trusted private network. Unencrypted database traffic on a shared or cloud network is interceptable; unencrypted storage means a stolen disk or an improperly decommissioned backup exposes everything on it in plaintext.

- **DON'T:** Treat storage-level ("at rest") encryption as sufficient protection for the most sensitive fields in a table (payment details, government identifiers, health data) without additional field/column-level encryption. Storage-level encryption protects against a stolen physical disk or backup file; it does nothing against a compromised database credential or a SQL injection vulnerability, both of which see the data in plaintext exactly as the application does — sensitive fields that need protection against those threats need to be encrypted at the application/column level, independently of storage encryption.
```pseudocode
// Column-level encryption: the database itself never sees the plaintext
encryptedSsn = encrypt(ssn, applicationManagedKey)
db.execute("INSERT INTO users (ssn_encrypted) VALUES (?)", encryptedSsn)
```

- **DO:** Manage encryption keys through a dedicated key management service (KMS) with proper access controls and rotation policies, rather than hardcoding an encryption key in application configuration or source code. A leaked application config that also contains the encryption key defeats the entire purpose of encrypting the data it protects.

- **DON'T:** Encrypt a column that needs to be searched, filtered, or joined on using standard, non-deterministic encryption, then be surprised that `WHERE encrypted_email = ?` no longer works. Understand the real tradeoff: deterministic encryption enables equality lookups but leaks whether two encrypted values are equal; non-deterministic (randomized) encryption is more secure but sacrifices searchability — for search needs, a separate blind index (a keyed hash of the plaintext, stored alongside the encrypted value) is a common way to get both.

- **DO:** Extend encryption-in-transit requirements to every hop sensitive data travels, not only the primary application-to-database connection — replication traffic between primary and replicas, connections from analytics/reporting tools, and any data pipeline reading from the database, all need the same TLS requirement applied consistently.

### Slow Query Logging & Continuous Performance Monitoring

- **DO:** Enable and actively monitor the database's slow query log (or equivalent — `pg_stat_statements` in PostgreSQL, the slow query log in MySQL, Query Store in SQL Server) in every environment that matters, with a threshold tuned to catch genuinely problematic queries without drowning the log in noise from routine queries. A slow query log is the primary tool for finding performance regressions that unit tests and casual manual testing will never catch, because they only show up under real production data volume and concurrency.

- **DON'T:** Treat slow query monitoring as a reactive tool used only after users complain. Reviewing slow query logs regularly (a weekly triage, or an automated alert on new queries crossing a latency threshold) catches regressions introduced by a recent deploy while they're still small and easy to fix, instead of after they've compounded into a production incident.

- **DO:** Track query performance trends over time, not just point-in-time snapshots, so a query that's gradually gotten slower as a table has grown (a classic symptom of a missing index that was fine at a smaller scale) is caught by the trend, not only once it crosses an absolute threshold that finally triggers an alert.

- **DON'T:** Attribute every logged slow query automatically to "the database is slow" without checking whether the actual cause is application-side (a slow serialization step, an N+1 pattern generating hundreds of individually-fast-but-collectively-slow queries, lock contention from an unrelated long-running transaction). The slow query log shows individual query duration; correlating it with what the application was doing around that time is what actually diagnoses the cause.

### Large Objects & External File Storage

- **DO:** Store large binary content (images, videos, PDFs, large file uploads) in dedicated object storage (S3, Google Cloud Storage, Azure Blob Storage) rather than directly in database columns, and store only a reference (a URL or object key) in the database. Object storage is purpose-built for large binary content — cheaper per byte, natively supports streaming/range reads, and doesn't bloat the database's backup size or working set the way large blobs stored inline do.
```sql
-- Store a reference, not the file itself
CREATE TABLE documents (
  id BIGINT PRIMARY KEY,
  file_url TEXT NOT NULL,       -- points to object storage
  file_size_bytes BIGINT,
  content_type TEXT
);
```

- **DON'T:** Store large binary objects directly in database rows (`BLOB`/`BYTEA` columns holding multi-megabyte files) as a default pattern. Every backup, replication stream, and full-table scan now has to move that binary data along with the rest of the table, which measurably slows routine database operations that have nothing to do with the file content itself.

- **DO:** Use the database's native large object type (`BYTEA`/`BLOB`) deliberately for genuinely small binary data that's tightly coupled to a row's lifecycle and benefits from transactional consistency with the rest of the row (a small thumbnail, a signature image a few kilobytes in size) — this is a legitimate, bounded exception, not a general pattern for "files."

- **DON'T:** Let an application read an entire large object into memory before streaming it to a client, when the database driver and the storage layer both support streaming. Loading a multi-gigabyte file fully into application memory before forwarding it is a common, avoidable cause of memory pressure and slow response times for large-file downloads.

### Internationalization & Localized Content Schema Design

- **DO:** Model translatable content with a dedicated structure — either a separate translations table keyed by (entity ID, locale) or a `JSONB`/document column keyed by locale — rather than adding a new column per language (`name_en`, `name_fr`, `name_de`), which doesn't scale as new languages are added and requires a schema migration for every new locale.
```sql
CREATE TABLE product_translations (
  product_id INT REFERENCES products(id),
  locale TEXT NOT NULL,
  name TEXT NOT NULL,
  description TEXT,
  PRIMARY KEY (product_id, locale)
);
```

- **DON'T:** Store locale-formatted values (a pre-formatted date string, a pre-formatted currency amount with a locale-specific symbol and separator) instead of the underlying raw value plus the locale used to format it. Storing only the formatted output makes it impossible to correctly reformat for a different locale later, and makes the value unusable for any actual date/numeric computation.

- **DO:** Store a locale-neutral canonical value (an ISO date, an amount in a fixed currency and unit, a country code from a fixed standard like ISO 3166) as the source of truth, and perform locale-specific formatting only at the display layer, at read time — never persisting the formatted, locale-specific version as the stored value.

- **DON'T:** Assume every entity needs translation for every field. Translating fields that are actually language-neutral (a SKU, a numeric measurement, an internal status code) adds unnecessary complexity — decide per field whether translation is genuinely needed rather than defaulting every text field into the translation system.

### Polymorphic Associations

- **DO:** Prefer a supertype/subtype table structure (a shared base table plus per-type tables, joined by a shared primary key) over a polymorphic foreign key (a `commentable_id` column paired with a `commentable_type` string) when a real database-enforced foreign key constraint matters. A polymorphic foreign key cannot be declared as a real `FOREIGN KEY` constraint pointing at more than one target table, so the database can never enforce that the referenced row actually exists — referential integrity for that relationship becomes entirely the application's responsibility.
```sql
-- Supertype/subtype: a real, enforceable foreign key
CREATE TABLE commentable_entities (id BIGINT PRIMARY KEY, entity_type TEXT NOT NULL);
CREATE TABLE posts (id BIGINT PRIMARY KEY REFERENCES commentable_entities(id), title TEXT);
CREATE TABLE photos (id BIGINT PRIMARY KEY REFERENCES commentable_entities(id), url TEXT);
CREATE TABLE comments (
  id BIGINT PRIMARY KEY,
  commentable_id BIGINT NOT NULL REFERENCES commentable_entities(id),  -- one real FK
  body TEXT
);
```

- **DON'T:** Reach for a polymorphic association (`{type, id}` pair with no real foreign key) as the default way to model "this thing can belong to several different kinds of parent." Beyond the lost referential integrity, it also prevents an index from being used as cleanly (queries typically need to filter on both the type and the ID together) and it isn't obvious from the schema alone which tables are valid values for the type column without checking application code.

- **DO:** Accept a polymorphic association as a pragmatic, explicitly-flagged exception when the number of possible parent types is large, changes often, or the relationship is genuinely a lightweight, non-critical one (activity feed entries referencing many different entity kinds) — but compensate with application-level validation and, ideally, a periodic integrity check that scans for orphaned references the database itself cannot prevent.

- **DON'T:** Mix a polymorphic association's type discriminator values inconsistently (`'Post'` in one place, `'post'` or `'posts'` elsewhere) across the codebase. Since the database can't validate these values against a real constraint, inconsistent casing or pluralization silently creates orphaned-looking references that are actually just mismatched strings — enforce the discriminator's valid values with an application-level enum and/or a `CHECK` constraint against a fixed list.

### Optional 1-to-1 Extension Tables

- **DO:** Split a wide table's optional, sparsely-populated, or feature-specific columns into a separate 1-to-1 extension table (joined by a shared primary key) when those columns apply to only a subset of rows or are only relevant to a specific feature area, rather than accumulating dozens of nullable columns on the core table. This keeps the core table lean for its common access pattern and makes each extension's ownership boundary explicit.
```sql
CREATE TABLE users (id BIGINT PRIMARY KEY, email TEXT NOT NULL, created_at TIMESTAMPTZ);
-- Seller-specific fields only exist for the subset of users who are sellers
CREATE TABLE seller_profiles (
  user_id BIGINT PRIMARY KEY REFERENCES users(id),
  business_name TEXT, tax_id TEXT, payout_schedule TEXT
);
```

- **DON'T:** Split a table into extension tables so finely that a common read path now requires several joins just to assemble one entity's normal, always-needed data. Extension tables are for genuinely optional or feature-specific data, not a way to fragment core, always-present attributes that belong together on the primary table.

- **DO:** Enforce the "optional" part of a 1-to-1 extension table explicitly — the extension row simply doesn't exist for entities that don't have that feature — rather than creating a row with every column `NULL` as a substitute for "this doesn't apply." A missing row is a cleaner, more queryable signal ("does a seller_profiles row exist for this user") than an all-null row that requires checking every column to determine the same thing.

- **DON'T:** Use an extension table where the relationship isn't actually 1-to-1 but could become 1-to-many over time (a user having potentially more than one of something the schema currently assumes is singular). Confirm the cardinality is genuinely fixed at 1-to-1 (or 1-to-0/1) before committing to that shape, since converting it later is a real migration, not a trivial change.

### Hierarchical Data in Relational Tables

- **DO:** Choose a tree-representation strategy deliberately based on the dominant access pattern: an adjacency list (`parent_id` self-reference) for simplicity and easy single-level or recursive-CTE traversal; a materialized path (storing the full ancestor path as a string or array on each row) for fast "get all descendants" queries without recursion; or a nested set model for fast subtree queries at the cost of expensive inserts/moves. Each trades off differently between read speed, write complexity, and query simplicity — there is no single universally-correct tree representation.
```sql
-- Adjacency list: simple, natural for recursive CTEs, but "get all
-- descendants of X" requires a recursive query rather than a single scan
CREATE TABLE categories (id INT PRIMARY KEY, parent_id INT REFERENCES categories(id), name TEXT);

-- Materialized path: descendants found with a simple prefix match,
-- no recursion needed, at the cost of updating paths on any move
CREATE TABLE categories (id INT PRIMARY KEY, path TEXT, name TEXT);
-- path = '1/4/17/' means: descendant of 1, then 4, then this row is 17
SELECT * FROM categories WHERE path LIKE '1/4/%';
```

- **DON'T:** Default to the adjacency list model and then reach for application-side recursive queries (one query per level, in a loop) to fetch a subtree, reintroducing the N+1 problem for tree traversal specifically. Use a recursive CTE (a single round trip) or switch to a materialized-path/nested-set model if subtree queries are frequent and the recursive-CTE performance isn't sufficient at the tree's actual depth and size.

- **DO:** Bound the maximum tree depth explicitly, either with an application-level check or a `CHECK` constraint-adjacent mechanism, when using an adjacency list with recursive CTEs, since an unexpectedly deep or (due to a bug) cyclic tree can make a recursive query run far longer than expected or, in the cyclic case, loop until a database-enforced recursion limit kills it.

- **DON'T:** Use the nested set model for a tree that's frequently modified (nodes added, removed, or moved often). Nested sets make subtree queries very fast but require renumbering a potentially large portion of the tree's `left`/`right` values on every insert or move — a poor tradeoff for a frequently-changing tree, where an adjacency list or materialized path with more modest write costs is usually the better choice.

### Array & Composite Column Types

- **DO:** Use a native array column type (PostgreSQL's array types) for a genuinely small, bounded, order-relevant, or non-relational-needing set of scalar values tightly owned by a single row — a list of tags directly on a post, for instance — when that list never needs to be queried, joined, or constrained independently of its parent row.
```sql
CREATE TABLE posts (id INT PRIMARY KEY, title TEXT, tags TEXT[]);
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);  -- supports tag containment queries
SELECT * FROM posts WHERE tags @> ARRAY['postgresql'];
```

- **DON'T:** Use an array column for a relationship that actually needs referential integrity, independent querying, or attributes of its own (a tag that has a color, a description, or needs its own uniqueness enforced across posts). An array column can't hold a foreign key constraint on its individual elements, can't easily support "how many posts use this tag" aggregation without unnesting, and can't attach metadata to the relationship itself — a proper junction table serves all of these needs directly.

- **DO:** Recognize that array (and other composite/nested) columns are, in a normalization sense, a deliberate exception to 1NF's "atomic values" rule, and use them consciously for that reason — as a denormalization tool for a specific, bounded, tightly-coupled case — not as a reflexive substitute for a junction table whenever "a list of things" comes up.

- **DON'T:** Assume array column support is portable across database engines. Native array types are a PostgreSQL-specific feature without a direct equivalent in MySQL or SQL Server (which instead push toward JSON columns or junction tables for the same need) — code relying on array-specific operators and functions won't translate directly if the target database changes.

### Table Bloat & Storage Reclamation

- **DO:** Monitor table and index bloat (the gap between a table's logical data size and its actual on-disk size, caused by MVCC's old row versions and deleted rows not yet reclaimed) as an operational metric, especially on tables with heavy update/delete churn, and run the database's reclamation tooling (`VACUUM`/autovacuum tuning in PostgreSQL, `OPTIMIZE TABLE` in MySQL) proactively rather than only after bloat has already degraded performance.

- **DON'T:** Assume routine `DELETE` or `UPDATE` operations immediately shrink a table's on-disk footprint. Under MVCC, deleted and old-version rows occupy space until a cleanup process reclaims them — a table that's had millions of rows deleted can still occupy roughly the same disk space (and be just as slow to scan) as before the delete, until vacuum/cleanup actually runs and completes.

- **DO:** Schedule and monitor autovacuum (or the equivalent automatic maintenance process) tuning specifically for high-churn tables, since the default settings are often conservative and can fall behind on a table with a very high rate of updates/deletes — a table where autovacuum can't keep up accumulates bloat continuously, degrading both query performance and storage efficiency over time.

- **DON'T:** Run a blocking, full-table-rewrite reclamation operation (`VACUUM FULL` in PostgreSQL, certain `OPTIMIZE TABLE` operations in MySQL) against a large, actively-written production table without planning for the exclusive lock it holds for the operation's full duration. These operations genuinely reclaim space, but at the cost of blocking reads and writes to the table while they run — schedule them for a maintenance window, or use an online/concurrent alternative where the database offers one.

### Wide Tables vs Many Narrow Tables for Write-Heavy Workloads

- **DO:** Consider row-locking and update granularity when deciding whether to keep frequently-and-independently-updated attributes on one wide table or split them into separate tables joined 1-to-1. Some databases (PostgreSQL, for instance, via its Heap-Only Tuple optimization) can update a row more cheaply when the update doesn't touch an indexed column, but in general, two application flows that each frequently update a *different* subset of the same row's columns will contend with each other over the same row-level lock even though they're logically touching unrelated data.

- **DON'T:** Assume splitting a table always helps write contention without considering the added join cost on the read side. Splitting a wide, frequently-read table into several narrow tables to reduce write contention on a rarely-touched high-frequency column trades write-side contention for read-side join overhead — measure whether the actual contention is real and significant before restructuring around it.

- **DO:** Isolate a specific high-write-frequency column (a view counter, a last-seen timestamp) into its own narrow table when it's updated far more often than the rest of the row's data and its updates would otherwise contend with less-frequent, more-important updates to the same row. This keeps the hot, frequently-locked column's contention isolated from the rest of the entity's data.

- **DON'T:** Over-apply this pattern preemptively to every column that might conceivably be updated frequently "someday." Splitting off a column into its own table adds a permanent join cost to every read that needs both pieces of data — reserve it for a column with an actually measured, high, and specifically isolated write pattern.

### Composite & Multi-Column Foreign Keys

- **DO:** Use a composite foreign key (referencing a composite unique or primary key made of more than one column) when the natural identity of the referenced relationship genuinely requires more than one column — a table keyed by `(tenant_id, order_id)` being referenced by a child table that must match both columns together, not just `order_id` alone, which matters in a multi-tenant schema where `order_id` might not be globally unique on its own.
```sql
CREATE TABLE orders (tenant_id INT, order_id INT, PRIMARY KEY (tenant_id, order_id));
CREATE TABLE order_items (
  tenant_id INT, order_id INT, item_id INT,
  FOREIGN KEY (tenant_id, order_id) REFERENCES orders (tenant_id, order_id),
  PRIMARY KEY (tenant_id, order_id, item_id)
);
```

- **DON'T:** Reference only part of a composite key from a child table when the full composite key is what actually establishes the correct relationship. Referencing just `order_id` when the parent's real identity is `(tenant_id, order_id)` can match a row against the wrong tenant's order if `order_id` values aren't globally unique across tenants — a subtle multi-tenant data integrity gap that a single-column foreign key can't catch.

- **DO:** Carry every column of a composite foreign key consistently through every table that needs to join back to it, including in indexes, so the composite relationship's benefits (correct matching, index usability) aren't lost partway through a longer chain of related tables.

- **DON'T:** Use a composite foreign key as a substitute for a proper surrogate key when the "composite" is really just a workaround for not having assigned a single well-chosen primary key to the parent table in the first place. Composite foreign keys are for relationships that are genuinely composite by nature (a multi-tenant natural key, a versioned entity keyed by ID plus version) — not a routine default.

### Money & Multi-Currency Accounts at the Schema Level

- **DO:** Store an amount and its currency together as an inseparable pair everywhere money is represented in the schema — never a bare numeric amount with currency assumed or tracked elsewhere. A `balance NUMERIC(14,2)` column with no adjacent currency column is a schema-level bug waiting to happen the moment the system needs to support more than one currency, or even just needs an unambiguous audit trail of what currency a historical value was actually recorded in.
```sql
CREATE TABLE account_balances (
  account_id BIGINT PRIMARY KEY,
  amount NUMERIC(14,2) NOT NULL,
  currency CHAR(3) NOT NULL   -- always paired with amount, never implicit
);
```

- **DON'T:** Sum or compare monetary amounts across rows without first confirming they share the same currency. `SUM(amount)` over a table containing mixed currencies produces a number that's numerically valid SQL but financially meaningless — enforce currency consistency at the query level (`GROUP BY currency`) or convert explicitly to a common reporting currency using a recorded, appropriate exchange rate before combining.

- **DO:** Model a multi-currency account (one that can hold balances in more than one currency simultaneously) as multiple rows — one per currency — rather than a single row with an assumed "primary" currency and ad hoc conversion elsewhere, so each currency's balance is tracked with its own precision and audit trail.

- **DON'T:** Use a floating-point or imprecise numeric type for currency conversion calculations, compounding the general money-as-float problem discussed earlier with the additional complexity of exchange-rate math — use fixed-point decimal arithmetic throughout, including for intermediate conversion calculations, not just for the final stored value.

### Schema Design for Event Sourcing

- **DO:** Model an event-sourced entity's state as the fold/replay of an ordered, immutable sequence of events, with the events table (not a mutable "current state" table) as the actual source of truth, when the business genuinely needs a complete, auditable history of every state transition and the ability to reconstruct state as of any point in time — not merely "we might want an audit log someday."
```sql
CREATE TABLE order_events (
  id BIGINT PRIMARY KEY,
  order_id BIGINT NOT NULL,
  event_type TEXT NOT NULL,      -- 'OrderCreated', 'ItemAdded', 'OrderShipped', ...
  event_data JSONB NOT NULL,
  sequence_number BIGINT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (order_id, sequence_number)
);
-- Current state is derived by folding events in sequence order, not stored directly
```

- **DON'T:** Adopt event sourcing as a default architecture for entities with straightforward CRUD needs and no real requirement for historical replay or audit. Event sourcing trades simple, direct state queries for the ability to reconstruct history — every "what is this entity's current state" read now requires replaying (or reading from a maintained projection of) its event history, a real ongoing cost not justified for data that has no actual need for that capability.

- **DO:** Maintain a materialized "current state" projection/read model alongside the event log for any event-sourced entity that needs to be queried efficiently by its current attributes, rebuilding or incrementally updating the projection as new events arrive, rather than replaying the full event history on every read.

- **DON'T:** Allow events in an event-sourced system to be edited or deleted once written, even to "fix" a mistake. The entire correctness model of event sourcing depends on the event log being append-only and immutable — a correction to a past mistake should itself be a new, forward-looking event (a compensating event), never a retroactive edit to history.

## Query Design & Performance

### Avoiding SELECT *

- **DO:** Name the exact columns a query needs instead of `SELECT *`, both in application code and in views. Explicit column lists mean a later `ALTER TABLE ADD COLUMN` never silently changes the shape of existing result sets, and the database can sometimes satisfy the query from an index alone instead of touching every column's storage.
```sql
-- BAD: pulls every column, including ones this code never uses
SELECT * FROM orders WHERE customer_id = ?;

-- GOOD: only what the caller actually needs
SELECT id, status, total, created_at FROM orders WHERE customer_id = ?;
```

- **DON'T:** Use `SELECT *` in application queries that map results onto typed objects/DTOs. Adding a new column to the table then silently appears in every `SELECT *` result, which can break serialization, leak a column that was never meant to be exposed (like a password hash or an internal flag), or bloat payloads with data nothing downstream reads.

- **DO:** Use `SELECT *` sparingly and consciously in ad hoc debugging/exploration queries run directly against a database console, where the goal is genuinely "show me everything about this row." That is a different context from application code that runs on every request.

- **DON'T:** Assume `SELECT *` and an explicit column list perform identically "because the database reads the whole row anyway" on a row-store engine. Even on a row store, fetching unused large columns (blobs, long text, wide JSON) wastes I/O and network bandwidth, and on a column-store/analytics engine the difference is often an order of magnitude because the engine can skip entire unread columns.

- **DO:** Explicitly enumerate columns in `INSERT` statements (`INSERT INTO orders (customer_id, status, total) VALUES (...)`) rather than relying on positional `INSERT INTO orders VALUES (...)`. An implicit-column insert breaks the moment a column is added, removed, or reordered by a migration, and it is impossible to tell which value maps to which column just by reading the statement.

### Avoiding N+1 Queries

- **DO:** Fetch related/child records with a single batched query (a `JOIN`, or a follow-up `WHERE id IN (...)` query, or the ORM's eager-loading feature) instead of issuing one query per parent row in a loop. An N+1 pattern turns "one page load" into hundreds or thousands of round trips, and its cost scales linearly with result set size — invisible in development with 5 test rows, catastrophic in production with 5,000.
```pseudocode
// BAD: N+1 — one query per order, inside a loop
orders = db.query("SELECT * FROM orders WHERE customer_id = ?", customerId)
for order in orders:
    items = db.query("SELECT * FROM order_items WHERE order_id = ?", order.id)  // N queries

// GOOD: two queries total, regardless of how many orders there are
orders = db.query("SELECT * FROM orders WHERE customer_id = ?", customerId)
orderIds = orders.map(o => o.id)
items = db.query("SELECT * FROM order_items WHERE order_id IN (?)", orderIds)
itemsByOrder = groupBy(items, "order_id")
```

- **DON'T:** Trust an ORM's lazy-loading default without checking what it actually does under a loop. Accessing `order.items` inside a `for order in orders` loop is a classic N+1 trigger in nearly every major ORM (ActiveRecord, Django ORM, Hibernate, Sequelize, Entity Framework) unless the association was explicitly eager-loaded before the loop began.
```python
# BAD (Django): order.items triggers one query per order inside the loop
for order in Order.objects.filter(customer_id=customer_id):
    for item in order.items.all():   # N additional queries
        ...

# GOOD: prefetch_related batches the child query up front
for order in Order.objects.filter(customer_id=customer_id).prefetch_related("items"):
    for item in order.items.all():   # served from the prefetched cache
        ...
```

- **DO:** Reach for the ORM's eager-loading primitives deliberately (`.includes`/`.eager_load` in Rails, `select_related`/`prefetch_related` in Django, `JOIN FETCH` in JPA/Hibernate, `.with()` in Laravel) whenever code accesses an association inside a loop, and treat "does this touch an association in a loop" as a standing question during code review.

- **DON'T:** Fix an N+1 by eager-loading everything on every query "to be safe." Eager-loading associations that a particular code path never actually reads wastes bandwidth and memory just as surely as the N+1 wasted round trips — load exactly what the specific code path needs.

- **DO:** Use request-scoped batching/dataloader patterns (e.g., the DataLoader pattern popularized by GraphQL) when the access pattern is dynamic and per-request rather than a single static loop, so that many logically-separate "give me this related record" calls within one request still collapse into one batched query.

- **DON'T:** Assume N+1 problems only happen in read paths. The same pattern shows up in write loops — issuing one `UPDATE`/`INSERT` per item inside a loop instead of a single batched statement — with the same linear-scaling cost, plus the added overhead of a round trip and (often) a transaction per statement.

### Query Plans & EXPLAIN

- **DO:** Run `EXPLAIN` (or `EXPLAIN ANALYZE` for actual runtime numbers, not just estimates) on any query that is slow, runs frequently, or touches a large table, before guessing at a fix. The plan tells you directly whether the database is doing a sequential scan where an index scan was expected, doing a poor join order, or misestimating row counts — guessing wastes time trying fixes that don't address the actual bottleneck.
```sql
EXPLAIN ANALYZE
SELECT o.id, o.total FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE c.region = 'EU' AND o.created_at > now() - interval '30 days';
```

- **DON'T:** Add an index in response to a slow query without first confirming, via `EXPLAIN`, that a missing index (rather than a bad join order, an unnecessary sort, a function wrapped around an indexed column, or an outdated statistics estimate) is actually the cause. An index added to fix the wrong problem adds write overhead while leaving the real bottleneck untouched.

- **DO:** Watch specifically for a sequential scan (`Seq Scan` in PostgreSQL, `type: ALL` in MySQL's `EXPLAIN`) on a large table where an indexed lookup was expected — this is the single most common signal that either an index is missing, or a function/cast on the filtered column is preventing the planner from using an index that does exist.
```sql
-- A function applied to the indexed column defeats the index:
SELECT * FROM users WHERE LOWER(email) = 'a@example.com';  -- can't use a plain email index

-- Fix: index the expression itself, or normalize on write
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
```

- **DON'T:** Trust query plan estimates blindly on a table whose statistics are stale — a planner working from outdated row-count/distribution statistics can choose a badly wrong plan (e.g., a nested loop join where a hash join would be far cheaper) even though the schema and indexes are fine. Run `ANALYZE` (PostgreSQL) or update statistics (`UPDATE STATISTICS` in SQL Server, `ANALYZE TABLE` in MySQL) after large data changes, and check whether autovacuum/auto-analyze is actually keeping up.

- **DO:** Compare the planner's estimated row count against the actual row count in `EXPLAIN ANALYZE` output — a large gap between estimated and actual rows is a strong signal of stale statistics or a correlation the planner can't see (e.g., two filtered columns that are correlated in practice, which the planner assumes are independent).

- **DON'T:** Treat a query as "optimized" just because it returns quickly on a development database with a few hundred rows. A plan that looks fine at small scale can flip from an index scan to a full scan, or from a good join order to a bad one, once the table crosses a size threshold in production — always validate against production-representative data volumes, or at minimum sanity-check the plan itself.

- **DO:** Look at the join order and join algorithm the planner chose (nested loop vs. hash join vs. merge join) for multi-table queries, since a nested loop over a large outer table is often the actual source of slowness even when every individual table has the "right" indexes.

### Offset Pagination

- **DO:** Use `LIMIT`/`OFFSET` (or the dialect equivalent) for simple, low-page-count pagination such as an admin table the user pages through a handful of times, where the simplicity of implementation matters more than deep-page performance. It is easy to implement, easy to reason about ("give me page 3"), and adequate when nobody realistically pages past the first few dozen pages.
```sql
SELECT id, name FROM products ORDER BY id LIMIT 20 OFFSET 40;  -- page 3, 20/page
```

- **DON'T:** Use `OFFSET`-based pagination for large, deep, or frequently-accessed datasets (API endpoints, infinite scroll, data exports). The database still has to scan and discard every row before the offset, so `OFFSET 1000000` does real work proportional to a million rows even though it returns only 20 — the deeper the page, the slower the query, with no upper bound.

- **DON'T:** Paginate a result set with `OFFSET` while the underlying data is being inserted or deleted between page requests. Rows can shift between pages — a row can be skipped entirely (if a row before the offset was deleted) or duplicated across two pages (if a row was inserted) — because offset pagination has no stable anchor, only a row count to skip.

- **DO:** Combine `OFFSET` pagination with a stable, deterministic `ORDER BY` that includes a tiebreaker column (typically the primary key) whenever ties are possible on the primary sort column. `ORDER BY created_at LIMIT 20 OFFSET 40` with many rows sharing the same `created_at` value can return rows in a different, inconsistent order across requests, causing rows to be skipped or repeated across pages.
```sql
-- BAD: ties on created_at make page boundaries non-deterministic
ORDER BY created_at LIMIT 20 OFFSET 40;

-- GOOD: a unique tiebreaker guarantees a stable, repeatable order
ORDER BY created_at, id LIMIT 20 OFFSET 40;
```

### Keyset / Cursor Pagination

- **DO:** Use keyset (cursor-based) pagination — filtering with `WHERE (sort_col, id) > (last_seen_sort_col, last_seen_id)` instead of skipping rows — for large or infinite-scroll datasets, and for any API where clients page deeply or the underlying data changes between requests. Keyset pagination's cost is constant per page regardless of how deep the client has paged, because it uses an index seek to the cursor position instead of scanning and discarding everything before it.
```sql
-- Cursor pagination: constant-time regardless of page depth,
-- and stable even if rows are inserted/deleted between requests
SELECT id, name, created_at FROM products
WHERE (created_at, id) > (:last_created_at, :last_id)
ORDER BY created_at, id
LIMIT 20;
```

- **DON'T:** Build keyset pagination on a sort column that isn't unique (or paired with a unique tiebreaker) — without a tiebreaker, rows sharing the same value on the sort column can be skipped or repeated at page boundaries, the same failure mode keyset pagination is meant to avoid.

- **DO:** Encode the pagination cursor (the last-seen sort key values) opaquely — base64 or otherwise encoded, not raw exposed values the client can tamper with — when the cursor is exposed through a public API. This keeps the pagination implementation free to change later without breaking API compatibility, and avoids letting clients construct arbitrary cursor values that bypass intended query bounds.

- **DON'T:** Assume keyset pagination lets a client jump to an arbitrary page number ("show me page 47") the way offset pagination does. Keyset pagination only supports "give me the next/previous page relative to where I am," which is the right tradeoff for infinite scroll and APIs but a real limitation for UIs that need numbered page jumps — pick the pagination style that matches what the UI actually needs, or offer both with different cost profiles.

- **DO:** Index the exact columns used in the keyset predicate as a composite index matching the `ORDER BY`, so the cursor filter is served by an index seek rather than a scan. Cursor pagination's performance advantage disappears entirely if the underlying comparison isn't index-backed.

### Avoiding Unbounded Queries

- **DO:** Put an explicit upper bound (`LIMIT`, a page size cap, a maximum date range) on every query that could return an unbounded number of rows, especially any query reachable from user input or an external API call. An unbounded query is one bad input, one large customer, or one data-growth milestone away from returning millions of rows and taking down the process that tries to hold them all in memory.
```sql
-- BAD: no bound at all — grows without limit as the table grows
SELECT * FROM audit_log WHERE user_id = ?;

-- GOOD: explicit, sane upper bound regardless of how much data exists
SELECT * FROM audit_log WHERE user_id = ? ORDER BY created_at DESC LIMIT 500;
```

- **DON'T:** Trust that "this table will always stay small" as a substitute for an explicit limit. Tables described as small at design time routinely grow past that assumption in production, and by the time an unbounded query becomes a problem it is usually already causing timeouts or memory pressure for real users, discovered in an incident rather than a review.

- **DO:** Enforce a maximum page size server-side for any paginated endpoint that accepts a client-supplied page size, in addition to a default. A client (or attacker) requesting `?page_size=1000000` should be capped, not honored, or a single request can be used to force an enormous, resource-exhausting query.

- **DON'T:** Load an entire large table into application memory to "filter/process it in the application layer" when the same filtering can be pushed down to the database in the query itself. Pulling millions of rows across the network just to discard 99% of them in a loop wastes I/O, memory, and time that a `WHERE` clause would have avoided entirely.

- **DO:** Stream or cursor through large result sets (server-side cursors, chunked/keyset iteration) rather than materializing an entire large query result in memory at once, for exports, batch processing, or any operation over a table too big to comfortably fit in RAM.
```pseudocode
// Stream in chunks instead of loading the whole table into memory
lastId = 0
loop:
    batch = db.query("SELECT * FROM events WHERE id > ? ORDER BY id LIMIT 1000", lastId)
    if batch is empty: break
    process(batch)
    lastId = batch.last.id
```

- **DON'T:** Write a query with an unbounded `IN (...)` list built from user-controllable input (e.g., an unfiltered list of IDs from a client request) without a size cap. Extremely large `IN` lists can blow past a database's parameter limit, produce a terrible query plan, or simply be a vector for resource-exhaustion abuse.

### Batch Operations vs Row-by-Row

- **DO:** Use a single batched `INSERT`/`UPDATE`/`DELETE` statement (multi-row `VALUES`, `WHERE id IN (...)`, or a bulk-load utility) instead of looping and issuing one statement per row from application code. Each round trip to the database carries fixed overhead (network latency, parsing, planning); batching amortizes that overhead across many rows instead of paying it once per row.
```sql
-- BAD: one round trip per row
INSERT INTO events (user_id, type) VALUES (1, 'login');
INSERT INTO events (user_id, type) VALUES (2, 'login');
-- ... hundreds more, one round trip each

-- GOOD: one round trip for the whole batch
INSERT INTO events (user_id, type) VALUES (1, 'login'), (2, 'login'), (3, 'login');
```

- **DON'T:** Batch an unbounded number of rows into a single statement without a chunk size limit. A single `INSERT` or `UPDATE` touching millions of rows can hold locks for a long time, generate a huge amount of write-ahead log/undo log in one transaction, and risk hitting statement size or parameter count limits — chunk large batch operations into reasonably sized transactions (e.g., a few thousand rows at a time) with a short pause or commit between chunks.

- **DO:** Use the database's native bulk-loading mechanism (`COPY` in PostgreSQL, `LOAD DATA INFILE` in MySQL, `bcp`/`BULK INSERT` in SQL Server) for genuinely large one-time or scheduled bulk loads, rather than any row-by-row `INSERT` approach, even a batched one. Native bulk loaders bypass much of the per-row overhead of normal SQL execution and are typically an order of magnitude faster for large loads.

- **DON'T:** Wrap each row of a batch operation in its own transaction when the whole batch is logically one unit of work. Committing after every single row multiplies the fsync/durability cost by the row count and leaves the batch in a confusing partially-applied state if it fails partway through; wrap a bounded chunk in one transaction so it commits (or rolls back) as a unit.

- **DO:** Use `UPSERT` (`INSERT ... ON CONFLICT DO UPDATE` in PostgreSQL/SQLite, `ON DUPLICATE KEY UPDATE` in MySQL, `MERGE` in SQL Server/Oracle) for "insert or update" logic in a single atomic statement instead of a select-then-branch pattern from application code, which is both slower (two round trips) and race-prone under concurrency.
```sql
INSERT INTO inventory (sku, quantity) VALUES ('WIDGET-1', 10)
ON CONFLICT (sku) DO UPDATE SET quantity = inventory.quantity + EXCLUDED.quantity;
```

- **DON'T:** Disable or ignore constraint/index maintenance as a shortcut to speed up a large batch load unless you have a specific, measured reason and a plan to re-enable/rebuild afterward. Dropping indexes before a huge bulk load and rebuilding them after is a legitimate, well-known technique for very large one-time loads — but doing this casually on a live table other queries depend on removes correctness guarantees for everyone else during the window.

### Joins, Subqueries & CTEs

- **DO:** Prefer a `JOIN` over a correlated subquery when checking for existence or pulling related columns, and let the query planner choose the join strategy, rather than hand-rolling row-by-row logic in application code. A correlated subquery re-executed conceptually once per outer row can be far slower than a semantically equivalent join or semi-join the planner can optimize as a set operation.
```sql
-- Often slower: correlated subquery evaluated per outer row
SELECT * FROM customers c
WHERE (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) > 0;

-- Usually faster and clearer: semi-join via EXISTS
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

- **DON'T:** Use `IN (SELECT ...)` with a subquery that can return `NULL` values, especially combined with `NOT IN`. `NOT IN` against a subquery result containing even one `NULL` causes the entire condition to evaluate to unknown/false for every row — a well-known correctness trap, not just a performance one; use `NOT EXISTS` instead, which handles `NULL` correctly.
```sql
-- BAD: silently returns zero rows if orders.customer_id ever contains NULL
SELECT * FROM customers WHERE id NOT IN (SELECT customer_id FROM orders);

-- GOOD: correct regardless of NULLs in the subquery
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

- **DO:** Use Common Table Expressions (CTEs, `WITH ... AS (...)`) to break a complex query into named, readable steps, especially for multi-stage aggregation or filtering logic that would otherwise be an unreadable wall of nested subqueries.
```sql
WITH recent_orders AS (
  SELECT customer_id, SUM(total) AS spend
  FROM orders WHERE created_at > now() - interval '90 days'
  GROUP BY customer_id
)
SELECT c.name, ro.spend FROM customers c
JOIN recent_orders ro ON ro.customer_id = c.id
WHERE ro.spend > 1000;
```

- **DON'T:** Assume a CTE is always materialized separately (and therefore always an "optimization fence") or always inlined — this differs by database and version (e.g., PostgreSQL inlines non-recursive CTEs by default since version 12, but earlier versions and other databases may materialize them). Check the actual planner behavior for the specific database and version in use before relying on a CTE either as a readability tool or as a deliberate optimization barrier.

- **DO:** Use recursive CTEs for genuinely hierarchical/graph-shaped data (org charts, category trees, bill-of-materials explosions) instead of application-side recursive querying that issues one query per level. A recursive CTE resolves the entire hierarchy in one round trip; application-side recursion re-triggers the N+1 problem one level at a time.
```sql
WITH RECURSIVE subordinates AS (
  SELECT id, manager_id, name FROM employees WHERE id = :root_id
  UNION ALL
  SELECT e.id, e.manager_id, e.name
  FROM employees e JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

- **DON'T:** Join tables you don't need columns or filters from just because a foreign key relationship exists. An unnecessary join can silently duplicate rows (a one-to-many join fanning out a one-to-one query) and adds planner and execution cost for no benefit — join only what the specific query actually requires.

- **DO:** Use window functions (`ROW_NUMBER()`, `RANK()`, `SUM(...) OVER (...)`, `LAG`/`LEAD`) for "top N per group," running totals, and row-comparison-to-neighbor queries instead of self-joins or correlated subqueries that are harder to read and often slower.
```sql
-- Top 3 orders per customer, in one pass, no self-join needed
SELECT * FROM (
  SELECT o.*, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total DESC) AS rn
  FROM orders o
) ranked
WHERE rn <= 3;
```

### Aggregate & Window Function Correctness

- **DO:** Filter rows before aggregation with `WHERE`, and filter on aggregate results after grouping with `HAVING` — never the reverse. Putting a per-row condition in `HAVING` forces the database to aggregate every row first and only then discard some groups, doing far more work than filtering rows out before grouping ever began.
```sql
-- Correct: filter rows first (WHERE), then filter groups (HAVING)
SELECT customer_id, COUNT(*) AS order_count
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING COUNT(*) > 5;
```

- **DON'T:** Forget that `COUNT(column)` and `COUNT(*)` behave differently — `COUNT(column)` skips rows where that column is `NULL`, while `COUNT(*)` counts every row regardless. Using the wrong one silently under-counts rows with legitimate `NULL` values in the counted column, a subtle bug that only shows up when someone notices the numbers don't add up.

- **DO:** Be explicit about how `NULL` values participate in aggregates and comparisons — `SUM`, `AVG`, `COUNT(column)`, `MAX`, `MIN` all ignore `NULL`s, and any comparison against `NULL` (`= NULL`, `<> NULL`) evaluates to unknown rather than true or false, which silently excludes those rows from `WHERE` filters entirely.
```sql
-- BAD: rows where discount_code IS NULL are silently excluded, not matched
SELECT * FROM orders WHERE discount_code = NULL;   -- always zero rows

-- GOOD
SELECT * FROM orders WHERE discount_code IS NULL;
```

- **DON'T:** Group by a column that isn't actually functionally determined by the grouping key while selecting it in a non-strict SQL mode — the database is then free to return an arbitrary value from within the group for that column, producing results that look plausible but are not deterministic. Either add it to `GROUP BY`, wrap it in an aggregate function, or run in strict SQL mode (e.g., MySQL's `ONLY_FULL_GROUP_BY`) so this is caught as an error instead of silently returning an unpredictable value.

- **DO:** Use `FILTER (WHERE ...)` (PostgreSQL/standard SQL) or `CASE WHEN` inside an aggregate function to compute multiple conditional aggregates in a single pass over the data, instead of running several separate queries or self-joining the same table multiple times.
```sql
SELECT
  COUNT(*) FILTER (WHERE status = 'completed') AS completed,
  COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled
FROM orders WHERE created_at > now() - interval '30 days';
```

### Connection Pooling & Timeouts

- **DO:** Use a connection pool sized to the database's actual capacity (not the application's thread/request concurrency) and share it across requests, rather than opening a new database connection per request. Establishing a connection is expensive (TCP handshake, authentication, session setup); a pool amortizes that cost and caps how many concurrent connections the database has to manage.

- **DON'T:** Size a connection pool arbitrarily large "to avoid ever waiting for a connection." A database has a hard limit on concurrent connections and finite memory per connection — an oversized pool from even one application instance can starve other services, and oversized pools across multiple instances can exhaust the database's connection limit entirely, causing every service to fail simultaneously.

- **DO:** Set an explicit statement timeout and a connection-acquisition timeout, both at the database client/pool level, rather than letting a runaway query or a starved pool hang a request indefinitely. A query with no timeout can hold a connection (and its locks) forever if something goes wrong, cascading into a pool exhaustion incident for every other request.
```sql
-- PostgreSQL: cap how long any single statement may run in this session
SET statement_timeout = '5s';
```

- **DON'T:** Open a new connection for every unit of work inside a loop instead of reusing one connection (or the pool) across the whole operation. Repeatedly connecting and disconnecting inside a batch job multiplies connection-establishment overhead unnecessarily and can itself exhaust the pool under concurrent load.

- **DO:** Release/return connections to the pool promptly — close cursors, commit or roll back transactions, and return the connection — even on the error path, using the language's structured resource-management construct (`try/finally`, `with`, `using`, context managers) rather than manual cleanup that can be skipped on an exception.

### ORM-Specific Pitfalls

- **DO:** Inspect the actual SQL an ORM generates for a given call, especially for anything involving associations, aggregations, or dynamic filters, rather than assuming the ORM's abstraction always produces reasonable SQL. ORMs regularly generate suboptimal queries — unnecessary joins, missing indexes on generated foreign keys, or N+1 patterns — that are invisible until you look at the generated SQL or the query log.

- **DON'T:** Let an ORM's convenience methods (`.all()`, `.find_each` without a batch size, unscoped `.get()`) silently load an entire table into memory. Many ORMs default to eagerly materializing the full result set of a query unless told otherwise; use the ORM's batching/streaming primitives for large tables.

- **DO:** Drop down to raw SQL (or the ORM's raw-query escape hatch, always with parameterized values) for genuinely complex queries — multi-table aggregations, window functions, recursive CTEs — where forcing the query through the ORM's query-builder DSL produces unreadable code or a worse query plan than hand-written SQL would.

- **DON'T:** Let an ORM's automatic schema migrations run directly against a production database without review. Auto-generated migrations from model diffs can guess wrong about column types, nullability, or — most dangerously — can decide to drop and recreate a column (losing data) when a rename would have preserved it; always review the generated migration before it runs anywhere with real data.

- **DO:** Understand and control the ORM's default transaction boundaries (does each save wrap in its own transaction? does a request get one shared transaction?) rather than assuming a default that may not match the actual consistency needs of a multi-step operation.

### Stored Procedures & Database-Side Logic

- **DO:** Use stored procedures, triggers, and database functions deliberately and sparingly for narrow, performance-critical, or data-integrity-critical operations — a trigger that maintains a denormalized aggregate, a function that enforces an invariant no `CHECK` constraint can express — and document why the logic lives in the database rather than the application. A justified, documented exception is very different from a schema that has quietly accumulated business logic no application developer knows to look for.

- **DON'T:** Move general business logic (pricing rules, workflow state transitions, validation that changes with product requirements) into stored procedures or triggers as a default architectural choice. Logic hidden in the database is invisible to the application's normal code review, testing, and version-control history in the same way application code is, and is a common source of "why did this happen" debugging sessions when nobody remembers a trigger exists.

- **DO:** Version-control database-side logic (stored procedures, functions, triggers) in the same repository and migration pipeline as the rest of the schema, with the same review process, rather than letting a DBA or ad hoc console session modify it directly in production. Database logic that isn't in version control is exactly as risky as application code that isn't.

- **DON'T:** Rely on a trigger to silently fix up or override data the application just wrote, without the application knowing the trigger exists. A trigger that quietly overwrites a value the application explicitly set produces confusing bugs where the application's own logic appears not to work, and the actual cause is invisible without knowing to check for triggers.

- **DO:** Consider stored procedures for genuinely performance-critical batch operations where minimizing round trips between the application and database matters enough to justify the reduced portability and testability — a nightly reconciliation job doing heavy set-based computation entirely within the database, for instance.

### Query Readability & Maintainability

- **DO:** Format complex SQL queries with consistent, readable indentation and capitalization (keywords capitalized or not, consistently; one clause per line for anything beyond a trivial query) so a reviewer can follow the query's logic without mentally reformatting it first. A query that's hard to read is a query that's hard to review correctly, and bugs slip through unreadable code more easily than readable code.
```sql
-- Readable: each clause on its own line, consistent style
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE c.status = 'active'
GROUP BY c.name
ORDER BY order_count DESC;
```

- **DON'T:** Write deeply nested subqueries or long chains of implicit joins when a CTE, a named intermediate view, or simply splitting the logic into named steps would make the same query's intent legible. A query nested five subqueries deep is nearly impossible to verify correct by reading; break it into named, individually-checkable pieces.

- **DO:** Add a short comment explaining *why* for any query with a non-obvious business rule embedded in it (a specific date cutoff, an unusual join condition, a filter that looks redundant but isn't) — the same discipline applied to comments in application code. A future reader (human or AI assistant) modifying the query without knowing why a condition exists is likely to "simplify" it into a bug.

- **DON'T:** Use `SELECT DISTINCT` as a reflexive fix for unexpected duplicate rows without understanding why the duplicates appeared in the first place. `DISTINCT` masking a fan-out from an unintended one-to-many join hides the actual bug (and adds a real sorting/deduplication cost) instead of fixing the join that produced the duplication.

- **DO:** Give computed columns and aggregate results meaningful aliases (`AS total_revenue`, not the default `sum` or an unlabeled column) so query results are self-describing in logs, generated reports, and downstream code that consumes them by column name.

### Avoiding Implicit Cartesian Products

- **DO:** Specify an explicit `JOIN` condition for every table referenced in a query, and double-check multi-table queries for a missing condition that would silently produce a cartesian product (every row of one table paired with every row of another). A three-table query missing just one join condition doesn't fail — it silently returns a syntactically valid but wildly wrong, massively inflated result set.
```sql
-- BAD: missing join condition between order_items and customers —
-- produces a row for every (order_item, customer) pair, a cartesian product
SELECT * FROM orders o, order_items oi, customers c
WHERE o.id = oi.order_id;   -- customers has no join condition at all

-- GOOD: every table has an explicit, correct join condition
SELECT * FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN customers c ON c.id = o.customer_id;
```

- **DON'T:** Use old-style comma-separated `FROM` clauses (`FROM a, b, c WHERE ...`) for multi-table queries, where a missing `WHERE` condition silently becomes a cartesian product rather than a syntax error. Explicit `JOIN ... ON` syntax makes a missing condition immediately visible as a syntax error instead of a silent semantic bug.

- **DO:** Sanity-check the row count of a multi-table query's result against a rough expectation (does the result have roughly as many rows as the "many" side of the relationship, not the product of two tables' sizes) whenever a join-heavy query's result looks suspiciously large. A result set in the millions from a query joining two tables with thousands of rows each is a strong signal of an accidental cartesian product.

- **DON'T:** Assume a `CROSS JOIN` used unintentionally will be caught by testing on small development data. A cartesian product between two development tables with 5 rows each produces a "plausible-looking" 25-row result that passes a casual glance; the same query against production tables with 50,000 rows each produces 2.5 billion rows and can take down the database.

### Query Hints & Forcing Plans

- **DO:** Treat query hints and plan-forcing directives (`FORCE INDEX`/`USE INDEX` in MySQL, planner GUCs in PostgreSQL, query hints in SQL Server) as a targeted, last-resort tool for a specific, well-understood case where the planner is measurably and persistently choosing a bad plan — not as a routine way to write queries. A hint locks in today's assumption about the best plan, which can become the wrong assumption as data volume and distribution change over time.

- **DON'T:** Reach for a query hint before exhausting simpler fixes — updating stale statistics, adding a missing index, rewriting the query to help the planner reason about it, or checking whether a parameter-specific issue ("parameter sniffing" in SQL Server) is the real cause. A hint that works around a symptom without addressing the underlying statistics or indexing problem often stops working, silently, once the data changes enough.

- **DO:** Document, next to any query hint left in production code, exactly why it was needed and what the planner was doing wrong without it, so a future maintainer knows whether the hint is still necessary rather than treating it as an unexplainable magic incantation nobody dares remove.

- **DON'T:** Copy a query hint from one query to a superficially similar one without verifying that the same underlying planner misestimate applies. Hints are specific to the exact query, the exact data distribution, and the exact database version they were tuned against — assuming they generalize is often wrong.

- **DO:** Re-verify any long-standing query hint periodically (after a major database version upgrade, or after a significant change in data volume/distribution) since planner improvements or changed statistics can mean a hint that was once necessary is now actively preventing the planner from choosing a better plan on its own.

### Serverless & Edge Connection Management

- **DO:** Use a connection pooler/proxy (PgBouncer, RDS Proxy, Cloud SQL Auth Proxy, or the platform's equivalent) between a serverless or edge compute layer and the database, rather than opening a direct database connection per function invocation. Serverless functions scale by spinning up many concurrent short-lived instances, each potentially opening its own connection — without a pooler in front of the database, a traffic spike can open far more connections than the database's connection limit allows, causing every function invocation to start failing at once.

- **DON'T:** Assume a database connection pool configured for a traditional long-running server process (a fixed pool of N connections reused across many requests within one process) works the same way in a serverless environment where each function invocation may be its own short-lived process with its own pool. Without externalizing pooling to a proxy, "pool size per instance" multiplied by "number of concurrent instances" can vastly exceed what the database can actually handle.

- **DO:** Reuse a database connection across invocations within the same serverless execution environment when the platform allows it (keeping a connection alive in module-level/global scope so a "warm" function invocation reuses it instead of reconnecting), while still accounting for the possibility of a "cold start" needing a fresh connection.

- **DON'T:** Ignore connection limits when load-testing or capacity-planning a serverless architecture. The database's maximum connection count is a hard, shared ceiling regardless of how elastically the compute layer scales — plan and test against that real ceiling, with a pooler in front of it, rather than assuming serverless scaling automatically means the database layer scales along with it.

### GraphQL & API Layer Query Amplification

- **DO:** Use a batching/dataloader pattern at the GraphQL resolver layer (or equivalent for any API layer that lets clients request nested, composable data) so that resolving a nested field across many parent objects in one response collapses into batched database queries, exactly the same discipline as avoiding N+1 in any other data-access code — except here the "loop" is implicit in the shape of the client's query rather than an explicit loop in application code, which makes it easier to miss.

- **DON'T:** Assume a GraphQL (or similarly flexible) API layer is safe from N+1 query problems just because no explicit loop appears in the resolver code. A naive per-field resolver that independently queries the database for each object in a list, with no batching layer in front of it, produces exactly as many database round trips as a hand-written N+1 loop would — the client's ability to request arbitrarily nested data just makes the problem easier to trigger accidentally and harder to spot in code review.

- **DO:** Set query depth limits, complexity/cost limits, or pagination requirements on flexible query APIs (GraphQL, OData, or any client-composable query layer) to prevent a single client-constructed query from fanning out into an unbounded number of database operations. A deeply nested query with no depth limit can force the server to resolve an exponentially growing number of database calls from what looks like one API request.

- **DON'T:** Expose a flexible query API's full field/relationship graph without considering which paths through it correspond to genuinely expensive database operations. Not every field is equally cheap to resolve — a field that triggers a full-text search or a cross-table aggregation needs its own cost accounting in any complexity-limiting scheme, not a flat per-field cost that under-counts it.

### Query Timeout & Circuit Breaking for Downstream Protection

- **DO:** Set an application-level timeout on every database call, distinct from (and typically shorter than) any database-side statement timeout, so a slow or hung query fails the specific request that triggered it quickly rather than tying up an application thread/connection indefinitely while waiting on a response that may never come.

- **DON'T:** Let a database performance degradation cascade into a full application outage through unbounded retry loops or unbounded connection-pool waiting. Without a circuit breaker (a mechanism that detects repeated failures to a downstream dependency and briefly stops sending it new requests, giving it room to recover), a struggling database can be driven further into overload by an application that keeps retrying at full volume exactly when the database most needs load relief.
```pseudocode
// Circuit breaker: stop hammering a struggling database, fail fast instead,
// and give it a chance to recover rather than piling on more load
if circuitBreaker.isOpen("primary_db"):
    return cachedFallbackOrError()
try:
    result = db.query(..., timeout=2s)
    circuitBreaker.recordSuccess("primary_db")
    return result
except TimeoutError:
    circuitBreaker.recordFailure("primary_db")
    raise
```

- **DO:** Design a defined degraded-mode behavior (serve stale cached data, return a partial response, queue the write for later) for critical user-facing paths when the database is unavailable or overloaded, rather than letting every database dependency be a hard single point of failure with no fallback.

### Prepared Statements & Plan Caching

- **DO:** Use prepared/parameterized statements consistently, not only for their security benefit (preventing injection) but for their performance benefit — the database can parse and plan the statement once and reuse that plan across repeated executions with different parameter values, avoiding the parse/plan overhead on every single call for a frequently-run query shape.

- **DON'T:** Assume a cached query plan is always optimal for every parameter value it's reused with. Some databases (SQL Server and, under some configurations, PostgreSQL) can cache a plan based on the first parameter values seen and then reuse that same plan for very different, less-representative parameter values later — a phenomenon known as parameter sniffing — which can produce a plan that's excellent for one value and terrible for another.

- **DO:** Recognize the specific symptoms of a parameter-sniffing problem (a query that's fast for most inputs but occasionally, consistently slow for certain specific parameter values) and address it with the database-specific tool for the job — plan hints, `OPTION (RECOMPILE)` in SQL Server for a genuinely variable query, or adjusting `plan_cache_mode` in PostgreSQL — rather than assuming the query itself is simply inconsistent for no discoverable reason.

- **DON'T:** Disable prepared-statement plan caching wholesale as a blunt fix for an occasional bad-plan-reuse problem, without first confirming that's actually the cause. Losing plan caching entirely trades away a real, ongoing performance benefit to fix what's often a narrower, specific issue that has a more targeted fix.

### Streaming Large Result Sets to Clients

- **DO:** Stream a large query result directly to the client (a CSV/JSON export endpoint, a large report download) using a database cursor and a chunked HTTP response, rather than materializing the entire result set in application memory before sending any of it. Streaming keeps memory usage bounded regardless of result size and lets the client start receiving data immediately instead of waiting for the entire query and serialization to finish first.
```pseudocode
// Stream rows to the client as they're fetched, never holding the whole
// result set in memory at once
response.setHeader("Content-Type", "text/csv")
cursor = db.queryCursor("SELECT * FROM large_table WHERE ...")
for row in cursor:  // fetched in batches under the hood, not all at once
    response.write(toCsvRow(row))
response.end()
```

- **DON'T:** Build an export feature by first loading the full query result into an in-memory array/list and only then converting and sending it. For a large table, this is both a memory-usage risk (a big enough export can exhaust the process's memory) and an unnecessary latency cost (the client waits for the entire operation to complete before receiving the first byte).

- **DO:** Set an explicit, documented maximum export size (or require the request to be scoped by a filter/date range below some cap) for any user-triggered bulk export feature, so a single export request can't become an unbounded query against the entire table regardless of how the streaming is implemented.

- **DON'T:** Forget that a long-running streamed response holds its underlying database cursor and connection open for the entire duration of the stream, which can be a long time for a large export over a slow client connection. Account for this in connection pool sizing, and set a reasonable idle/total-duration timeout so a slow or abandoned client connection doesn't hold a database connection open indefinitely.

### Safe Dynamic Filtering for Search & Filter UIs

- **DO:** Build dynamic `WHERE` clauses for search/filter UIs (many optional filters, any combination of which might be applied) by conditionally appending parameterized clauses joined with `AND`, keeping every value as a bound parameter regardless of which filters are actually active, rather than assembling the condition list via string interpolation of the values themselves.
```pseudocode
// Safe dynamic filtering: conditions vary, but every value stays parameterized
clauses = []
params = []
if filters.status:
    clauses.append("status = ?"); params.append(filters.status)
if filters.minPrice:
    clauses.append("price >= ?"); params.append(filters.minPrice)
sql = "SELECT * FROM products" + (clauses.length ? " WHERE " + clauses.join(" AND ") : "")
db.query(sql, params)
```

- **DON'T:** Build a search/filter query by interpolating each filter value directly into the SQL string, even when validating that the value "looks like" the expected type first. Type-looking validation is not equivalent to parameterization and is easy to get subtly wrong across every filter combination — use bound parameters for every value uniformly, with no exceptions carved out for values that seem safe.

- **DO:** Use the query-builder or ORM's native conditional/dynamic query construction methods when available, which are designed specifically for this exact "compose optional conditions safely" pattern, rather than hand-rolling string concatenation logic that has to correctly handle every combination of active and inactive filters, `AND`/`OR` boundaries, and parameter ordering.

- **DON'T:** Let dynamically-composed filter queries run with no bound at all on the number of active filters or the complexity of the resulting query, especially when filters can include user-composed logical combinations (arbitrary AND/OR nesting). An unbounded, user-composable filter query is a route to both a resource-exhaustion risk and, if a corner case in the query-building logic is wrong, a filter-bypass bug that returns data the filter should have excluded.

### LIMIT Without ORDER BY: Non-Deterministic Results

- **DO:** Always pair `LIMIT` with an explicit, fully-deterministic `ORDER BY` (including a unique tiebreaker column) whenever the specific rows returned matter — which is nearly always. Without an `ORDER BY`, a database is free to return any rows it likes in any order for a `LIMIT` query, and while it often appears to return a consistent order in practice (following physical storage order), that behavior is an implementation detail, not a guarantee, and can change silently between query runs, after a `VACUUM`/reorganization, or after an index is added.
```sql
-- Which 10 rows come back, and in what order, is not actually guaranteed
SELECT * FROM orders LIMIT 10;

-- Deterministic: always the same 10 rows, always in the same order
SELECT * FROM orders ORDER BY created_at DESC, id DESC LIMIT 10;
```

- **DON'T:** Rely on a `LIMIT`-only query's apparent consistency observed during development or testing as evidence that it's actually deterministic. A query that happens to always return the same rows on a small, rarely-modified development database can behave completely differently in production once the table is large, actively written, and has a different physical layout — the absence of `ORDER BY` is a latent bug regardless of what's been observed so far.

- **DO:** Treat a `LIMIT` query with no `ORDER BY` as an explicit, deliberate choice only for the rare cases where truly any rows would do (a query that only needs to check "does at least one row matching this condition exist," for instance) — and even then, `EXISTS` is usually the more direct way to express that intent.

- **DON'T:** Use `LIMIT` with no `ORDER BY` in a pagination implementation, a "top N" report, or any test/snapshot assertion that compares specific returned rows. Each of these depends on getting the *same* rows back reliably — exactly the guarantee an unordered `LIMIT` doesn't provide — and flaky, hard-to-reproduce bugs in exactly this shape are a common, avoidable consequence.

### Avoiding Correlated Subqueries in SELECT Lists

- **DO:** Replace a subquery placed in the `SELECT` list (executed once per outer row) with a `JOIN` against a pre-aggregated derived table, or a window function, whenever the same aggregate/lookup is needed for every row of a result set. A subquery in the select list is a correlated subquery evaluated once per outer row, which scales linearly with the outer result set size in a way a single join or window-function pass typically does not.
```sql
-- Slower: subquery re-executed once per customer row
SELECT c.name,
  (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS order_count
FROM customers c;

-- Faster: aggregated once, joined in a single pass
SELECT c.name, COALESCE(oc.order_count, 0) AS order_count
FROM customers c
LEFT JOIN (
  SELECT customer_id, COUNT(*) AS order_count FROM orders GROUP BY customer_id
) oc ON oc.customer_id = c.id;
```

- **DON'T:** Assume the query planner will always transparently rewrite a correlated `SELECT`-list subquery into an equivalent efficient join-based plan. Some planners do optimize simple cases, but this isn't guaranteed across engines or query shapes — verify with `EXPLAIN` rather than relying on the planner to always save you from an expensive pattern.

- **DO:** Use window functions (`SUM(...) OVER (PARTITION BY ...)`, `COUNT(...) OVER (...)`) as a often-cleaner and often-faster alternative to a `SELECT`-list subquery when the aggregate needs to appear alongside each individual row rather than collapsed into a grouped result.

- **DON'T:** Nest multiple correlated subqueries in the same `SELECT` list, each independently re-scanning related tables per outer row. Each additional correlated subquery compounds the per-row cost — if several aggregates are needed per row, compute them together in one pre-aggregated join rather than as several separate per-row subqueries.

### Numeric vs String Comparison Pitfalls

- **DO:** Match the data type of a query parameter to the actual column type exactly — pass a numeric value for a numeric column, a string for a text column — rather than relying on the database to implicitly cast a mismatched type. An implicit cast between a numeric column and a string parameter (or vice versa) can silently prevent the planner from using an index on that column, since the index is built on the column's actual type, not the type of whatever gets compared against it.
```sql
-- If order_id is an INTEGER column, passing it as a string can defeat
-- the index on some engines/configurations by forcing an implicit cast
SELECT * FROM orders WHERE order_id = '1042';  -- string literal vs int column

-- Match the type explicitly
SELECT * FROM orders WHERE order_id = 1042;
```

- **DON'T:** Assume every database handles a numeric-vs-string type mismatch identically. Some engines/configurations will transparently and efficiently coerce the comparison; others will defeat an otherwise-usable index, or in stricter modes, reject the comparison outright — verify the actual behavior for the specific engine and column type rather than assuming implicit coercion is always free.

- **DO:** Be especially careful with type mismatches in application code that builds query parameters dynamically from request data (URL query parameters, form fields), which typically arrive as strings regardless of the target column's real type — parse and convert them to the correct native type in application code before passing them to the query, rather than passing the raw string through and relying on the database to sort it out.

### Estimating Result Size Before Running Expensive Ad Hoc Queries

- **DO:** Run `EXPLAIN` (without `ANALYZE`, which actually executes the query) to check the planner's estimated row count and cost before running an unfamiliar, potentially expensive ad hoc query directly against a production database, especially one without a `LIMIT`. This gives a cheap, non-executing preview of roughly how much work the query is about to do, catching an accidentally unbounded or cartesian-product query before it actually runs.

- **DON'T:** Run an unfamiliar ad hoc query directly against production with no size estimate first, especially a query built somewhat speculatively (an exploratory join, an aggregate across a table of unknown size). A query that turns out to scan and return far more data than expected can degrade the database for other concurrent users for as long as it runs, and canceling it after the fact doesn't undo the load it already placed on the system.

- **DO:** Add a `LIMIT` (or run against a read replica, or wrap the exploration in an explicit read-only transaction with a statement timeout) as standard practice when exploring or testing a new query interactively against a production or production-like database, treating the safety measure as a default habit rather than something added only after a close call.

- **DON'T:** Trust an `EXPLAIN` row-count estimate blindly when the table's statistics might be stale (right after a large bulk load, for instance) — cross-check against a quick, cheap sanity check (an approximate count, or querying a similar smaller time range first) when the stakes of getting the estimate wrong are high enough to matter.

## Migrations

### Reversibility

- **DO:** Write every schema migration with both an "up" (apply) and a "down" (revert) path, even if the down path is rarely exercised in practice. A migration with no rollback path turns any deploy that needs to be reverted into an emergency hand-written fix under pressure, instead of a routine, pre-tested `migrate down` command.
```sql
-- up.sql
ALTER TABLE users ADD COLUMN phone_number TEXT;

-- down.sql
ALTER TABLE users DROP COLUMN phone_number;
```

- **DON'T:** Write a migration whose "down" method is a no-op stub (`def down; end`) purely to satisfy the framework's requirement for one, without actually thinking through what reverting the change means. A fake rollback gives false confidence that the migration is reversible, right up until someone actually needs to run it during an incident and discovers it does nothing.

- **DO:** Treat some migrations as legitimately, explicitly irreversible (most destructive operations — see below) and mark them as such clearly, rather than pretending every migration can be perfectly undone. An honest "this migration cannot be automatically reversed; restore from backup if needed" is more useful than a `down` method that silently does the wrong thing.

- **DON'T:** Assume a migration is reversible just because its inverse operation exists in the migration framework's DSL. `add_column` reversing cleanly to `remove_column` says nothing about whether the *data* that existed in that column is recoverable — reverting a migration that dropped a column does not restore the values that were in it before the drop.

- **DO:** Test the down migration, not just the up migration, in CI or a staging environment before merging — actually run `migrate up`, then `migrate down`, then `migrate up` again, and confirm the schema ends up in a consistent, expected state at each step.
```pseudocode
// CI migration test
run("migrate up")
assertSchemaMatches(expectedAfterUp)
run("migrate down")
assertSchemaMatches(expectedBeforeUp)
run("migrate up")   // re-apply to confirm idempotent up/down cycle
assertSchemaMatches(expectedAfterUp)
```

- **DO:** Keep each migration small and focused on one logical schema change (one new table, one column addition, one index) rather than bundling many unrelated changes into a single migration file. Small migrations are easier to review, easier to reason about reverting individually, and less likely to partially fail midway through a large multi-statement change.

### Destructive Migrations & Backups

- **DO:** Take (or confirm the existence of) a verified, restorable backup immediately before running any migration that drops a column, drops a table, truncates data, or otherwise destroys information that cannot be regenerated from other tables. A backup that has never been test-restored is not a real safety net — verify restore procedures periodically, not only when disaster has already struck.

- **DON'T:** Run `DROP COLUMN`, `DROP TABLE`, or `TRUNCATE` against production as part of the same deploy that stops writing to that column/table. If the new code has a bug and needs to be rolled back, the data it depended on is already gone — separate "stop using this data" from "delete this data" by at least one full deploy cycle (often measured in days or weeks), so there is a safe rollback window.
```pseudocode
// Deploy 1: stop reading/writing the old column; keep it in the schema
// Deploy 2 (days/weeks later, after confirming Deploy 1 is stable):
//   migration that actually drops the now-unused column
```

- **DO:** Rename destructive migrations distinctly in the codebase or PR description (e.g., a clear "DESTRUCTIVE" label or comment) so they get extra scrutiny in review, rather than blending in with routine additive changes. A reviewer skimming a stack of migrations should immediately spot the one that deletes data.

- **DON'T:** Run an untested destructive migration directly against production "because it worked in the migration file review." Dry-run destructive migrations against a realistic staging copy of production data first, and where feasible, run them inside a transaction with a final manual confirmation step before commit for one-off/manual destructive operations outside the normal migration pipeline.

- **DO:** Prefer soft, reversible alternatives to hard deletion when the business requirement is genuinely ambiguous about whether data will be needed later — renaming a column with a deprecation prefix and dropping it later, or archiving rows to a separate table before purging them from the primary table.
```sql
-- Archive-then-delete pattern: data isn't gone, it's relocated
INSERT INTO orders_archive SELECT * FROM orders WHERE created_at < now() - interval '7 years';
DELETE FROM orders WHERE created_at < now() - interval '7 years';
```

- **DON'T:** Assume "we have backups" alone is sufficient justification to run a risky destructive change carelessly. Restoring from backup after a bad migration typically means real downtime, lost data for the window since the last backup, and a much worse incident than simply being careful in the first place — backups are a last resort, not a substitute for caution.

### Zero-Downtime Migration Patterns (Expand/Contract)

- **DO:** Use the expand/contract (a.k.a. parallel change) pattern for any schema change that the running application depends on — add the new structure, migrate/dual-write data, switch reads to the new structure, then remove the old structure — so the database is never in a state incompatible with the currently-deployed application code. Because deploys are rarely instantaneous and rollback must remain possible, the old and new schema shapes must coexist for a transition period.
```pseudocode
// Expand/contract for renaming a column, e.g. `name` -> `full_name`

// 1. EXPAND: add the new column, don't touch the old one
ALTER TABLE users ADD COLUMN full_name TEXT;

// 2. BACKFILL: populate the new column from the old one, in batches
UPDATE users SET full_name = name WHERE full_name IS NULL;  // batched, not one giant UPDATE

// 3. DUAL WRITE: deploy app code that writes both columns on every write

// 4. SWITCH READS: deploy app code that reads only from full_name

// 5. CONTRACT: once fully rolled out and stable, stop dual-writing, then:
ALTER TABLE users DROP COLUMN name;
```

- **DON'T:** Rename a column directly (`ALTER TABLE users RENAME COLUMN name TO full_name`) as a single-step deploy when old application code (still running on other instances during a rolling deploy, or subject to rollback) references the old column name. The instant the rename applies, every instance still running old code starts failing every query that touches that column.

- **DO:** Add new columns as nullable (or with a default) first, backfill data in batches, and only add a `NOT NULL` constraint once the backfill is confirmed complete. Adding a `NOT NULL` column with no default to a table that already has rows fails outright (or, worse, locks the table while rewriting every row) unless the database supports fast metadata-only defaults for the version in use.
```sql
-- Step 1: nullable, no lock-heavy rewrite
ALTER TABLE users ADD COLUMN country_code TEXT;

-- Step 2: backfill in batches (see batch operations)

-- Step 3: only after backfill is verified complete
ALTER TABLE users ALTER COLUMN country_code SET NOT NULL;
```

- **DON'T:** Change a column's type in a single blocking statement on a large, actively-written table without checking whether that specific type change requires a full table rewrite on the database engine in use. Some type changes are fast metadata-only operations; others (e.g., narrowing a type, or certain changes on MySQL/older PostgreSQL versions) rewrite the entire table and hold a blocking lock for the duration.

- **DO:** Create indexes on large, live tables using the non-blocking/concurrent variant the database provides (`CREATE INDEX CONCURRENTLY` in PostgreSQL, online DDL / `ALGORITHM=INPLACE` in MySQL/InnoDB) instead of a plain blocking `CREATE INDEX`, which takes a lock that blocks writes (and in some databases, reads) to the table for the entire build duration.
```sql
-- PostgreSQL: builds the index without holding a long write lock
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders (customer_id);
```

- **DON'T:** Forget that `CREATE INDEX CONCURRENTLY` (and similar online DDL tools) cannot run inside a transaction block in PostgreSQL, and can fail partway through leaving an invalid index behind that must be dropped and retried — check for and clean up invalid indexes after a failed concurrent build rather than assuming failure left no trace.

- **DO:** Split adding a foreign key constraint to an existing large table into two steps where the database supports it: add the constraint as `NOT VALID` (skipping the initial full-table validation scan under a lock), then `VALIDATE CONSTRAINT` separately, which can run with a lighter lock than the initial creation.
```sql
-- PostgreSQL: add the constraint without validating existing rows first
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer
  FOREIGN KEY (customer_id) REFERENCES customers(id) NOT VALID;

-- Validate separately — still scans the table, but doesn't block new writes
-- with as strong a lock as combining both steps would
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_customer;
```

- **DO:** Sequence expand/contract steps across separate deploys with a stabilization/monitoring period between each, rather than trying to compress the whole expand-backfill-switch-contract cycle into one release. Each step should be independently safe to roll back without requiring the next step to have happened.

### Shared History Discipline

- **DO:** Treat a migration as immutable the moment it has been applied to any shared environment (any environment other than the author's own local machine) — staging, a shared dev database, or production. Editing an already-applied migration file changes what future clones of the repository will apply, while environments that already ran the old version silently diverge from what the migration file now claims to do.

- **DON'T:** Edit the contents of a migration file that has already run in a shared environment, even to fix a typo or a small mistake. Anyone who already applied it has a database that doesn't match the corrected file; anyone applying it fresh gets the corrected version — the two groups now have silently different schemas with no record of the discrepancy. Write a new migration that corrects the mistake instead.
```pseudocode
// WRONG: editing 20260101_add_status_column.sql after it's already
// been applied in staging — anyone who already ran it now has a
// database that no longer matches what the file says it does.

// RIGHT: a new migration that fixes the earlier one
// 20260115_fix_status_column_default.sql
ALTER TABLE orders ALTER COLUMN status SET DEFAULT 'pending';
```

- **DO:** Use a migration tool that tracks which migrations have been applied per environment (a schema_migrations/version table) so that "has this run here yet" is answered definitively by the database itself, not by developer memory or a changelog. This is what makes it safe to detect drift and refuse to silently re-run or skip a migration.

- **DON'T:** Reorder or renumber migration files after they've been created, even before deploying, once more than one person's local environment might already have pulled and applied them. Renumbering a migration that a teammate already ran locally under its old identifier can desynchronize their applied-migrations record from the new file layout, causing a confusing "already applied" or "out of order" error.

- **DO:** Resolve migration conflicts from parallel feature branches (two branches both add a migration intended to run "next") by renumbering the later-merged one to a new timestamp/sequence number at merge time, rather than trying to force both to occupy the same sequence position. Most timestamp-based migration systems are designed to make this the normal, expected resolution.

- **DON'T:** Manually edit a production database's schema out-of-band (a DBA running `ALTER TABLE` directly in a console to "fix something quickly") without also creating a corresponding migration file. An out-of-band change leaves the migration history lying about the actual state of the schema, and the next migration run — or the next environment provisioned from migrations alone — won't reflect that manual fix.

- **DO:** Include the migration tool's applied-migration tracking table/mechanism in the standard backup and disaster-recovery plan, so that restoring a database from backup also restores an accurate record of which migrations have and haven't been applied.

### Migration Tooling & Review

- **DO:** Run every migration against a realistic copy of production data (or at minimum a representative-scale synthetic dataset) in a staging environment before it reaches production, especially for any migration touching a large table. A migration that runs instantly against an empty local database can lock a multi-million-row production table for minutes.

- **DON'T:** Assume a migration framework's "safe" defaults are safe for the specific database engine and table sizes in production. Some frameworks generate migrations that work fine on small tables but silently choose a blocking, full-table-rewrite strategy on a specific engine/version for changes that could have been done online — verify against the actual database and version, not the framework's general claims.

- **DO:** Set (and enforce in CI where possible) a maximum lock-timeout for migrations run against production, so a migration that cannot acquire the lock it needs quickly fails fast and rolls back, instead of queueing behind other traffic and blocking every subsequent query on that table indefinitely.
```sql
-- PostgreSQL: fail fast rather than waiting indefinitely for a lock
SET lock_timeout = '2s';
ALTER TABLE orders ADD COLUMN priority INT;
```

- **DON'T:** Run a schema migration and a data backfill in the same transaction/statement when the backfill touches a large number of rows. A combined operation holds whatever lock the schema change requires for the entire duration of the backfill, instead of a brief lock for the schema change followed by a separately-batched, lower-impact backfill.

- **DO:** Require migrations to go through the same code review process as application code, with reviewers specifically checking for locking behavior, reversibility, and whether the change fits an expand/contract sequence when the table is large or the change is breaking. A migration is production infrastructure change, not a formality to rubber-stamp.

- **DON'T:** Let a migration silently depend on application-level logic (a Rails model callback, an application-defined function) to backfill or transform data during the migration. Migrations should be self-contained SQL/DDL wherever possible; depending on application code that can change or be deleted later means a migration re-run months later (on a fresh environment, or during a restore) may behave completely differently than it did the first time.

- **DO:** Log or record migration execution time and lock duration in CI/staging runs for migrations against large tables, and treat an unexpectedly long-running migration as a signal to investigate before it ever reaches production, not after an incident report.

### Squashing & Consolidating Migration History

- **DO:** Squash a long chain of historical migrations into a single baseline schema definition periodically for a mature project, once every environment that matters has already applied the individual migrations being squashed and the old chain is purely historical baggage for new environments. A five-year-old project with eight hundred individual migration files makes provisioning a fresh database (for a new developer, a test environment, a disaster recovery restore) needlessly slow and fragile.

- **DON'T:** Squash migrations that any environment still needs to apply individually — squashing is only safe for history that every live environment has already fully applied; an environment that's behind and still needs to run the pre-squash migrations individually will break if the individual files are removed from under it.

- **DO:** Keep the pre-squash migration files archived (in a separate directory, a tag, or a branch) even after consolidating them into a baseline, rather than deleting them outright. They remain useful as historical documentation of how the schema evolved and as a reference if an old backup or environment ever needs to be replayed from scratch.

- **DON'T:** Treat migration squashing as a substitute for actually fixing a bad migration. Squashing consolidates history for provisioning speed; it does not retroactively make a badly-designed past migration well-designed — a genuinely wrong past decision (an FK that should cascade differently, a column that should never have been added) still needs its own forward-fixing migration, squashed history or not.

- **DO:** Automate a check (in CI, or in the migration tool itself) that fails loudly if an environment's applied-migrations record doesn't match what the current migration files expect, especially right after a squash, so that a stale environment is caught immediately rather than silently drifting.

### Backfills as Part of Migrations

- **DO:** Chunk any data backfill that's part of a migration into small, bounded batches (by primary key range or a fixed row count) with a brief pause or a fresh transaction between batches, rather than a single `UPDATE` touching the whole table. Chunking keeps each individual transaction's lock duration short, keeps replication lag bounded (each chunk replicates quickly instead of one giant burst), and makes the backfill resumable if it's interrupted partway through.
```sql
-- Batched backfill: bounded lock duration per iteration, resumable, and
-- throttleable — run in a loop from a script or migration job, not as
-- one giant statement
UPDATE users SET normalized_email = LOWER(email)
WHERE id BETWEEN :batch_start AND :batch_start + 999
  AND normalized_email IS NULL;
```

- **DON'T:** Run a large backfill at full speed with no throttling on a production database that's also serving live traffic. Even a well-chunked backfill can still saturate I/O or replication bandwidth if it runs as fast as possible with no pause between batches — add a small delay between batches and monitor replication lag and query latency while it runs, backing off if either degrades.

- **DO:** Make a backfill script idempotent and safely re-runnable (skip rows already processed, e.g., via a `WHERE column IS NULL` guard on the target column) so that if it's interrupted or needs to be restarted, re-running it from the beginning does not redo already-completed work or produce incorrect results.

- **DON'T:** Run an ad hoc backfill script by hand against production with no logging of progress, no way to resume from where it stopped, and no record afterward of exactly what was changed. Treat a production backfill with the same operational rigor as a migration — logged, monitored, and reproducible — not as a one-off manual task run from someone's laptop.

- **DO:** Schedule large backfills during lower-traffic windows when possible, and coordinate with whoever monitors production during the backfill window, so an unexpected performance impact is caught and addressed quickly rather than discovered independently as a user-facing incident.

### Feature Flags & Schema-Dependent Rollouts

- **DO:** Gate application code that depends on a new schema shape behind a feature flag, decoupling "the migration has run" from "the new code path is active." This lets the schema change and the behavior change roll out independently — the migration can be verified as applied and stable in production before the flag is flipped, and the flag can be flipped back instantly if the new code path misbehaves, without needing a second migration to undo anything.
```pseudocode
if featureFlags.isEnabled("use_new_pricing_column"):
    price = order.newPricingColumn
else:
    price = computeLegacyPrice(order)
```

- **DON'T:** Ship a schema migration and the application code that depends on it in a way that makes rollback of the application code alone impossible. If rolling back a bad deploy also requires rolling back the migration (which, per the expand/contract discipline, often isn't safe or fast to do), the team has lost the fast, low-risk rollback path that feature flags and expand/contract are both meant to preserve.

- **DO:** Remove the feature flag and the old code path deliberately, as a follow-up cleanup step, once the new path has been stable in production for a defined period — flags that stay in the codebase indefinitely accumulate as dead branches and untested combinations of flag states that nobody remembers the purpose of.

- **DON'T:** Leave both the old and new code paths permanently active behind a flag with no plan or ticket to retire the old path. A flag meant to be temporary that never gets cleaned up effectively doubles the schema's maintenance burden forever, since both the old and new column/table now need to be kept correct indefinitely.

### Backup & Disaster Recovery Testing

- **DO:** Define explicit Recovery Point Objective (RPO — how much data loss is acceptable, measured in time) and Recovery Time Objective (RTO — how long recovery is allowed to take) for every production database, and configure the actual backup frequency, replication topology, and restore procedure to meet those numbers deliberately, rather than backing up "regularly" with no explicit target tied to real business tolerance for loss and downtime.

- **DON'T:** Assume automated backups are working correctly just because the backup job reports success. A backup job can complete "successfully" while producing a backup that's corrupted, incomplete, or missing a critical piece of the restore chain (a base backup with no matching WAL/transaction log archive needed to reach a consistent point) — the only real confirmation that a backup works is a successful test restore.

- **DO:** Perform periodic test restores of production backups into an isolated environment, on a defined schedule, and verify the restored data is actually complete and consistent — not just that the restore process ran without an error. A disaster recovery plan that has never been rehearsed is a hypothesis, not a plan; the first real test of most untested backup strategies happens during an actual outage, which is the worst possible time to discover a gap.
```pseudocode
// Scheduled DR drill, not a one-time setup task
restoreLatestBackupToIsolatedEnvironment()
assertRowCounts(expected, restored)
assertSampleRecordsMatch(expected, restored)
measureAndLogRestoreDuration()   // compare against the defined RTO
```

- **DON'T:** Store backups only in the same physical location, account, or region as the primary database. A backup that's destroyed by the same regional outage, account compromise, or physical incident that took out the primary database provides no actual disaster recovery — store backups in a genuinely independent location/account with its own access controls.

- **DO:** Document and rehearse the actual restore procedure (not just "we have backups") as a runbook that anyone on the on-call rotation can follow under pressure, including how to restore to a specific point in time (point-in-time recovery), not only to the most recent full backup.

### Database Version Upgrades

- **DO:** Treat a major database engine version upgrade as a planned project with its own testing phase — reviewing the release notes for breaking changes, deprecated features, and changed default behaviors, and running the full application test suite plus a representative production-data workload against the new version in a staging environment before upgrading production.

- **DON'T:** Assume a major version upgrade is purely additive and backward compatible without checking. Major version releases of every mainstream database periodically change default configuration values, deprecate or remove syntax, or subtly change query planner behavior (a plan that was previously chosen might no longer be chosen after an upgrade) — verify explicitly rather than assuming.

- **DO:** Plan a rollback path for a database version upgrade before starting it — whether that's a tested downgrade procedure, a pre-upgrade backup with a defined restore time, or a blue/green cutover that keeps the old version running and reachable until the new version is verified — since some upgrades are not cleanly reversible once completed (a schema/storage format change may be one-directional).

- **DON'T:** Upgrade a production database's major version with no maintenance window or rollback plan just because a minor point release "should be safe." Even routine-seeming upgrades occasionally surface an edge case specific to a particular workload; treat every production database version change with the same care as a significant infrastructure change, proportional to that database's actual criticality.

### Cross-Database Portability Tradeoffs

- **DO:** Decide deliberately, upfront, whether a project genuinely needs to support multiple database engines (an open-source project that must run on whatever the user has, a product sold to customers who mandate a specific vendor) versus committing to one engine and using its full feature set without artificial restraint. Writing every migration and query as if it must be portable to every possible database, when the project will in fact only ever run on one, sacrifices real, useful engine-specific features (partial indexes, `RETURNING`, native JSON operators) for a portability guarantee nobody needs.

- **DON'T:** Use an ORM or query builder's "database agnostic" abstraction as a substitute for actually testing against every database engine the project claims to support. An abstraction layer reduces surface-level syntax differences but does not guarantee identical behavior — transaction isolation defaults, case sensitivity, `NULL` sorting order, and numeric precision handling all still differ by engine underneath a portable-looking query builder call.

- **DO:** Isolate genuinely engine-specific code (a raw SQL block using a feature only one target engine supports) behind a clearly-marked abstraction point when multi-engine support is a real requirement, so it's obvious which parts of the codebase need engine-specific handling and testing, rather than scattering engine-specific assumptions implicitly throughout.

- **DON'T:** Claim or imply multi-database support in documentation or configuration options without actually running the full test suite against each claimed-supported engine in CI. An untested "also works with MySQL" claim next to a codebase only ever run against PostgreSQL is often false the first time someone actually tries it, at the exact moment they're relying on the claim being true.

### Migrating Between Database Engines

- **DO:** Treat a database engine migration (MySQL to PostgreSQL, on-premises to a cloud-managed equivalent, a legacy engine to a modern one) as a project with its own explicit data-type mapping plan, dialect-difference audit, and a parallel-run validation phase, rather than assuming a mechanical dump-and-restore will produce an equivalent, correctly-behaving system on the other side. Two engines that both call themselves "SQL databases" can differ in exactly the ways that silently break an application — implicit type coercion rules, default isolation levels, auto-increment behavior, and NULL sort ordering all commonly differ.

- **DON'T:** Migrate application code's SQL wholesale to a new engine without checking every dialect-specific construct — proprietary functions, non-standard syntax, and engine-specific behaviors (like MySQL's historically more permissive handling of invalid dates, or differing string comparison defaults) — for behavioral differences the new engine will not silently replicate.

- **DO:** Run the source and destination databases in parallel for a defined validation window during a migration, comparing key business metrics and sampled query results between the two, before fully cutting over. This catches subtle behavioral differences in real production traffic that a pre-migration test suite, built from developers' assumptions about what matters, is likely to miss.

- **DON'T:** Attempt a database engine migration as a single big-bang cutover for a system that can't tolerate meaningful downtime, without a dual-write or CDC-based sync strategy that lets the cutover happen with the destination already caught up and continuously kept in sync until the actual switch.

### Blue-Green & Shadow Database Deployments

- **DO:** Use a blue-green database deployment (standing up a fully separate, up-to-date replica environment, validating it independently, then switching traffic to it) for major changes too risky or disruptive to apply in place — a major version upgrade, a significant schema restructuring, or an engine migration — when the ability to switch back instantly by routing traffic to the still-intact "blue" environment is worth the cost of running two full environments temporarily.

- **DON'T:** Assume blue-green deployment eliminates the need for the expand/contract discipline within the new ("green") environment's own schema changes. Blue-green solves "how do we safely cut over to a new environment"; it does not, by itself, solve "how do we make a breaking schema change without an app-code compatibility window" — the two techniques address different problems and are often used together, not as substitutes for each other.

- **DO:** Keep the "blue" (old) environment fully intact and capable of receiving traffic again for a defined rollback window after cutting over to "green," rather than tearing it down immediately, so a problem discovered shortly after cutover has a fast, tested path back rather than an emergency restore-from-backup.

- **DON'T:** Let data written to the new ("green") environment after cutover go unsynchronized back to the old ("blue") environment during the rollback window, if a rollback might still happen. A rollback that reverts traffic to "blue" but leaves it missing everything written to "green" since cutover silently loses that data — plan explicitly for how (or whether) data written post-cutover is preserved through a potential rollback.

### Coordinating Migrations Across Multiple Services or Teams

- **DO:** Establish an explicit communication and sequencing process for schema changes to a database shared across multiple services or teams — announcing upcoming breaking changes, agreeing on a deprecation timeline for a column or table other teams still use, and confirming before a migration runs that every current consumer has been accounted for. A shared database with no cross-team coordination process is exactly where "I didn't know anyone else used that table" incidents come from.

- **DON'T:** Assume a migration is safe to run just because it doesn't break the migrating team's own code paths. In a shared database, a change that's invisible from one team's application code (dropping an "unused" column, changing a type) can silently break another team's reporting query, batch job, or service that reads the same table without the migrating team's knowledge.

- **DO:** Prefer splitting a genuinely shared database into per-service schemas (or, longer-term, separate databases with explicit APIs between services) once cross-team migration coordination becomes a frequent source of friction or incidents — the coordination overhead is often a symptom that the database boundary no longer matches the team/service boundary.

- **DON'T:** Let a migration owned by one team silently depend on a migration owned by another team completing first, with no explicit ordering or dependency declared between them. An implicit cross-team migration ordering dependency, coordinated only informally ("just run yours after mine, I'll ping you"), is exactly the kind of process that breaks the first time the informal communication doesn't happen.

### Handling Migration Failures Mid-Deploy

- **DO:** Know whether the target database supports transactional DDL (schema changes that can be rolled back as part of a failed transaction) before assuming a partially-applied migration can simply be retried from the start. PostgreSQL supports transactional DDL — a failed multi-statement migration rolls back cleanly, leaving no partial change. MySQL largely does not — several DDL statements implicitly commit, meaning a migration that fails partway through can leave the schema in a partially-applied state that a naive retry of the whole migration will then fail against (e.g., "column already exists").
```sql
-- PostgreSQL: if the second statement fails, the whole block rolls back —
-- the ADD COLUMN never actually takes effect
BEGIN;
ALTER TABLE orders ADD COLUMN priority INT;
ALTER TABLE orders ADD CONSTRAINT chk_priority CHECK (priority BETWEEN 1 AND 5);
COMMIT;

-- MySQL: ALTER TABLE statements each implicitly commit — if the second
-- statement fails, the first one's change has already taken effect and
-- WILL NOT be rolled back by a surrounding transaction
```

- **DON'T:** Write a migration for a non-transactional-DDL database as if it were atomic, with no idempotency guard on each individual step. On such a database, every step of a multi-step migration needs to be safely re-runnable on its own (`ADD COLUMN IF NOT EXISTS`-style guards, or a check before each step) so that retrying after a partial failure doesn't fail again on the already-applied steps.

- **DO:** Have the migration tool record success/failure per individual step, not only per whole migration file, for databases without transactional DDL, so that after a partial failure it's possible to know precisely which steps already succeeded and resume correctly, rather than needing to manually inspect the schema to figure out where things stopped.

- **DON'T:** Treat a migration failure in production as something to fix by manually running the "missing" remaining statements by hand in a console, without also updating the migration tool's own tracking so it reflects what actually happened. A manual fix that isn't reflected in the migration history leaves the tracked state lying about what's actually been applied, exactly the shared-history integrity problem discussed earlier.

### Renaming Tables Safely

- **DO:** Use a compatibility view (creating a view under the old table name that points at the newly-renamed table) as a transitional shim when renaming a table that other code, other services, or ad hoc reporting queries might still reference by its old name, so that a direct `ALTER TABLE ... RENAME` doesn't instantly break every reference to the old name the moment it runs.
```sql
ALTER TABLE customers RENAME TO accounts;
-- Transitional shim: old name still resolves, buying time for every
-- consumer to update to the new name before the shim is removed
CREATE VIEW customers AS SELECT * FROM accounts;
```

- **DON'T:** Rename a table directly in a single step when its old name is embedded in code, dashboards, or reports outside the immediate deploy's control (a BI tool's saved queries, another team's script, a data analyst's ad hoc notebook). A rename that breaks external, less-visible consumers is a common source of "why is the nightly report suddenly failing" incidents discovered well after the migration that caused it.

- **DO:** Track down and update every known reference to the old table name systematically (a codebase-wide search, checking BI tool saved queries, checking other services' code) before removing the transitional compatibility view, treating the view's removal as its own separate, deliberate step — not an automatic cleanup that happens right after the rename.

- **DON'T:** Leave a compatibility view in place indefinitely once every consumer has migrated to the new name. Like a feature flag left in forever, an unremoved compatibility view becomes permanent, confusing indirection — schedule and actually execute its removal once its purpose is served.

### Concurrent Migration Development & Merge Conflicts

- **DO:** Expect and plan for two developers independently creating migrations on separate branches around the same time, and resolve the resulting ordering conflict at merge time by renumbering the later-merged migration to a fresh timestamp/sequence number, as covered under shared history discipline — treat this as a routine, expected event in any team of more than one, not an exceptional situation needing an ad hoc fix each time.

- **DON'T:** Let two migrations that both alter the same table in incompatible ways merge without anyone noticing the conflict, just because the migration filenames don't literally collide. A file-naming/sequence conflict is easy for tooling to catch automatically; a *semantic* conflict (two migrations both trying to add a column with the same name, or one dropping a column the other still expects) requires an actual reviewer or a schema-diff check to catch, since the migration tool alone often won't.

- **DO:** Run the full migration chain (applying every migration in sequence from a clean database) as part of CI on every pull request that adds a migration, so an ordering or semantic conflict between concurrently-developed migrations is caught automatically before merge, rather than discovered by the next developer who pulls `main` and finds their local database won't migrate.

- **DON'T:** Resolve a migration merge conflict by editing the content of a migration that's already been applied anywhere shared (even just a teammate's local environment, if migrations are meant to be shared history) — per the shared-history discipline, the fix is a new migration or a renumbering of the not-yet-applied one, never an edit to one that's already run somewhere that matters.

### Data Type Widening vs Narrowing

- **DO:** Treat type-widening changes (`INT` to `BIGINT`, `VARCHAR(50)` to `VARCHAR(200)`, `NUMERIC(10,2)` to `NUMERIC(14,2)`) as generally safe, non-lossy changes that preserve every existing value — though still worth checking whether the specific database performs this as a fast metadata-only change or a full table rewrite, since that varies by engine and by the specific type change.

- **DON'T:** Treat a narrowing change (`BIGINT` to `INT`, reducing a `VARCHAR`'s max length, reducing a `NUMERIC`'s precision or scale) as safe without first verifying that every existing value in the column actually fits within the new, smaller type. A narrowing migration applied to a column that already contains out-of-range values will fail outright (best case) or, on a database with permissive coercion, silently truncate or corrupt existing data (worst case).
```sql
-- Verify BEFORE narrowing, not after — find any value that wouldn't fit
SELECT COUNT(*) FROM orders WHERE quantity > 32767;  -- would overflow a SMALLINT
```

- **DO:** Run an explicit pre-check query against production data for any narrowing type change, confirming zero rows would be affected/truncated by the change, and treat any non-zero result as a blocker requiring a data-cleanup or business-decision step before the migration can safely proceed.

- **DON'T:** Narrow a column's type as a "cleanup" pass without first confirming with whoever owns the data's business meaning that the narrower range or precision is actually always sufficient going forward, not just sufficient for the data that happens to exist today. A type that's safely narrowed against current data can still be too narrow for legitimate future values, causing new failures shortly after the "cleanup."

## NoSQL

### Document Database Schema Design

- **DO:** Design document schemas around the application's actual access patterns — what a single screen or single API response needs to render — rather than mechanically mirroring a normalized relational schema inside documents. Document databases are optimized for retrieving one document in one lookup; a schema that still requires five separate document fetches to render one screen has thrown away the main advantage of the model.

- **DON'T:** Copy a relational schema's table structure directly into a document database, one collection per "table" with foreign-key-style references everywhere. This produces the worst of both worlds — none of the join performance or referential integrity guarantees of a relational database, and none of the locality benefits that make document databases fast for their intended access patterns.

- **DO:** Default toward embedding related data within a single document when that related data is only ever accessed together with its parent, has a bounded and predictable size, and does not need to be queried or updated independently. Embedding is what lets a document store answer "give me everything about this order" with a single lookup instead of a fan-out of queries.
```json
// Embedded: order and its line items are always read/written together
{
  "_id": "order_123",
  "customer_id": "cust_456",
  "items": [
    { "sku": "WIDGET-1", "qty": 2, "price": 9.99 },
    { "sku": "GADGET-2", "qty": 1, "price": 24.99 }
  ],
  "total": 44.97
}
```

- **DON'T:** Embed a collection that grows without bound inside a parent document (e.g., embedding every comment ever made on a post directly inside the post document). Most document databases enforce a maximum document size (16MB in MongoDB, for instance), and even well under that limit, an ever-growing embedded array means rewriting the entire parent document on every single append, which gets slower as the array grows.

- **DO:** Design the shape of embedded sub-documents around what a single query needs to return, accepting some duplication of frequently-read, rarely-changed fields (like a product's name and price snapshotted onto an order line item) as a deliberate tradeoff for read performance, exactly as with relational denormalization.

- **DON'T:** Assume schema flexibility (the fact that a document database doesn't enforce a rigid schema) means schema design doesn't matter or that no validation is needed. Every document store still benefits from a deliberate, consistent document shape — enforced by application-layer validation, or by native schema validation features where available (e.g., MongoDB's `$jsonSchema` validator) — otherwise the collection accumulates inconsistent, hard-to-query documents as the application evolves.
```javascript
// MongoDB: enforce a schema shape even in a "schemaless" database
db.createCollection("orders", {
  validator: {
    $jsonSchema: {
      required: ["customer_id", "items", "total"],
      properties: {
        total: { bsonType: "decimal", minimum: 0 }
      }
    }
  }
});
```

### Embedding vs Referencing

- **DO:** Reference (store an ID and look up separately) instead of embedding when the related data is large, grows unboundedly, is shared across many parent documents, or needs to be queried, updated, or paginated independently of its "parent." A product referenced by thousands of orders should live in its own document, updated once, not duplicated (and now inconsistent) across thousands of order documents.
```json
// Referenced: product catalog entries are shared across many orders,
// updated independently, and would bloat every order if embedded
{ "_id": "order_123", "customer_id": "cust_456", "product_ids": ["prod_1", "prod_2"] }
```

- **DON'T:** Reference when the "join" it requires happens on nearly every read of the parent document — that just re-creates the N+1 problem inside a database that has no efficient native join, forcing the application to do the join in code with multiple round trips (or forcing a database-specific `$lookup`-style aggregation that is often far more expensive than a relational join over indexed foreign keys).

- **DO:** Use a hybrid approach — embed a small, frequently-needed snapshot of related data (e.g., a customer's display name and avatar URL embedded in a comment) for what most reads need, while also storing a reference to the full related document for the cases that need complete/current data. This trades a bounded amount of duplication for avoiding a lookup on the hot path, while keeping full data reachable when needed.

- **DON'T:** Treat "MongoDB doesn't have foreign keys" as license to skip referential integrity entirely. Even without a database-enforced constraint, decide and implement (in application code, via a background consistency job, or via database features that do exist for this in some stores) how orphaned references get detected and cleaned up when a referenced document is deleted — otherwise references silently rot over time.

- **DO:** Reconsider the embed-vs-reference decision as access patterns change, the same as reconsidering relational denormalization — a field that seemed always-embedded-together at launch may need independent querying a year later, and the schema (and migration script to reshape existing documents) should evolve deliberately rather than accumulating inconsistent document shapes.

- **DON'T:** Assume the one-to-many "embed the many side" pattern is always correct for one-to-many relationships. When the "many" side is unbounded or needs independent access (all orders a customer has ever placed, potentially thousands), reference from the many side back to the one (each order references its customer_id) rather than trying to embed an unbounded array into the customer document.

### Key-Value Store Usage Patterns

- **DO:** Design key-value store keys with a clear, consistent, hierarchical naming convention (e.g., `user:1042:profile`, `session:abc123`, `cache:product:99:price`) that encodes what the value represents and supports the store's key-scanning/pattern-matching tools for operational debugging. An opaque or inconsistent key scheme makes production debugging and cache-key collisions much more likely.

- **DON'T:** Use a key-value store (Redis, Memcached, DynamoDB used purely as KV) for data that genuinely needs relational querying — filtering, joining, or aggregating across many keys by attributes other than the key itself. Key-value stores are fast because lookups are by exact key; forcing ad hoc cross-key queries (scanning every key to find ones matching some attribute) defeats the model and performs far worse than the equivalent query would in a database designed for it.

- **DO:** Set an explicit TTL/expiration on key-value entries that represent genuinely transient data (sessions, rate-limit counters, short-lived caches) rather than letting them accumulate forever. An in-memory key-value store with no expiration policy on ephemeral keys will eventually exhaust memory and start evicting or refusing writes under memory pressure, often unpredictably.

- **DON'T:** Store data in a key-value store as the sole system of record when that data has any requirement for durability guarantees, backup/restore, or complex querying beyond key lookup, unless the specific store is explicitly designed and configured for durable primary storage (e.g., DynamoDB, properly persisted Redis with AOF/RDB). Plain in-memory caches (default Memcached, Redis without persistence configured) can lose all data on a restart.

- **DO:** Use atomic operations the key-value store provides natively (`INCR`, `HINCRBY`, conditional `SET ... NX`, compare-and-swap) for counters and simple concurrent updates, instead of a read-modify-write cycle done in application code, which is race-prone under concurrent access.
```pseudocode
// BAD: race condition — two concurrent requests can both read 5, both write 6
count = redis.get("page_views:123")
redis.set("page_views:123", count + 1)

// GOOD: atomic increment, safe under any concurrency
redis.incr("page_views:123")
```

- **DON'T:** Use unbounded key patterns (e.g., a key per user per day per event type, generated forever with no cleanup path) without a retention/eviction strategy. Key-value stores that hold everything in memory scale with the total live key count, not with query complexity — an ever-growing key space is a slow-motion outage.

### Wide-Column Stores

- **DO:** Design wide-column store (Cassandra, HBase, Bigtable, ScyllaDB) tables around the specific queries they need to serve, choosing the partition key to distribute writes evenly across the cluster and the clustering columns to match the sort order the query needs — often meaning the same logical entity is stored in several differently-keyed tables, one per query pattern ("query-first design"). Unlike a relational database, there is no cheap ad hoc join or secondary aggregation to fall back on later; the access pattern must be known at design time.
```sql
-- Cassandra: a table specifically shaped to serve "get a user's recent
-- orders, newest first" — the partition key distributes writes by user,
-- the clustering column gives sorted retrieval for free
CREATE TABLE orders_by_customer (
  customer_id UUID,
  order_id TIMEUUID,
  status TEXT,
  total DECIMAL,
  PRIMARY KEY (customer_id, order_id)
) WITH CLUSTERING ORDER BY (order_id DESC);
```

- **DON'T:** Choose a partition key with low cardinality or with a "hot" value that a disproportionate share of traffic hits (e.g., partitioning by `status` when 90% of rows are `'active'`). A poorly chosen partition key creates a hot partition that receives far more reads/writes than the rest of the cluster, which defeats the horizontal scalability the store exists to provide and can overload individual nodes.

- **DO:** Denormalize aggressively and duplicate data across multiple tables, each shaped for one specific query, when using a wide-column store. This is the expected, idiomatic design pattern for these systems (sometimes summarized as "one table per query"), not a compromise — unlike a relational schema, minimizing duplication is not the design goal here.

- **DON'T:** Expect ACID transactions, secondary-index performance, or ad hoc multi-attribute filtering from a wide-column store the way you would from a relational database. Most wide-column stores offer only limited transactional guarantees (often single-partition only) and treat secondary indexes as a narrow, sometimes discouraged feature rather than a general-purpose tool — plan queries around the primary key design instead of leaning on secondary indexes as a crutch.

- **DO:** Understand the specific consistency model and tunable consistency levels of the wide-column store in use (e.g., Cassandra's per-query consistency level from `ONE` to `ALL`, or `QUORUM`) and choose them deliberately per operation based on the actual consistency-vs-availability tradeoff that operation needs, rather than accepting a store-wide default without understanding what it implies.

### When NoSQL Is (and Isn't) the Right Choice

- **DO:** Choose a relational database as the default for data with well-defined structure, relationships that need to be queried in varied and evolving ways, and a need for strong transactional/consistency guarantees across multiple related pieces of data (financial records, inventory, anything involving money or scarce resources). A relational database's flexible ad hoc querying and strong consistency guarantees are hard to fully replicate in most NoSQL stores.

- **DO:** Choose a NoSQL store when the actual, driving requirement matches what that category is built for — massive horizontal write scale with a simple, known access pattern (wide-column), sub-millisecond key lookups (key-value), deeply nested/variable document shapes accessed as a unit (document), highly connected graph traversal queries (graph), or full-text/relevance search (search engines like Elasticsearch/OpenSearch). Match the tool to the actual shape of the problem, not to its popularity.

- **DON'T:** Choose a NoSQL database because it is perceived as more modern, more scalable, or more "webscale" than a relational database, without a specific requirement that a relational database and its ecosystem (read replicas, partitioning, caching) genuinely cannot satisfy. The large majority of applications, even ones with significant traffic, run comfortably on a well-tuned relational database; NoSQL trades away general-purpose querying and (in many configurations) strong consistency for scale characteristics most applications never actually need.

- **DON'T:** Pick a document database and then immediately need complex multi-document transactions, ad hoc joins across collections, and strict schema enforcement everywhere — those needs are a strong signal the actual problem is relational, and forcing it into a document model will fight the tool at every step rather than benefit from it.

- **DO:** Consider polyglot persistence — using a relational database as the system of record for core transactional data while using a specialized store (Elasticsearch for search, Redis for caching/sessions, a time-series database for metrics) for the specific access pattern each one is genuinely best at — rather than trying to force one database technology to serve every access pattern in the system equally well.

- **DON'T:** Add a second database technology to the stack for a narrow use case without weighing the very real operational cost — a new system to back up, monitor, secure, patch, and staff expertise for. A specialized store is worth the added operational surface only when the access pattern it solves is both real and not reasonably solvable within the primary database.

- **DO:** Revisit "NoSQL for scale" decisions made early in a project's life once real production data proves out (or disproves) the original scaling assumption. Plenty of systems are built on a NoSQL store anticipating a scale they never reach, paying an ongoing complexity cost (weaker consistency, harder ad hoc querying, more application-side logic to compensate) for a scale requirement that turned out to be smaller than assumed.

- **DO:** Evaluate NewSQL / distributed SQL systems (CockroachDB, YugabyteDB, Google Spanner, Aurora/PlanetScale-style managed offerings) when the actual requirement is both relational semantics (joins, strong consistency, familiar SQL) and horizontal write scale beyond what a single-primary relational database can offer — this category exists specifically to avoid the false choice between "relational with strong guarantees" and "NoSQL with horizontal scale."

### Graph Databases

- **DO:** Choose a graph database (Neo4j, Amazon Neptune, ArangoDB's graph engine) when the actual workload is dominated by variable-depth relationship traversal — "friends of friends within 3 hops," recommendation paths, fraud-ring detection through chains of shared attributes — where the number of joins needed in a relational model would be unknown ahead of time or grow with the depth of the query. Graph databases index relationships as first-class citizens, so traversal cost stays roughly proportional to the size of the traversed subgraph rather than the size of the whole dataset.
```text
// Cypher (Neo4j): find friends-of-friends within 2 hops, a query whose
// relational equivalent would need a variable, unknown number of self-joins
MATCH (me:Person {id: $id})-[:FRIENDS_WITH*1..2]-(fof:Person)
WHERE fof <> me
RETURN DISTINCT fof.name
```

- **DON'T:** Reach for a graph database for relationships that are simple, fixed-depth, and already well served by a handful of `JOIN`s in a relational schema (a one-to-many "author has books" relationship does not need a graph database). Graph databases earn their complexity specifically for variable-depth or highly interconnected traversal patterns, not for ordinary foreign-key relationships.

- **DO:** Model both nodes and relationships with their own meaningful properties in a graph schema (a `FRIENDS_WITH` relationship can itself carry a `since` date, a `WORKS_AT` relationship can carry a `role` and `start_date`) rather than only modeling entities as nodes and losing the relationship-level detail relational foreign keys can't easily express either.

- **DON'T:** Try to force a graph database to also serve as the primary system of record for tabular, non-relationship-heavy business data (financial ledgers, inventory counts) just because it's already in the stack. Graph databases are usually a complement to a primary relational or document store for the specific relationship-heavy subset of the data, not a full replacement for it.

### Search Engines & Vector Stores

- **DO:** Use a dedicated search engine (Elasticsearch, OpenSearch, Meilisearch, Algolia) when the requirement genuinely needs relevance-ranked full-text search across large, unstructured or semi-structured text at scale, faceted filtering combined with free-text search, or typo-tolerant/fuzzy matching — capabilities well beyond what a relational database's native full-text search index typically offers at scale.

- **DON'T:** Stand up a separate search engine as the primary source of truth for data that also needs strong consistency and transactional guarantees. Search engines are typically eventually consistent (indexing happens asynchronously after a write to the primary store) and are not designed as a transactional system of record — keep the relational/primary database as the source of truth and treat the search index as a derived, rebuildable copy.

- **DO:** Design an explicit, monitored synchronization pipeline (CDC, an outbox pattern, or a scheduled reindex) to keep a search index in sync with its source-of-truth database, and monitor indexing lag as its own metric. A search index that silently falls behind its source data returns results that don't match what a direct database query would show, which surfaces to users as "I just added/edited this and it's not showing up in search."

- **DO:** Choose a vector database or a vector-search extension (pgvector for PostgreSQL, a dedicated vector store like Pinecone/Weaviate/Milvus, or a vector index within an existing search engine) when the requirement is similarity search over embeddings — semantic search, recommendation-by-similarity, retrieval-augmented generation — since exact-match indexes (B-tree, hash) cannot answer "find the N nearest vectors" queries efficiently at all.
```sql
-- pgvector: nearest-neighbor search over embeddings directly in PostgreSQL,
-- avoiding a separate vector database when scale doesn't require one
CREATE EXTENSION IF NOT EXISTS vector;
ALTER TABLE documents ADD COLUMN embedding vector(1536);
CREATE INDEX idx_documents_embedding ON documents USING hnsw (embedding vector_cosine_ops);
SELECT content FROM documents ORDER BY embedding <=> :query_embedding LIMIT 5;
```

- **DON'T:** Assume every application doing any kind of "semantic search" needs a dedicated, separately-operated vector database from day one. An extension like pgvector added to an already-running relational database is often sufficient until scale or query-pattern requirements genuinely exceed what it can do — evaluate the added operational cost of a separate vector store against the actual scale before adopting one.

- **DO:** Understand the specific approximate-nearest-neighbor (ANN) index type in use (HNSW, IVFFlat, and others each trade off build time, query speed, and recall differently) and tune its parameters deliberately for the actual data size and required recall, rather than accepting default index parameters that may not suit the dataset's scale.

### Time-Series Databases

- **DO:** Choose a time-series-optimized database or extension (TimescaleDB, InfluxDB, Prometheus for metrics specifically) when the workload is dominated by high-volume, append-mostly, timestamped data queried primarily by time range and aggregated over time windows — application metrics, IoT sensor readings, financial tick data. These systems are built around exactly that access pattern, with automatic time-based partitioning, efficient time-range compression, and downsampling/retention features a general-purpose relational database has to be manually configured to approximate.

- **DON'T:** Store high-cardinality, high-frequency time-series data in a general-purpose relational table with no partitioning strategy and expect it to remain performant as it grows. A metrics table accumulating millions of rows per day with no time-based partitioning quickly becomes an indexing, storage, and query-performance problem that a purpose-built time-series store (or a well-partitioned relational table) handles far more gracefully.

- **DO:** Define an explicit data retention and downsampling policy for time-series data from the start — raw high-resolution data kept for a short window, progressively downsampled/aggregated data kept longer — rather than accumulating every raw data point indefinitely. Time-series data volume grows continuously and predictably; a retention policy decided upfront avoids an emergency cleanup once storage costs or query performance become a problem.
```sql
-- TimescaleDB: automatically compress and drop old raw data on a schedule
SELECT add_compression_policy('sensor_readings', INTERVAL '7 days');
SELECT add_retention_policy('sensor_readings', INTERVAL '90 days');
```

- **DON'T:** Query a time-series table without a bounded time range in the `WHERE` clause. An unbounded time-series query is exactly the unbounded-query problem described earlier, made worse by the fact that time-series tables are specifically expected to grow very large very fast — always scope by a reasonable time window, and page/downsample beyond that.

### Consistency Models in Distributed NoSQL Stores

- **DO:** Learn and choose the specific consistency model a distributed NoSQL store actually offers per operation — strong/linearizable reads, eventual consistency, or a tunable middle ground (quorum reads/writes) — and pick deliberately per use case, since many distributed NoSQL stores default to eventual consistency for performance and availability reasons that a relational-database background doesn't prepare you to expect.

- **DON'T:** Assume a read immediately after a write on a distributed NoSQL store will see that write, unless the specific store and configuration guarantee it (a "read-your-writes" or strongly consistent read mode). Under eventual consistency, a read routed to a different replica than the write can return stale data for a window after the write — this is a documented, expected property of the system, not a bug, and application logic that assumes otherwise will intermittently misbehave.
```pseudocode
// Under eventual consistency, this is not guaranteed to see the write
// that just happened, if the read is served by a replica that hasn't
// caught up yet
write(key, newValue)
value = read(key)   // may return the OLD value briefly
```

- **DO:** Understand how the specific store resolves write conflicts when two writes to the same key happen concurrently on different nodes before an eventually-consistent system converges — last-write-wins (which silently discards one write), vector clocks, or CRDTs (conflict-free replicated data types) that merge concurrent writes deterministically. Which strategy is in play determines whether a concurrent-write scenario can silently lose data.

- **DON'T:** Treat "eventually consistent" as a vague, unquantified promise. Ask or verify what the actual convergence window looks like under normal and degraded conditions for the specific store and configuration in production — an eventual-consistency window of tens of milliseconds behaves very differently for application logic than one that can stretch to seconds under replica lag or network partition.

- **DO:** Use CRDTs or an application-level merge strategy deliberately for data that's genuinely expected to be written concurrently from multiple locations with eventual convergence (collaborative editing state, distributed counters, shopping cart merges across devices) rather than accepting whatever the store's default conflict resolution happens to do without understanding it.

### Fan-Out on Write vs Fan-Out on Read

- **DO:** Choose fan-out-on-write (pre-computing and storing a result for every consumer at write time — e.g., pushing a new post into every follower's pre-built timeline the moment it's published) when reads vastly outnumber writes and read latency matters most, accepting the write-time cost and storage duplication in exchange for near-instant reads. This is the standard pattern behind high-fan-out social feeds and notification systems built on document or wide-column stores.

- **DON'T:** Use fan-out-on-write for an entity with an extremely large number of downstream consumers per write (a celebrity account with tens of millions of followers) without a specific mitigation, since a single write would otherwise need to fan out into tens of millions of individual writes synchronously — the classic "celebrity problem." Handle high-fan-out writers with a hybrid approach (fan out to typical followers, compute the celebrity's content on read instead) rather than applying the same strategy uniformly regardless of fan-out size.

- **DO:** Choose fan-out-on-read (computing the combined result at query time, by fetching and merging from each source) when writes are frequent relative to reads, or when the number of potential consumers per write is unbounded or very large, accepting higher read latency in exchange for avoiding an expensive or unbounded write amplification.

- **DON'T:** Mix fan-out-on-write and fan-out-on-read inconsistently for the same logical data without a clear reason, since each strategy implies different staleness, latency, and storage tradeoffs — a system that's inconsistent about which pattern applies where becomes hard to reason about and debug when something looks stale or slow in only some cases.

- **DO:** Process fan-out-on-write asynchronously (via a queue/background worker) rather than synchronously within the original write's request/response cycle whenever the fan-out count is large enough to add meaningful latency, so the original write returns quickly and the fan-out happens in the background at its own pace.

### Secondary Indexes & Query Limitations in NoSQL

- **DO:** Understand the real cost and consistency model of secondary indexes in the specific NoSQL store before relying on them the way a relational secondary index is relied on — some distributed NoSQL stores maintain secondary indexes asynchronously (meaning a just-written row may not yet appear in a secondary-index-based query), and some limit how many secondary indexes a table can have or how they can be combined in a single query.

- **DON'T:** Assume a NoSQL secondary index supports the same flexible, arbitrary combination of filters a relational database's query planner handles automatically. Many NoSQL stores only efficiently support querying by one indexed attribute at a time (or a specific, limited combination declared upfront), forcing either multiple queries merged in application code or a redesigned primary key/partition strategy for genuinely multi-attribute filtering.

- **DO:** Design the primary key (partition key, and clustering/sort key where applicable) around the most critical, highest-frequency query pattern, and treat secondary indexes as a supplement for less-critical, lower-volume query patterns — not as the primary mechanism for the store's most important queries, which secondary indexes in many NoSQL systems aren't optimized to serve at the same performance level.

- **DON'T:** Add a secondary index to a NoSQL table without understanding its write-amplification cost. Just as in a relational database, every additional secondary index adds overhead to every write — often more so in distributed systems, where maintaining a global secondary index can itself require cross-partition coordination.

- **DO:** Consider maintaining a separate, purpose-built table (the query-first "one table per access pattern" design already discussed for wide-column stores) instead of a secondary index when a query pattern is both important and not well served by the primary key's natural ordering — this is often more predictable and performant than leaning on the store's built-in secondary indexing feature.

### Multi-Region NoSQL & Geo-Replication

- **DO:** Choose a multi-region replication topology (active-passive with a single writable region, or active-active with writes accepted in multiple regions) deliberately based on the actual latency and conflict-tolerance requirements, since active-active writability across regions reintroduces the same conflict-resolution questions covered under distributed consistency models, now across a wide-area network with meaningfully higher latency between regions than within one.

- **DON'T:** Enable multi-region active-active writes for data with strict uniqueness or sequential-ordering requirements (an auto-incrementing invoice number, a strict FIFO queue) without a specific conflict-resolution or partitioning strategy for those requirements. Two regions independently accepting writes for what's supposed to be a single global sequence will produce collisions or ordering violations that a single-writer system never has to think about.

- **DO:** Account explicitly for cross-region replication latency when reasoning about consistency guarantees — a write accepted in one region is not instantly visible to a read in another region, and the actual lag depends on real network distance and the specific replication mechanism, which can be tens to hundreds of milliseconds even in the best case, not zero.

- **DON'T:** Assume a "multi-region" or "globally distributed" label on a managed NoSQL service means every operation is automatically globally consistent with no configuration. Most of these services offer per-operation or per-table consistency tuning (region-local reads vs. globally consistent reads, for instance) precisely because global consistency for every operation carries a real latency cost — choose the tuning per use case rather than accepting whichever default the service ships with.

### Document Database Indexing

- **DO:** Index document database collections around the actual query patterns just as deliberately as relational indexing — a compound index ordered to match equality-then-range-then-sort filters, exactly the same left-to-right prefix logic that applies to composite indexes in a relational database. The underlying B-tree mechanics (and therefore the column-order discipline) are the same even though the syntax and terminology differ by store.
```javascript
// MongoDB compound index: equality filter first, then range/sort
db.orders.createIndex({ customer_id: 1, status: 1, created_at: -1 });
// Serves: find({customer_id, status}).sort({created_at: -1})
```

- **DON'T:** Assume a document database automatically indexes every field or that queries on unindexed fields perform acceptably at scale, the same false assumption that causes trouble in relational databases. An unindexed filter on a large collection forces a full collection scan exactly as it would force a sequential scan in a relational table.

- **DO:** Index fields inside embedded arrays/sub-documents (multikey indexes) deliberately when queries filter on them, and understand the specific limitations that come with array indexing (many document stores restrict a compound index to at most one array field) rather than assuming array fields index exactly like scalar fields.

- **DON'T:** Over-index a document collection with a large, growing number of narrow indexes "to cover every possible query," which imposes exactly the same per-write index-maintenance cost discussed for relational over-indexing — the tradeoff and the discipline for managing it are the same regardless of which category of database is doing the indexing.

### Migrating Data Between NoSQL and Relational Models

- **DO:** Redesign the schema deliberately around the target model's actual strengths when moving data between a relational and a document/NoSQL representation, rather than mechanically translating tables to collections (or the reverse) one-to-one. A straight one-to-one translation of a normalized relational schema into a document store reproduces exactly the anti-pattern described earlier (a document schema that still needs a fan-out of lookups for one screen) — the migration is an opportunity to actually redesign around access patterns, not just a format conversion.

- **DON'T:** Migrate from a document store to a relational database (or the reverse) without first cataloguing every distinct document shape actually present in the source data. A document collection's "schemaless" flexibility often means, in practice, that documents have accumulated several inconsistent shapes over the application's history — discovering this only mid-migration, after committing to a single target relational schema, causes migration failures or silent data loss for the documents that don't fit the assumed shape.

- **DO:** Run a validation pass comparing record counts, key aggregates, and a sample of individual records between source and destination after a cross-model migration, the same reconciliation discipline used for any other data migration, since a model-translation migration has more opportunities for subtle logic errors (an embedded array flattened incorrectly, a reference resolved to the wrong parent) than a same-model migration does.

- **DON'T:** Assume a generic, automated schema-conversion tool fully handles the semantic differences between the two models without manual review. Automated conversion tools are good at mechanical structure translation but cannot infer the actual business meaning behind a document's nested structure or a table's normalization choices — always manually review the tool's output against the actual intended target design before trusting it with production data.

### Change Streams & Real-Time Triggers in Document Databases

- **DO:** Use the document database's native change stream feature (MongoDB Change Streams, Firestore listeners, DynamoDB Streams) to react to data changes in real time — triggering downstream processing, cache invalidation, or search index updates — instead of polling the collection for changes, for the same reasons CDC is preferred over polling in the relational world: change streams capture every change in order, including deletes, with much lower latency than a poll-based approach.
```javascript
// MongoDB change stream: react to inserts/updates/deletes as they happen,
// instead of polling the collection on an interval
const changeStream = db.collection("orders").watch();
changeStream.on("change", (event) => {
  if (event.operationType === "update") reindexSearchDocument(event.documentKey._id);
});
```

- **DON'T:** Assume a change stream connection is guaranteed to stay open forever with no gaps. Change stream connections can be interrupted (a network blip, a server restart, a resume token expiring past the store's retention window) — design the consumer to detect a gap and use the store's resume-token mechanism to pick back up from the correct point, or explicitly reconcile against the source after any interruption, rather than silently missing whatever changes happened during the gap.

- **DO:** Treat change-stream-driven side effects (search reindexing, cache invalidation, downstream notifications) with the same idempotency discipline as any other event-driven consumer, since most change stream implementations offer at-least-once delivery to consumers, not exactly-once.

- **DON'T:** Put heavy, slow processing directly in the change stream event handler in a way that causes the consumer to fall behind the stream's actual rate of change. A slow handler causes the consumer's position in the stream to lag further and further behind, increasing both the latency of downstream effects and the risk of falling outside the store's resume-token retention window entirely — offload heavy processing to a queue instead of doing it inline in the stream handler.

### Choosing Between Managed and Self-Hosted NoSQL

- **DO:** Default to a managed NoSQL offering (DynamoDB, Cosmos DB, MongoDB Atlas, a cloud provider's managed Cassandra/Redis) for most teams and most workloads, since operating a distributed database well — handling node failures, rebalancing, backups, patching, monitoring cluster health — is genuinely specialized operational work that a managed service absorbs, and getting it wrong on a self-hosted cluster is a common source of data-loss incidents for teams without dedicated database operations expertise.

- **DON'T:** Assume self-hosting a distributed NoSQL cluster is simply "install the software and go." Distributed systems have real operational subtleties — quorum configuration, network partition handling, rebalancing during scale events, backup consistency across nodes — that differ meaningfully from operating a single-node database, and getting them wrong tends to fail silently until an outage or a partition exposes the gap.

- **DO:** Choose self-hosting deliberately when there's a specific, concrete reason a managed offering can't meet — a compliance requirement mandating physical infrastructure control, a cost structure that only makes sense at very large scale with in-house expertise already available, or a need for a specific configuration/version a managed service doesn't offer — rather than defaulting to self-hosting out of a general preference for infrastructure control.

- **DON'T:** Underestimate the cost comparison between self-hosted and managed by only counting infrastructure spend and ignoring the engineering time cost of operating the cluster. The ongoing operational burden of a self-hosted distributed database (on-call load, upgrade projects, incident response) is a real, often underestimated cost that needs to be weighed against a managed service's higher sticker price.

### TTL & Automatic Expiration in NoSQL Stores

- **DO:** Use a NoSQL store's native TTL feature (MongoDB's TTL indexes, DynamoDB's TTL attribute) for data that should automatically expire and be removed from the primary dataset — session data, temporary tokens, event data with a defined retention window — rather than building a separate scheduled deletion job to do the same thing. Native TTL expiration is handled by the store's own background process, is far less error-prone than a custom cleanup job, and doesn't compete for application resources.
```javascript
// MongoDB: rows automatically removed some time after expiresAt passes
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

- **DON'T:** Assume a store's native TTL expiration happens at the exact instant the TTL elapses. Most implementations run a periodic background sweep (MongoDB's default TTL monitor runs roughly every 60 seconds, for instance; DynamoDB's TTL deletion can take up to 48 hours in some cases) — if precise-timing expiration matters for correctness (not just eventual cleanup), don't rely on native TTL alone; check the expiration condition explicitly in the read path as well.

- **DO:** Understand whether the store's native TTL deletion is itself a normal, indexed/tracked delete (consuming write capacity, generating change-stream events) or a special background operation with different cost characteristics, since this affects both cost planning (in a consumption-priced managed service) and whether downstream consumers relying on change streams will see TTL-driven deletions as events.

- **DON'T:** Rely on TTL as a substitute for actually deciding a real data retention policy. TTL is a mechanism for enforcing a retention decision, not a replacement for making that decision deliberately (per the data classification, compliance, and business-need considerations already discussed under PII handling) — set the TTL value to match an actual decided retention period, not an arbitrary convenient number.

### Denormalized Counters & Aggregate Maintenance

- **DO:** Use the store's native atomic increment/decrement operator (MongoDB's `$inc`, DynamoDB's atomic counter update expressions) to maintain a denormalized counter on a parent document (a post's `like_count`, a product's `review_count`) rather than reading the current value, computing a new value in application code, and writing it back. A read-modify-write cycle from application code is a race condition under concurrent updates in a document store exactly as it is in a relational one — the atomic operator is the store's built-in fix for precisely this.
```javascript
// Atomic, race-safe increment — no read-modify-write cycle
db.posts.updateOne({ _id: postId }, { $inc: { likeCount: 1 } });
```

- **DON'T:** Trust a denormalized counter to stay perfectly accurate forever with no reconciliation check, especially across a system with multiple write paths (a like added through the API, a like removed through an admin tool, a bulk data-fix script) that might not all go through the same atomic-increment code path consistently. Periodically recompute and reconcile denormalized counters against their actual source data (counting the real underlying records) and alert on drift, the same discipline used for any other denormalized value.

- **DO:** Consider whether a denormalized counter needs strict real-time accuracy or can tolerate periodic batch recomputation instead — a "like count" displayed to users can often tolerate being refreshed every few minutes via a batch job rather than maintained with a real-time atomic increment on every single write, which simplifies the write path considerably.

- **DON'T:** Maintain the same denormalized counter through multiple independent, inconsistent update paths (one code path using an atomic increment, another recomputing and overwriting the whole value) within the same system. Mixing update strategies for the same counter is a reliable way to reintroduce the exact race condition the atomic operator was meant to eliminate.

### Document Size Limits & Splitting Large Documents

- **DO:** Know the specific document/item size limit of the store in use (16MB per document in MongoDB, 400KB per item in DynamoDB, for example) and design embedded structures with headroom well under that limit, since a document that grows past the limit fails to write outright — a hard, unrecoverable-at-write-time failure, not a graceful degradation.

- **DON'T:** Embed an unbounded or fast-growing sub-collection inside a document without a plan for what happens as it approaches the size limit. An embedded array of comments, log entries, or history events that grows without bound will eventually hit the document size ceiling — well before that point, split it into a separately referenced collection instead of embedding, per the embedding-vs-referencing tradeoffs already discussed.

- **DO:** Monitor document sizes in production for collections with embedded, growable substructures, treating an approaching size limit as an actionable warning to migrate that data out to a separate collection before it becomes a hard failure in the write path.

- **DON'T:** Assume splitting a large document into several smaller linked documents is a purely mechanical, risk-free fix applied at the last minute under write-failure pressure. Restructuring an embedded relationship into a referenced one changes the query patterns needed to read the data — plan and test the restructuring properly rather than improvising it in response to a production incident.

### Multi-Document ACID Transactions in Modern Document Databases

- **DO:** Verify the actual, current transactional capability of the specific document database and version in use before assuming it either has or lacks multi-document ACID transactions — this genuinely varies and has changed over time (MongoDB, for instance, added multi-document ACID transaction support in version 4.0, which was not true of earlier versions), so an assumption formed from older general knowledge about "NoSQL has no transactions" can be outdated for a specific modern store.

- **DON'T:** Assume that because a document database supports multi-document transactions, using them liberally carries the same low relative cost as a single-node relational transaction. Multi-document (and especially multi-shard) transactions in a distributed document store typically carry a real performance cost specifically because they require cross-node coordination — reserve them for cases that genuinely need atomicity across documents, and prefer single-document atomic operations (which most document stores guarantee natively and cheaply) wherever a single document can be structured to make that sufficient.

- **DO:** Design the document schema, where possible, so that operations needing atomicity naturally fit within a single document (embedding related data specifically to take advantage of guaranteed single-document atomicity) rather than reaching for a multi-document transaction as the default tool for every operation that touches more than one piece of data.

- **DON'T:** Assume multi-document transaction support eliminates the need to think about the embedding-vs-referencing tradeoffs discussed earlier. Transactions solve the atomicity problem for operations that must span multiple documents; they don't remove the performance and complexity reasons to prefer a well-embedded document model for data that's naturally accessed together in the first place.

## Transactions & Consistency

### ACID Basics

- **DO:** Wrap any set of writes that must succeed or fail together as a unit in a single database transaction, relying on the database's atomicity guarantee rather than trying to hand-roll a "rollback" in application code. If a multi-step operation (debit one account, credit another) is not transactional, a crash or error between the two steps leaves the data permanently inconsistent with no automatic recovery.
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'a';
UPDATE accounts SET balance = balance + 100 WHERE id = 'b';
COMMIT;  -- both succeed or both are rolled back; never a partial application
```

- **DON'T:** Perform multiple related writes as separate, independent statements/round trips outside of a transaction, assuming "they'll basically always both succeed." The failure case — a crash, a network blip, an application error between the two calls — is exactly the case atomicity exists to protect against, and it is precisely the case that "basically always" ignores until it happens in production.

- **DO:** Understand what durability actually means for the specific database and configuration in use — whether a committed transaction is guaranteed to survive a crash immediately, or whether the configuration trades some durability for throughput (e.g., asynchronous replication, relaxed fsync settings, `synchronous_commit = off` in PostgreSQL). Know explicitly what data loss window, if any, a crash could expose given the current durability configuration, rather than assuming "committed" always means "unconditionally safe."

- **DON'T:** Assume consistency (in the ACID sense — every transaction leaves the database in a state that satisfies all defined constraints) is automatically guaranteed without actually defining the constraints (foreign keys, check constraints, uniqueness) that should hold. ACID's "C" enforces whatever invariants are declared; a schema with no constraints has nothing for the database to protect, no matter how transactional the writes are.

- **DO:** Treat isolation (the "I" in ACID) as a spectrum with real tradeoffs to choose deliberately per use case (see isolation levels below), not as a single guarantee that "transactions are isolated" without further thought. The specific isolation level chosen determines exactly which concurrency anomalies are possible.

- **DON'T:** Assume every data store advertising "ACID" provides the same guarantees at the same strength. Some NoSQL and distributed systems offer ACID only within a single document/partition/shard, or offer a weaker default isolation level than users assume — read the specific system's actual documented guarantees rather than treating "ACID" as a single, universal promise.

### Isolation Levels and Their Tradeoffs

- **DO:** Choose an explicit isolation level per transaction (or per connection) based on the actual concurrency risk of that specific operation, rather than accepting whatever the database's default happens to be without evaluating it. The right isolation level trades off correctness guarantees against concurrency/throughput, and that tradeoff differs by operation — a report query has very different needs than a funds transfer.

- **DO:** Understand the four standard isolation levels and which anomalies each one prevents: `READ UNCOMMITTED` (allows dirty reads), `READ COMMITTED` (prevents dirty reads, allows non-repeatable reads and phantom reads), `REPEATABLE READ` (prevents dirty and non-repeatable reads, may allow phantom reads depending on the database), and `SERIALIZABLE` (prevents all standard anomalies by behaving as if transactions ran one at a time).
```sql
-- Explicitly choosing the isolation level for a transaction that
-- needs a stable, consistent view for its whole duration
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 'a';
-- ... business logic based on that balance ...
UPDATE accounts SET balance = balance - 100 WHERE id = 'a';
COMMIT;
```

- **DON'T:** Assume `READ COMMITTED` (the default isolation level in PostgreSQL, Oracle, and SQL Server) protects against re-reading the same row twice within one transaction and getting different values. Under `READ COMMITTED`, each individual statement sees a fresh snapshot, so two `SELECT`s of the same row in the same transaction can return different results if another transaction committed a change in between — a non-repeatable read.

- **DO:** Use `SERIALIZABLE` isolation (or an equivalent optimistic-concurrency mechanism) for operations where correctness genuinely depends on the transaction behaving as if it ran alone against the database — inventory decrements that must never oversell, or any check-then-act sequence where a concurrent transaction changing the checked condition would produce a wrong result.

- **DON'T:** Default every transaction to `SERIALIZABLE` "to be safe" without considering the throughput cost. `SERIALIZABLE` isolation, especially implemented via optimistic concurrency control (serializable snapshot isolation), can cause a meaningful rate of transaction aborts/retries under contention — reserve it for the operations that actually need it, and use a lighter isolation level plus explicit locking or application logic for everything else.

- **DO:** Know the specific phantom-read and write-skew anomalies that a chosen isolation level does not protect against, and mitigate them explicitly (with a stricter isolation level, or an explicit `SELECT ... FOR UPDATE`) when the business logic is vulnerable to them. Write skew — where two transactions each read overlapping data, each make a decision based on what they read, and both commit changes that are individually valid but jointly violate an invariant — is a notoriously easy-to-miss bug even under `REPEATABLE READ`.
```pseudocode
// Write skew example: an on-call rule requires at least one doctor on duty.
// Two doctors, both currently on duty, each independently check
// "is at least one OTHER doctor on duty?" under REPEATABLE READ,
// both see "yes", and both go off duty in the same window — now
// zero doctors are on call, even though each individual transaction
// looked correct in isolation.
```

- **DON'T:** Mix isolation levels inconsistently across the different code paths that touch the same critical data, assuming they'll compose safely. A funds-transfer path running at `SERIALIZABLE` next to a balance-adjustment path running at `READ COMMITTED` on the same table gives the weaker path's anomalies a way into data the stronger path was trying to protect.

### Long-Running Transactions

- **DO:** Keep transactions as short as possible — open the transaction right before the writes it needs to protect, and commit or roll back as soon as those writes are done. A long-running transaction holds its locks (and, in MVCC databases, prevents old row versions from being vacuumed/garbage-collected) for its entire duration, which can block other transactions and cause table/index bloat.

- **DON'T:** Perform slow, non-database work (calling an external API, sending an email, waiting on user input, doing heavy in-application computation) while a database transaction is open. Any lock the transaction is holding stays held for the entire duration of that slow external call, turning an unrelated network hiccup into a database-wide contention problem.
```pseudocode
// BAD: external API call happens while the transaction (and its locks) is open
BEGIN
  UPDATE orders SET status = 'processing' WHERE id = ?
  callPaymentGateway()   // could take seconds; locks held the whole time
  UPDATE orders SET status = 'paid' WHERE id = ?
COMMIT

// GOOD: do the slow external call outside the transaction, then commit quickly
callPaymentGateway()     // happens first, no locks held
BEGIN
  UPDATE orders SET status = 'paid' WHERE id = ?
COMMIT
```

- **DO:** Use a background job with its own short transactions, plus a durable record of intermediate state, for any workflow that spans slow external steps — rather than trying to hold one long database transaction open across the entire multi-step process.

- **DON'T:** Leave a transaction open indefinitely because application code hit an unhandled exception, forgot to call commit/rollback, or is stuck waiting on something that will never resolve inside a transaction context manager that doesn't guarantee cleanup. An orphaned open transaction (visible in `pg_stat_activity`'s "idle in transaction" state, for example) can hold locks and block vacuum/cleanup for as long as the connection stays open — always use the language's structured resource-cleanup construct so a transaction is guaranteed to close on every code path, including exceptions.

- **DO:** Monitor for long-running and idle-in-transaction sessions in production and alert on them, since they are a common, often invisible cause of mysterious lock contention and table bloat that has no other obvious symptom until it becomes an incident.
```sql
-- PostgreSQL: find transactions that have been open a suspiciously long time
SELECT pid, state, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle' AND xact_start < now() - interval '5 minutes';
```

- **DON'T:** Batch a huge amount of work (e.g., updating millions of rows) inside a single transaction just because it is logically "one operation." A single enormous transaction holds locks and accumulates undo/WAL data proportional to its size for its entire duration — chunk large logical operations into a series of smaller transactions, accepting that the whole operation is no longer atomic as a unit, and design for that (idempotency, resumability) instead.

### Optimistic vs Pessimistic Locking

- **DO:** Use pessimistic locking (`SELECT ... FOR UPDATE`, or an equivalent explicit row lock) when contention on the same rows is frequent and a conflict is expensive or awkward to retry — for example, decrementing a scarce inventory count where a lost update would oversell a limited item. Pessimistic locking blocks other transactions from touching the locked row until the current one finishes, guaranteeing no concurrent modification slips through.
```sql
BEGIN;
SELECT quantity FROM inventory WHERE sku = 'WIDGET-1' FOR UPDATE;  -- locks the row
-- application checks quantity > 0, then:
UPDATE inventory SET quantity = quantity - 1 WHERE sku = 'WIDGET-1';
COMMIT;
```

- **DON'T:** Use pessimistic locking as the default strategy for low-contention data (rows that are almost never touched concurrently by two transactions at once). Pessimistic locks that are held but rarely actually contested add latency and reduce throughput for no real benefit; optimistic concurrency control is usually the better default when conflicts are the exception rather than the rule.

- **DO:** Use optimistic locking (a version/row-revision column checked and incremented on update, failing the update if the version has changed since it was read) for the common case of low-to-moderate contention, since it avoids holding locks at all during the "think time" between reading data and writing an update.
```sql
-- Optimistic locking via a version column
UPDATE products SET price = 29.99, version = version + 1
WHERE id = 42 AND version = 7;   -- fails to match (0 rows updated) if someone
                                   -- else updated the row since it was read
-- application checks affected row count; on 0, reload and retry or surface a conflict
```

- **DON'T:** Implement optimistic locking without a clear, deliberate conflict-resolution path for the caller. An optimistic update that silently no-ops when the version doesn't match, with nothing checking the affected-row-count and surfacing the conflict, produces a confusing bug where user changes appear to save successfully but silently didn't apply.

- **DO:** Choose pessimistic locking for short, well-bounded critical sections where the lock will be held only briefly, and choose optimistic locking for longer "think time" scenarios (a user editing a form for minutes before submitting) where holding a database lock the whole time would be far too costly.

- **DON'T:** Combine both locking strategies inconsistently on the same data — some code paths taking `SELECT ... FOR UPDATE` and others relying on a version check — since a pessimistic lock does nothing to protect against a concurrent optimistic-style update that never took the lock in the first place. Pick one strategy per data/operation and apply it consistently across every code path that writes it.

- **DO:** Set a lock timeout when using pessimistic locking so a transaction waiting on a contested lock fails fast with a clear error rather than hanging indefinitely, especially in interactive/user-facing code paths where an indefinite hang is a worse user experience than a clear "please retry" error.

### Idempotency for Retries

- **DO:** Design any operation that a client or infrastructure might retry (network timeout, a queue redelivering a message, an at-least-once delivery guarantee) to be idempotent — running it twice with the same input produces the same end state as running it once. Retries are a fact of life in distributed systems; treating "exactly once" as achievable without idempotency is a reliable source of duplicate charges, duplicate emails, and double-counted records.

- **DON'T:** Implement a "process payment" or "send notification" endpoint as a bare `INSERT`/side-effect with no duplicate-request detection. A network timeout after the server successfully processed the request but before the client received the response leads the client to retry — with no idempotency guard, that retry charges the customer twice.
```pseudocode
// BAD: retried request creates a second charge
function chargeCustomer(customerId, amount):
    charge = paymentGateway.charge(customerId, amount)
    db.insert("charges", charge)

// GOOD: an idempotency key makes a retried request a safe no-op
function chargeCustomer(customerId, amount, idempotencyKey):
    existing = db.query("SELECT * FROM charges WHERE idempotency_key = ?", idempotencyKey)
    if existing: return existing   // already processed; return the same result
    charge = paymentGateway.charge(customerId, amount)
    db.insert("charges", { ...charge, idempotency_key: idempotencyKey })
    return charge
```

- **DO:** Enforce idempotency with a database-level uniqueness constraint on the idempotency key, not merely an application-layer check-then-insert, for the same race-condition reason a uniqueness business rule always needs a database constraint — two concurrent retries of the same request can both pass an application-level check before either has inserted.
```sql
ALTER TABLE charges ADD CONSTRAINT uq_charges_idempotency_key UNIQUE (idempotency_key);
```

- **DON'T:** Assume `INSERT`-only operations are automatically idempotent just because they don't modify existing rows. Retrying an unprotected `INSERT` creates a second row with the same logical content — idempotency has to be actively designed in (a uniqueness constraint on a natural or client-supplied key, an upsert keyed on that identity) rather than assumed as a side effect of the operation type.

- **DO:** Make `UPDATE` operations idempotent by expressing them as absolute state changes or conditioned on the current state, rather than as relative deltas that compound if replayed. `SET status = 'shipped'` run twice leaves the same end state; `SET quantity = quantity - 1` run twice (due to a retry) silently decrements twice.

- **DON'T:** Rely on message queue "exactly-once delivery" claims as a substitute for idempotent consumers. Most real-world messaging systems provide at-least-once delivery in practice (or exactly-once only within narrow, specific conditions) — consumer-side idempotency is the actual mechanism that makes duplicate delivery safe, regardless of what delivery guarantee the queue advertises.

- **DO:** Include a client-generated idempotency key in the request itself (a UUID the client generates once and resends unchanged on every retry of the same logical operation) for any API where the client controls retry behavior, so the server can recognize "this is the same request being retried" versus "this is a genuinely new request."

### Distributed Transactions & Sagas

- **DO:** Use the saga pattern — a sequence of local transactions, each with a defined compensating action that undoes it — for a business operation that spans multiple databases or services, rather than attempting a single distributed transaction across systems that don't support one. Sagas trade strict atomicity for a defined, testable recovery path when one step fails partway through, which is the realistic option for most cross-service workflows.
```pseudocode
// Saga: book a trip across three independently-owned services
step1 = reserveFlight(booking)         // compensating action: cancelFlight
step2 = reserveHotel(booking)          // compensating action: cancelHotel
step3 = chargePayment(booking)         // compensating action: refundPayment

// If step3 fails, run compensations for the already-completed steps in reverse:
// cancelHotel(step2), cancelFlight(step1)
```

- **DON'T:** Assume two-phase commit (2PC) is a practical, low-cost way to get atomicity across multiple independent databases or services. 2PC requires every participant to be available and responsive during the commit window, holds locks across all participants for the duration, and a coordinator failure can leave participants blocked indefinitely — it doesn't scale well and is rarely the right tool for modern distributed/microservice architectures, which is why the saga pattern is the more common practical choice.

- **DO:** Make every step of a saga idempotent and design explicit compensating actions for every step that has a side effect, since a saga's failure-recovery correctness depends entirely on those compensations actually undoing what their forward step did — an untested or partially-correct compensating action leaves the saga's failure path just as broken as having no plan at all.

- **DON'T:** Design a saga's compensating actions to always be exact mirror-image reversals without checking that reversal is actually possible for that specific side effect. Some actions (an email already sent, funds already transferred to a third party outside the system) cannot be cleanly undone — the compensation for those needs to be a different kind of remediation (a follow-up notification, a manual reconciliation flag) rather than a naive "undo," and that should be designed deliberately, not discovered during an incident.

- **DO:** Use the outbox pattern (writing the event/message to an "outbox" table in the same local transaction as the business data change, then relaying it to a message broker via a separate process) when a step needs to both commit a database change and reliably publish an event about it, since publishing to a message broker cannot itself be part of the same database transaction. This avoids the classic dual-write problem where the database commit succeeds but the message publish fails (or vice versa), leaving the two systems inconsistent.
```sql
BEGIN;
UPDATE orders SET status = 'confirmed' WHERE id = :id;
INSERT INTO outbox_events (event_type, payload) VALUES ('order_confirmed', :payload);
COMMIT;
-- A separate relay process reads outbox_events and publishes to the message
-- broker, retrying until success — the DB write and the eventual publish
-- can never silently disagree, because the event only exists if the
-- transaction that created it actually committed.
```

### Deadlocks

- **DO:** Access tables and rows in a consistent, agreed-upon order across every code path that might lock more than one of them within a transaction, to avoid deadlocks caused by two transactions acquiring the same two locks in opposite order. If every transaction always locks `accounts` before `transactions`, for instance, two concurrent transactions can never deadlock on those two tables against each other.
```pseudocode
// BAD: inconsistent lock order between two code paths creates a deadlock risk
// Path A: locks account_1 then account_2
// Path B: locks account_2 then account_1
// If both run concurrently, each can end up waiting on the lock the other holds

// GOOD: always lock in a consistent order, e.g. by ascending primary key
ids = sort([account_1_id, account_2_id])
lockInOrder(ids)
```

- **DON'T:** Assume the database will always detect and resolve a deadlock cleanly with no consequence to the application. Deadlock detection does resolve the deadlock (by aborting one of the transactions), but the aborted transaction's work is lost and must be retried — application code that doesn't catch the specific deadlock error and retry the transaction will surface it to the user as an unhandled failure instead of a transparent retry.

- **DO:** Catch the database's specific deadlock error code and retry the whole transaction automatically (with a short backoff) for operations known to be subject to lock contention, since a deadlock is often a transient condition that succeeds cleanly on retry once the competing transaction has released its locks.
```pseudocode
function withDeadlockRetry(fn, maxRetries=3):
    for attempt in range(maxRetries):
        try:
            return runInTransaction(fn)
        except DeadlockError:
            if attempt == maxRetries - 1: raise
            sleep(backoff(attempt))
```

- **DON'T:** Hold a lock longer than necessary while waiting on something unrelated (user input, a slow external call, an unrelated computation) — the longer a lock is held, the larger the window in which another transaction can request a conflicting lock and create deadlock potential. This is the same discipline as keeping transactions short in general, applied specifically to deadlock risk.

- **DO:** Reduce lock scope and granularity where possible — locking specific rows (`SELECT ... FOR UPDATE` on exactly the rows needed) rather than an entire table, and keeping the set of rows locked within one transaction as small as the business logic allows — since deadlock probability increases with the number and breadth of locks any two concurrent transactions might contend over.

- **DON'T:** Ignore recurring deadlocks in production logs as "the database handled it, no harm done." A deadlock that recurs regularly under normal load is a signal of a real lock-ordering or transaction-scope problem that will get worse under higher concurrency — investigate and fix the root cause (lock ordering, transaction length, lock granularity) rather than relying on retry logic to paper over a systemic issue indefinitely.

### CAP Theorem & Distributed Consistency Tradeoffs

- **DO:** Understand the CAP theorem's actual claim precisely — under a network partition, a distributed system must choose between remaining fully consistent (every read sees the latest write) or remaining fully available (every request gets a response), and cannot guarantee both simultaneously — and use it to reason clearly about a specific distributed data store's documented tradeoff, rather than as a vague buzzword invoked to justify any architecture decision.

- **DON'T:** Treat CAP as a simple binary "CP or AP" label that fully describes a real system's behavior. In practice, most systems are consistent and available under normal operation and only have to make the CAP tradeoff during an actual partition, and many systems offer tunable consistency (adjustable per-operation, per-region) rather than a single fixed choice — describe the actual, specific behavior of the system in use rather than reducing it to a two-letter label.

- **DO:** Extend the reasoning beyond CAP to latency as well when it matters practically (sometimes framed as PACELC — during a Partition, choose Availability or Consistency; Else, even with no partition, choose Latency or Consistency), since even without a partition, a strongly consistent distributed system typically pays a latency cost (coordinating with a quorum or a primary) that an eventually consistent one avoids.

- **DON'T:** Assume a single-node relational database's ACID guarantees translate directly, unchanged, once that database is replicated, sharded, or otherwise distributed. Consistency guarantees that were "free" on a single node (immediate visibility of a committed write to every subsequent read) require deliberate design and often a real cost once multiple nodes are involved — verify what guarantee the specific distributed configuration (multi-region replicas, a distributed SQL engine) actually provides rather than assuming ACID travels unchanged.

- **DO:** Choose the consistency point deliberately per operation in systems that offer tunable consistency, rather than applying one blanket setting everywhere — a financial balance check needs strong consistency; a "likes" counter displayed on a post can tolerate eventual consistency without any real business impact.

### Read Replica Consistency & Replication Lag

- **DO:** Account explicitly for replication lag when reading from a read replica — a replica's data can be seconds (or, under load, much longer) behind the primary, and any code path that reads its own recent write needs to either read from the primary or tolerate seeing stale data. Treating a replica as "basically the same as the primary, just for load distribution" ignores this real and sometimes significant gap.
```pseudocode
// A user updates their profile, then the very next request reads it back
// from a lagging replica — without special handling, they can see their
// own change appear to have been silently reverted
updateProfile(userId, newData)     // writes to primary
profile = readProfile(userId)      // reads from a replica that hasn't caught up yet
// profile may still show the OLD data
```

- **DON'T:** Route read-after-write flows (a user submitting a form and immediately being shown the result, a payment confirmation page) to a read replica without a read-your-writes strategy — either reading that specific request's follow-up from the primary, using a "read from primary for N seconds after this user's write" rule, or checking replica lag and falling back to the primary when it's too far behind.

- **DO:** Monitor replication lag as a first-class metric with alerting, since lag that grows unexpectedly (from a heavy write burst, a replica falling behind due to its own load, or a network issue) silently increases how stale every replica-served read is, without any error or exception marking the degradation.

- **DON'T:** Assume every read can be safely routed to a replica just because the query is a `SELECT`. Reads that are part of a read-then-write business logic sequence (check inventory, then decide whether to allow a purchase) need consistency with the data the subsequent write will act on — reading stale inventory from a lagging replica can lead to a decision based on data the primary has already moved past.

- **DO:** Design the application's read-routing logic explicitly and centrally (a data-access layer that knows which queries are safe to send to a replica and which must go to the primary) rather than leaving that decision to be made ad hoc, inconsistently, at each call site.

### Idempotency Key Storage & Expiry

- **DO:** Store idempotency keys with enough context to return the original operation's actual result on a retry (not just a boolean "already processed" flag) — the response payload or a reference to it — so a retried request gets the same response the original request would have returned, which is what callers actually expect from an idempotent API.
```sql
CREATE TABLE idempotency_keys (
  key TEXT PRIMARY KEY,
  request_hash TEXT NOT NULL,   -- detect a reused key with a different payload
  response_body JSONB NOT NULL,
  status_code INT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- **DON'T:** Let an idempotency key be reused for a genuinely different request body/payload without detecting the mismatch. If the same idempotency key arrives with different request content than the first time it was used, that's a client bug or a key-collision risk — detect it (by storing and comparing a hash of the original request) and reject the mismatched retry explicitly rather than silently returning the first request's unrelated result.

- **DO:** Set a defined expiry/retention window for stored idempotency keys, long enough to cover realistic retry windows (client timeout and retry policies, including exponential backoff with several attempts) but not indefinite, since idempotency keys accumulate forever otherwise and are rarely useful past the window in which a client could plausibly still be retrying.

- **DON'T:** Handle idempotency key lookup and the operation it guards as two separate, non-atomic steps (check if the key exists, then separately perform the operation and record the key). Two concurrent requests with the same idempotency key can both pass the "does not exist yet" check before either has recorded it — use a database-level uniqueness constraint on the key as the actual enforcement mechanism, with the operation and key-recording happening in the same transaction.

### Cache Eviction Policies

- **DO:** Understand the eviction policy the caching layer uses when it runs out of memory (LRU — evict least recently used, LFU — evict least frequently used, or a TTL-driven expiry-only policy with no active eviction) and choose one that matches the actual access pattern — LRU generally suits typical web caching workloads well, while LFU can better protect consistently popular items from being evicted by a short burst of one-time accesses.

- **DON'T:** Let a cache grow unbounded with no eviction policy or memory limit configured, assuming "it'll be fine." An unconfigured cache that's allowed to consume unbounded memory will eventually exhaust available memory on its host and can crash the caching process — or, worse, get OOM-killed at the worst possible moment — configure an explicit `maxmemory` (or equivalent) and eviction policy deliberately.

- **DO:** Distinguish which of the cache's entries are safe to lose to eviction (typical cache-aside data, always re-derivable from the source of truth) from any that are not (data written with write-behind semantics, not yet flushed to durable storage) — and never let genuinely un-flushed, not-yet-durable data sit in a cache configured with an eviction policy that could discard it before it's persisted.

- **DON'T:** Assume a cache's eviction policy protects against a single oversized value or a small number of very large values dominating available memory. A handful of unusually large cached objects (an unbounded query result cached whole, for instance) can crowd out many smaller, more valuable cached entries regardless of the eviction algorithm — bound the size of what gets cached, not only the cache's total memory budget.

### Savepoints & Partial Rollback

- **DO:** Use savepoints within a larger transaction when part of a multi-step operation might fail in a recoverable way and the transaction as a whole should continue rather than aborting entirely — a batch import processing many independent records within one transaction, where a single bad record should be skipped, not abort every already-processed record in that same transaction.
```sql
BEGIN;
INSERT INTO orders (...) VALUES (...);
SAVEPOINT before_risky_step;
-- an operation that might fail, e.g. calling a trigger that could raise an error
UPDATE inventory SET quantity = quantity - 1 WHERE sku = 'X';
-- on failure, roll back only to the savepoint, not the whole transaction:
ROLLBACK TO SAVEPOINT before_risky_step;
COMMIT;  -- the orders insert from before the savepoint is preserved
```

- **DON'T:** Use savepoints as a substitute for properly scoping what actually needs to be atomic. A transaction sprinkled with many savepoints to selectively undo pieces of unrelated work is often a sign the operation should be broken into genuinely separate, smaller transactions instead — savepoints are for handling an expected, recoverable failure within one logically single operation, not for stitching together operations that don't actually need shared atomicity.

- **DO:** Understand that a savepoint does not release the locks acquired since it was set when rolling back to it — rolling back to a savepoint undoes the data changes made after that point, but locks taken during that span may still be held until the enclosing transaction actually commits or rolls back entirely, which matters for how long the transaction as a whole contends with others.

- **DON'T:** Assume every database supports savepoints identically or that ORMs expose them consistently — verify the specific database and ORM/driver's savepoint support (nested savepoints, naming rules, interaction with autocommit) before relying on them for anything beyond the simplest single-level use.

### Read Committed Snapshot & MVCC Mechanics

- **DO:** Understand, at least at a conceptual level, how the database's concurrency control actually works — most modern relational databases use multi-version concurrency control (MVCC), where a reader sees a consistent snapshot of data as of some point in time by reading older row versions, rather than being blocked by a concurrent writer. This is why, under MVCC, readers generally don't block writers and writers generally don't block readers — a very different mental model than pure lock-based concurrency control.

- **DON'T:** Assume every update immediately deletes the old row version, freeing its space right away. Under MVCC, an old row version has to stick around as long as any open transaction might still need to read it under its own snapshot — long-running transactions therefore directly cause old row versions to accumulate (table and index bloat) until they can finally be reclaimed, which circles back to why long-running transactions are specifically harmful.

- **DO:** Run and monitor the database's cleanup/garbage-collection process for old row versions (`VACUUM`/autovacuum in PostgreSQL, purge threads in MySQL/InnoDB) as a first-class operational concern, not an invisible background detail. A database falling behind on this cleanup accumulates bloat that degrades both storage efficiency and query performance over time, and in PostgreSQL specifically, an autovacuum that falls far enough behind risks transaction ID wraparound, a serious operational problem.

- **DON'T:** Be surprised that a `SELECT COUNT(*)` is expensive on an MVCC database with many concurrent transactions touching the table. Because different transactions can see different row versions as "current" under MVCC, a database like PostgreSQL generally cannot maintain a simple, always-accurate row count the way some other systems can — an exact count typically requires actually scanning (at least an index) rather than reading a cached number, which surprises people coming from a database or mental model where row counts are free.

### Transaction Retry Storms & Backoff

- **DO:** Add randomized backoff (jitter) to any automatic transaction retry logic, rather than retrying immediately or on a fixed delay. When a spike of concurrent transactions all fail together (a deadlock affecting several transactions, a brief lock contention spike) and all retry on the same fixed delay, they collide again at the same moment — randomized jitter spreads the retries out so they don't repeatedly pile onto each other.
```pseudocode
function retryWithJitter(fn, maxAttempts=5):
    for attempt in range(maxAttempts):
        try:
            return fn()
        except (DeadlockError, SerializationFailure):
            if attempt == maxAttempts - 1: raise
            delay = min(baseDelay * 2**attempt, maxDelay) * random(0.5, 1.5)
            sleep(delay)
```

- **DON'T:** Retry a failed transaction indefinitely with no maximum attempt count or escalation path. An operation that keeps failing (due to a genuine, non-transient conflict, not a passing contention spike) and keeps retrying forever consumes resources without ever succeeding or surfacing the problem to anyone who could fix it — cap retries and surface a clear failure once the cap is reached.

- **DO:** Distinguish between errors that are safe and sensible to retry automatically (a transient deadlock, a serialization failure under optimistic concurrency, a connection blip) and errors that are not (a constraint violation reflecting genuinely invalid data, a business-rule rejection) — blindly retrying every database error treats a permanent, non-transient failure as if it might eventually succeed, which it never will.

- **DON'T:** Let retry logic at multiple layers of the stack (the ORM, the connection pool, application-level retry code, an API gateway) all retry the same failed operation independently, multiplying the effective retry rate far beyond what was intended at any single layer. Coordinate retry policy explicitly — usually at exactly one layer — rather than letting it happen implicitly and cumulatively across several.

### Testing Concurrency & Race Conditions

- **DO:** Write tests that deliberately trigger concurrent access to the same data — spawning multiple simultaneous requests/transactions against the same row or resource — for any code path where a race condition would have real consequences (inventory decrements, unique-constraint-guarded signups, balance transfers). A single-threaded test suite that never exercises true concurrency will never catch a race condition, no matter how thorough its sequential test coverage is.
```pseudocode
test("concurrent purchases never oversell limited inventory"):
    setInventory(sku: "LIMITED-1", quantity: 1)
    results = runConcurrently([
        () => purchase("LIMITED-1"),
        () => purchase("LIMITED-1"),
    ])
    successes = results.filter(r => r.success)
    assert successes.length == 1   // exactly one should succeed, not both
```

- **DON'T:** Rely solely on code review or manual reasoning to verify that a piece of concurrent-access logic is race-free. Race conditions are notoriously difficult to reason about correctly by inspection alone, precisely because they only manifest under specific timing interleavings that are easy to overlook when reading code sequentially — an actual concurrent test that exercises real parallelism is a much stronger check.

- **DO:** Use the database's own tools for artificially inducing contention during testing where available (holding a lock deliberately in one test connection while another attempts a conflicting operation) to verify the expected blocking, timeout, or retry behavior actually happens as designed, rather than only testing the happy path where no contention ever occurs.

- **DON'T:** Treat a race-condition test that passes once as sufficient proof of correctness. Race conditions are often probabilistic — a test that happens to pass on one run because of favorable scheduling can still be exercising a genuinely racy code path; run concurrency tests multiple times (or with intentionally exaggerated delays inserted at the critical section) to increase confidence that the fix is a real fix and not a lucky timing coincidence.

### Exactly-Once Processing Illusions

- **DO:** Understand "exactly-once processing" as, in almost every real distributed system, actually meaning "at-least-once delivery combined with idempotent processing that makes repeated delivery indistinguishable from exactly-once from the consumer's point of view" — not a literal guarantee that a message physically arrives exactly one time. Systems that market themselves as offering exactly-once semantics are almost always doing exactly this under the hood, and understanding that is what lets you reason correctly about their actual failure modes.

- **DON'T:** Skip idempotent consumer design because a messaging system or stream processing framework advertises "exactly-once semantics." Even frameworks with strong exactly-once guarantees typically scope that guarantee narrowly (exactly-once *within* the framework's own state management, for instance) and can still redeliver a message to application code outside that scope under specific failure and recovery scenarios — verify precisely what the guarantee covers rather than trusting the label alone.

- **DO:** Treat true exactly-once delivery (not effectively-once via idempotency) as effectively unachievable across an arbitrary network boundary in the general case, a consequence of the fundamental difficulty of guaranteeing both delivery and non-duplication when the network itself can drop, delay, or duplicate messages and the sender can't always know for certain whether the receiver actually processed a message before a failure.

- **DON'T:** Treat this as a purely theoretical concern that doesn't apply to a specific system in practice. It shows up constantly in ordinary operational events — a consumer crashing after processing a message but before acknowledging it, a network partition causing a producer to retry a message the consumer already received, a load balancer routing a retried request to a different instance than the one that actually completed the original request — all of which are routine, not exotic.

### Ledger Design for Financial Correctness

- **DO:** Model financial state as an append-only ledger of immutable transactions (each row records a movement of value, never updated or deleted) with the current balance always derived by summing the ledger, rather than storing only a mutable "current balance" column that gets directly incremented and decremented. An append-only ledger gives a complete, auditable history of exactly how a balance reached its current value, and makes it structurally impossible for a bug to silently corrupt history the way an in-place balance update can.
```sql
CREATE TABLE ledger_entries (
  id BIGINT PRIMARY KEY,
  account_id BIGINT NOT NULL,
  amount NUMERIC(14,2) NOT NULL,   -- positive for credit, negative for debit
  reference TEXT NOT NULL,          -- what this entry is for
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Current balance is always derivable, never independently stored-and-drifted
SELECT SUM(amount) FROM ledger_entries WHERE account_id = :id;
```

- **DON'T:** Rely solely on a mutable `balance` column updated via `balance = balance + amount` as the only record of a financial account's state, with no accompanying transaction history. If that balance is ever wrong — from a bug, a race condition, or a disputed transaction — there's no way to reconstruct how it got that way or to verify it independently; every financial discrepancy investigation starts from nothing.

- **DO:** Use double-entry bookkeeping (every transaction recorded as at least two balanced ledger entries — a debit from one account and a matching credit to another, always summing to zero across the pair) for any system tracking movement of value between parties, since this structurally enforces that value is conserved (never created or destroyed by a bug) and makes an out-of-balance condition immediately, mechanically detectable.

- **DON'T:** Treat a cached/materialized current-balance column as anything other than a derived, recomputable performance optimization over the ledger. It's reasonable to maintain a cached balance for fast reads, but it must always be reconcilable against (and, on any doubt, recomputable from) the ledger — never treat the cached balance as the actual source of truth the ledger is checked against, which inverts which one is authoritative.

- **DO:** Make every ledger entry idempotent via a unique reference to the business event that caused it (an order ID, a payment gateway's transaction ID), so that a retried operation can never double-post the same financial event — this is the same idempotency discipline discussed earlier, applied specifically to the place where duplicate application is most costly.

### Nested Transactions & Transaction Propagation in Frameworks

- **DO:** Understand exactly what the application framework or ORM does when a transactional method is called from inside another already-open transaction (Spring's `@Transactional` propagation settings, an ORM's nested `transaction do ... end` blocks), since frameworks differ meaningfully here — some create a real nested transaction using savepoints, some simply join the existing outer transaction and share its commit/rollback fate, and some default to a behavior that surprises developers coming from a different framework's convention.

- **DON'T:** Assume an inner "transaction" block automatically means an independent unit that can commit or roll back on its own, unaffected by the outer transaction. In many frameworks, a nested transactional call by default simply participates in the already-open outer transaction — code that expects the inner block's rollback to be isolated from the outer one (expecting the outer transaction's other work to survive) can be surprised when the whole outer transaction rolls back instead.

- **DO:** Use the framework's explicit propagation/nesting configuration (`REQUIRES_NEW` in Spring, an explicit savepoint-backed nested transaction where the ORM supports it) when a specific inner operation genuinely needs to commit or roll back independently of its caller's transaction, rather than assuming the default propagation behavior provides that isolation.

- **DON'T:** Nest transactional calls many levels deep across service boundaries without a clear mental model of where the transaction actually begins and ends. Deeply nested transactional calls, especially across different classes/services each independently marked "transactional," make it easy to lose track of the actual commit boundary and end up with a transaction far larger (and longer-held) than anyone intended.

### Ordering Side Effects Relative to Transaction Commit

- **DO:** Trigger external, non-transactional side effects (sending an email, calling a webhook, publishing to a message queue outside an outbox pattern) only after the database transaction that depends on them has actually committed successfully, never from inside the still-open transaction. An action taken from inside a transaction that later rolls back has already happened in the outside world and cannot be undone — the customer already received the "your order shipped" email for an order that, from the database's final point of view, was never actually created.
```pseudocode
// BAD: the email might send even if the transaction later rolls back
BEGIN
  createOrder(orderData)
  sendConfirmationEmail(orderData)   // side effect happens before commit is certain
COMMIT

// GOOD: the side effect only happens once commit has actually succeeded
BEGIN
  createOrder(orderData)
COMMIT
sendConfirmationEmail(orderData)     // only reached if COMMIT succeeded
```

- **DON'T:** Assume a side effect triggered "at the end of the function" is safe just because it's textually after the database write call, without confirming the transaction has actually committed by that point. In code with implicit or framework-managed transaction boundaries (a request-scoped transaction that commits automatically only after the whole request handler returns), a side effect fired mid-handler can still run before the eventual commit, and still needs the same care.

- **DO:** Use the outbox pattern (discussed earlier under distributed transactions) when a side effect genuinely needs a strong guarantee of happening if and only if the transaction commits, since "call the side effect right after commit" in application code is still vulnerable to the process crashing in the gap between the commit and the side-effect call — the outbox pattern closes that gap by making the intent to trigger the side effect itself part of the committed transaction.

- **DON'T:** Fire a side effect from inside a database trigger as a way to guarantee it happens exactly when the row changes, for any side effect that involves a network call (a trigger calling out to an HTTP webhook, for instance). A network call from inside a database trigger ties the transaction's commit latency and reliability to an external system's availability, and most databases either don't support this cleanly or make it a significant operational hazard.

### Clock Skew & Distributed Timestamps

- **DO:** Avoid relying on wall-clock timestamps from different machines to determine the order of events across a distributed system, since clocks on different machines are never perfectly synchronized — even with NTP, meaningful skew (milliseconds to, occasionally, much more under NTP failure) is normal, and "whichever timestamp is later" is not a reliable way to determine which of two events on different nodes actually happened first.

- **DON'T:** Use `created_at` timestamps generated independently on different application servers as the sole tiebreaker for "which write wins" in a last-write-wins conflict resolution scheme, without accounting for clock skew between those servers. Two writes that actually happened in one order can be timestamped in the opposite order if the writing servers' clocks disagree, silently picking the wrong "winner."

- **DO:** Use a logical clock (a monotonically increasing counter, a vector clock, or a hybrid logical clock that combines wall-clock time with a logical counter) instead of raw wall-clock time when the actual causal ordering of events across nodes matters for correctness, not just for approximate, human-readable timestamps.

- **DON'T:** Assume a single database server's own internal timestamp generation is safe to treat as a global ordering authority once more than one node (a read replica, a different region, a different service) can also produce timestamps that get compared against it. A timestamp is only a safe absolute ordering signal within a single, unambiguous clock source; the moment several independent clocks are involved, treat exact tie-breaking by timestamp alone as approximate, not exact.

- **DO:** Use the database's own transaction commit ordering (rather than an application-generated timestamp) when strict ordering of writes within a single database is what actually matters — a database's internal transaction sequencing is a real, exact ordering guarantee in a way that comparing `NOW()` values captured in different places is not.

## Caching

### Cache-Aside vs Write-Through vs Write-Behind

- **DO:** Use cache-aside (lazy loading) — the application checks the cache first, and on a miss reads from the database and populates the cache — as the default caching pattern for read-heavy data that tolerates brief staleness. It's simple to reason about, only caches data that is actually requested, and degrades gracefully (a cache outage just means every request falls through to the database) rather than failing outright.
```pseudocode
function getProduct(id):
    cached = cache.get("product:" + id)
    if cached: return cached
    product = db.query("SELECT * FROM products WHERE id = ?", id)
    cache.set("product:" + id, product, ttl=300)
    return product
```

- **DON'T:** Use cache-aside for data where staleness is unacceptable even briefly (real-time pricing during a flash sale, an account balance shown right after a transfer) without an explicit invalidation path triggered by the write, not just a TTL. A pure TTL-only cache-aside implementation guarantees staleness for up to the TTL duration on every write, which is fine for a product description and unacceptable for a live balance.

- **DO:** Use write-through caching — every write goes to the cache and the database together, synchronously, before the write is considered complete — when reads must never see stale data and the write-latency cost of updating both stores is acceptable. Write-through keeps the cache always consistent with the database because it is never allowed to fall behind.

- **DON'T:** Use write-through as a blanket default without accounting for its cost: every write now pays the latency of two systems instead of one, and a cache outage during a write either blocks the write entirely or requires a fallback policy — decide explicitly what happens to a write when the cache is unavailable rather than leaving it undefined.

- **DO:** Use write-behind (write-back) caching — writes land in the cache immediately and are asynchronously flushed to the durable store later — only when the workload can tolerate a real risk of losing the most recent writes if the cache crashes before flushing, in exchange for very low write latency. This is a genuine durability tradeoff, not a free performance win.

- **DON'T:** Choose write-behind caching for data with strict durability requirements (financial transactions, anything the business cannot afford to lose) without an explicit, tested plan for what happens on a cache node crash between the write and the flush. The performance gain of write-behind caching is not worth silently losing writes that matter.

### Cache Invalidation Strategies

- **DO:** Invalidate (delete or update) the specific cache entries a write affects at write time, rather than relying purely on a TTL to eventually clear stale data. Active invalidation bounds staleness to "until the next write's invalidation runs," while TTL-only invalidation bounds staleness to "up to the TTL duration," which is a much weaker guarantee for frequently-read, occasionally-written data.
```pseudocode
function updateProductPrice(id, newPrice):
    db.execute("UPDATE products SET price = ? WHERE id = ?", newPrice, id)
    cache.delete("product:" + id)   // explicit invalidation, not just waiting for TTL
```

- **DON'T:** Update the database and forget to invalidate (or update) the corresponding cache entry in the same code path. A write path that updates the source of truth but leaves the cache holding the old value produces exactly the "the UI shows old data after I just changed it" bug that is one of the most common and most confusing classes of caching bugs.

- **DO:** Invalidate by specific, precise cache keys tied to the entity that changed, rather than flushing the entire cache (or a broad prefix) on every write. Precise invalidation keeps the rest of the cache warm; flushing everything on every write causes a stampede of cache misses hitting the database right after every write, which can be worse than not caching at all under write-heavy load.

- **DON'T:** Rely on cache invalidation logic scattered independently across many different write code paths that all touch the same entity, hoping each one remembers to invalidate correctly. Centralize invalidation in the same layer that performs the write (a repository method, a model hook) so there is exactly one place responsible for keeping the cache correct for that entity, not N independent chances to forget.

- **DO:** Consider using the database's own change-notification mechanism (PostgreSQL `LISTEN`/`NOTIFY`, change data capture via a tool like Debezium, or a message published on write) to drive cache invalidation in systems with multiple writers or services, so invalidation isn't dependent on every single writer remembering to call the cache-clearing code.

- **DON'T:** Design cache keys that make invalidation ambiguous or impossible to target precisely — for example, caching an aggregate report under a generic key with no way to know which underlying rows fed into it. If a cached value depends on many underlying rows, either version the key so a new computation gets a new key, or explicitly track and invalidate based on which source rows can affect it.

### Avoiding Stale-Cache Bugs

- **DO:** Version cache keys with a value that changes when the underlying schema or computation logic changes (e.g., include a version number or a content hash in the key), so a code deploy that changes what a cached value means doesn't serve old-format data under a new code path that expects a different shape.
```pseudocode
// Bumping v1 -> v2 in the key instantly invalidates every old cache entry
// for this computation, without needing to explicitly flush anything
cacheKey = "recommendations:v2:user:" + userId
```

- **DON'T:** Cache a value that depends on the requesting user's identity, permissions, or context under a key that doesn't include that context. Caching a personalized or permission-scoped response under a shared/generic key can leak one user's data to another user entirely — this is a caching bug that becomes a security incident.
```pseudocode
// BAD: shared key regardless of who's asking — permission bypass risk
cache.get("dashboard_data")

// GOOD: key scoped to the requesting user
cache.get("dashboard_data:user:" + userId)
```

- **DO:** Treat "read-your-own-writes" as an explicit requirement to design for when it matters (a user who just updated their profile should immediately see the update reflected back), rather than an incidental property of whatever cache invalidation happens to do. Depending on the cache/invalidation architecture, a user's own subsequent read can otherwise hit a stale cached value or a lagging read replica right after their own write.

- **DON'T:** Assume a distributed cache (multiple cache nodes, or a cache plus a CDN layer in front of it) invalidates atomically and instantly everywhere. Invalidation across a distributed cache typically propagates with some delay or requires an explicit fan-out to every node/edge location — know the actual propagation characteristics of the caching layer in use rather than assuming instantaneous global consistency.

- **DO:** Add a "cache stampede" / thundering herd guard (request coalescing, a short lock/mutex around cache population, or serving stale-while-revalidate) for expensive-to-compute cache values with high read concurrency, so that a cache expiration doesn't trigger dozens or hundreds of concurrent requests all recomputing the same expensive value against the database simultaneously.
```pseudocode
// Stale-while-revalidate: serve the (slightly) stale value immediately,
// but kick off exactly one background refresh instead of N concurrent ones
function getExpensiveReport(key):
    entry = cache.get(key)
    if entry and not entry.expired:
        return entry.value
    if entry and entry.staleButUsable and not entry.refreshInProgress:
        entry.refreshInProgress = true
        backgroundRefresh(key)   // one refresh, not one per concurrent request
        return entry.value       // serve stale value while it refreshes
    return computeAndCache(key)  // true cold miss: compute synchronously
```

- **DON'T:** Cache error responses or empty/null results indefinitely without a distinctly shorter TTL than successful results. Caching "not found" or an error from a transient failure for the same duration as a successful lookup can hide a real record's existence, or hide a resolved transient failure, from users for far longer than intended.

### TTL Discipline

- **DO:** Set an explicit, deliberately chosen TTL on every cache entry based on how quickly the underlying data changes and how costly staleness is for that specific data — not a single blanket TTL applied uniformly across completely different kinds of data. A product catalog entry, a user session, and a real-time inventory count have very different acceptable staleness windows and should not share a default.

- **DON'T:** Cache data with no TTL at all ("cache forever") unless invalidation is fully and reliably handled by explicit write-time invalidation with no gaps. A cache entry with no expiration and an invalidation path that has even one missed code path becomes permanently stale with no self-healing mechanism — a TTL acts as a safety net that bounds the damage of a missed invalidation.

- **DO:** Add jitter (a small random variation) to TTLs for a large batch of cache entries that would otherwise expire at exactly the same time, to avoid a synchronized mass expiration that causes a burst of simultaneous cache misses hitting the database all at once.
```pseudocode
// Avoid every entry set in the same batch expiring in the same instant
baseTtl = 300
jitter = random(0, 30)
cache.set(key, value, ttl = baseTtl + jitter)
```

- **DON'T:** Set a TTL so short that the cache provides negligible benefit relative to the overhead of maintaining it, or so long that staleness routinely becomes a user-visible problem before the next natural write-triggered invalidation. Tune the TTL against measured staleness tolerance and cache hit rate, not an arbitrary round number picked without data.

- **DO:** Differentiate TTLs by data volatility within the same system — a rarely-changing configuration value can have a TTL of hours, while a frequently-updated counter or a "currently online" status needs a TTL of seconds, if it is cached at all.

- **DON'T:** Assume the cache and the database will never disagree just because a TTL exists. A TTL bounds *how long* staleness can persist, but does not prevent staleness from occurring at all — code that reads from cache still needs to tolerate the fact that the value may be slightly out of date, and any code path where that's unacceptable should bypass the cache entirely rather than relying on a short-enough TTL to "usually" be fine.

### Cache Warming & Preloading

- **DO:** Pre-populate ("warm") a cache with known-hot data before it's needed — at deploy time, on a schedule, or immediately after a cache flush — for data whose absence from the cache during a cold start would cause a painful spike of expensive database queries. Warming turns a predictable cold-start problem into a controlled, off-peak operation instead of an unplanned load spike during real traffic.

- **DON'T:** Deploy a change that clears or replaces the entire cache during peak traffic hours without a warming plan. A fully cold cache immediately after a deploy means every request for previously-cached data now falls through to the database simultaneously — exactly the thundering-herd problem cache invalidation strategies are meant to avoid, just triggered by a deploy instead of a TTL expiration.

- **DO:** Warm caches incrementally or in the background after a deploy (rolling the cache-clearing deploy out gradually, or explicitly re-populating known-hot keys immediately after) rather than accepting a full cold start as unavoidable. Even a partial warm-up of the highest-traffic keys meaningfully reduces the post-deploy load spike.

- **DON'T:** Warm a cache with stale or synthetic data that doesn't reflect current reality "to have something there." A cache pre-populated with wrong values is worse than an empty cache that correctly falls through to the database — verify that warmed data is genuinely current before relying on it.

### Multi-Layer Caching

- **DO:** Understand and deliberately design for every caching layer actually in the request path — browser cache, CDN/edge cache, a shared application-level cache (Redis/Memcached), and any in-process/local cache — since each layer has its own invalidation mechanism, and a fix applied at only one layer while others still serve a stale value produces confusing, layer-dependent staleness bugs.

- **DON'T:** Set caching headers (`Cache-Control`, `ETag`) without considering how they interact with a CDN or reverse proxy sitting in front of the application. A response cached aggressively at the CDN layer can remain stale for its full TTL even after the application-level cache and the database have both been correctly updated and invalidated — the staleness just moved to a layer that's easy to forget about.

- **DO:** Use a short-lived local/in-process cache (in application memory) as an additional layer on top of a shared distributed cache only for extremely hot, small, and TTL'd data, and understand explicitly that an in-process cache is inherently inconsistent across multiple application instances — each instance's local cache can hold a different value until its own TTL expires.

- **DON'T:** Assume invalidating the shared/distributed cache also invalidates every application instance's local in-process cache. If a local cache layer exists, its invalidation has to be handled explicitly (a pub/sub invalidation broadcast to all instances, or a deliberately very short local TTL) — it does not automatically follow from invalidating the shared cache underneath it.

- **DO:** Document which layer is authoritative for which kind of staleness tolerance, so a developer debugging "why am I seeing old data" knows which layer to check first (browser cache for a single user's stale view, CDN for a stale view affecting everyone in a region, application cache for a stale view affecting everyone) instead of guessing across four different systems.

### What Not to Cache

- **DO:** Keep highly sensitive data (authentication tokens, full payment details, anything subject to strict regulatory handling requirements) out of general-purpose shared caches entirely, or cache it only in a cache specifically configured and audited to meet the same security/compliance bar as the primary datastore. A shared cache is often operated with weaker access controls, weaker encryption-at-rest guarantees, and a different audit trail than the primary database — caching sensitive data there can silently widen its exposure.

- **DON'T:** Cache data that changes on every read or is inherently request-specific (a computed value that depends on the exact current timestamp, a per-request random value, truly real-time data where "as of a few seconds ago" is unacceptable) just because caching is the default pattern for read paths in the codebase. Caching data with no real reuse potential adds cache management overhead and cache-invalidation surface area for zero benefit.

- **DO:** Think specifically about whether cached data is safe to share across different users/tenants before caching it under a shared key — the single most consequential caching-scope mistake is caching personalized or permission-scoped data under a key that doesn't include the requester's identity, which turns a caching bug into a cross-user data leak.

- **DON'T:** Cache the result of an operation that has side effects or that the caller expects to actually execute every time (sending an email, incrementing an external counter, writing an audit log entry) as if it were a pure read. Caching is for idempotent reads of otherwise-expensive-to-recompute data, not for skipping operations that are supposed to happen on every call.

### Cache Key Design

- **DO:** Build cache keys from a deterministic, complete description of everything that affects the cached value — entity ID, relevant filter/query parameters, locale, user context where applicable, and a version marker — so that two logically different requests can never collide on the same key, and the same logical request always produces the same key.
```pseudocode
// Deterministic key including every input that affects the result
cacheKey = "search:v3:" + query.hash() + ":locale=" + locale + ":page=" + page
```

- **DON'T:** Build a cache key from only part of what determines the cached value (for example, keying a filtered/paginated list result only by the entity type and page number, omitting the filter parameters). Two different filtered queries landing on the same key means one request's result gets served, incorrectly, to a completely different request.

- **DO:** Use a consistent, collision-resistant hashing strategy for cache keys built from complex or large inputs (a full query object, a long list of parameters) rather than a naive string concatenation that could produce the same key for two different inputs. A short, well-distributed hash of the full canonicalized input is safer than an ad hoc concatenation that might accidentally collide.

- **DON'T:** Let cache key construction logic be duplicated independently across multiple call sites that cache and invalidate the same logical data. If the code that writes a cache key and the code that invalidates it construct the key slightly differently, invalidation silently misses the cached entry — centralize key construction in one function/module used by every caller that touches that cache entry.

- **DO:** Keep cache keys human-readable and greppable where practical (a clear prefix and structure, not an opaque hash for the entire key) so that operators can inspect, debug, and selectively clear entries in production without needing to reverse-engineer the key format from source code.

### CDN & HTTP Caching Specifics

- **DO:** Set `Cache-Control` headers explicitly and deliberately for every HTTP response that could benefit from caching (`max-age`, `s-maxage` for shared/CDN caches specifically, `private` vs `public`), rather than leaving caching behavior to whatever default the framework or CDN happens to apply. Explicit headers are the actual contract between the application and every cache sitting in front of it — an unset or wrong header either fails to cache data that should be cached, or caches data that should never be shared across users.
```text
# A public, shared-cacheable response with a short freshness window
Cache-Control: public, max-age=60, stale-while-revalidate=30

# A response containing user-specific data — must never be cached by a shared CDN
Cache-Control: private, no-store
```

- **DON'T:** Mark a personalized or authenticated response `public` (cacheable by shared/CDN caches) by default. A CDN that caches one user's personalized dashboard response as `public` and then serves that same cached response to a different user is a data leak caused entirely by a caching header mistake, not by an application logic bug.

- **DO:** Use `ETag` or `Last-Modified` conditional-request headers for resources that change infrequently, so a client can send `If-None-Match`/`If-Modified-Since` and receive a cheap `304 Not Modified` instead of re-downloading unchanged content — this reduces bandwidth and load without sacrificing freshness the way a long `max-age` alone would.

- **DON'T:** Set an aggressive, long `max-age` on a resource whose content can change in a way clients need to see promptly, without a cache-busting mechanism (a version/hash in the URL, or explicit purge/invalidation capability at the CDN). A long TTL with no way to invalidate specific cached responses means a bad deploy or an urgent content fix can stay stuck in caches around the world for the full TTL duration with no way to force it out sooner.

- **DO:** Use `stale-while-revalidate` (or the CDN's equivalent) for content where serving a slightly stale response while asynchronously refreshing it in the background is an acceptable tradeoff for consistently fast responses — this avoids every cache expiration turning into a slow, synchronous rebuild for whichever request happens to be first after expiry.

### Write-Through Cache Failure Handling

- **DO:** Decide explicitly, in advance, what happens when one half of a write-through cache update fails — the database write succeeds but the cache write fails, or vice versa — rather than treating this as an edge case too rare to plan for. A reasonable default is to treat the durable store (the database) as authoritative: if the cache write fails after a successful database write, invalidate/evict the cache key rather than leaving it holding a stale or absent value that a subsequent cache-aside read would need to reconcile.
```pseudocode
function updateProduct(id, newData):
    db.update("products", id, newData)   // durable write happens first
    try:
        cache.set("product:" + id, newData)
    except CacheError:
        cache.delete("product:" + id)    // fail safe: evict rather than risk stale data
```

- **DON'T:** Write to the cache before confirming the database write has actually succeeded. If the cache write happens first and the subsequent database write fails, the cache now holds a value that was never actually durably committed — any reader hitting the cache sees data that doesn't exist in the source of truth and never will unless the write is retried successfully.

- **DO:** Monitor and alert on write-through cache failures as their own distinct signal, separate from database write failures, since a cache-only failure (the database write succeeded) is a different, lower-severity situation than a database write failure, but still needs visibility — a cache that's silently failing to update on every write will gradually serve entirely stale data with no other symptom.

- **DON'T:** Assume a cache write failing is harmless just because the cache is "just a cache." A stale value left in the cache after a failed write-through update can persist and be served to every subsequent reader until its TTL expires or something else invalidates it — treat a write-through failure as a correctness issue requiring cleanup, not a shrug-worthy non-event.

### Caching Computed Aggregates & Expensive Joins

- **DO:** Cache the result of expensive, frequently-repeated aggregate queries and multi-table joins specifically — a dashboard's summary statistics, a leaderboard, a report that scans a large table — since these are exactly the queries where caching provides the largest return: expensive to compute, and often read far more often than their underlying data actually changes.

- **DON'T:** Cache the result of a cheap, fast query just because caching is available. A simple indexed primary-key lookup that already takes a couple of milliseconds gains little from caching and adds cache-management overhead (invalidation logic, memory usage, another thing that can go stale) for a negligible speedup — reserve caching effort for genuinely expensive operations.

- **DO:** Invalidate a cached aggregate based on changes to any of its underlying inputs, tracked explicitly (a write to any of the source tables that feed the aggregate triggers invalidation of the cached result), rather than relying on a TTL alone for data where staleness has a real, visible cost (a leaderboard that's stuck showing yesterday's ranking, for instance).

- **DON'T:** Cache an expensive aggregate under a single global key when it's actually parameterized (a report scoped by date range, region, or filter). A single shared cache key for a parameterized query either serves the wrong parameters' results to different requests, or never gets reused at all because every request's parameters differ — key the cache correctly by every parameter that affects the result, as covered under cache key design.

### Local Development & Testing Without a Live Cache Dependency

- **DO:** Provide an in-memory fake or a local containerized instance of the caching layer for local development and automated tests, so tests don't depend on a shared, stateful, network-reachable cache instance that different test runs (or different developers) could interfere with each other on. Tests that share a real cache instance across parallel runs commonly fail intermittently because one test's leftover cached state affects another's expectations.

- **DON'T:** Let automated tests silently share cache state with other tests, other developers, or a persistent local cache across runs. A test that passes only because a previous test happened to warm the cache with the right value (or fails only because a previous run left stale data behind) is testing the cache's leftover state, not the code under test — clear or isolate cache state explicitly between test runs.

- **DO:** Test cache-miss and cache-hit code paths both explicitly, since a bug in the cache-population or invalidation logic can easily hide behind a code path that always happens to run against a warm cache during casual manual testing but never gets exercised cold.

- **DON'T:** Assume behavior verified against a fast, low-latency local cache generalizes to production, where network latency to a real distributed cache is a meaningfully different (and non-zero) cost. Code that awaits a cache lookup synchronously on every request might be fine against a local in-memory fake with near-zero latency and still be a real bottleneck against a network-hop cache in production — validate latency assumptions against realistic infrastructure, not just correctness against a fake.

### Rate Limiting Using a Cache or Key-Value Store

- **DO:** Use a fast key-value store (Redis is the common choice) with atomic increment-and-expire operations to implement rate limiting, since rate limiting is fundamentally a high-frequency counter operation that needs to be both fast and safe under heavy concurrent access — exactly what an in-memory key-value store with atomic primitives is built for.
```pseudocode
// Fixed-window rate limiting using atomic INCR + EXPIRE
key = "ratelimit:" + userId + ":" + currentWindow()
count = redis.incr(key)
if count == 1:
    redis.expire(key, windowDurationSeconds)
if count > limit:
    reject("rate limit exceeded")
```

- **DON'T:** Implement rate limiting against the primary relational database with a read-then-write pattern (`SELECT` the current count, check it, `UPDATE` it). This is both a race condition under concurrent requests (two requests can both read the same count before either writes back an incremented value, letting both through when only one should have been) and adds unnecessary load to the primary database for what is fundamentally ephemeral, high-frequency counter state that doesn't need durable relational storage.

- **DO:** Choose the rate-limiting algorithm deliberately (fixed window, sliding window, or token bucket) based on the actual fairness and burst-tolerance requirements — a fixed window is simplest but allows a burst of up to double the limit right at a window boundary; a sliding window or token bucket smooths that out at the cost of slightly more complex bookkeeping.

- **DON'T:** Let rate-limit state accumulate without an expiration matching the rate-limiting window. Every rate-limit key needs a TTL that reflects the window it's counting — without one, the key-value store accumulates rate-limit counters forever for every user and every window that's ever existed, exactly the unbounded-key-growth problem discussed earlier for key-value stores generally.

## Data Pipelines & ETL

### Idempotent Pipeline Design

- **DO:** Design every pipeline stage so that running it twice on the same input produces the same end state as running it once — the same principle as idempotent API operations, applied to batch and streaming data processing. Pipelines fail and get re-run constantly (a transient error, a manual retry, an orchestrator retrying a failed task); a non-idempotent pipeline turns every retry into a data-correctness risk.

- **DON'T:** Write a pipeline stage that appends/inserts new rows on every run without a mechanism to detect "this batch/record was already processed." A naive `INSERT` re-run after a partial failure duplicates every row that was already successfully written before the failure, silently corrupting downstream aggregates and reports.
```sql
-- BAD: re-running this after a partial failure creates duplicate rows
INSERT INTO daily_sales_summary SELECT date, SUM(amount) FROM sales GROUP BY date;

-- GOOD: idempotent via upsert keyed on the natural identity of the row
INSERT INTO daily_sales_summary (date, total)
SELECT date, SUM(amount) FROM sales GROUP BY date
ON CONFLICT (date) DO UPDATE SET total = EXCLUDED.total;
```

- **DO:** Make batch/window-based pipeline runs idempotent by having each run fully own and replace its own output partition or date range (`DELETE` then `INSERT`, or an atomic partition swap) rather than incrementally appending. A pipeline that owns "everything for 2026-03-15" and rewrites that partition wholesale on every run of that date can be re-run safely any number of times with an identical result.
```sql
BEGIN;
DELETE FROM daily_sales_summary WHERE date = :run_date;
INSERT INTO daily_sales_summary SELECT date, SUM(amount) FROM sales
  WHERE date = :run_date GROUP BY date;
COMMIT;
```

- **DON'T:** Design a pipeline stage around "checkpoint and resume from where it left off" without also making the resumed portion idempotent relative to whatever partial work already committed before the failure. A naive resume that just continues from a saved offset, without checking whether records at or near that offset were already written, can double-process the boundary records.

- **DO:** Give every pipeline run a unique, deterministic run identifier (derived from the batch's logical time window or an explicit run ID, not simply "now") so that reprocessing the same logical batch — whether triggered manually or by an orchestrator retry — is recognizable as the same logical unit of work rather than a new one.

- **DON'T:** Build a streaming pipeline on top of a message queue's at-least-once delivery guarantee without deduplication logic on the consumer side. At-least-once means every message is guaranteed to arrive at least once, not exactly once — a consumer that processes every delivered message as if it were guaranteed-unique will double-process messages that were redelivered after a consumer crash or a slow acknowledgment.

### Schema Evolution

- **DO:** Version data schemas explicitly (a schema registry, a version field embedded in each record, or a well-defined versioned file format) for any data that crosses a pipeline boundary between independently-deployed producers and consumers. Without an explicit schema contract, a producer that changes its output format silently breaks every downstream consumer that assumed the old shape.

- **DON'T:** Make a breaking schema change (renaming a field, changing a field's type, removing a field consumers depend on) to a data format that other, independently-deployed systems already consume, without a compatibility plan. A producer that changes `"amount": 19.99` to `"amount_cents": 1999` with no transition period breaks every consumer at the exact moment the new producer version deploys, with no coordinated rollout possible.

- **DO:** Prefer additive, backward-compatible schema changes (adding a new optional field, adding a new event type) over changes that remove or repurpose existing fields, following the same compatibility discipline used for public API versioning. Consumers that don't know about a new field can safely ignore it; consumers that expect a field that no longer exists cannot.

- **DON'T:** Silently change the meaning of an existing field without renaming it (e.g., a `status` field that used to mean order status starts meaning shipment status after a refactor, with no schema-level signal that anything changed). A field that keeps its name but changes its meaning is far more dangerous than an obviously-renamed field, because nothing forces consumers to notice and update.

- **DO:** Use a schema format that supports explicit forward/backward compatibility rules (Avro, Protocol Buffers, JSON Schema with a registry) for high-fan-out data (many independent consumers) rather than an ad hoc, undocumented JSON shape that consumers reverse-engineer from examples. A schema registry can actively reject a producer change that would break existing consumers, catching the problem before it reaches production instead of after.

- **DON'T:** Assume a downstream consumer will "just ignore" unexpected fields or a changed structure without verifying that assumption. Some parsers/deserializers fail loudly on unrecognized fields or strict schema mismatches by design (strict JSON schema validation, strongly-typed deserialization) — know the actual failure behavior of every real consumer before treating additive changes as automatically safe for all of them.

- **DO:** Maintain a defined deprecation window for removing a field or event type entirely — announce it, keep emitting it (or accepting it) for a defined period, monitor for consumers still using it, and only remove it once usage has genuinely dropped to zero, mirroring the same discipline used for deprecating a public API.

### Data Validation at Ingestion

- **DO:** Validate incoming data against explicit rules (types, required fields, value ranges, referential consistency) at the point of ingestion, rather than trusting upstream sources and discovering malformed data only when a downstream query or dashboard produces obviously wrong numbers. Validating at the boundary means a bad record is caught, logged, and handled deliberately — instead of silently propagating through every downstream transformation.
```pseudocode
function ingestRecord(raw):
    if raw.amount is None or raw.amount < 0:
        quarantine(raw, reason="invalid amount")
        return
    if raw.timestamp not in reasonableRange():
        quarantine(raw, reason="implausible timestamp")
        return
    write(raw)
```

- **DON'T:** Let a pipeline silently coerce or discard invalid values with no record of what was dropped or why. A row that fails a type conversion and gets silently skipped (or worse, silently coerced to `0`/`null`/some default) corrupts downstream aggregates in a way that is extremely hard to trace back to its cause, because there is no trail showing anything was ever wrong.

- **DO:** Route records that fail validation to a distinct quarantine/dead-letter destination rather than either blocking the entire pipeline run on one bad record or silently dropping it. A dead-letter queue/table preserves the bad data for inspection and reprocessing while letting the rest of the valid batch proceed, and gives you an auditable count of how much data is failing validation over time.

- **DON'T:** Assume upstream systems (a third-party API, another team's service, a user-submitted file upload) will always send well-formed data because their own documentation/schema says they will. Real-world upstream data routinely violates its own documented contract — a required field missing, a date in the wrong format, an enum value that isn't in the documented list — and a pipeline that trusts the contract blindly breaks on the first violation.

- **DO:** Validate referential/business-level consistency, not just per-field type correctness, where it matters — a foreign key reference that doesn't exist in the dimension table, an order total that doesn't match the sum of its line items, a percentage field outside 0–100. Type-only validation catches malformed data but misses data that is well-typed and still wrong.

- **DON'T:** Perform validation only at the very end of a long pipeline, after several expensive transformation stages have already run on the bad data. Validate as early as possible (ideally right at ingestion) so that invalid data is caught before the pipeline spends compute on records that were always going to be rejected, and so the failure is attributed to the actual source rather than to whichever downstream stage happened to choke on it.

### Avoiding Silent Data Loss

- **DO:** Make every stage of a pipeline count records in and records out, and alert when the counts don't reconcile within an expected tolerance (accounting for legitimate filtering/deduplication). A stage that silently drops rows — due to a join that unintentionally filters out unmatched rows, a parsing error swallowed by a broad `except`/`catch`, or a timeout on part of a batch — is invisible unless something is actively watching the row counts.
```pseudocode
inputCount = countRows(source)
outputCount, errorCount = processAndCount(source, sink)
expectedLoss = errorCount  // rows we deliberately quarantined
if inputCount - outputCount != expectedLoss:
    alert("unexplained row count discrepancy in pipeline stage X")
```

- **DON'T:** Use an inner join where a left join was needed (or vice versa) without deliberately checking which rows a mismatched join type will silently exclude. An inner join between a fact table and a dimension table quietly drops every fact row whose dimension key doesn't (yet) exist in the dimension table — a very common, very quiet source of missing data in reporting pipelines, especially with late-arriving dimension data.

- **DO:** Catch and log exceptions at the level of an individual record within a batch, rather than letting one malformed record's exception propagate and either crash the entire batch or (worse) get swallowed by an overly broad `try/except` around the whole batch that silently skips everything after the failure point.
```pseudocode
// BAD: one bad record silently aborts everything after it, or a broad
// catch swallows the error and the rest of the batch is never processed
try:
    for record in batch:
        process(record)
except Exception:
    log("something went wrong")   // which record? how many were lost?

// GOOD: isolate failures per record, keep processing, and report precisely
failures = []
for record in batch:
    try:
        process(record)
    except Exception as e:
        failures.append((record, e))
if failures:
    reportFailures(failures)  // exact records and reasons, not a vague count
```

- **DON'T:** Treat "the pipeline job finished with exit code 0" as proof that all the data was processed correctly. A job can complete successfully while having silently skipped, truncated, or malformed a meaningful fraction of its input — success/failure of the job process and correctness of its output are two different things that need two different kinds of verification.

- **DO:** Build reconciliation checks that compare pipeline output against an independent source of truth periodically (row counts, checksums, key business totals like "sum of revenue in the warehouse should match sum of revenue in the source system for the same period") rather than trusting the pipeline's own internal logging exclusively. A pipeline verifying its own correctness using only its own instrumentation cannot catch a bug in that instrumentation itself.

- **DON'T:** Silently truncate string fields, drop precision on numeric fields, or lose timezone information during a format conversion between pipeline stages (e.g., loading into a column narrower than the source data, or parsing a timestamp without explicit timezone handling). Any lossy conversion should either fail loudly when data doesn't fit, or the loss should be an explicit, documented, deliberate decision — not an unnoticed side effect of a default cast.

### Batch vs Streaming Tradeoffs

- **DO:** Choose batch processing as the default for data where near-real-time freshness isn't actually a business requirement — most reporting, most analytics, most nightly reconciliation. Batch pipelines are simpler to build, test, debug, and reason about than streaming pipelines, and "the report is an hour old" is very often a perfectly acceptable answer that a streaming architecture would be paying unnecessary complexity to avoid.

- **DON'T:** Build a streaming pipeline for a use case that would be entirely well served by a batch job run every few minutes or hours, just because streaming feels more sophisticated or scalable. Streaming architectures add real, ongoing operational complexity — exactly-once semantics, watermarking for late-arriving data, stateful stream processing, backpressure handling — that is only worth paying for when genuine low-latency requirements demand it.

- **DO:** Choose streaming when the actual business requirement is measured in seconds (fraud detection that must act before a transaction completes, real-time dashboards for operational monitoring, alerting that needs to fire promptly) — cases where the value of the data decays quickly enough that batch latency would defeat the purpose.

- **DON'T:** Assume streaming automatically guarantees lower end-to-end latency than a frequent, well-tuned batch job. A streaming pipeline with significant processing overhead per event can, in practice, have worse tail latency than a batch job that runs every minute and processes a bounded window efficiently — measure actual latency requirements against actual achievable latency for both approaches before committing to the more complex one.

- **DO:** Design explicit handling for late-arriving data in both batch and streaming pipelines — a batch job needs a defined reprocessing/backfill window for data that arrives after its batch already ran; a streaming job needs explicit watermarking and a defined policy for how late an event can arrive and still be included in its intended window. Neither architecture makes "data arrives exactly on time, every time" a safe assumption.

- **DON'T:** Mix batch and streaming logic for the same metric without reconciling them, producing two different numbers for "the same" thing depending on which pipeline a consumer happens to query (a common "lambda architecture" pitfall when the batch and speed layers' logic silently drifts apart over time). If both a real-time approximate number and a batch-corrected authoritative number are needed, label them clearly as such rather than letting them look like the same number with two different values.

- **DO:** Consider a hybrid approach deliberately (streaming for a low-latency approximate view, batch for periodic authoritative correction/backfill) when the business genuinely needs both fast approximate numbers and accurate final numbers, and document explicitly which number is which so consumers know which one to trust for which purpose.

### CDC & Event-Driven Pipelines

- **DO:** Use change data capture (CDC) — reading a database's transaction/replication log (e.g., via Debezium, native logical replication) — to feed downstream pipelines and other services with a reliable, ordered, low-latency stream of every change, instead of periodically polling the source table for "what changed since last time." CDC captures every intermediate state change (including deletes) in order, which polling based on an `updated_at` column can miss entirely.

- **DON'T:** Build change detection by polling with `WHERE updated_at > :last_poll_time` as the sole mechanism for a pipeline that needs to capture every change, including deletes. This approach misses hard deletes entirely (a deleted row has no `updated_at` to poll for), can miss rapid updates to the same row between polls (only the latest state is seen, intermediate values are lost), and is vulnerable to clock skew between the application and the polling job.

- **DO:** Treat the CDC/event stream as the pipeline's actual source of truth for downstream consumers, and design consumers to be resilient to replaying the stream from an earlier offset (for recovery, backfill, or adding a new consumer) — which circles back to idempotent consumer design.

- **DON'T:** Let an event-driven pipeline assume events arrive in a strict global order across different partitions/topics unless the specific messaging system and partitioning scheme actually guarantee that ordering. Many systems guarantee ordering only within a single partition/key, not globally — a consumer that assumes global ordering can process events in the wrong sequence relative to each other when they land in different partitions.

- **DO:** Include a clear, unambiguous event type and schema version in every event payload, along with the entity's primary key and, for CDC-derived events, the operation type (insert/update/delete) and, where possible, both the before and after state for updates. A consumer that can't tell whether an event represents a create, update, or delete has to guess, and a guess is eventually wrong.

- **DON'T:** Couple downstream consumers directly to the source database's internal schema/table structure via CDC without an explicit translation layer. Exposing raw CDC events straight from internal tables means any internal refactor of those tables (a column rename, a table split) becomes a breaking change for every external consumer — publish a stable, intentionally-designed event schema instead of the raw internal row shape.

### Observability for Data Pipelines

- **DO:** Track data quality metrics as first-class monitored signals — row counts, null rates on critical fields, distribution shifts on key metrics, freshness (time since the last successful update) — with the same seriousness as uptime and latency metrics for a service. A pipeline can be "up" (running without errors) while silently producing wrong or incomplete data, and only data-quality-specific monitoring catches that failure mode.

- **DON'T:** Treat "the pipeline job succeeded" as the only signal worth alerting on. A job that completes successfully every single run while its output row count silently drops by 90% (because an upstream API started returning an empty page, or a filter condition regressed) will never trigger a job-failure alert — only a check on the data itself catches this class of problem.
```pseudocode
// Freshness + volume check, run after every pipeline execution
lastUpdate = getMaxTimestamp("fact_orders")
if now() - lastUpdate > expectedFreshnessSLA:
    alert("fact_orders is stale")

rowCount = countRows("fact_orders", since=today)
if rowCount < historicalAverage(rowCount) * 0.5:
    alert("fact_orders row count is anomalously low today")
```

- **DO:** Alert on anomalies relative to a historical baseline (a metric that's 3 standard deviations off its trailing average, a day-over-day change beyond an expected range) rather than only on fixed absolute thresholds, since "normal" volume for many pipelines varies by day of week, seasonality, or business cycle in ways a single static threshold can't capture without generating constant false alarms.

- **DON'T:** Bury data pipeline failures in a log file nobody actively watches, with no alerting hooked up to an on-call rotation or notification channel. A pipeline failure discovered by a stakeholder noticing "the dashboard looks wrong" days later, instead of by an automated alert within minutes, has already caused whatever downstream damage the delay allowed.

- **DO:** Maintain data lineage — a record of which source tables, files, and transformation logic version produced a given downstream dataset — so that when a data quality issue is discovered, it's possible to trace it back to its origin and to identify every downstream consumer that was affected by it, rather than starting the investigation from zero.

- **DON'T:** Rely purely on manual, ad hoc spot-checks ("someone looks at the dashboard occasionally and would probably notice if something looked wrong") as the data quality strategy for a pipeline that feeds decisions that matter. Manual spot-checks catch large, obvious anomalies inconsistently and catch subtle, gradual data quality regressions almost never — automate the checks that matter.

- **DO:** Version and track the transformation logic (dbt model versions, pipeline code commit hash, configuration) alongside the data it produced, so that a data quality investigation can determine whether an anomaly correlates with a specific code or configuration change, not only with a specific point in time.

### Orchestration, Backfills & Dependency Management

- **DO:** Express pipeline task dependencies explicitly as a directed acyclic graph (DAG) — using an orchestrator (Airflow, Dagster, Prefect) or a build-style dependency tool (dbt's model dependency graph) — rather than relying on cron jobs scheduled at times that merely happen to give the previous step enough of a head start. An implicit time-based dependency ("job B runs 30 minutes after job A because A usually finishes by then") breaks silently the first time A runs slower than usual.
```pseudocode
// Explicit dependency, not an assumed time gap
task_extract >> task_transform >> task_load   // orchestrator enforces this order,
                                                // and only starts B once A actually succeeds
```

- **DON'T:** Chain pipeline stages via fixed cron offsets when a proper dependency-aware orchestrator is available. A time-based chain has no way to know whether the upstream step actually succeeded before the downstream step starts — it will happily run on stale or missing data if the upstream job failed or ran late.

- **DO:** Design every pipeline task to be independently re-runnable for a specific historical time period (a backfill), not only capable of processing "today's" data — parameterize tasks by the logical date/window they're processing rather than hardcoding "now." Backfills are routine (fixing a bug that affected the last month of data, adding a new derived field retroactively) and a pipeline that can only ever process the current day makes every backfill a special, manually-hacked one-off.
```pseudocode
// Parameterized by logical date, not wall-clock "now" — trivially backfillable
function runDailyAggregation(logicalDate):
    data = extract(logicalDate)
    result = transform(data)
    load(result, partition=logicalDate)

// Backfilling three months is then just: for each date in range, run it
```

- **DON'T:** Let a backfill silently skip the same validation, deduplication, and idempotency guarantees the regular scheduled run has. A backfill script hacked together separately from the regular pipeline logic often reintroduces exactly the bugs (duplicate rows, missing validation) the regular pipeline was designed to avoid — reuse the same code path for both, parameterized by date range, rather than maintaining two versions.

- **DO:** Set explicit retry policies, timeouts, and failure-alerting per task in the orchestrator, with a clear owner notified on failure, rather than leaving a failed task to be discovered only when someone notices downstream data is missing or wrong. A failed task with no alert is a silent pipeline break.

- **DON'T:** Build implicit, undeclared dependencies between pipelines owned by different tasks/teams (Pipeline B silently reads a table that Pipeline A happens to populate, with no explicit contract between them). If Pipeline A's schedule or logic changes, Pipeline B breaks with no warning to either team — make cross-pipeline dependencies explicit and discoverable, ideally through the orchestrator's own dependency graph rather than tribal knowledge.

### Data Contracts & Ownership

- **DO:** Establish an explicit, versioned "data contract" between a data-producing team/system and its consumers — the schema, the semantics of each field, the expected freshness/SLA, and how breaking changes will be communicated — for any dataset with more than one consuming team. A data contract turns "please don't break this table without telling us" from an informal hope into a documented, enforceable agreement.

- **DON'T:** Let a table or dataset's schema and meaning be defined only implicitly by whatever the producing pipeline currently happens to output, with consumers reverse-engineering the meaning of each column from example data. Undocumented, contract-free data is a constant source of "we didn't know that field's meaning changed" incidents whenever the producing team refactors.

- **DO:** Assign a clear, discoverable owner (a team, not just an individual) to every dataset used by more than one downstream consumer, so a consumer who finds a data quality issue or has a schema-change question knows exactly who to reach, rather than the dataset being an orphaned artifact nobody feels responsible for.

- **DON'T:** Treat a one-time export or ad hoc extract as a stable, reusable data source once other people start depending on it. An extract that was originally a quick one-off query result, now being read by three other pipelines, has no contract, no freshness guarantee, and no notification path if the person who created it stops maintaining it — formalize it as a proper, owned pipeline output once real dependents exist.

- **DO:** Include automated schema and semantic checks in CI for producing pipelines that validate against the declared data contract before deploying a change, so a breaking change to a contracted dataset is caught before it ships, not after a downstream consumer's pipeline breaks in production.

### PII & Sensitive Data in Pipelines

- **DO:** Identify and explicitly classify personally identifiable information (PII) and other sensitive fields (health data, financial account details, government identifiers) as data enters a pipeline, and apply the handling policy that classification requires — masking, tokenization, field-level encryption, or exclusion — deliberately rather than letting sensitive fields flow through every transformation and destination unexamined.

- **DON'T:** Copy production data containing real PII into a development, staging, analytics, or data-science environment without masking or anonymizing it first. "We'll just use a copy of prod for testing" is one of the most common ways sensitive data ends up in a less-secured environment, accessible to a much wider set of people and tools than the production database ever was.

- **DO:** Apply masking, pseudonymization, hashing, or tokenization to sensitive fields specifically for non-production environments (a consistent hash of an email address instead of the real email, for instance, if referential consistency across tables is still needed for testing) so development and analytics work can proceed without ever exposing real individuals' data outside its authorized scope.
```pseudocode
// Deterministic masking preserves joinability across tables without
// exposing the real value in a non-production environment
maskedEmail = hash(realEmail + salt) + "@masked.example"
```

- **DON'T:** Log sensitive fields in plaintext as part of routine pipeline debugging/error logging. A stack trace or debug log that includes a full row's contents on failure can leak PII into log aggregation systems that have much broader access and much longer, less-audited retention than the primary database — redact or omit sensitive fields from logs by default, and require deliberate opt-in for the rare case a specific field genuinely needs to appear in a log for debugging.

- **DO:** Build data deletion (the operational side of "right to erasure" requirements like GDPR/CCPA) into pipeline design from the start — knowing which pipelines and derived datasets (including backups, caches, and search indexes) hold a copy of a given individual's data, and having an automatable way to purge or anonymize it across all of them, rather than discovering at request time that the data is scattered across a dozen systems with no inventory.

- **DON'T:** Assume anonymization achieved by simply removing obvious direct identifiers (name, email, SSN) from a dataset is sufficient to make it non-identifying. Well-documented re-identification techniques can often re-associate "anonymized" records with real individuals using a combination of remaining quasi-identifying fields (birth date, zip code, gender, in one well-known finding, uniquely identifies a majority of the U.S. population) — apply an actual anonymization methodology (k-anonymity, differential privacy, or simply not retaining the derived dataset at all) rather than assuming identifier removal alone is enough.

- **DO:** Set explicit, enforced data retention limits for pipelines that store historical raw data containing PII, deleting or further anonymizing data once it's past the retention period the business actually needs and any applicable regulation allows, rather than retaining every raw record indefinitely by default because storage is cheap.

### ETL vs ELT

- **DO:** Choose ELT (extract, load raw data into the warehouse first, then transform it there — typically with a tool like dbt) when the destination is a modern analytical warehouse with strong, scalable compute (Snowflake, BigQuery, Redshift) and the priority is flexibility to iterate on transformation logic without re-extracting from the source every time. Loading raw data first means transformation logic can be changed, re-run, and version-controlled independently of the (often slower, rate-limited) extraction step.

- **DON'T:** Default to ELT without considering the cost and governance implications of landing completely raw, unfiltered, unvalidated data (potentially including PII, secrets accidentally present in source data, or malformed records) into a shared warehouse before any validation happens. ELT's "load first, validate later" convenience is not free — apply access controls and at least basic validation to the raw landing zone, not only to the final transformed tables.

- **DO:** Choose traditional ETL (transform before loading) when the destination system has limited compute for transformations, when strict data validation/masking must happen before data ever lands in a shared destination (compliance-sensitive data, for instance), or when the source system can only be queried in a narrow, expensive way that makes doing more work per-extraction worthwhile.

- **DON'T:** Assume ELT eliminates the need for the same data validation and quality discipline that ETL enforces upfront — it only moves *when* that work happens, from before loading to after. An ELT pipeline that never validates the raw layer and only spot-checks transformed output can still let bad data sit undetected in the raw tables, which downstream ad hoc queries may read directly, bypassing the transformation layer's own checks entirely.

- **DO:** Preserve raw, untransformed data in its own clearly-labeled layer (a "bronze"/raw schema) even in an ELT pipeline, separate from cleaned/transformed ("silver") and business-ready aggregated ("gold") layers, so that a bug discovered in transformation logic can be fixed and the data re-transformed from the untouched raw layer, rather than needing to re-extract from the original source (which may have already changed or deleted the data).

### Testing Data Pipelines & Transformations

- **DO:** Write unit tests for individual transformation functions/logic using small, hand-crafted input datasets with known expected outputs, especially for any transformation with non-obvious business logic (currency conversion, deduplication rules, date-window boundaries). A transformation bug caught by a fast unit test costs minutes; the same bug caught only after it's corrupted a production table costs a very different order of magnitude of cleanup effort.
```pseudocode
test("deduplicates orders by order_id, keeping the latest updated_at"):
    input = [
        { order_id: 1, updated_at: "2026-01-01", status: "pending" },
        { order_id: 1, updated_at: "2026-01-02", status: "shipped" },
    ]
    result = deduplicateOrders(input)
    assert result == [{ order_id: 1, updated_at: "2026-01-02", status: "shipped" }]
```

- **DON'T:** Test a pipeline only end-to-end against a full copy of production data, with no isolated unit tests for individual transformation steps. End-to-end-only testing is slow, makes it hard to pin down exactly which step introduced a regression, and often can't cover edge cases (empty inputs, malformed records, boundary dates) that are rare in a production snapshot but still need to be handled correctly.

- **DO:** Use a data quality testing framework (dbt tests, Great Expectations, or equivalent) to declare and automatically check expectations about pipeline output — not-null constraints, uniqueness, referential integrity between derived tables, value ranges, expected row-count relationships between source and target — as part of the regular pipeline run, not as a separate manual step.
```sql
-- dbt schema test example: declaring expectations as data, checked automatically
-- models/schema.yml
-- - name: order_id
--   tests: [unique, not_null]
-- - name: total
--   tests:
--     - dbt_utils.accepted_range: {min_value: 0}
```

- **DON'T:** Treat a pipeline's tests as a one-time setup task, written once and never revisited as the pipeline's logic evolves. Transformation logic that changes without a corresponding test update leaves tests either failing for the wrong reason (blocking legitimate changes) or, worse, silently passing while no longer actually verifying the behavior that matters — review and update pipeline tests as part of every meaningful transformation-logic change.

- **DO:** Test schema evolution and malformed-input handling explicitly — feed the pipeline a record missing an expected field, a field with an unexpected type, or a schema version it hasn't seen before — and verify it fails safely (quarantines the record, alerts) rather than crashing the whole run or silently corrupting output. This is the class of failure real production data triggers far more often than clean, well-formed test fixtures would suggest.

### Slowly Changing Dimensions

- **DO:** Choose a slowly changing dimension (SCD) handling strategy deliberately per dimension attribute in a data warehouse, based on whether history needs to be preserved: Type 1 (overwrite the old value, no history kept) for corrections and attributes where history genuinely doesn't matter; Type 2 (insert a new row with validity date ranges, keeping the old row intact) when historical accuracy matters — e.g., "what was this customer's address when this order was placed."
```sql
-- SCD Type 2: a new address doesn't overwrite history, it closes the old
-- row's validity window and inserts a new current row
UPDATE dim_customer SET valid_to = now(), is_current = FALSE
WHERE customer_id = 42 AND is_current = TRUE;

INSERT INTO dim_customer (customer_id, address, valid_from, valid_to, is_current)
VALUES (42, '456 New St', now(), NULL, TRUE);
```

- **DON'T:** Default to Type 1 (overwrite) for dimension attributes where historical reporting accuracy actually matters. Overwriting a customer's region on every address change means a report of "revenue by region last year" silently uses this year's region for all historical transactions, quietly rewriting history and producing numbers that don't match what actually happened at the time.

- **DO:** Use a Type 2 dimension's validity date range (or a current-row flag alongside it) consistently when joining fact tables to dimensions for historical reporting — joining a fact table's transaction date against the dimension row that was valid *at that time*, not simply joining to whichever dimension row is currently marked "current."
```sql
-- Correct: join to the dimension row that was actually valid
-- at the time of the fact, not to today's current row
SELECT f.amount, d.address
FROM fact_orders f
JOIN dim_customer d
  ON d.customer_id = f.customer_id
  AND f.order_date BETWEEN d.valid_from AND COALESCE(d.valid_to, 'infinity');
```

- **DON'T:** Mix Type 1 and Type 2 handling inconsistently for related attributes on the same dimension without a clear, documented rule for which attributes get which treatment. An undocumented mix leaves every report author guessing whether a given dimension column reflects history-as-it-was or history-as-it-is-now.

- **DO:** Consider Type 3 (keep both the current and one previous value as separate columns on the same row) only for the narrow case where exactly one prior state needs to be compared against the current one, and prefer Type 2 for anything that might need more than a single step of history — Type 3 cannot represent more than the one prior value it was designed to hold.

### Dimensional Modeling: Star & Snowflake Schemas

- **DO:** Model analytical/warehouse data using a star schema — one central fact table holding measurable events (order line items, page views, transactions) with foreign keys out to surrounding dimension tables (customer, product, date, region) that describe the context of each fact — as the default pattern for BI and reporting workloads. A star schema is optimized specifically for the aggregate-heavy, filter-by-dimension queries that reporting and dashboards run constantly.
```sql
-- Star schema: one fact table, denormalized dimension tables around it
CREATE TABLE fact_sales (
  date_key INT REFERENCES dim_date(date_key),
  customer_key INT REFERENCES dim_customer(customer_key),
  product_key INT REFERENCES dim_product(product_key),
  quantity INT, revenue NUMERIC(12,2)
);
```

- **DON'T:** Model a data warehouse's fact and dimension tables in fully normalized (3NF) form the way an OLTP transactional schema would be modeled. A deeply normalized warehouse schema requires many more joins per report query, which directly fights the aggregate-query performance a warehouse exists to provide — deliberate, bounded denormalization within dimension tables is the expected, idiomatic warehouse design, not a shortcut.

- **DO:** Use a date dimension table (`dim_date`) with pre-computed calendar attributes (day of week, fiscal quarter, is_holiday, week number) rather than computing date logic repeatedly in every report query. A date dimension turns "revenue by fiscal quarter" into a simple join and filter instead of repeated date-arithmetic expressions scattered across dozens of queries.

- **DON'T:** Snowflake a dimension (normalizing a dimension table further into sub-dimension tables) without a specific reason — reduced storage for a very large, highly repetitive dimension attribute, or a genuine shared sub-dimension used by multiple dimension tables. Snowflaking adds joins back into queries a star schema was specifically designed to avoid; use it selectively, not as a default modeling habit carried over from OLTP normalization instincts.

- **DO:** Grain the fact table explicitly and consistently — decide and document exactly what one row represents (one order line item, one daily aggregate per customer) — since an ambiguous or inconsistent grain is the most common root cause of double-counted or incorrectly-aggregated warehouse metrics.

### Choosing File Formats for Data Interchange

- **DO:** Choose a columnar, typed, self-describing binary format (Parquet, ORC) for large analytical datasets that will be read repeatedly by query engines, over a row-oriented text format like CSV. Columnar formats let a query engine read only the columns a specific query needs, carry an embedded schema and type information (removing the "is this column a string or a number" guessing that plain CSV forces on every reader), and compress dramatically better than text.

- **DON'T:** Default to CSV for data interchange between systems just because it's simple and universally supported, without weighing its real costs: no native type information (every value is text until something downstream decides how to parse it), fragile handling of embedded delimiters/quotes/newlines within fields, no compression, and no schema enforcement — any of which can silently corrupt or misparse data if the writer and every reader don't agree by convention alone.

- **DO:** Use a schema-embedded format (Avro, Protocol Buffers, Parquet) for streaming or event-driven data interchange specifically because the schema travels with the data (or is referenced via a schema registry), which is what makes the schema-evolution compatibility guarantees discussed earlier actually enforceable, rather than merely hoped for.

- **DON'T:** Assume JSON is an adequate long-term storage format for large-scale structured data at rest just because it's convenient for APIs. JSON is a reasonable choice for interchange over an API or for genuinely variable/nested structures, but for large batch datasets it carries meaningful overhead (repeated field names in every record, no native compression, no columnar read pruning) compared to a purpose-built analytical format.

- **DO:** Consider the actual downstream consumer's needs when choosing a format — a format optimized for analytical scan performance (Parquet) is not the best choice for a system that needs low-latency single-record lookups (better served by a database or key-value store), and a human-readable format may be worth its overhead specifically for data that people need to inspect directly.

### Cost-Aware Pipeline Design

- **DO:** Design queries and pipeline stages against a cloud data warehouse to minimize the amount of data scanned — filtering on partition columns, using columnar formats that support column pruning, and avoiding `SELECT *` on wide tables — since many cloud warehouses (BigQuery, Snowflake, Redshift Spectrum, Athena) charge based on data scanned or compute consumed, meaning an unnecessarily broad query has a direct, measurable dollar cost, not merely a latency cost.
```sql
-- Scans the entire table's date column, in every partition, to filter —
-- expensive if the table isn't partitioned by date and the engine can't prune
SELECT * FROM events WHERE DATE(event_timestamp) = '2026-03-01';

-- Scans only the relevant partition, if the table is partitioned on this
-- column in a way the engine can prune against directly
SELECT * FROM events WHERE event_date = '2026-03-01';
```

- **DON'T:** Run exploratory or ad hoc analytical queries against full production-scale tables in a consumption-priced warehouse without first sampling or scoping the query to a smaller, representative slice of data, especially while iterating on a query's logic. Iterating with `LIMIT` and a narrow date range during development, then removing the limit for the final run, avoids paying the full-scan cost repeatedly for each iteration of getting the logic right.

- **DO:** Set up cost monitoring and alerting on data warehouse spend (per-query cost limits, budget alerts, or a cost-tracking dashboard by team/pipeline) so an unexpectedly expensive query or a runaway pipeline is caught quickly, rather than discovered at the end of a billing cycle as a surprise invoice.

- **DON'T:** Assume storage is the dominant cost in a cloud data platform and optimize accordingly while ignoring compute/query cost, which is very often the larger and more variable line item in a consumption-priced warehouse. A well-partitioned, well-compressed table still costs real money to scan repeatedly by inefficient queries — cost-aware design has to address both storage layout and query patterns together.

### Watermarks & Late-Arriving Data

- **DO:** Define an explicit watermark — a declared threshold for how late an event can arrive relative to its own timestamp and still be included in the window it logically belongs to — for any streaming or windowed-aggregation pipeline, and choose that threshold based on real, measured data about how late events actually arrive from the specific upstream source (mobile clients with intermittent connectivity arrive far later, on average, than server-side events).

- **DON'T:** Assume events arrive in the pipeline in the same order they were generated, or arrive within a "reasonable" time of their timestamp, without measuring the actual distribution. Mobile clients that buffer events while offline, retried failed deliveries, and clock skew on the producing device are all common, ordinary reasons an event's arrival time can lag its logical timestamp by minutes or hours — a windowed aggregation with no late-arrival handling will simply miscount or drop those events with no visible error.

- **DO:** Decide and document explicitly what happens to data that arrives after its watermark has already passed and its window has already been finalized — drop it, emit a correction/late-update to already-published results, or route it to a distinct "late data" destination for separate handling — rather than leaving the behavior as an unexamined default of whatever streaming framework is in use.
```pseudocode
// Explicit late-data policy, not an unexamined default
if event.timestamp < currentWatermark - allowedLateness:
    routeToLateDataTable(event)   // handled separately, not silently dropped
else:
    includeInWindow(event)
```

- **DON'T:** Set an unbounded or extremely generous lateness allowance "to be safe," without recognizing the real tradeoff — a longer lateness window means results for any given time window take longer to be considered final, which delays every downstream consumer that needs a "done" signal for that window. Tune the watermark to the actual, measured lateness distribution, not to an arbitrary conservative guess.

- **DO:** Emit and clearly label intermediate (non-final) results separately from finalized results, when a pipeline needs to provide both fast approximate numbers before the watermark passes and accurate final numbers afterward, so downstream consumers know which guarantee they're actually getting.

### Data Pipeline Deployment & Environment Promotion

- **DO:** Version-control pipeline transformation code (dbt models, Airflow/Dagster DAG definitions, Spark jobs) in the same way application code is version-controlled, with code review, CI checks, and a defined promotion path from development through staging to production — rather than editing transformation logic directly in a production orchestrator's UI or notebook.

- **DON'T:** Let a data analyst or engineer edit a scheduled production pipeline's logic directly through an orchestrator's web UI, bypassing version control and review entirely. A change made this way has no record of who changed what or why, no review before it affects production data, and no straightforward way to revert if it turns out to be wrong.

- **DO:** Run new or changed pipeline logic against a staging environment with representative (ideally production-scale or realistically sampled) data before promoting it to production, and compare its output against the current production pipeline's output for the same input to catch unintended behavioral differences before they reach real downstream consumers.

- **DON'T:** Promote a pipeline change to production without a rollback plan — the ability to revert to the previous version of the transformation logic quickly if the new version produces bad output. Since a bad pipeline deploy can corrupt data that downstream consumers then read and act on, the ability to roll back the *code* quickly is necessary but not sufficient — also plan for how already-corrupted output data gets identified and corrected.

### Handling Schema Drift from Third-Party & External Sources

- **DO:** Treat every third-party API, vendor data feed, or external file source as capable of changing its schema without warning, and build ingestion code that detects and reacts explicitly to an unexpected field, missing field, or changed type — rather than assuming the external contract, however well documented, will hold indefinitely. Vendors change their APIs, deprecate fields, and fix their own bugs in ways that can silently alter what they send, on their own schedule, with no coordination with downstream consumers.

- **DON'T:** Assume a third-party data source's documented schema matches what it actually sends in every case. Real-world API responses routinely include undocumented fields, omit documented-as-required fields in edge cases, or return a type that doesn't match the documentation for certain inputs — validate against the schema you actually observe in practice, and treat the documentation as a starting hypothesis, not a guarantee.

- **DO:** Alert distinctly and immediately when an external source's schema changes in a way ingestion code doesn't already handle (a new unexpected field, a field disappearing, a type change), so a human can assess the change quickly, rather than letting the ingestion code either crash opaquely or silently drop/misinterpret the changed data.

- **DON'T:** Hardcode assumptions about a third-party source's field order, exact casing, or undocumented implementation details that happen to be true today but aren't part of any actual guarantee. Code that works only because of an accidental, unguaranteed detail of the current API response is fragile in a way that's invisible until the vendor's next unrelated internal change breaks it.

- **DO:** Maintain a versioned record of exactly what schema was observed from each external source at each point in time (alongside the ingested data, or in a schema registry), so that when a downstream data quality issue is traced back to a source schema change, it's possible to confirm exactly when the change happened and what changed, rather than reconstructing it from memory or vendor changelogs after the fact.

### Data Versioning & Reproducibility

- **DO:** Snapshot or version datasets that feed reports, machine learning models, or any output that might need to be exactly reproduced or audited later, rather than allowing the "current" state of a mutable table to be the only representation of the data that ever existed. A report generated today from a table that's since been updated, corrected, or reprocessed cannot be exactly reproduced later without a way to reconstruct the data as it was at generation time.

- **DON'T:** Assume a "latest" table or view is sufficient for any use case that requires reproducibility — a regulatory report, a model training run whose results need to be explainable later, an A/B test analysis someone might need to re-verify. Without versioned or immutable snapshots, "what did the data look like when this decision was made" becomes unanswerable once the underlying table has moved on.

- **DO:** Use partition-based or timestamp-based dataset versioning (each pipeline run writes to a new, immutable, dated partition or table version rather than overwriting the previous one in place) for data that feeds reproducible processes, keeping enough historical versions to satisfy the actual reproducibility requirement, then aging out older versions per a defined retention policy.

- **DON'T:** Conflate "we have backups" with "we have dataset versioning for reproducibility." A backup is for disaster recovery (restoring the whole system to a point in time); dataset versioning is for being able to say precisely which version of a specific dataset fed a specific report or model — the two serve different purposes and typically need different mechanisms.

### Multi-Tenant Data in Shared Pipelines

- **DO:** Enforce tenant isolation explicitly within shared data pipeline infrastructure — tagging every record with its tenant ID from ingestion through every transformation stage, and scoping every downstream output (a shared warehouse table, a shared search index) by tenant — with the same structural discipline used for multi-tenant application schemas, rather than assuming pipeline code is inherently safe just because it's "just ETL."

- **DON'T:** Let a pipeline bug or a missing filter cause one tenant's data to leak into another tenant's output — a join that's missing a tenant-scoping condition, an aggregation that accidentally groups across tenants, or a shared cache/lookup table that isn't properly scoped. A cross-tenant data leak through a pipeline bug is just as serious a security and compliance incident as one through an application bug, even though it happens in batch/offline code that feels lower-stakes.

- **DO:** Test pipeline transformations explicitly with multiple tenants' data present in the same test run, specifically checking that no tenant's output includes any other tenant's data, rather than testing only with a single tenant's data where a missing isolation filter would produce no visible symptom at all.

- **DON'T:** Assume that because a pipeline runs "internally" and isn't directly user-facing, tenant isolation matters less there than in the application layer. Pipeline output frequently does become user-facing eventually (a report a customer downloads, a search index a customer's users query) — by the time a leak surfaces there, it's already happened upstream, further from where it's easy to trace and fix.

### Schema-on-Read vs Schema-on-Write

- **DO:** Choose schema-on-write (validating and enforcing structure at ingestion time, before data is stored — the traditional relational and data-warehouse approach) when downstream consumers need reliable, consistent guarantees about the data's shape without each one re-validating it independently. Enforcing structure once, at the point of entry, is far cheaper in aggregate than every consumer defensively re-checking and re-interpreting loosely structured data.

- **DON'T:** Adopt schema-on-read (storing raw, loosely-structured data in a data lake and interpreting/validating its structure only at query time) without accounting for the real cost this shifts onto every consumer. Schema-on-read's flexibility is genuinely valuable for exploratory or highly variable data, but it means every query against that data has to handle malformed or unexpected records itself — a cost schema-on-write would have paid once, centrally, instead of repeatedly, per consumer.

- **DO:** Use schema-on-read deliberately for the specific cases it's actually suited to — a raw landing zone for data whose eventual shape isn't fully known yet, exploratory data science work, or genuinely heterogeneous data where forcing a single schema upfront would lose information — rather than as a blanket policy adopted because it's less upfront work.

- **DON'T:** Let a schema-on-read raw zone become the de facto interface other teams query directly and build dependencies on, without it ever graduating to a validated, schema-enforced layer. A raw landing zone that was only ever meant to be an intermediate staging area, but that other teams started querying directly because it was convenient, inherits all of schema-on-read's downsides (every consumer handling malformed data itself) without anyone having decided that tradeoff deliberately.

### Duplicate Detection & Fuzzy Matching at Scale

- **DO:** Choose a deduplication/entity-resolution strategy appropriate to the actual matching requirement — exact-match deduplication (a straightforward `GROUP BY`/uniqueness check) for data with a reliable natural key, versus fuzzy/probabilistic matching (similarity scoring on name, address, and other attributes, often via a dedicated entity-resolution library or service) for data that represents the same real-world entity without a shared reliable identifier, such as reconciling customer records from two merged systems with no common ID.

- **DON'T:** Attempt exact-match deduplication on data where the "duplicate" relationship is inherently fuzzy (slightly different spellings of the same company name, addresses formatted differently across sources) and expect a simple `GROUP BY`/`DISTINCT` to find them. Exact matching will simply fail to identify these as duplicates at all, understating the actual duplication rate and leaving the fragmented records unmerged.

- **DO:** Run fuzzy matching at scale using blocking/indexing techniques (grouping records into smaller candidate buckets by an approximate key — first three letters of a name, a postal code — before doing expensive pairwise similarity comparison only within each bucket) rather than comparing every record against every other record, which scales quadratically and becomes infeasible past a fairly small dataset size.

- **DON'T:** Treat automated fuzzy-match results as ground truth to merge or delete records without human review for any deduplication with real consequences (merging customer accounts, deduplicating financial records). Fuzzy matching produces confidence scores, not certainties — an aggressive similarity threshold applied automatically can incorrectly merge two genuinely distinct entities that happen to look similar, which is a much harder mistake to reverse than leaving two true duplicates unmerged a little longer.

- **DO:** Preserve a record of which original records were merged into a deduplicated result (a crosswalk/mapping table linking the original IDs to the resulting canonical entity) so a merge decision can be audited, and if necessary reversed, rather than performing the merge destructively with no trace of the original, separate records.

### Currency, Units & Measurement Consistency Across Pipeline Sources

- **DO:** Store the unit or currency explicitly alongside every measured or monetary value ingested from multiple sources, and normalize to one canonical unit/currency at ingestion time (or make the unit an unavoidable part of every downstream calculation) rather than assuming every source reports in the same unit. A pipeline combining sales data from a US system (USD) and a European system (EUR) without an explicit currency field will silently sum incompatible values as if they were the same currency.
```sql
-- Explicit unit, not an assumption
CREATE TABLE revenue_events (
  amount NUMERIC(14,2) NOT NULL,
  currency CHAR(3) NOT NULL,   -- ISO 4217, e.g. 'USD', 'EUR' — never implicit
  event_time TIMESTAMPTZ NOT NULL
);
```

- **DON'T:** Assume every upstream source reports measurements in the same unit system without verifying it explicitly per source. Mixing metric and imperial measurements (kilograms vs. pounds, kilometers vs. miles), or mixing currencies, without normalizing or explicitly tagging each value's unit, produces aggregates that are silently, arbitrarily wrong — and wrong in a way that often looks plausible enough not to be caught by casual inspection.

- **DO:** Apply currency conversion (or unit conversion) using a rate captured at a specific, recorded point in time relevant to the transaction (the exchange rate at the time of the transaction, not today's rate), and store which rate and rate-date was used, for any pipeline that needs to reconcile amounts to a single reporting currency across historical data — using today's exchange rate to convert a transaction from a year ago produces a number that doesn't match what actually happened financially at that time.

- **DON'T:** Silently drop or coerce a value's unit/currency field when it doesn't match an expected default during ingestion, on the assumption that "it's probably the same as everything else." An ingestion pipeline that encounters an unexpected currency code should flag and quarantine that record for review, per the data validation discipline covered earlier — not silently assume it's a data entry mistake and proceed as if it were the default currency.

### Incremental vs Full Refresh Loading Strategies

- **DO:** Choose incremental loading (extracting and processing only records that are new or changed since the last run, tracked via a high-water-mark column like `updated_at` or a CDC stream) for large source tables where a full reload would be prohibitively slow or expensive, and full-refresh loading (truncate and reload everything, every run) for small reference/dimension tables where simplicity and guaranteed correctness matter more than efficiency and the full reload cost is negligible.
```sql
-- Incremental load: only pull what's changed since the last successful run
SELECT * FROM source_orders WHERE updated_at > :last_watermark;
```

- **DON'T:** Use a naive `updated_at`-based watermark for incremental loading without also handling hard deletes (which have no `updated_at` to detect) and clock skew between the source system and the pipeline (which can cause a record updated right at the watermark boundary to be missed or double-counted) — the same limitations already discussed for polling-based change detection apply directly here.

- **DO:** Periodically run a full reconciliation pass (or a full reload) alongside an incremental pipeline, even when incremental loading is the normal operating mode, to catch drift that accumulates from edge cases the incremental logic doesn't perfectly handle — a scheduled weekly full-refresh safety net alongside daily incremental loads is a common, pragmatic hybrid.

- **DON'T:** Assume an incremental pipeline that's run correctly every day for months has no accumulated drift just because no error has ever been reported. Small, silent misses in incremental logic (a boundary condition on the watermark, a delete that was never captured) compound over time without ever producing a visible failure — this is exactly why a periodic reconciliation or full-refresh check matters even for a pipeline with a clean error history.

## Common AI-Assistant Mistakes in Data Code

### Inventing SQL Dialect Features

- **DO:** Confirm which specific database engine and version a query targets before writing anything beyond basic ANSI SQL, and check that engine's actual documentation for dialect-specific syntax rather than generating syntax that "seems right" from general SQL familiarity. `LIMIT`, `TOP`, `FETCH FIRST`, and `ROWNUM` are four different, mutually incompatible ways different databases express "give me N rows," and using the wrong one produces a syntax error, not a warning.
```sql
-- PostgreSQL / MySQL / SQLite
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- SQL Server
SELECT TOP 10 * FROM orders ORDER BY created_at DESC;

-- Oracle (12c+)
SELECT * FROM orders ORDER BY created_at DESC FETCH FIRST 10 ROWS ONLY;
```

- **DON'T:** Generate a function call or syntax construct that belongs to a different database engine than the one the user is actually targeting, presented as if it's universal — `STRING_AGG` (PostgreSQL/SQL Server) versus `GROUP_CONCAT` (MySQL) versus `LISTAGG` (Oracle) for string aggregation, or PostgreSQL's `RETURNING` clause (not available in MySQL or SQL Server the same way), or MySQL's `ON DUPLICATE KEY UPDATE` versus PostgreSQL's `ON CONFLICT DO UPDATE`. These are not interchangeable spellings of the same feature; assuming they are produces code that fails outright against the real target database.

- **DO:** Flag explicitly, in the response, when a requested feature genuinely does not exist in the target database (e.g., MySQL has no native array type the way PostgreSQL does; SQLite has no `RIGHT JOIN` before a certain version and limited `ALTER TABLE` support) rather than silently substituting invented syntax that will fail. State the actual limitation and offer the real workaround the engine supports.

- **DON'T:** Assume a feature available in one major version of a database is available in whatever version the user actually runs. Window function support, `JSON` column types, `CHECK` constraint enforcement (MySQL only started enforcing `CHECK` constraints by default in 8.0.16), and generated columns all have real version cutoffs — verify or explicitly caveat the assumed version rather than presenting version-specific syntax as universally available.

- **DO:** Distinguish between a database's core engine capabilities and the capabilities of a specific hosted/managed variant (Amazon Aurora's MySQL/PostgreSQL compatibility layers, PlanetScale's Vitess-based MySQL, CockroachDB's PostgreSQL wire compatibility) which can each have meaningful gaps or differences from the upstream engine they're compatible with — don't present every PostgreSQL feature as automatically available on every "PostgreSQL-compatible" system.

- **DON'T:** Fabricate a plausible-sounding function or clause name when unsure whether the specific dialect supports the requested operation, rather than checking or saying so. A confidently-generated `SELECT DATE_TRUNC_WEEK(created_at)` for a database that has no such function wastes the user's time discovering it doesn't exist, when a direct "I'm not certain this engine supports that; here's the closest verified equivalent" would have been more useful.

### Migrations That Lock Large Tables in Production

- **DO:** Check, before generating a schema-altering statement for a table the user indicates is large or actively written, whether that specific operation requires a full table rewrite/exclusive lock on the target database and version, and generate the non-blocking variant (concurrent index creation, `NOT VALID` + `VALIDATE CONSTRAINT`, expand/contract for column changes) by default rather than the naive blocking form.

- **DON'T:** Generate `ALTER TABLE ... ADD COLUMN ... NOT NULL` with no default for an existing large table without flagging that on many database versions this requires rewriting every existing row under a lock, or without offering the safe nullable-then-backfill-then-constrain sequence instead. Presenting the single-statement blocking form as the standard way to add a required column is a common way AI-generated migrations cause a production incident.

- **DO:** Default to `CREATE INDEX CONCURRENTLY` (PostgreSQL) or the equivalent online/non-blocking DDL mechanism for the target database when generating an index-creation migration for a table described as large or high-traffic, rather than plain `CREATE INDEX`, and explain the tradeoff (concurrent builds are slower and can't run inside a transaction) rather than silently picking the blocking form because it's simpler to write.

- **DON'T:** Generate a migration that combines a schema change with a data backfill of unknown size in a single statement or single transaction, on a table whose size isn't known, without asking about table size or explicitly flagging the risk. `UPDATE large_table SET new_col = compute(old_col)` with no `WHERE` clause and no batching, run as part of a schema migration, is one of the most common causes of an AI-generated migration locking a production table for an unacceptable duration.

- **DO:** Ask about (or explicitly caveat the assumption about) approximate table size and write traffic before recommending a specific migration strategy, when that information isn't already given, since the correct approach for a 10,000-row table (any straightforward statement is fine) is often actively wrong for a 500-million-row table (needs batching, online DDL, and a rollout plan).

- **DON'T:** Recommend `ALTER TABLE ... RENAME COLUMN` or a column type change as a single-step operation without asking whether other, independently-deployed application instances or services are still reading/writing the old column name/type — a rename that's perfectly safe on a database with no live traffic is a production outage waiting to happen against a database serving a currently-running application during a rolling deploy.

### Ignoring Existing Schema & Duplicating Logic

- **DO:** Read the actual current schema (via the provided migration files, an ORM's model definitions, or a direct schema introspection query) before generating a new table, column, index, or query against it, rather than inferring the schema from the conversation's context or from what a "typical" schema for this kind of application usually looks like. A generated query against a guessed schema fails immediately, or worse, silently targets the wrong column/table if a same-named-but-differently-typed column happens to exist.

- **DON'T:** Propose adding a new column, table, or index without first checking whether an equivalent already exists under a different name, or whether the existing schema already has a way to derive the same information. Duplicating an existing `status` column as a new `order_status` column because the existing one wasn't checked creates two sources of truth for the same fact, with no mechanism keeping them in sync.

- **DO:** Search the existing codebase for similar queries, existing repository/service methods, or existing indexes before writing new data-access code from scratch, and reuse or extend what's already there rather than writing a parallel, slightly different implementation of the same logic. A second, independently-written "get active orders for a customer" query that filters slightly differently from the existing one is a subtle correctness bug waiting to surface as two different numbers shown in two different parts of the same application.

- **DON'T:** Generate a migration that recreates a table, column, or constraint that a prior migration already created, without checking the migration history first. This is especially likely when working from a truncated view of the codebase or a summarized description of the schema rather than the actual migration files — always ground schema changes in the real, current migration history, not a remembered or inferred prior state.

- **DO:** Check for and respect existing naming conventions, ID generation strategies (auto-increment vs. UUID), and soft-delete patterns already established in the schema before introducing a new table that does it differently. A new table using a UUID primary key when every other table in the schema uses auto-increment integers (or vice versa) is a consistency regression, not a neutral choice, even if UUIDs are "generally a fine choice" in isolation.

- **DON'T:** Assume a column exists with a particular name, type, or nullability based on common convention (assuming every table has `created_at`, assuming a `users` table has an `email` column that's unique, assuming money is stored in a `NUMERIC` column) without verifying against the actual schema when it's available to check. Confident guesses about schema shape that turn out wrong produce code that either fails outright or, worse, silently references the wrong column if a similarly-named one happens to exist.

### SQL Injection via String Concatenation

- **DO:** Always generate parameterized queries (bound parameters/placeholders) for any query that incorporates a value from outside the hard-coded query text — user input, another service's response, configuration read at runtime — rather than string-concatenating or f-string-interpolating that value directly into the SQL text. Parameterization is not a style preference; it is the actual, complete defense against SQL injection, and no amount of manual "sanitization" of a concatenated string reliably replaces it.
```python
# NEVER generate this pattern — classic SQL injection vulnerability
query = f"SELECT * FROM users WHERE email = '{user_input}'"
cursor.execute(query)

# ALWAYS generate this pattern — the driver handles escaping correctly
cursor.execute("SELECT * FROM users WHERE email = %s", (user_input,))
```

- **DON'T:** Present string concatenation as an acceptable shortcut "for a quick script" or "since this is just internal tooling." Internal tools and one-off scripts get copy-pasted into production code, get exposed via an unexpected internal API, or simply process user-influenced data eventually — the discipline of parameterized queries should be the default in every context, not a production-only requirement introduced later.

- **DO:** Use parameterized queries even for values that seem inherently "safe" from injection at first glance — numeric IDs, enum-like status strings, dates — since the actual risk isn't limited to values that obviously look like attacker-controlled strings; a numeric-looking field can still originate from user input that wasn't validated as strictly numeric before reaching the query.

- **DON'T:** Attempt to build dynamic `WHERE` clauses, `ORDER BY` columns, or table/column names by concatenating user-supplied strings, even when parameterization is used for the *values*. Parameter placeholders can bind values, not identifiers (column names, table names, sort direction) — dynamic identifiers must be validated against a fixed allow-list of known-safe names, never interpolated directly from user input, because no query parameter mechanism protects against injection through an identifier position.
```python
# BAD: sort_column comes from user input and is concatenated directly —
# an attacker can inject arbitrary SQL here even though '?' is used elsewhere
query = f"SELECT * FROM products ORDER BY {sort_column} LIMIT ?"

# GOOD: validate the identifier against a fixed allow-list first
ALLOWED_SORT_COLUMNS = {"name", "price", "created_at"}
if sort_column not in ALLOWED_SORT_COLUMNS:
    raise ValueError("invalid sort column")
query = f"SELECT * FROM products ORDER BY {sort_column} LIMIT ?"
```

- **DO:** Use the query-building/ORM layer's own parameter-binding methods when generating code through an ORM or query builder, rather than dropping into raw string-built SQL "for convenience" inside otherwise-parameterized ORM code. An ORM makes it easy to accidentally introduce one raw, concatenated fragment (a raw `.where()` clause with a format string) inside code that otherwise looks safe.

- **DON'T:** Generate example/tutorial/demo code that uses string concatenation for SQL "to keep the example simple," even with an explicit disclaimer. Example code gets copied into real projects far more often than the disclaimer gets read — model the secure pattern by default in every example, not only in production-labeled code.

### Assuming a Schema Without Checking the Actual One

- **DO:** Ask for or actively retrieve the real schema (via `\d table_name`, `SHOW CREATE TABLE`, an ORM's model files, an actual migration history, or a schema-introspection query) before writing any nontrivial query or migration against an existing database, rather than proceeding from an assumed or remembered schema shape. The cost of checking is a single query; the cost of a wrong assumption is a failed deploy, a silent bug, or — for a destructive statement — real data loss.

- **DON'T:** Carry forward an earlier version of a schema described or seen previously in the conversation as still accurate, once meaningful time or unrelated work has passed, without re-verifying it. Schemas change via migrations that may have run outside the current conversation entirely; treat a previously-seen schema as a snapshot that can go stale, not as a permanently valid source of truth.

- **DO:** Explicitly state the assumption when the actual schema genuinely isn't available to check ("assuming a standard `users` table with `id`, `email`, `created_at` — please confirm this matches your actual schema") rather than presenting a guessed schema as verified fact. Naming the assumption lets the user catch a wrong guess before it's built on top of.

- **DON'T:** Generate a data migration or backfill script that assumes a specific data distribution or absence of edge cases (assuming every row has a non-null value in a column that's actually nullable, assuming a date column never contains a sentinel value like `'0000-00-00'`, assuming referential integrity holds when it might not have been enforced historically) without checking. Production data accumulated over time reliably contains edge cases that a freshly-designed schema assumption doesn't anticipate.

- **DO:** Treat differences between a local/development database and the actual production schema as a real possibility, particularly in codebases where migrations might be applied inconsistently across environments — verify against the environment the code will actually run against, not only against a conveniently available local copy.

- **DON'T:** Assume the presence of indexes, constraints, or specific column types based on what "should" be there for good design, when generating a query whose performance or correctness depends on that assumption (e.g., assuming a foreign key column is indexed, assuming a `status` column has a `CHECK` constraint limiting its values). Good design isn't guaranteed just because it would be a reasonable choice — verify what's actually enforced before relying on it.

### Fake/Mock Data Leaking Into Seed Scripts

- **DO:** Clearly separate and label test/fixture data generation from production seed data, using distinct scripts, distinct naming, and (where possible) a distinct environment guard, so that mock data intended purely for local development or automated tests cannot accidentally run against a staging or production database.
```pseudocode
// Guard against accidental execution outside intended environments
if environment == "production":
    raise Error("seed_test_fixtures.py must never run in production")
```

- **DON'T:** Generate placeholder/mock data using patterns that resemble real-looking personal information (plausible-looking real names, real-format but fabricated SSNs/credit card numbers, realistic-looking email addresses at real domains) without using an established, clearly-fake convention. Realistic-looking fake data is exactly the kind of data most likely to be mistaken for real records if it leaks into a shared or production environment, and most likely to violate a data classification/compliance policy if it does.

- **DO:** Use obviously-fake, clearly-flagged placeholder values for generated test data — reserved example domains (`example.com`, `example.org` per RFC 2606), a distinct clearly-fictional naming pattern, sentinel values, or test-card numbers officially designated by payment processors for testing — rather than data that could pass as real.
```sql
-- GOOD: unambiguously fake, uses reserved test domains and clearly-marked names
INSERT INTO users (name, email) VALUES ('Test User One', 'test.user1@example.com');

-- AVOID: looks enough like a real record that it could be mistaken for one
-- if it ends up somewhere it shouldn't
INSERT INTO users (name, email) VALUES ('Sarah Mitchell', 'sarah.mitchell1988@gmail.com');
```

- **DON'T:** Write a seed script that both creates realistic test data *and* is wired into the same migration/deployment pipeline that runs against production, without an explicit environment check. A seed script meant only for local onboarding that silently also runs during a production deploy is how demo/test accounts, fake transactions, or placeholder records end up live in a real system.

- **DO:** Generate a bounded, clearly-sized amount of seed/fixture data appropriate for its purpose (a handful of rows for local development ergonomics, a larger realistic-scale set specifically for load/performance testing) rather than an arbitrary amount that doesn't match either purpose, and document what the seed data is for so a future reader isn't left guessing whether it's safe to keep, modify, or delete.

- **DON'T:** Hard-code secrets-shaped values (API keys, tokens, passwords) into seed data, even fake ones formatted like real credentials, without a clear "this is not a real secret" marker. A fake-but-realistic-looking API key in a seed script can trigger false-positive secret-scanning alerts, or worse, get mistaken for a real leaked credential during an incident investigation, wasting responder time.

### Hallucinated ORM & Library APIs

- **DO:** Verify that a specific ORM method, query-builder function, or migration DSL call actually exists in the exact version of the library the project uses, rather than generating a plausible-sounding method name based on the general shape of similar libraries. ORMs (SQLAlchemy, Prisma, TypeORM, Sequelize, ActiveRecord, Django ORM) each have their own specific method names for similar operations, and confusing them (`prisma.user.findOne` vs. the current `findUnique`, or inventing a `.batchUpsert()` that doesn't exist) produces code that fails at runtime, not at generation time.

- **DON'T:** Blend method names or configuration options across different ORMs or across different major versions of the same ORM as if they're interchangeable. A method that existed in an ORM's older major version and was renamed or removed in a newer one (a common occurrence across most actively-developed ORMs) will produce a runtime `AttributeError`/`undefined is not a function` that looks like a typo but is actually a version mismatch.

- **DO:** Check the project's actual dependency manifest (`package.json`, `requirements.txt`/`pyproject.toml`, `Gemfile`, etc.) for the exact installed version of the database library/ORM/driver before relying on version-specific syntax or recently-changed APIs, since a codebase pinned to an older version won't have access to a newer version's methods even if that's what most current documentation shows.

- **DON'T:** Invent a configuration option, connection string parameter, or migration command flag that sounds plausible but doesn't exist in the actual tool (a made-up `--safe-mode` flag for a migration CLI, a nonexistent `pool.strategy` config key). A confidently-presented but fabricated option is worse than admitting uncertainty, because it looks correct until someone tries to actually use it.

- **DO:** Distinguish clearly between what the raw database driver supports, what the query builder layer supports, and what the full ORM layer supports, since a project may use these in combination and a feature available at one layer isn't automatically available or spelled the same way at another.

### Ignoring Transaction Boundaries

- **DO:** Explicitly decide and state where a transaction begins and ends when generating multi-step data-modifying code, rather than leaving transaction boundaries implicit or unaddressed. Multi-statement operations that should be atomic (deduct inventory, create an order, charge a payment) need an explicit transaction wrapping them; presenting them as separate, unwrapped statements silently drops the atomicity guarantee the operation actually needs.

- **DON'T:** Generate a sequence of related `INSERT`/`UPDATE` statements across multiple tables with no transaction wrapper, when the operation is clearly one logical unit that should succeed or fail together. This produces code that appears to work correctly in every normal test run and only reveals its flaw when a failure occurs partway through in production, leaving inconsistent data with no automatic recovery.

- **DO:** Flag when a requested multi-step operation spans a boundary a single database transaction cannot cover — a database write plus a call to an external payment API plus a write to a different database — and recommend the correct pattern for that situation (the outbox pattern, a saga, idempotency keys plus compensating actions) rather than presenting a single `BEGIN`/`COMMIT` as if it could make the whole cross-system sequence atomic, which it cannot.

- **DON'T:** Wrap unrelated, independent operations into a single unnecessary shared transaction "for safety." Bundling operations that don't actually need atomicity with each other into one transaction increases lock duration and blast radius (an unrelated failure now rolls back things that didn't need to be tied to it) without adding any real correctness benefit.

- **DO:** Call out explicitly when generated code runs inside a context (a web framework request handler, an ORM's unit-of-work pattern, a background job framework) that already manages transaction boundaries automatically, so the reader knows whether to add an explicit transaction or whether one is already implicitly present — silently assuming either way produces either redundant nested transactions or a missing one.

### Silent Truncation & Type Coercion

- **DO:** Flag a mismatch between a value's expected size/precision and the target column's declared size/precision explicitly, rather than generating an `INSERT`/`UPDATE` statement that will silently truncate or round the value on databases/modes where that's the default behavior. A `VARCHAR(50)` column receiving a 60-character string, or a `NUMERIC(10,2)` column receiving a value with more decimal precision, can silently lose data depending on the database's strict-mode configuration.

- **DON'T:** Assume every database rejects out-of-range or oversized values by default. Some databases and configurations (notably MySQL outside strict SQL mode, in older defaults) silently truncate strings and coerce out-of-range numeric values instead of raising an error — generated code that relies on "the database will catch this" needs to actually confirm that assumption holds for the specific database and mode in use, or validate the value in application code before it ever reaches the database.

- **DO:** Be explicit and correct about type coercion across language/database type boundaries — how the application language's date/time, decimal, and boolean types map to the database's native types — rather than letting an implicit conversion happen and hoping it's correct. A language's native float type silently used to populate a `NUMERIC` money column, or a naive local datetime silently written to a `TIMESTAMPTZ` column without explicit timezone handling, are both common silent-coercion bugs.

- **DON'T:** Generate code that compares or joins on values of visibly different types (a string column joined against an integer column, relying on implicit cross-type comparison) without an explicit, deliberate cast. Implicit type coercion rules differ by database and can silently produce unexpected results, unexpectedly prevent an index from being used, or in stricter databases, simply raise an error — an explicit `CAST`/`::type` documents the intent and avoids relying on engine-specific coercion behavior.

- **DO:** Preserve full timestamp precision and timezone information through every stage of a pipeline or query chain, and flag explicitly whenever a step would lose it (casting `TIMESTAMPTZ` to `DATE`, converting to a naive datetime, truncating sub-second precision) rather than letting it happen as an unremarked side effect of a type conversion.

### Over-Recommending Complexity

- **DO:** Recommend the simplest architecture that satisfies the stated requirements — a single well-indexed relational table before a sharded multi-database setup, a scheduled batch job before a streaming pipeline, a cache-aside pattern before a custom multi-tier caching system — and only introduce additional complexity when the user's actual described scale, consistency, or latency requirements genuinely demand it. A recommendation sized for a requirement the user doesn't have adds real implementation and operational cost for no corresponding benefit.

- **DON'T:** Default to recommending NoSQL, sharding, microservice-per-table, or a specialized store (search engine, time-series database, graph database) whenever a database question comes up, treating "modern, distributed, specialized" as inherently better advice than "boring, relational, single-instance." Match the recommendation to evidence about actual scale and access patterns the user has provided, not to which architecture sounds most sophisticated in an answer.

- **DO:** Ask about actual scale (rows, writes per second, read/write ratio, latency requirements) before recommending an architecture change motivated by scale, when that information isn't already given, rather than assuming the user's traffic justifies a complex solution. Most schema and query problems brought to an AI assistant are solvable with an index, a query rewrite, or a bounded pagination fix — not a wholesale architecture migration.

- **DON'T:** Present a sophisticated-sounding solution (a distributed transaction saga, a custom write-behind cache, a bespoke sharding scheme) as the default answer to a problem a standard, well-tested built-in feature already solves adequately (a database's native `UPSERT`, a standard connection pool, an existing managed caching service). Reinventing infrastructure the ecosystem already provides, reliably, is a common way generated advice quietly increases a project's maintenance burden.

- **DO:** Explicitly name the tradeoff being introduced whenever recommending a more complex approach — "this adds eventual consistency," "this requires an additional service to operate," "this makes local development harder" — so the user can weigh the recommendation with the real cost visible, not just the benefit.

### Excessive or Insufficient Privileges in Generated DDL

- **DO:** Generate database user/role creation statements with the minimum privileges the stated use case actually requires, by default, rather than reflexively granting broad or superuser-level access because it "will definitely work" for whatever the code needs to do. Defaulting to least privilege in generated code is the same principle as defaulting to parameterized queries — the safe choice should be the path of least resistance, not an opt-in the user has to remember to ask for.
```sql
-- AVOID generating this by default for an application's normal DB user
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';

-- GENERATE this instead, scoped to what the application actually needs
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'%';
```

- **DON'T:** Generate a connection string or role grant with a wildcard host (`'%'` in MySQL, an unrestricted `pg_hba.conf` entry) or public network exposure as the default example, without at least flagging that this should be restricted in any real deployment. Presenting an insecure-by-default example as the standard pattern means it gets copy-pasted into real, internet-facing systems.

- **DO:** Flag explicitly when a generated migration or setup script grants a new privilege beyond what previously existed, so a reviewer notices the privilege expansion during review rather than it blending in silently among routine schema changes.

- **DON'T:** Generate example code that embeds real-looking credentials, connection strings with plaintext passwords, or API keys directly in SQL/migration files or seed scripts, even as a "replace this with your own" placeholder that looks like a genuine secret. Use an obviously non-functional placeholder format and point to environment variables or a secrets manager as the actual mechanism, so the example doesn't normalize hardcoding real secrets into version-controlled files.

### Proposing Migrations Where a Query or Index Fix Would Do

- **DO:** Reach for a targeted fix — a new index, a query rewrite, an added `LIMIT`, a corrected join — as the first response to a reported slow query or correctness bug, and reserve a schema migration for cases that genuinely require a structural change. A slow query caused by a missing index needs an index, not a denormalization project or a new caching layer bolted on top of the symptom.

- **DON'T:** Jump directly to "add a denormalized column" or "add a new table" in response to a performance complaint before ruling out a missing index or a bad query plan via `EXPLAIN`. Recommending a schema change before diagnosing the actual cause risks solving the wrong problem while leaving the real one (and its underlying missing index, which would also help every other query touching those columns) untouched.

- **DO:** Distinguish clearly, in the response, between "this needs a migration" and "this needs a query change" so the user understands the actual scope and risk of the proposed fix — a query rewrite is typically zero-risk and instantly reversible; a migration carries deployment, locking, and rollback considerations that are a different category of change entirely.

- **DON'T:** Propose a schema change that duplicates data (a new denormalized column, a new summary table) to solve a performance problem without first checking whether a well-chosen index or a materialized view would solve the same problem with less ongoing write-side maintenance burden and less risk of the duplicate falling out of sync.

### Silently Changing Query Semantics While "Optimizing"

- **DO:** Verify that a rewritten or "optimized" query returns the same results as the original for the same inputs — including edge cases around `NULL` handling, duplicate rows, and ties in sort order — before presenting the rewrite as a drop-in performance improvement. A faster query that returns different results is not an optimization; it's a different query wearing an optimization's clothing.
```sql
-- Original: LEFT JOIN preserves customers with zero orders
SELECT c.name, COUNT(o.id) FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id GROUP BY c.name;

-- "Optimized" rewrite that silently changes semantics: an inner JOIN
-- drops every customer with zero orders — not the same result set,
-- even though it may run faster and look like a reasonable simplification
SELECT c.name, COUNT(o.id) FROM customers c
JOIN orders o ON o.customer_id = c.id GROUP BY c.name;
```

- **DON'T:** Swap `DISTINCT` for `GROUP BY` (or the reverse), change a subquery to a join, or replace `NOT IN` with `NOT EXISTS` (or vice versa) purely for a perceived performance or style benefit without checking whether `NULL` handling, duplicate-row behavior, or short-circuit semantics differ between the two forms for the actual data in question. Several of these pairs are equivalent only under specific conditions (no `NULL`s in the relevant column, no duplicate keys) that don't automatically hold.

- **DO:** Call out explicitly, when proposing a query rewrite for performance, exactly what (if anything) changes about the result semantics — "this returns the same rows in the same order," or "this drops rows where X is null, which the original included; confirm that's acceptable" — rather than presenting every rewrite as a pure performance win with no behavioral difference.

- **DON'T:** Reorder `UNION` to `UNION ALL` as a performance shortcut without checking whether the original relied on `UNION`'s deduplication behavior. `UNION ALL` is faster because it skips deduplication, which is exactly the wrong tradeoff if the query's correctness depended on duplicates being removed.

- **DO:** Re-run (or ask the user to re-run) both the original and the rewritten query against representative data and diff the results whenever the rewrite is non-trivial, rather than relying solely on manual reasoning about equivalence — manual reasoning about SQL semantic equivalence is genuinely error-prone, including for an AI assistant, and an actual result diff is a much stronger check than confidence alone.

### Fabricating Performance Claims Without Evidence

- **DO:** Base any specific performance claim ("this will be 10x faster," "this reduces query time from 2s to 200ms") on an actual measurement — a real `EXPLAIN ANALYZE` run, a benchmark against representative data — or phrase it as a qualitative, hedged expectation ("this should reduce full-table scans, which is typically a significant improvement on a table this size") when no measurement is available. A specific fabricated number is more misleading than an honest qualitative statement, because its precision implies evidence that doesn't exist.

- **DON'T:** State a specific before/after latency, throughput, or row-count number for a change that was never actually run or measured against the real schema and data. A confidently-stated fake benchmark number can end up in a PR description or a design doc as if it were real evidence, misinforming a decision that deserved actual data.

- **DO:** Recommend the specific verification step the user should run to confirm a proposed change's actual impact (a before/after `EXPLAIN ANALYZE`, a load test, a staging benchmark) rather than substituting a confident-sounding claim for that verification.

- **DON'T:** Present a general, well-known truth about database performance ("indexes speed up lookups," "batching reduces round trips") with fabricated specific numbers attached, as if the general principle's direction of effect also validates a specific, unmeasured magnitude. The direction of a well-established principle can be stated with confidence; its magnitude for this specific query, on this specific data, cannot be, without measuring it.

### Recommending Destructive "Cleanup" Without Confirming Business Rules

- **DO:** Ask explicit clarifying questions before proposing a deletion, merge, or "cleanup" of data that looks like duplicates, orphans, or dead rows (duplicate customer records, orders with no matching customer, old rows past some apparent cutoff) rather than assuming the apparent anomaly is safe to remove. What looks like a duplicate or an orphan from a schema-only view can be a deliberate business pattern (multiple customer records intentionally kept separate for legal entities, orders intentionally retained past a deletion cutoff for a specific compliance reason) that isn't visible from the data alone.

- **DON'T:** Generate a `DELETE`/merge script for "duplicate" or "orphaned" data based purely on a pattern match, and present it as safe to run, without flagging what assumption it makes and what could go wrong if that assumption is incorrect. A script that looks technically correct but is built on a wrong business assumption is exactly the kind of AI-generated mistake that causes real, hard-to-reverse data loss when run with confidence and without independent review.

- **DO:** Propose a non-destructive first step — flag/mark candidate rows, or move them to a quarantine/review table — for any data-cleanup task where the actual disposition (delete, merge, keep) genuinely depends on business context the available schema and data can't fully answer, so a human with that context can review and confirm before anything is actually removed.

- **DON'T:** Treat "the data looks messy" as sufficient justification for a cleanup operation on its own. Messy-looking data is very often working as intended for reasons not visible in the schema (soft-deleted rows kept for audit, historical records intentionally inconsistent with current validation rules because they predate them) — recommend investigation and confirmation over immediate remediation whenever the reason for the apparent mess isn't already established.

### Ignoring Case Sensitivity & Collation Rules

- **DO:** Check the target database's actual case-sensitivity and collation configuration before generating a string comparison, `WHERE` clause, or uniqueness constraint that assumes case-insensitive matching — or assumes case-sensitive matching — since this genuinely differs by engine and by configuration: MySQL's default collations are typically case-insensitive for comparisons, PostgreSQL's default `=` comparison is case-sensitive, and SQL Server's default collation is usually case-insensitive, all of which can silently change whether `WHERE email = 'Foo@Example.com'` matches a row stored as `'foo@example.com'`.
```sql
-- Case sensitivity is NOT the same by default across engines —
-- verify rather than assume, and be explicit when it matters:
SELECT * FROM users WHERE LOWER(email) = LOWER(:input);  -- explicit, portable
```

- **DON'T:** Generate a uniqueness constraint on a text column (like email) without considering whether the intended uniqueness rule is case-sensitive or case-insensitive, and without checking whether the column's actual collation matches that intent. A `UNIQUE` constraint on a case-sensitive collation will happily accept both `alice@example.com` and `Alice@Example.com` as two distinct "unique" values, which is very often not what the business rule actually means.

- **DO:** Verify how the specific database sorts and compares Unicode text — collation affects not only case sensitivity but accent sensitivity, locale-specific sort order (does "ö" sort near "o" or at the end of the alphabet?), and how `ORDER BY` behaves on non-ASCII text — rather than assuming a generic "alphabetical" sort behaves identically everywhere.

- **DON'T:** Assume identifier (table/column name) case handling is consistent across databases when generating DDL. PostgreSQL folds unquoted identifiers to lowercase; some other systems fold to uppercase or preserve case as typed — a generated `CREATE TABLE "Users"` (quoted, case-preserved) versus `CREATE TABLE Users` (folded per the engine's rule) can silently produce a table that later unquoted references can't find, purely due to a case mismatch introduced by inconsistent quoting.

- **DO:** Flag explicitly when a generated query's correctness depends on an assumption about case sensitivity or collation that hasn't been verified against the actual target database and column configuration, rather than presenting the query as unconditionally correct.

### Generating Non-Idempotent Retry Logic for Data Operations

- **DO:** Check whether an operation is actually idempotent before wrapping it in automatic retry logic, and if it isn't, either make it idempotent first (an idempotency key, an upsert instead of an insert, a conditional update) or make the retry logic itself safe (checking current state before retrying, rather than blindly repeating the original call). Generating a generic "retry on failure" wrapper around a database write and applying it universally, without checking what that specific write does on a second execution, is a direct path to duplicate records, double charges, or double-decremented inventory.
```pseudocode
// BAD: generic retry wrapper applied to a non-idempotent INSERT —
// a retry after a timeout (where the first attempt actually succeeded
// server-side) creates a duplicate row
retry(() => db.execute("INSERT INTO charges (customer_id, amount) VALUES (?, ?)", ...))

// GOOD: idempotency built into the operation itself, THEN wrapped in retry
retry(() => chargeWithIdempotencyKey(customerId, amount, idempotencyKey))
```

- **DON'T:** Present a retry-wrapper utility as a generically safe addition to any database call without noting the idempotency precondition it relies on. A retry helper is a genuinely useful, reusable pattern — the mistake is applying it silently to every database call as if idempotency were guaranteed rather than a property that has to actually hold for the specific operation being wrapped.

- **DO:** Distinguish, when generating retry logic, between operations that are naturally idempotent (a `SELECT`, a `DELETE` by ID, an `UPDATE` that sets an absolute value) and operations that are not (an `INSERT` with no uniqueness guard, an `UPDATE` that increments/decrements a value) and apply retry logic only to the former without modification, flagging the latter as needing an idempotency mechanism first.

- **DON'T:** Assume a retry is safe just because the retried call is wrapped in a database transaction. A transaction guarantees that a single execution either fully applies or fully rolls back — it says nothing about what happens when the *same logical operation* is executed a second time in a *separate* transaction after a retry; that's precisely the idempotency question a transaction boundary alone doesn't answer.

### Overfitting Solutions to a Toy or Example Dataset Size

- **DO:** State explicitly when a proposed query, schema, or pipeline design has only been reasoned about (not actually tested) against a small example, and flag what might change at real production scale — an index that isn't needed at 100 rows but is essential at 100 million, a query plan the optimizer would choose differently once table statistics reflect real data volume, a pagination approach that works fine for a 3-page result set but not a 3,000-page one.

- **DON'T:** Validate a generated query or schema only against a small hand-crafted example (a handful of sample rows provided in the conversation) and present the result as production-ready with no caveat about scale. A query that returns correct results instantly against 5 example rows can still contain a missing index, an N+1 pattern, or an unbounded result set that only becomes a problem once real data volume is involved — none of which a 5-row example would ever reveal.

- **DO:** Ask about, or explicitly flag assumptions about, the real expected data volume, growth rate, and query frequency when they materially affect which design is appropriate, rather than silently designing for whatever scale happens to be implied by the example data shown in the conversation.

- **DON'T:** Generalize a specific fix that happened to work for the example at hand into a universal recommendation without noting the conditions under which it applies. A recommendation that solved a specific slow query for a specific data distribution isn't automatically the right general-purpose pattern for every superficially similar query — be explicit about what made the specific fix work, so the user can judge whether the same reasoning applies elsewhere.

### Confusing Similarly-Named Concepts Across Databases

- **DO:** Verify what a term actually means in the specific database/store being discussed before using it, since the same word carries meaningfully different guarantees across systems — an "index" in Elasticsearch is closer to a whole database/table than to a relational secondary index; a "transaction" in a system with only single-partition atomicity is a much weaker guarantee than a relational database's cross-table ACID transaction; a "table" in Cassandra is designed around a single query pattern in a way a relational table isn't; "consistency" itself means something different in ACID (constraint validity) versus in CAP (linearizability).

- **DON'T:** Carry an assumption from one database's terminology directly into a discussion of a different database that happens to use the same word. Advising on Elasticsearch "index" design using intuitions built for relational secondary indexes, or assuming a NoSQL "transaction" provides the same isolation guarantees as a relational one, produces advice that sounds fluent but is quietly wrong because the underlying concept isn't actually the same thing.

- **DO:** Define the specific term explicitly when there's genuine risk of cross-system confusion, especially in a response that discusses more than one kind of data store side by side, so the reader isn't left to guess which sense of an overloaded term applies where.

- **DON'T:** Assume performance or consistency characteristics transfer from a well-known system to a less-familiar one just because both are broadly labeled the same category ("both are NoSQL," "both are SQL databases"). The category label is a much weaker guarantee of shared behavior than it might imply — verify the specific system's actual documented behavior rather than reasoning from the category name.

### Not Flagging When an "Optimization" Trades Away Correctness or Safety

- **DO:** Explicitly call out, whenever a request to "make this faster" or "simplify this" would require removing a constraint, validation check, transaction boundary, or parameterized-query safety in order to comply, that the tradeoff exists and let the user decide — rather than silently complying with the literal performance/simplicity request at the cost of correctness or safety the user likely didn't intend to give up.

- **DON'T:** Remove a `NOT NULL`/`CHECK` constraint, drop a transaction wrapper, switch to string-concatenated SQL, or skip input validation as a way to make code "simpler" or "faster" in response to a general request, without flagging that the change has a safety or correctness cost beyond the stated goal. A user asking for faster code almost never means "and it's fine if it becomes vulnerable to injection" or "and it's fine if invalid data can now be written" — those tradeoffs need to be surfaced, not silently assumed as acceptable.

- **DO:** Offer the safe version of a requested optimization by default, and mention the unsafe-but-marginally-faster alternative only as an explicitly labeled option with its tradeoff named, rather than defaulting to the riskier version because it happens to satisfy the literal request more directly.

- **DON'T:** Treat "the user asked for X" as sufficient justification to produce code that quietly regresses correctness, security, or data integrity in service of X. The most useful response to a request that has a hidden tradeoff is one that surfaces the tradeoff plainly, not one that optimizes for literal compliance with the stated ask at the expense of what the user actually needs.

### Assuming a Migration or Query Succeeded Without Verifying

- **DO:** Actually check the result of a database command that was run — the return status, the row count affected, the query output, an explicit follow-up query confirming the expected state — before reporting that a migration, backfill, or data fix succeeded. Reporting success based on the absence of a visible error, without confirming the actual outcome, can be wrong in ways a missing-error-check wouldn't catch: a statement that ran but affected zero rows, a migration tool that reported success while silently skipping a step, a script that completed without ever reaching the intended change.
```pseudocode
// Don't just run it — confirm what actually happened
result = db.execute("UPDATE orders SET status = 'archived' WHERE created_at < :cutoff")
if result.rowsAffected == 0:
    flagForReview("expected to archive some rows, but zero were affected — verify the cutoff and data before assuming this is correct")
```

- **DON'T:** Report a database operation as complete based only on the absence of an exception or error message. Many failure modes in database operations are silent by nature — a `WHERE` clause that matched nothing, a migration that no-ops because a guard condition was already satisfied, a connection that silently used the wrong database/schema — and none of these raise an error that a simple "did it throw" check would catch.

- **DO:** Re-query and display the actual resulting state after a schema change or data modification when reporting it as done, particularly for anything run against a real (not purely local/example) database — showing the new schema, the affected row count, or a sample of the changed data as concrete evidence, rather than asserting success as a conclusion without evidence attached.

- **DON'T:** Assume a multi-step operation (a migration plus a backfill plus an index build, for instance) succeeded in full because the last step in the sequence reported success. An earlier step failing silently or partially, while a later independent step still completes and reports its own success, can leave the overall operation in a broken or incomplete state that a check of only the final step's outcome would miss entirely — verify each meaningful step, not just the last one.

### Presenting Deprecated Syntax or APIs as Current Best Practice

- **DO:** Check whether a recommended syntax, function, or library API is still current and non-deprecated for the specific version in use, rather than defaulting to whatever pattern was most common in older, more widely-represented training material. Database ecosystems evolve — a pagination approach, an ORM method, or a configuration option that was standard practice several years ago can be deprecated, discouraged, or superseded by a better-supported alternative by the time it's suggested.

- **DON'T:** Recommend a database version, engine, or tool that is past its end-of-life/end-of-support date as if it were an unremarkable current choice, without flagging its status. Suggesting a specific major version with no mention that it's no longer receiving security patches leaves the user to discover that fact later, potentially after already building on it.

- **DO:** Note explicitly when a suggested approach is the older, still-functional-but-superseded way of doing something, alongside the current recommended approach, so the user can make an informed choice rather than unknowingly adopting a pattern the ecosystem has moved past — particularly relevant for ORMs and query builders that go through frequent API changes across major versions.

- **DON'T:** Assume that because a pattern appears frequently and consistently across general knowledge, it's still the current best practice for a fast-moving area (cloud-managed database offerings, specific ORM APIs, specific cloud data-warehouse SQL dialects change relatively quickly). When there's genuine uncertainty about whether a specific recommendation is current, say so, and suggest the user verify against the specific tool's current documentation rather than presenting a possibly-stale pattern with unwarranted confidence.

## Quick Checklist
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
