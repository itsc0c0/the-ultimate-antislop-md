# Ruby

## Idiomatic Style (Ruby Style Guide Conventions)

- **DO:** Use `snake_case` for methods, local variables, and file names; `CamelCase` for classes and modules; and `SCREAMING_SNAKE_CASE` for constants, matching the community-wide Ruby Style Guide conventions that nearly every gem and Rails app already follows.
- **DO:** Name a file to match the constant it defines, translated to `snake_case` (`UserProfile` lives in `user_profile.rb`, `HTTP::Client` lives in `http/client.rb`), so Ruby's/Rails' autoloading (Zeitwerk) can find it without manual `require` statements, and so any reader can predict a class's file location from its name alone.
- **DO:** Use two-space indentation (the near-universal Ruby convention) consistently, and never mix it with tabs — most Ruby tooling (Rubocop, most editors' default Ruby modes) assumes two spaces, and inconsistent indentation produces noisy diffs.
- **DO:** Omit parentheses on method calls that read naturally without them, especially for DSL-style calls and simple one-argument invocations (`puts "hello"`, `expect(result).to eq 5`), while keeping parentheses for method *definitions* and for calls where omitting them would be ambiguous — this is a stylistic convention with some team-to-team variance, so match whatever the project's existing code and Rubocop config already do.
- **DON'T:** Use `and`/`or` for boolop control flow (as opposed to their much lower operator precedence making them occasionally useful for flow control like `raise "boom" unless valid? or return`) in place of `&&`/`||` for boolean logic — `and`/`or` have surprising, much lower precedence than `&&`/`||`, which has caused real bugs when someone expects C-like precedence (`x = false and true` assigns `false` to `x`, not the boolean result of the whole expression).
  ```ruby
  # BAD — precedence surprise: this assigns `1` to x, not the "and"ed result
  x = 1 and 2 # x == 1, not (1 and 2)

  # GOOD — && has the precedence most people expect
  x = 1 && 2  # x == 2
  ```
- **DO:** Use `||=` for memoization and lazy default assignment (`@cache ||= {}`) as the idiomatic Ruby shorthand for "assign only if currently nil/false," instead of a manual `if @cache.nil? then @cache = {} end`.
- **DO:** Name predicate methods (ones returning `true`/`false`) with a trailing `?` (`empty?`, `valid?`, `admin?`) and destructive/in-place-mutating methods with a trailing `!` (`sort!`, `strip!`, `save!` for the "raise on failure" variant) — Ruby uses punctuation in method names precisely to communicate these two properties, and skipping the convention removes information callers rely on.
  ```ruby
  # BAD — no signal that this mutates in place or raises
  def sort_users(users)
    users.sort_by! { |u| u.name }
  end

  # GOOD — the `!` on the method name matches its `!`-suffixed mutation
  def sort_users!(users)
    users.sort_by! { |u| u.name }
  end
  ```
- **DO:** Prefer implicit `return` (the value of the last evaluated expression) over an explicit `return` keyword at the end of a method, reserving explicit `return` for early exits. This matches idiomatic Ruby and keeps method bodies reading as expressions rather than imperative statement lists.
- **DO:** Use string interpolation (`"Hello, #{name}!"`) instead of string concatenation with `+` or `<<` when building a string from mixed literal and variable parts — it's clearer to read and avoids repeated intermediate string allocations.
- **DO:** Prefer `unless` over `if !condition` for a single negative condition, and prefer a trailing conditional modifier (`do_thing if condition`) for a single guard-style statement, reserving multi-line `if`/`unless` blocks for branches with real bodies.
  ```ruby
  # GOOD — trailing modifier for a simple guard
  return unless user.active?

  # GOOD — unless reads better than if !valid?
  raise InvalidRecordError unless record.valid?
  ```
- **DON'T:** Use `unless...else`. The negative-condition-then-positive-branch reads backwards; restructure as a positive `if`/`else`, or split into two guard clauses.
- **DO:** Prefer symbols (`:status`) over strings (`"status"`) for internal identifiers like hash keys, enum-like values, and method arguments that aren't user-facing text — symbols are immutable and interned, making comparisons cheap and the intent ("this is an identifier, not data") explicit.
- **DO:** Use Ruby's safe navigation operator (`user&.address&.city`) to traverse a chain of possibly-nil objects instead of a nested chain of manual `if obj && obj.method` checks — it reads far more clearly and doesn't lose the important distinction between "nil short-circuits" and "false short-circuits."
- **DON'T:** Overuse the safe navigation operator as a substitute for actually deciding whether `nil` is a legitimate state at that point in the code. If a method should never receive `nil` there, let it raise (a `NoMethodError` on `nil` is a legitimate, debuggable signal) rather than silently no-op-ing the whole chain with `&.`, which can hide a real bug.
- **DO:** Use guard clauses at the top of a method to handle edge cases and exit early, keeping the main logic path unindented, rather than wrapping the entire method body in a single large conditional.
- **DO:** Favor Ruby's built-in `Enumerable` methods (`map`, `select`, `reject`, `reduce`, `each_with_object`, `group_by`, `tally`) over hand-rolled loops with manual accumulator variables — they express intent directly and avoid a class of off-by-one and mutation bugs.
  ```ruby
  # BAD — manual loop and accumulator
  active_names = []
  users.each { |u| active_names << u.name if u.active? }

  # GOOD — expresses the transformation directly
  active_names = users.select(&:active?).map(&:name)
  ```
- **DO:** Use the `&:method_name` shorthand for a block that just calls one method on its argument (`users.map(&:name)`) instead of the verbose equivalent `users.map { |u| u.name }`, when the block genuinely does nothing but a single method call.
- **DON'T:** Chain more than a handful of `Enumerable` calls in a single expression when it hurts readability or performance (each link allocates a new intermediate array). Break a long chain into named intermediate variables, or fold multiple steps into one `each_with_object`/`reduce` when performance on large collections matters.
- **DO:** Keep methods short and focused on one responsibility — as a rule of thumb, a method that doesn't fit on one screen, or that mixes multiple levels of abstraction (low-level string parsing next to high-level business rules), is a candidate for extraction.
- **DO:** Use keyword arguments for methods with more than two or three parameters, especially when some are optional or boolean, so call sites stay self-documenting instead of a run of unlabeled positional values.
  ```ruby
  # BAD — call site says nothing about what true/false mean
  def create_user(name, email, true, false)

  # GOOD — self-documenting at the call site
  def create_user(name:, email:, admin: false, verified: false)
  ```
- **DON'T:** Use global variables (`$config`, `$current_user`) to pass state around. Ruby's globals bypass encapsulation entirely and make it impossible to reason locally about what a method depends on; pass dependencies explicitly or use a properly scoped singleton/service object instead.
- **DO:** Use multiple assignment and destructuring for readable unpacking of arrays and small fixed-shape structures, instead of indexing into an array positionally at each use site.
  ```ruby
  # BAD — indexing by position obscures what each value represents
  result = [200, "OK", payload]
  status = result[0]
  message = result[1]

  # GOOD — destructuring names each value at the point of unpacking
  status, message, payload = fetch_result
  ```
- **DO:** Use `Struct.new` or a plain `Data.define` (Ruby 3.2+) for small, immutable value objects that just bundle a few named attributes, instead of a full `class` with a hand-written `initialize`, `attr_reader`s, and `==` when nothing beyond that is needed.
  ```ruby
  # GOOD — Data.define gives you an immutable value object with
  # generated accessors, ==, and a readable inspect for free
  Point = Data.define(:x, :y)
  origin = Point.new(x: 0, y: 0)
  ```
- **DO:** Freeze string literals project-wide (`# frozen_string_literal: true` at the top of each file, or as a Rubocop-enforced default) so accidental in-place string mutation raises immediately instead of silently corrupting a value that's shared or reused elsewhere, and so literal strings aren't re-allocated on every execution.
- **DON'T:** Rescue `NoMethodError` as a substitute for checking whether an object actually responds to a method before calling it. Use `respond_to?` for a genuine duck-typing check, and let an unexpected `NoMethodError` on a value that *should* have the method surface as the bug report it actually is.
- **DO:** Use `Comparable` (implementing `<=>`) when a type has a natural ordering, so it automatically gets `<`, `>`, `<=`, `>=`, `==`, and `between?` for free, instead of hand-implementing each comparison operator separately.
- **DO:** Prefer `tap` for a debugging/side-effect insertion point in a method chain that shouldn't change the chain's value (`user.tap { |u| logger.info(u.id) }.save`), and prefer `then`/`yield_self` when transforming a value once as part of a pipeline, keeping both uses intentional rather than reaching for either as a general-purpose escape hatch.
- **DO:** Use case/pattern matching (`case value; in {status: "active", role:}` — Ruby 2.7+'s `in` clause) for destructuring and matching against a value's shape, which reads more directly than a chain of `is_a?`/`respond_to?` checks for structured data like parsed JSON.
  ```ruby
  case response
  in { status: 200, body: { "id" => id } }
    process(id)
  in { status: 404 }
    handle_not_found
  end
  ```
- **DO:** Use `Object#then`/`Object#yield_self` to pipe a value through a short transformation chain in-place, especially when the alternative would require introducing an otherwise-unneeded intermediate local variable purely to hold a one-step transformation.
- **DO:** Prefer `Array#dig`/`Hash#dig` for safely reading a value several levels deep into a nested structure (`config.dig(:database, :pool, :size)`), instead of a chain of `&.[]` or repeated `.fetch` calls, since `dig` short-circuits to `nil` cleanly the moment any intermediate key is missing.
  ```ruby
  # BAD — verbose and still crashes if any intermediate key is absent
  size = config[:database] && config[:database][:pool] && config[:database][:pool][:size]

  # GOOD — short-circuits to nil safely at any missing level
  size = config.dig(:database, :pool, :size)
  ```
- **DO:** Use `Hash#fetch` with an explicit default or a block, instead of `hash[key]`, when a missing key should be treated as an error or needs a computed (not merely static) fallback — `fetch` raises `KeyError` by default on a missing key, surfacing a bug immediately instead of silently returning `nil` and deferring the failure to wherever that `nil` eventually causes a `NoMethodError`.
  ```ruby
  # hash[:timeout] silently returns nil if the key is missing;
  # fetch makes the requirement explicit and fails fast if it's absent
  timeout = config.fetch(:timeout) { raise "timeout must be configured" }
  ```
- **DON'T:** Reach for `method_missing`-based dynamic attribute access (as some older ActiveRecord-adjacent code does) when a plain `Struct`, `OpenStruct` (careful — it's comparatively slow and loosely typed), or explicit `attr_accessor` declarations would express the same data shape more transparently and with better tooling support (autocomplete, static analysis).
- **DO:** Use `Kernel#Array()`/`Kernel#Integer()`/`Kernel#String()` conversion methods (which raise on truly invalid input) rather than the more permissive `.to_a`/`.to_i`/`.to_s` coercion methods when invalid input should be treated as an error rather than silently coerced to a zero-ish default (`"abc".to_i` silently returns `0`, while `Integer("abc")` raises `ArgumentError`).
- **DO:** Use heredocs (`<<~TEXT`, the squiggly variant that strips leading whitespace) for multi-line string literals embedded in otherwise-indented code, rather than manually concatenating several `+`-joined lines or accepting the older `<<-TEXT` variant's less intuitive indentation rules.
  ```ruby
  message = <<~TEXT
    Hello, #{user.name}!
    Your order ##{order.id} has shipped.
  TEXT
  ```

## Blocks, Procs, and Lambdas Usage

- **DO:** Reach for a block (`do...end` or `{ }`) as the default way to pass a chunk of code to a method — it's the idiomatic Ruby mechanism for iteration, resource cleanup (`File.open(path) { |f| ... }`), and callback-style APIs, and it doesn't need to be instantiated as an object first.
- **DO:** Use `{ }` for single-line blocks and `do...end` for multi-line blocks, matching the community-standard convention (the "Weirich convention"), so block style itself signals whether a block is a short inline transform or a longer procedural chunk.
- **DO:** Convert a block to a named `Proc` (via `&block` capture, or `Proc.new`/`proc { }`) only when it needs to be stored, passed around, or reused across multiple calls — for a block used once at a single call site, passing it directly is simpler and avoids the extra allocation and indirection.
- **DO:** Prefer `lambda`/`->` syntax over `proc`/`Proc.new` when you specifically want strict arity checking (a lambda raises `ArgumentError` on a wrong number of arguments, a proc silently ignores extras or fills missing ones with `nil`) and `return`-from-lambda-only semantics (a `return` inside a lambda returns from the lambda itself; a `return` inside a proc returns from the enclosing method, which can be a surprising non-local exit).
  ```ruby
  # A proc silently tolerates the wrong arity — a subtle bug source
  add = proc { |a, b| a.to_i + b.to_i }
  add.call(1)          # => 1, b silently became nil -> 0

  # A lambda enforces arity strictly
  add = ->(a, b) { a + b }
  add.call(1)           # => ArgumentError: wrong number of arguments
  ```
- **DON'T:** Use `return` inside a `proc` (or a block, which behaves like a proc for this purpose) that's stored and invoked later from a different call stack, expecting it to simply exit the block — a non-local `return` from a proc whose defining method has already returned raises a `LocalJumpError`. Use `next` to exit a block/proc early with a value instead.
- **DO:** Use `yield` and `block_given?` for the common case of a method that simply invokes a caller-supplied block once (or several times) without needing to store or pass it elsewhere — it's more efficient and more idiomatic than declaring an explicit `&block` parameter you never actually pass onward.
  ```ruby
  def with_logging
    puts "starting"
    result = yield if block_given?
    puts "finished"
    result
  end
  ```
- **DO:** Declare an explicit `&block` parameter only when the block genuinely needs to be passed to another method, stored as an attribute, or converted to a `Proc` for later use — not as a default habit for every method that takes a block.
- **DON'T:** Build deeply nested blocks-within-blocks (a callback whose block itself takes a callback with its own block) when the same logic could be flattened using intermediate named methods or `Enumerable` chaining. Nested blocks quickly become as hard to read as nested callbacks in any other language.
- **DO:** Use `Symbol#to_proc` (the `&:method_name` shorthand) and `method(:name).to_proc` interchangeably where they fit — both convert an existing callable into block form without writing a wrapper block by hand.
- **DO:** Use `curry` on a `Proc`/`lambda` when a function needs to be partially applied — supplied some arguments now and the rest later — instead of manually wrapping it in another lambda that closes over the first set of arguments.
  ```ruby
  add = ->(a, b, c) { a + b + c }
  add5 = add.curry[5]
  add5[10, 20] # => 35
  ```
- **DO:** Pass a block explicitly through several layers of delegation with `&block`/`yield` chaining when a wrapper method just forwards to an inner method that itself takes a block, rather than accidentally dropping the block by forgetting to re-yield or re-pass it.
  ```ruby
  def with_retries(&block)
    3.times { return yield } rescue nil
  end
  ```
- **DON'T:** Use a block where a named method would be clearer and reusable — a block defined once inline and never referenced again is fine, but a block whose logic is copy-pasted at multiple call sites should become a real method or a stored `Proc` constant instead.
- **DO:** Use `Enumerable#each_with_object` when building up a single mutable accumulator (a hash, an array) across an iteration, since it both returns the accumulator and makes the intent ("I'm folding into this object") clearer than `reduce`/`inject` used the same way with an extra wrapping step.
  ```ruby
  # GOOD — the accumulator's role is explicit in the method name itself
  grouped = orders.each_with_object(Hash.new(0)) { |o, h| h[o.status] += 1 }
  ```
- **DON'T:** Rely on a block's implicit return value in a context where the caller ignores it (`each` used for a return value it discards) — use `map`/`select`/`reduce` when the block's return value matters, and reserve `each` for iteration purely for side effects.

## Metaprogramming Discipline (When It Helps vs. Obscures)

- **DO:** Reach for metaprogramming (`define_method`, `method_missing`, `class_eval`, `instance_variable_get/set`) only when it removes real, otherwise-unavoidable repetition — generating a family of near-identical accessor or delegation methods from a small declarative list is a legitimate use; metaprogramming for a one-off method is not.
  ```ruby
  # A legitimate use: generating a known, bounded family of methods
  %i[draft published archived].each do |status|
    define_method("#{status}?") { self.status == status.to_s }
  end
  ```
- **DON'T:** Use `method_missing` as a substitute for a real, discoverable API when the set of dynamically-handled method names is knowable in advance — prefer `define_method` in a loop (which shows up in `instance_methods`, works with `respond_to?`, and is debuggable with a normal stack trace) over `method_missing`, which hides the actual method surface from tooling, documentation, and `respond_to?` unless you also override `respond_to_missing?`.
- **DO:** Always pair a `method_missing` override with a matching `respond_to_missing?` override, so `respond_to?`, `method(:name)`, and duck-typing checks by other code behave correctly instead of falsely reporting that the dynamically-handled method doesn't exist.
- **DON'T:** Monkey-patch (reopen and modify) core Ruby classes (`String`, `Array`, `Hash`, `Object`) to add general-purpose convenience methods in application code, especially in a shared library other teams depend on. A monkey-patch is global and can silently collide with a method another gem defines with the same name but different behavior, producing confusing bugs far from the patch site.
  ```ruby
  # AVOID in application/library code — global, silent, collision-prone
  class String
    def to_slug
      downcase.gsub(/\s+/, "-")
    end
  end

  # PREFER — explicit, scoped, no risk of colliding with another gem
  module Sluggable
    def self.slugify(string)
      string.downcase.gsub(/\s+/, "-")
    end
  end
  ```
- **DO:** Use Ruby's `refine`/`using` refinements instead of a global monkey-patch when you genuinely need to extend a core class's behavior only within a specific scope — refinements are lexically scoped and don't leak the modification into unrelated code.
- **DO:** Prefer composition (modules mixed in via `include`/`extend`, or plain delegation) over deep, dynamically-generated inheritance chains or heavy `class_eval` manipulation of a class's ancestry at runtime — the more a class's behavior is assembled dynamically, the harder it is for a reader (or an IDE's "jump to definition") to find where a given method actually lives.
- **DON'T:** Use `send`/`__send__` to call private methods from outside a class as a way to bypass encapsulation you find inconvenient. If a private method genuinely needs to be called from outside, that's a signal the method should be public (or the design should change), not that encapsulation should be silently defeated.
- **DO:** Comment non-obvious metaprogramming (why a `define_method` loop exists, what `method_missing` is standing in for) since dynamically generated methods are invisible to a plain-text search of the codebase — a future reader grepping for a method name that was metaprogrammed into existence will otherwise find nothing.
- **DO:** Keep `class_eval`/`instance_eval` blocks small and focused when used for DSL-building (a common, legitimate use in gems like RSpec or Rails routing) — the DSL's implementation should be easy to audit even though its *usage* is meant to read declaratively.
- **DO:** Use `Module#prepend` rather than monkey-patching a method's body directly when you need to wrap or intercept an existing method's behavior (adding logging around every call, adding a feature flag check) — `prepend` lets the original implementation still be called via `super`, preserving the original behavior as a composable layer instead of physically overwriting it.
  ```ruby
  module Loggable
    def save(*args)
      Rails.logger.info("Saving #{self.class}")
      super
    end
  end
  # class Order; prepend Loggable; end
  # Order#save still calls the real implementation via `super`
  ```
- **DON'T:** Reopen a class (`class String; def new_method; end; end`) when a `Module#prepend`-based approach, a refinement, or simply a differently-named helper method would achieve the same result with a smaller, more contained blast radius.
- **DO:** Use `define_method` with a closure over local variables when generating a family of methods that each need to capture different values from the enclosing scope — this is something `def` alone can't do, and it's one of the legitimate cases where `define_method` is strictly more capable than a static method definition, not just a stylistic alternative.
- **DO:** Inspect an object's actual method source location (`method(:foo).source_location`) when debugging a dynamically-defined method whose behavior is confusing, since a metaprogrammed method's origin is often not obvious from reading the class definition alone.
- **DON'T:** Use `instance_variable_get`/`instance_variable_set` to read or write another object's internal state from outside the object as a way to bypass its public interface. This breaks encapsulation just as thoroughly as calling a private method via `send`, and it silently stops working the moment the target object's internal implementation changes its ivar names.
- **DO:** Use `ObjectSpace`-based introspection and `TracePoint` only for genuinely exceptional tooling needs (a custom profiler, a debugging aid) — never as a mechanism application business logic depends on, since both operate outside Ruby's normal method-dispatch model and make behavior extremely difficult to reason about or test conventionally.
- **DO:** Validate metaprogrammed DSL input at the point the DSL method is called (raise immediately on an invalid argument to a `has_many`-style macro, for instance) rather than letting a malformed DSL call silently produce a broken or partially-configured object that only fails much later when the misconfiguration is actually exercised.

## Rails-Specific Conventions

- **DO:** Use Rails' built-in fragment/Russian-doll caching (`cache` in views, `Rails.cache.fetch` in Ruby code) for expensive-to-compute or expensive-to-render data that doesn't change on every request, with an explicit, deliberate cache key (including a version/timestamp component) and expiration policy — an uninvalidated or overly broad cache key is a common source of stale-data bugs that are hard to reproduce because they depend on cache state.
  ```ruby
  Rails.cache.fetch("dashboard_stats/#{account.id}/#{account.updated_at}", expires_in: 5.minutes) do
    compute_expensive_dashboard_stats(account)
  end
  ```
- **DO:** Keep controllers thin — a Rails controller action should orchestrate (find/authorize a record, call domain logic, choose a response) in a handful of lines, not contain validation rules, multi-step business logic, or raw SQL. Fat controllers become untestable without spinning up the full request/response cycle.
- **DON'T:** Let "fat models" become a dumping ground either — a model class accumulating dozens of callbacks, class methods, and unrelated business rules is just as much a maintainness liability as a fat controller, even though "fat model, skinny controller" is the traditional Rails guidance. Extract genuinely separate concerns (notification logic, complex multi-model workflows, external API orchestration) out of the model into service objects, form objects, or query objects.
- **DO:** Introduce a **service object** — a plain Ruby object with a single public entry point (often `.call`) — for a multi-step business operation that coordinates several models, external calls, or side effects that don't naturally belong on any single `ActiveRecord` model.
  ```ruby
  class ProcessRefund
    def self.call(order:, amount:)
      new(order, amount).call
    end

    def initialize(order, amount)
      @order, @amount = order, amount
    end

    def call
      PaymentGateway.refund(@order.payment_id, @amount)
      @order.update!(status: :refunded)
      RefundMailer.confirmation(@order).deliver_later
    end
  end
  ```
- **DO:** Watch for N+1 queries — a loop over an association that triggers one query per iteration instead of one query total — and eliminate them with eager loading (`includes`, `preload`, or `eager_load` depending on whether the association is also used in a `WHERE`/`ORDER` clause). This is one of the single most common Rails performance bugs, and it's invisible in development with small seed data but devastating in production with real data volumes.
  ```ruby
  # BAD — N+1: one query for posts, then one more per post for its author
  Post.all.each { |post| puts post.author.name }

  # GOOD — two queries total, regardless of how many posts there are
  Post.includes(:author).each { |post| puts post.author.name }
  ```
- **DO:** Use the `bullet` gem (or equivalent) in development/test to automatically flag N+1 queries and unused eager loads as they're introduced, rather than relying on manually noticing slow query logs after the fact.
- **DO:** Push validation and simple derived-attribute logic onto the model (Rails' built-in `validates` DSL, computed methods reading the model's own attributes) since that's genuinely model-appropriate responsibility — the "thin controller" principle doesn't mean "no logic in models," it means "no orchestration/HTTP-layer logic in controllers."
- **DON'T:** Perform database writes, external API calls, or other side effects directly inside a view template or helper. Views should render already-prepared data; any side-effecting logic belongs in a controller action, a background job, or a service object called before rendering.
- **DO:** Use strong parameters (`params.require(:user).permit(:name, :email)`) to explicitly allow-list mass-assignable attributes on every controller action that creates or updates a record, rather than permitting all params or bypassing the mechanism.
- **DO:** Prefer scopes (`scope :active, -> { where(active: true) }`) and query objects for reusable, composable query logic instead of duplicating the same `where`/`joins` chain across multiple controllers and models.
- **DON'T:** Put callbacks (`before_save`, `after_create`) on a model for logic that has effects outside that model's own data (sending emails, calling external APIs, touching unrelated records) — callback chains that reach far outside the model make it hard to predict what saving a record will actually trigger, and they run on every save including ones the caller didn't intend to have side effects. Prefer explicit calls from a service object or controller for anything with an external effect.
- **DO:** Use background jobs (`ActiveJob` with Sidekiq, GoodJob, or similar) for anything that doesn't need to complete before the response is sent — sending email, calling a slow third-party API, generating a report — rather than doing it synchronously inside the request cycle.
- **DO:** Use `counter_cache` for a frequently-read count of an association's size (e.g., a post's comment count shown on every listing page) instead of calling `.count` on the association repeatedly, which issues a fresh `COUNT(*)` query every time.
  ```ruby
  # Avoids a COUNT(*) query every time comment counts are displayed
  class Comment < ApplicationRecord
    belongs_to :post, counter_cache: true
  end
  ```
- **DO:** Use `find_each`/`find_in_batches` instead of `.all.each` when iterating over a large table, since `.all.each` loads every row into memory at once while `find_each` fetches and processes records in bounded-size batches.
  ```ruby
  # BAD — loads the entire table into memory before iterating
  User.all.each { |u| u.send_newsletter }

  # GOOD — fetches and processes in batches, bounded memory usage
  User.find_each(batch_size: 1000) { |u| u.send_newsletter }
  ```
- **DON'T:** Perform `.count`/`.length`/`.size` interchangeably without knowing the difference — `.count` issues a `SELECT COUNT(*)` query, `.length` loads the full association into memory and counts it in Ruby (expensive if not already loaded), and `.size` picks whichever is cheaper depending on whether the association is already loaded. Calling `.length` on a large not-yet-loaded association when `.count` would do is a common accidental performance regression.
- **DO:** Use database-level `NOT NULL`, uniqueness, and foreign-key constraints alongside (not instead of) Rails model-level validations — model validations can be bypassed by direct SQL, console access, or a code path that skips `save`'s validation callbacks (like `update_column` or `insert_all`), so the database should still enforce the same invariants as a backstop.
- **DO:** Use `strict_loading` (Rails 6.1+) on associations during development/test to surface N+1 queries as hard errors immediately, rather than relying entirely on the `bullet` gem's soft warnings, when a codebase wants to enforce eager-loading discipline strictly.
- **DON'T:** Put authorization logic (who is allowed to perform an action) directly and repeatedly inline in controller actions with ad hoc conditionals. Use a dedicated authorization gem (Pundit, CanCanCan) or a consistent, centralized policy pattern so authorization rules live in one auditable place instead of being scattered and potentially inconsistent across controllers.
- **DO:** Use `ActiveRecord::Base.transaction` to wrap a controller action or service object's multi-model writes that must succeed or fail together, exactly as with raw SQL transactions, so a failure partway through a multi-step save rolls back every change instead of leaving associated records partially updated.
  ```ruby
  ApplicationRecord.transaction do
    order.update!(status: :paid)
    inventory.decrement!(:quantity, order.item_count)
  end
  ```
- **DON'T:** Rely on `update`/`save` (which return `false` on validation failure) inside a context where a silent failure would go unnoticed — use the bang variants (`update!`/`save!`, which raise `ActiveRecord::RecordInvalid`) when a failure to persist should never pass silently, particularly inside a transaction block, where a plain `false`-returning `save` doesn't itself trigger a rollback the way a raised exception does.
- **DO:** Use view/presenter/decorator objects (a plain Ruby object wrapping a model to add view-specific formatting methods) to keep display-formatting logic out of both the model (which shouldn't know about view concerns) and the view template (which shouldn't contain non-trivial logic), when a project's views need enough formatting logic to justify the extra layer.
- **DO:** Use `ActiveModel::Serializer`, `jbuilder`, or a dedicated serialization library consistently for JSON API responses, rather than calling `.to_json`/`.as_json` directly on ActiveRecord objects from a controller — direct serialization tends to either leak internal model attributes that shouldn't be exposed, or requires ad hoc per-action attribute-whitelisting that drifts out of sync across endpoints.
- **DO:** Version an API deliberately (a URL path segment like `/api/v1/`, or an `Accept` header-based scheme) from the very first public endpoint, since retrofitting versioning onto an already-shipped, already-consumed API is far more disruptive than starting with a versioning scheme in place even before it's strictly needed.
- **DO:** Use Rails' built-in `ActiveSupport` core extensions (`5.days.ago`, `"CamelCase".underscore`, `array.in_groups_of(3)`, `hash.deep_merge`) where they clearly improve readability over the equivalent hand-written logic, since they're a well-tested, widely-understood part of the Rails ecosystem — but be aware they're Rails/ActiveSupport-specific and won't be available in a plain Ruby script or library that doesn't load ActiveSupport.
- **DON'T:** Reach for raw threads (`Thread.new`) for concurrency inside a typical Rails request/response cycle without understanding the implications for the app server's threading model and the Global VM Lock (GVL/GIL) — CRuby's GVL means threads provide concurrency for I/O-bound work but not true CPU parallelism, and an unmanaged thread spawned inside a request can outlive the request or contend with the app server's own thread pool in ways that are easy to get wrong; prefer a background job for anything beyond the simplest, well-understood concurrent I/O pattern.

## Error Handling

- **DO:** Raise specific, custom exception classes (`class InsufficientFundsError < StandardError; end`) for domain-specific failure conditions rather than raising generic `RuntimeError` or `StandardError` directly — a specific class lets callers `rescue` exactly the failure they know how to handle without accidentally catching unrelated errors.
- **DO:** Rescue the narrowest exception class that's actually expected, and avoid a bare `rescue` (which defaults to `rescue StandardError` in Ruby, but still catches far more than intended in most spots) unless the block genuinely needs to handle any standard error the same way.
  ```ruby
  # BAD — swallows everything, including bugs unrelated to the network call
  begin
    response = api_client.fetch(id)
  rescue => e
    nil
  end

  # GOOD — only the expected failure mode is handled specifically
  begin
    response = api_client.fetch(id)
  rescue ApiClient::TimeoutError => e
    logger.warn("API timeout for id=#{id}: #{e.message}")
    nil
  end
  ```
- **DON'T:** Rescue `Exception` (as opposed to `StandardError`) anywhere in application code. `Exception` is the root of Ruby's entire exception hierarchy and includes things like `SystemExit`, `NoMemoryError`, and interrupt signals — rescuing it can prevent the process from ever being able to shut down or respond to `Ctrl-C`.
- **DON'T:** Use exceptions for expected, non-exceptional control flow (e.g., raising and rescuing to signal "not found" on every lookup miss in a hot path). Reserve `raise`/`rescue` for genuinely exceptional conditions; use a `nil`/sentinel-value return, or a `find_by` (returns `nil`) vs. `find!`(raises) naming pair to make the expected-vs-exceptional distinction explicit at the call site.
- **DO:** Use `ensure` to guarantee cleanup (closing a connection, releasing a lock) runs whether or not an exception was raised, mirroring `defer`/`finally` in other languages.
  ```ruby
  connection = acquire_connection
  begin
    connection.execute(query)
  ensure
    connection.release
  end
  ```
- **DO:** Include enough context in a custom exception (the record ID, the operation attempted) either as a message or as additional attributes on the exception class, so a caught error is actionable in logs without needing to reproduce the failure.
- **DON'T:** Swallow an exception silently with an empty `rescue` block or a bare `rescue nil` modifier on a statement that can genuinely fail for reasons worth knowing about. At minimum log it; for anything user-facing, surface a meaningful message or fallback behavior.
- **DO:** Re-raise with added context when appropriate (`raise ProcessingError, "failed processing order #{order.id}: #{e.message}"` or wrapping via `raise CustomError.new(...), cause: e`) so a higher-level catch site sees what business operation failed, not just the low-level library error.
- **DO:** Use `retry` inside a `rescue` block deliberately, with a bounded attempt counter, for genuinely transient failures (a flaky network call) — an unbounded `retry` on every failure without a limit can spin forever on a persistent, non-transient error.
  ```ruby
  attempts = 0
  begin
    attempts += 1
    api_client.fetch(id)
  rescue ApiClient::TimeoutError
    retry if attempts < 3
    raise
  end
  ```
- **DO:** Define a base application-specific error class (`class ApplicationError < StandardError; end`) that all custom domain exceptions inherit from, so a boundary layer (a controller's top-level rescue, a background job's failure handler) can rescue every application-defined error in one place while still letting truly unexpected exceptions (bugs, `StandardError` subclasses from gems) surface differently.
- **DON'T:** Raise a `String` directly (`raise "something went wrong"`) as a default habit when a more specific, previously-defined exception class would communicate the failure mode better — a bare string-raised `RuntimeError` gives a `rescue` clause nothing to pattern-match against beyond the message text.
- **DO:** Include the original exception as the `cause` when wrapping an error (Ruby does this automatically when you `raise NewError, "..."` from inside a `rescue` block) so the full causal chain is visible in a backtrace, rather than losing the original low-level exception's information entirely.
- **DO:** Use `Timeout.timeout` (or a lower-level, driver-specific timeout option where available) around a call to an external service that has no reliable timeout of its own, so a single hung dependency can't block a request/worker indefinitely — but prefer a library's own native timeout configuration over `Timeout.timeout` where one exists, since Ruby's `Timeout` module has documented edge cases where it can interrupt code at an unsafe point.
- **DO:** Report unhandled exceptions to an error-tracking service (Sentry, Honeybadger, Rollbar, or equivalent) from a centralized location (a Rails `rescue_from ApplicationController`, a background-job failure hook) rather than adding ad hoc reporting calls scattered through business logic, so error visibility doesn't depend on remembering to add reporting at every call site.

## Testing (RSpec)

- **DO:** Structure specs with `describe`/`context`/`it` so the resulting output reads as a specification in plain English — `describe` for the class/method under test, `context` for the condition being varied, `it` for the specific expected behavior.
  ```ruby
  RSpec.describe OrderCancellation do
    context "when the order has already shipped" do
      it "raises an InvalidStateError" do
        expect { described_class.call(shipped_order) }
          .to raise_error(InvalidStateError)
      end
    end
  end
  ```
- **DO:** Use `let`/`let!` for lazily (or eagerly, for `let!`) memoized test fixtures shared across examples in a `describe`/`context` block, instead of repeating identical setup inside every `it` block.
- **DON'T:** Overuse shared `before` blocks and deeply nested `context`s to the point that an individual `it` example can't be understood without scrolling through several ancestor blocks to reconstruct its actual setup — if a spec file's nesting is more than 3-4 levels deep, consider splitting it or flattening some contexts.
- **DO:** Prefer expressive matchers (`expect(user).to be_valid`, `expect(response).to have_http_status(:ok)`, `expect { action }.to change(Order, :count).by(1)`) over generic `eql`/`==` assertions when a domain-specific matcher exists — they produce clearer failure messages and read closer to the intent being tested.
- **DO:** Use `instance_double`/`class_double` (verifying doubles) rather than a plain `double` when stubbing an existing class or object, so RSpec fails the test if the stubbed method doesn't actually exist on the real class — this catches drift between a test's mocks and the real API they're standing in for.
- **DON'T:** Hit real external services (payment gateways, third-party APIs, real email delivery) from specs. Stub them with `WebMock`/`VCR` or an injected fake, both for speed and so tests don't fail (or worse, silently succeed) based on an external system's availability.
- **DO:** Use factories (FactoryBot) with sensible, minimal defaults for building test records, and override only the specific attributes each test actually cares about, rather than constructing full records by hand or relying on fixtures that couple every test to a fixed shared dataset.
- **DO:** Write feature/request specs for the critical user-facing paths in addition to unit specs — a suite of only isolated unit tests can pass while the pieces don't actually work together through the real controller/routing/view stack.
- **DON'T:** Test private methods directly by reaching into an object's internals (`send(:private_method)`) as a matter of course. Test private behavior indirectly through the public methods that use it; a proliferation of tests that call private methods directly makes refactoring the implementation break tests that should have been implementation-agnostic.
- **DO:** Keep each `it` example asserting one logical behavior. A test with several unrelated `expect` calls checking unrelated outcomes makes it unclear, on failure, which behavior actually broke.
- **DO:** Use `subject`/`described_class` to reduce repetition when a spec file is primarily testing one class's instances, letting individual examples read as `expect(subject.valid?).to be true` rather than repeatedly constructing the same object by name.
  ```ruby
  RSpec.describe EmailValidator do
    subject { described_class.new(email) }

    context "with a valid email" do
      let(:email) { "user@example.com" }
      it { is_expected.to be_valid }
    end
  end
  ```
- **DO:** Use shared examples (`shared_examples`/`it_behaves_like`) to test common behavior across multiple types that all implement the same interface/contract (several classes conforming to a shared duck-typed protocol), instead of copy-pasting the same block of examples into each type's spec file.
- **DON'T:** Assert against a full object dump (`expect(response.body).to eq(huge_json_string)`) when only a few specific fields are actually relevant to the behavior under test — an assertion against unrelated fields makes the test break on any unrelated change to the payload shape, not just a regression in the behavior being tested.
- **DO:** Tag slow or environment-dependent specs (`:slow`, `:integration`) with RSpec metadata so they can be selectively excluded from the fast local feedback loop while still running in CI's full suite.
- **DO:** Use `before(:suite)`/`before(:all)` sparingly and only for genuinely expensive, safely-shareable setup (seeding a reference dataset that no test mutates) — shared state across examples set up this way is a common source of order-dependent test failures when one example does mutate what others assumed was untouched.
- **DO:** Run the suite with randomized example order (`config.order = :random` with a logged seed, RSpec's default) so hidden inter-test dependencies surface as flaky failures during development rather than lying dormant until a reordering (a new spec file, a parallelized CI run) exposes them in production-adjacent CI.
- **DO:** Use `aggregate_failures` (or a block passed to `expect` with multiple matchers) when a single logical behavior genuinely needs several related assertions that should all be reported together, so a failure shows every assertion that failed in one run instead of stopping at the first and requiring several fix-rerun cycles to see the rest.
  ```ruby
  it "returns a fully populated profile", :aggregate_failures do
    expect(profile.name).to eq("Alice")
    expect(profile.email).to eq("alice@example.com")
    expect(profile.verified?).to be true
  end
  ```
- **DON'T:** Stub or mock the exact method under test itself — a test that stubs the very method it's supposedly verifying always passes regardless of whether the real implementation is correct, which makes the test worthless as regression protection.
- **DO:** Use `travel_to`/`freeze_time` (from `ActiveSupport::Testing::TimeHelpers` or the `timecop` gem) to control the current time deterministically in tests involving dates/timestamps, instead of relying on the real wall-clock time, which makes date-boundary logic (a test that behaves differently right around midnight) flaky.
- **DO:** Assert on the actual public contract a collaborator promises (its documented return type/shape) when stubbing it with a verifying double's return value, not just whatever shape happens to make the current test pass — a stub that returns a plausible-but-wrong shape can mask a real integration bug that only appears once the double is replaced with the genuine collaborator.
- **DO:** Keep test descriptions (`it "..."`) written as a plain-English statement of expected behavior from the caller's perspective ("returns nil when the user is not found"), not as a description of the test's internal implementation ("calls find_by twice") — the former stays meaningful even after the implementation is refactored, the latter doesn't.

## Common AI-Assistant Mistakes

- **DON'T:** Monkey-patch core classes (`String`, `Array`, `Object`, `Hash`) as a default way to add a convenience method, without checking whether the project already has a pattern for this (a `lib/core_ext` directory, refinements, or simply "we don't do this here"). Global monkey-patches in generated code are a common source of silent, hard-to-trace collisions with gems.
- **DON'T:** Overuse metaprogramming (`method_missing`, dynamic `send`, runtime `class_eval` mutation) to solve problems that a straightforward, explicit method would solve just as well. Generated code that reaches for dynamic dispatch by default — rather than as a deliberate trade-off for removing real repetition — produces code that's harder to grep, harder to debug, and harder for the next contributor (human or AI) to modify safely.
- **DON'T:** Introduce N+1 queries in generated Rails code by looping over an association without eager loading. Any time generated code does `collection.each { |item| item.association.something }`, check whether `includes`/`preload` is needed first, and add it by default when the association is accessed inside a loop.
- **DON'T:** Ignore an existing project's conventions (RSpec vs. Minitest, service-object pattern vs. fat-model pattern, `frozen_string_literal` pragma usage, Rubocop configuration) and default to generic textbook Ruby style. Check for a `.rubocop.yml`, existing spec files, and existing model/service organization before generating new code, and match what's already there.
- **DON'T:** Rescue exceptions overly broadly (`rescue => e` or worse, `rescue Exception`) in generated error-handling code as a way to "make sure nothing crashes." This hides real bugs and can catch signals the process needs to receive (like `SIGINT`/`SystemExit`) when `Exception` is rescued; rescue the specific error classes actually expected.
- **DON'T:** Generate Rails code with mass-assignment left unguarded (skipping strong parameters, or permitting all attributes with a blanket `params.permit!`) — this is a well-known security footgun, and generated controller actions should always allow-list exactly the attributes intended to be settable from the request.
- **DON'T:** Default to synchronous execution for slow operations (external API calls, email sending, report generation) inside a Rails request cycle when the project already has a background job framework configured — check for `ActiveJob`/Sidekiq usage elsewhere in the codebase and use it for anything that doesn't need to block the response.
- **DON'T:** Generate `.all.each`-style unbounded iteration over what could be a large table — default to `find_each`/`find_in_batches` for any loop over a full model table in generated code, since the difference is invisible on a small seed dataset but severe on real production data.
- **DON'T:** Confuse `.count`, `.length`, and `.size` when generating ActiveRecord code — reach for `.count` (a `COUNT(*)` query) for an unloaded association, and understand that `.length` forces the whole association to load into memory first.
- **DON'T:** Generate error handling that rescues and logs an exception but then continues as if the operation succeeded (e.g., proceeding to the next line that assumes a variable was successfully set inside the now-failed block) — verify the control flow after a rescued exception actually reflects that something failed.
- **DON'T:** Skip database-level constraints (`NOT NULL`, foreign keys, uniqueness indexes) in a generated migration on the assumption that a model-level `validates` call is sufficient — generate both layers, since the database constraint is the backstop for the paths (raw SQL, bulk inserts, console access) that skip model validations entirely.
- **DON'T:** Generate a multi-step ActiveRecord write sequence without wrapping it in `transaction do ... end` — check whether generated code that touches more than one model or more than one row needs atomicity, and wrap it by default when it does.
- **DON'T:** Generate `save`/`update` calls (returning `false` on failure) inside logic that assumes success and immediately proceeds to use the now-possibly-stale record — either check the boolean return explicitly or use the bang variant (`save!`/`update!`) so a failure surfaces as an exception instead of silently falling through.
- **DON'T:** Default to hand-rolled inline authorization checks (`if current_user.admin? || current_user.id == resource.owner_id`) scattered across generated controller actions when the project already uses an authorization gem — check for Pundit/CanCanCan usage elsewhere and generate policy classes consistent with the existing pattern instead.

## Quick Checklist
- Use `snake_case` for methods/vars, `CamelCase` for classes/modules, `SCREAMING_SNAKE_CASE` for constants.
- Suffix predicate methods with `?` and destructive/raising methods with `!`.
- Prefer implicit return, string interpolation, and `unless`/trailing modifiers for simple conditionals; never `unless...else`.
- Use safe navigation (`&.`) deliberately — not as a blanket substitute for deciding whether `nil` is legitimate there.
- Favor `Enumerable` methods (`map`, `select`, `reduce`, `each_with_object`) over hand-rolled loops with accumulators.
- Use keyword arguments for methods with several parameters, especially booleans, so call sites self-document.
- Never use global variables (`$foo`) to pass state.
- Use `{ }` for single-line blocks, `do...end` for multi-line; convert to a `Proc`/`lambda` only when it must be stored or reused.
- Prefer `lambda`/`->` over `proc` when strict arity checking and local `return` semantics matter.
- Pair every `method_missing` override with `respond_to_missing?`.
- Never monkey-patch core classes for general application convenience; use modules, refinements, or explicit helpers instead.
- Prefer composition (mixins, delegation) over dynamically generated inheritance/`class_eval` mutation.
- Keep controllers thin (orchestration only) and don't let models balloon into unrelated-responsibility dumping grounds.
- Extract multi-step business operations into service objects with a single clear entry point.
- Eliminate N+1 queries with `includes`/`preload`/`eager_load`; verify with the `bullet` gem or query logs.
- Use strong parameters on every mass-assignable controller action; never `params.permit!`.
- Keep side-effecting logic (emails, external calls, DB writes) out of views and out of far-reaching model callbacks.
- Use background jobs for slow, non-blocking-required work instead of running it synchronously in the request cycle.
- Raise specific custom exception classes; rescue the narrowest class that's actually expected.
- Never rescue bare `Exception`; reserve broad `rescue` for genuinely intended catch-alls, not as a default habit.
- Use `ensure` for guaranteed cleanup, not manual duplication before every return/raise path.
- Structure RSpec with `describe`/`context`/`it`; use `let`/`let!` instead of repeating setup.
- Use verifying doubles (`instance_double`) over plain `double`; stub external services with WebMock/VCR.
- Use FactoryBot with minimal per-test overrides instead of hand-built records or shared fixtures.
- Don't test private methods directly; test them through the public interface that exercises them.
- Match the project's existing conventions (test framework, Rubocop config, service-object patterns) instead of defaulting to generic style.
- Use destructuring/multiple assignment instead of positional array indexing for readability.
- Prefer `Data.define`/`Struct.new` for small immutable value objects over a hand-rolled class.
- Enable `# frozen_string_literal: true` project-wide to catch accidental in-place string mutation.
- Use `Module#prepend` (with `super`) to wrap/intercept existing method behavior instead of physically overwriting it.
- Use `find_each`/`find_in_batches` for iterating large tables; never `.all.each` on an unbounded table.
- Know the difference between `.count`, `.length`, and `.size` on an ActiveRecord association before choosing one.
- Pair model-level validations with database-level constraints (`NOT NULL`, FKs, unique indexes) as a backstop.
- Bound every `retry` with an attempt counter; never retry unconditionally forever.
- Define a base application error class so a boundary layer can rescue every domain error uniformly.
- Use shared examples for common behavior across multiple types instead of duplicating specs.
