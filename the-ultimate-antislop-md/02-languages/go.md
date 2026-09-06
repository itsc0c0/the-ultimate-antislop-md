# Go

## Formatting & Naming Conventions

- **DO:** Run `gofmt` (or `goimports`, which also fixes import grouping) on every file before it is committed. Go's tooling treats formatting as non-negotiable, and unformatted code is an immediate signal of low-effort or non-native output.
- **DON'T:** Hand-align struct fields, comments, or `=` signs with manual spaces. `gofmt` owns alignment; manual spacing rots the moment a field is renamed and creates noisy diffs.
- **DO:** Use `MixedCaps` (or `mixedCaps` for unexported identifiers) instead of underscores. `max_retry_count` is not idiomatic Go; `maxRetryCount` or `MaxRetryCount` is.
  ```go
  // DON'T
  var max_retry_count = 3
  func Get_User(id int) (*User, error) { ... }

  // DO
  var maxRetryCount = 3
  func GetUser(id int) (*User, error) { ... }
  ```
- **DO:** Keep exported names free of stuttering with their package name. A type in package `user` should be `user.Profile`, not `user.UserProfile`, because callers already see the package qualifier.
- **DON'T:** Name a package `util`, `common`, `helpers`, or `base`. These are dumping grounds that say nothing about what the package does and tend to accumulate unrelated code; name packages after what they provide (`ratelimit`, `pathutil` only if genuinely about paths, `stripeclient`).
- **DO:** Keep package names short, lowercase, and free of underscores or mixedCaps — `httpclient`, not `HTTPClient` or `http_client`. The import site (`httpclient.New()`) is the actual API surface users read.
- **DO:** Prefer short, scoped variable names for short-lived local values (`i`, `err`, `buf`, `r`) and longer, descriptive names as scope and lifetime grow. A tight loop variable named `currentIterationIndex` is noise; a package-level exported field named `n` is not self-documenting.
- **DON'T:** Add a `Get` prefix to simple accessors. Go convention is `user.Name()`, not `user.GetName()`; reserve verbs like `Get`/`Fetch` for operations that actually do work (I/O, computation), not field access.
- **DO:** Name single-method interfaces with an `-er` suffix that describes the method: `Reader`, `Writer`, `Stringer`, `Closer`. This is one of the most recognizable idioms in the standard library and signals intent immediately.
- **DO:** Keep receiver names short and consistent across all methods of a type (typically one or two letters derived from the type name, e.g. `s *Server`). Switching between `this`, `self`, and full words across methods of the same type reads as generated, inconsistent code.
  ```go
  // DON'T — inconsistent receivers
  func (this *Server) Start() error { ... }
  func (s *Server) Stop() error { ... }
  func (server *Server) Addr() string { ... }

  // DO
  func (s *Server) Start() error { ... }
  func (s *Server) Stop() error { ... }
  func (s *Server) Addr() string { ... }
  ```
- **DON'T:** Use ALL_CAPS or SCREAMING_SNAKE for constants like other languages do. Go constants use the same `MixedCaps` rules as everything else (`const MaxRetries = 3`, not `const MAX_RETRIES = 3`), except for the rare case of mirroring an external protocol's literal names.
- **DO:** Write doc comments as a full sentence starting with the identifier's name, since `go doc` and godoc.org extract them verbatim. `// Marshal returns the JSON encoding of v.` is correct; `// this marshals stuff` is not.

## Package Layout & Project Structure

- **DO:** Organize packages by what they provide to callers (a cohesive API), not by architectural layer name. A `payments` package with types and functions is more idiomatic than parallel `models`, `services`, `controllers` packages mirroring a different language's MVC conventions.
- **DON'T:** Create a package per file mechanically, or one giant `pkg/` dumping ground with no internal boundaries. Package boundaries should reflect real API boundaries — group code that's used together and hide the rest.
- **DO:** Use an `internal/` directory to prevent packages from being imported outside the module (or outside a subtree). The Go toolchain enforces this at compile time, which is stronger than a comment saying "please don't import this."
  ```
  myapp/
    cmd/myapp/main.go        // thin entrypoint
    internal/server/         // not importable from other modules
    internal/store/
    pkg/publicapi/           // ok to import externally, if truly public
    go.mod
  ```
- **DON'T:** Put business logic in `main.go` or in a catch-all `cmd` package. Keep `main` limited to wiring: parsing flags/config, constructing dependencies, and calling into testable packages.
- **DO:** Avoid import cycles by designing dependencies to flow in one direction (e.g., domain packages know nothing about HTTP handlers; handlers import domain, not vice versa). Go refuses to compile circular imports, so this is a design problem, not a style nit.
- **DON'T:** Create a `models` package that every other package imports for shared structs, turning it into a de facto global namespace. Prefer defining types close to the code that owns their invariants; only extract a shared package when the sharing is real and stable.
- **DO:** Keep the module's root `go.mod` name matching its actual import path (e.g., a GitHub-hosted module's path is `github.com/org/repo`). Mismatches break `go get` for downstream consumers.
- **DO:** Group standard library imports, third-party imports, and local imports into separate blocks, and let `goimports` maintain the ordering and sorting automatically rather than hand-editing it.
  ```go
  import (
      "context"
      "fmt"

      "github.com/google/uuid"

      "github.com/myorg/myapp/internal/store"
  )
  ```
- **DON'T:** Export every type and function "just in case." Keep the exported surface minimal and intentional; unexported identifiers are easier to change later without breaking callers.

## Error Handling — Explicit Returns

- **DO:** Check every error returned by a call before using the result. Go's `(value, error)` convention exists precisely so failures are visible at the call site, not hidden behind an exception that might be caught three stack frames away.
  ```go
  // DON'T
  data, _ := os.ReadFile(path)
  process(data)

  // DO
  data, err := os.ReadFile(path)
  if err != nil {
      return fmt.Errorf("reading %s: %w", path, err)
  }
  process(data)
  ```
- **DON'T:** Assign an error to `_` to silence the compiler or linter unless the failure mode is genuinely inconsequential and that reasoning is spelled out in a comment. A discarded error is a discarded signal that something went wrong.
- **DO:** Return errors as the last return value, following the standard library convention `(T, error)`. Deviating (e.g., `(error, T)`) breaks reader expectations and idiomatic patterns like the early-return `if err != nil` check.
- **DO:** Return early on error rather than nesting the "happy path" inside an `if err == nil` block. Flat, guard-clause-style error handling keeps the success path visually unindented and easy to scan.
  ```go
  // DON'T — nested happy path
  if err == nil {
      if user != nil {
          return user.Name, nil
      }
  }
  return "", err

  // DO — guard clauses
  if err != nil {
      return "", err
  }
  if user == nil {
      return "", errors.New("user not found")
  }
  return user.Name, nil
  ```
- **DON'T:** Log an error and then also return it up the call stack, which typically causes the same failure to be logged multiple times as it propagates. Either handle and log the error at the point it's finally consumed, or return it — not both at every layer.
- **DO:** Use `errors.New` or `fmt.Errorf` for simple, unstructured errors, and define a custom error type only when callers need to programmatically distinguish or extract data from the failure.
- **DO:** Treat a `nil` interface holding a non-nil concrete error type as a real bug class and avoid it: never return a typed `*MyError` pointer through an `error`-typed return when the pointer can be `nil`, since the resulting interface value is non-nil even though it "looks" nil.
  ```go
  // DON'T — classic nil-interface trap
  func doWork() *MyError { return nil }
  var err error = doWork() // err != nil, even though doWork returned nil!

  // DO — return the interface type directly
  func doWork() error { return nil }
  ```
- **DON'T:** Use a boolean "ok" pattern (`(T, bool)`) as a substitute for real error reporting when the caller needs to know *why* something failed, not just *whether*. Reserve `(T, bool)` for lookups where absence is a normal, single-reason outcome (map access, type assertions).

## Error Handling — Wrapping & Sentinel Errors

- **DO:** Wrap errors with `fmt.Errorf("...: %w", err)` when adding context, so callers can still unwrap and inspect the original cause with `errors.Is` / `errors.As`. Wrapping preserves the chain instead of collapsing it into an opaque string.
  ```go
  func loadConfig(path string) (*Config, error) {
      f, err := os.Open(path)
      if err != nil {
          return nil, fmt.Errorf("loading config: %w", err)
      }
      defer f.Close()
      // ...
  }
  ```
- **DON'T:** Use `%v` when you mean `%w`. `%v` stringifies the error and destroys the wrapping chain, so `errors.Is(err, os.ErrNotExist)` silently stops working further up the stack.
- **DO:** Define sentinel errors as package-level `var Err... = errors.New("...")` values when callers need to check for a specific, well-known condition, and compare with `errors.Is`, never `==`, once wrapping is involved.
  ```go
  var ErrNotFound = errors.New("item not found")

  func Find(id string) (*Item, error) {
      if !exists(id) {
          return nil, fmt.Errorf("find %s: %w", id, ErrNotFound)
      }
      // ...
  }

  // caller
  if errors.Is(err, ErrNotFound) { ... }
  ```
- **DO:** Define a custom error struct implementing `Error() string` (and optionally `Unwrap() error`) when the caller needs structured data from the failure (e.g., a validation error with a field name), and extract it with `errors.As`.
- **DON'T:** Write custom error messages that start with a capital letter or end with punctuation. Go convention is lowercase, unpunctuated error strings, since they're frequently wrapped and concatenated into larger messages (`fmt.Errorf("reading config: %w", err)` reads oddly if `err` already starts with "Reading" and ends with a period).
- **DO:** Keep error messages specific enough to debug without a stack trace — include the operation and the relevant identifier (`fmt.Errorf("fetching user %d: %w", id, err)`), since Go doesn't provide automatic stack traces on error values.
- **DON'T:** Build ad hoc string matching against `err.Error()` to detect specific failures (`strings.Contains(err.Error(), "not found")`). This is brittle across error message changes and library versions; use sentinel errors or typed errors with `errors.Is`/`errors.As` instead.
- **DO:** Use `errors.Join` (Go 1.20+) when a function can legitimately fail for more than one independent reason at once (e.g., closing multiple resources), instead of discarding all but the first error.

## Panics, Recover, and When Not to Use Them

- **DO:** Reserve `panic` for truly unrecoverable programmer errors — invariant violations, impossible states, or startup-time failures where continuing would be unsafe (e.g., a required config file that can't be found before the server starts). It is not a substitute for normal error returns.
- **DON'T:** Use `panic`/`recover` as a control-flow mechanism for expected failure paths, the way exceptions are used in Java or Python. Callers of a Go function expect failures to surface through the returned `error`, and a library that panics on ordinary bad input (a malformed request, a missing key) breaks that contract.
  ```go
  // DON'T — panics as flow control for an expected condition
  func MustParse(s string) int {
      n, err := strconv.Atoi(s)
      if err != nil {
          panic(err) // caller has no way to handle this gracefully
      }
      return n
  }

  // DO — return the error, let the caller decide
  func Parse(s string) (int, error) {
      return strconv.Atoi(s)
  }
  ```
- **DO:** Use `recover` at well-defined boundaries — most commonly in an HTTP middleware that converts a panicking handler into a 500 response — so a single bad request cannot take down the whole process, while still surfacing the panic in logs/metrics.
- **DON'T:** Sprinkle `recover()` throughout business logic to "swallow" panics silently. Recovering without logging or re-evaluating program state hides real bugs and can leave data structures in a half-mutated, inconsistent state.
- **DO:** Name functions that can panic on invalid input with a `Must` prefix (`template.Must`, `regexp.MustCompile`) so the panicking behavior is visible at the call site, and reserve them for cases where the input is a compile-time constant, not user input.
- **DON'T:** Recover from a panic inside a goroutine and assume the rest of the program is fine. An unrecovered panic in any goroutine crashes the entire process (recover only works in the same goroutine, at a deferred call), so every long-running goroutine that isn't tightly supervised needs its own recover-and-log wrapper.
- **DO:** Let panics from truly unexpected conditions (nil pointer dereference, index out of range, a failed type assertion) surface and crash during development and testing — that visibility is a feature, since it means the bug gets fixed instead of being masked.

## Concurrency — Goroutines & Channels

- **DO:** Reach for channels when coordinating or communicating between goroutines, following "share memory by communicating" rather than manually coordinating shared state. A channel-based pipeline is often easier to reason about than a set of goroutines polling shared variables.
  ```go
  func worker(jobs <-chan int, results chan<- int) {
      for j := range jobs {
          results <- j * j
      }
  }
  ```
- **DON'T:** Launch a goroutine per unit of work with no bound when the input is large or externally controlled (e.g., one goroutine per incoming HTTP request body line). Unbounded goroutine creation can exhaust memory and scheduler resources; use a worker pool or a semaphore to cap concurrency.
- **DO:** Close a channel from the sender side only, and only once — never from the receiver, and never more than once (a double close panics). If multiple goroutines might send on the same channel, coordinate closing through a separate mechanism (e.g., a `sync.WaitGroup` plus a dedicated closer goroutine).
- **DON'T:** Send on a channel without a plan for what happens if no one is ever listening. An unbuffered send with no receiver blocks forever; always reason about buffer size, `select` with a `default`/timeout, or context cancellation.
  ```go
  // DON'T — can block forever if the receiver already gave up
  ch <- result

  // DO — respect cancellation
  select {
  case ch <- result:
  case <-ctx.Done():
      return ctx.Err()
  }
  ```
- **DO:** Prefer buffered channels with an explicit, justified capacity over unbuffered ones only when you have a concrete reason (e.g., decoupling producer/consumer bursts); default to unbuffered channels for straightforward synchronization since their blocking behavior is easier to reason about.
- **DO:** Pass a `context.Context` as the first parameter of any function that does I/O, calls out to another service, or might run long, and honor `ctx.Done()` for cancellation and timeouts. This is the standard mechanism for propagating deadlines and cancellation across goroutine and API boundaries.
  ```go
  func fetch(ctx context.Context, url string) ([]byte, error) {
      req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
      if err != nil {
          return nil, err
      }
      resp, err := http.DefaultClient.Do(req)
      // ...
  }
  ```
- **DON'T:** Store a `context.Context` inside a struct field for later use. Contexts are meant to flow explicitly through call chains as the first argument; stashing one in a struct hides its lifetime and makes cancellation semantics unclear.
- **DO:** Use `errgroup.Group` (from `golang.org/x/sync/errgroup`) when launching several goroutines that should all be canceled if one fails, instead of hand-rolling error aggregation with channels and a `WaitGroup`.

## Concurrency — sync Primitives & Data Races

- **DO:** Protect shared mutable state accessed from multiple goroutines with a `sync.Mutex` (or `sync.RWMutex` for read-heavy access) whenever channels aren't a natural fit. Both channels and mutexes are legitimate; use whichever makes the specific access pattern clearer.
  ```go
  type Counter struct {
      mu sync.Mutex
      n  int
  }

  func (c *Counter) Inc() {
      c.mu.Lock()
      defer c.mu.Unlock()
      c.n++
  }
  ```
- **DON'T:** Read or write a shared variable from more than one goroutine without synchronization "because it's just an int increment." Unsynchronized concurrent access is a data race regardless of how small the operation looks — it is undefined behavior in the Go memory model, not just a risk of a stale value.
- **DO:** Run tests and, ideally, production canaries with the race detector (`go test -race`, `go run -race`) whenever goroutines touch shared state. The race detector catches many concurrency bugs that are otherwise invisible until they cause an intermittent production failure.
- **DON'T:** Copy a `sync.Mutex` (or any type embedding one, like many stdlib synchronization types) by value. Copying a mutex duplicates its internal state, so the copy is not actually protecting the same critical section — `go vet` flags this, and it should never be silenced.
  ```go
  // DON'T — copies the mutex, breaking mutual exclusion
  func process(c Counter) { c.mu.Lock(); ... }

  // DO — pass by pointer
  func process(c *Counter) { c.mu.Lock(); ... }
  ```
- **DO:** Use `sync/atomic` (or Go 1.19+'s `atomic.Int64`/`atomic.Bool`/etc. wrapper types) for simple counters and flags accessed concurrently, when a full mutex is overkill — but only for the specific documented atomic operations, not as a general substitute for mutexes protecting compound state.
- **DO:** Use `sync.Once` to guarantee one-time initialization (e.g., lazy singleton setup) that's safe under concurrent first access, instead of hand-rolling a double-checked-locking pattern.
- **DON'T:** Hold a lock while calling into code you don't control (a callback, an interface method implemented elsewhere), since that code might call back into the same lock and deadlock, or might block for an unbounded time and stall every other goroutine waiting on that mutex.
- **DO:** Keep critical sections as small as possible — lock, mutate, unlock — and do expensive work (I/O, allocation-heavy computation) outside the lock whenever the data allows it.

## Concurrency — Avoiding Goroutine Leaks

- **DO:** Ensure every goroutine you start has a clear, reachable exit condition — a closed channel it ranges over, a `context.Done()` it selects on, or a bounded amount of work. A goroutine with no way to terminate leaks for the life of the process.
  ```go
  // DON'T — leaks if ctx is never the way this exits
  func poll(ch <-chan Event) {
      go func() {
          for e := range ch { handle(e) } // fine only if ch is guaranteed to close
      }()
  }

  // DO — give it an explicit way out
  func poll(ctx context.Context, ch <-chan Event) {
      go func() {
          for {
              select {
              case e := <-ch:
                  handle(e)
              case <-ctx.Done():
                  return
              }
          }
      }()
  }
  ```
- **DON'T:** Start a goroutine that sends a single result on an unbuffered channel without confirming a receiver will always show up. If the caller stops listening (e.g., due to a timeout elsewhere), that goroutine blocks on the send forever.
- **DO:** Use a `sync.WaitGroup` to track goroutines you expect to finish, and call `Wait()` at the point where the program (or test) needs to know they're all done — don't let "fire and forget" goroutines outlive their usefulness silently.
- **DO:** Set a deadline or use `context.WithTimeout`/`WithCancel` around any goroutine that waits on external I/O (network calls, long-running computation) so a hung dependency can't pin the goroutine indefinitely.
- **DON'T:** Assume `defer cancel()` alone protects you if the `context.CancelFunc` is captured by a goroutine that outlives the function where it was deferred. Verify the goroutine's own lifetime is tied to the same context, not just that `cancel` gets called eventually.
- **DO:** Audit long-running services periodically with `pprof`'s goroutine profile (`/debug/pprof/goroutine`) to catch a slow, cumulative goroutine leak before it becomes an outage — a rising goroutine count over time under steady load is the classic symptom.
- **DON'T:** Spawn a goroutine inside a loop that captures the loop variable incorrectly in Go versions before 1.22 (each iteration reused the same variable). Either capture it explicitly as a parameter or upgrade the toolchain's per-iteration semantics — but always verify which behavior your Go version has.
  ```go
  // Pre-1.22 pitfall: all goroutines may see the same, final i
  for i := 0; i < 3; i++ {
      go func() { fmt.Println(i) }()
  }

  // DO — pass i explicitly (works on any Go version)
  for i := 0; i < 3; i++ {
      go func(i int) { fmt.Println(i) }(i)
  }
  ```

## Interfaces & Composition over Inheritance

- **DO:** Define interfaces at the point of consumption (the caller's package), not alongside the concrete implementation. This is idiomatic Go: accept small interfaces, return concrete types, and let each consumer declare only the behavior it actually needs.
- **DON'T:** Design large, "just in case" interfaces with many methods up front, mirroring a Java-style abstract-base-class hierarchy. Small interfaces (one to three methods) are easier to implement, mock, and compose; `io.Reader` and `io.Writer` are the canonical example.
  ```go
  // DON'T — a bloated interface forcing every implementer to support everything
  type Storage interface {
      Get(key string) ([]byte, error)
      Put(key string, val []byte) error
      Delete(key string) error
      List() ([]string, error)
      Backup() error
      Restore() error
  }

  // DO — split by actual usage
  type Getter interface { Get(key string) ([]byte, error) }
  type Putter interface { Put(key string, val []byte) error }
  ```
- **DO:** Use struct embedding to compose behavior from smaller types, understanding that it is delegation with promoted methods, not inheritance — there is no dynamic dispatch back into the embedding type and no polymorphic "override" the way subclassing provides.
  ```go
  type Base struct{ Logger *log.Logger }
  func (b *Base) Log(msg string) { b.Logger.Println(msg) }

  type Service struct {
      Base // Service.Log(...) is promoted, but Base doesn't know about Service
      Name string
  }
  ```
- **DON'T:** Reach for embedding purely to save typing when a plain named field and explicit delegation would be clearer. Embedding silently promotes methods and fields into the outer type's API, which can produce surprising name collisions and an API surface the author didn't intend to expose.
- **DO:** Accept `io.Reader`, `io.Writer`, or other standard interfaces as parameters instead of concrete types like `*os.File` or `*bytes.Buffer` whenever the function only needs the interface's behavior — this is what makes stdlib I/O code composable across files, network connections, and in-memory buffers.
- **DO:** Verify interface satisfaction at compile time with a blank assignment when a package should reliably implement a given interface, catching drift immediately instead of discovering the mismatch when something tries to use it.
  ```go
  var _ io.Writer = (*MyWriter)(nil)
  ```
- **DON'T:** Model an "is-a" relationship by embedding a concrete struct and expecting virtual-method-style overriding, e.g., expecting a method on the embedded type to call back into an overridden method on the outer type. Go has no such mechanism; if you need that, pass the outer type's behavior in explicitly (a callback, or an interface field) instead of relying on embedding.
- **DO:** Keep the empty interface (`interface{}` / `any`) out of typed APIs wherever a concrete type or a generic type parameter can express the same thing; `any` throws away compile-time type checking that Go otherwise gives you for free.

## Testing — Table-Driven Tests

- **DO:** Write table-driven tests for functions with several input/output cases, iterating with `t.Run` to get named subtests, clear failure output, and the ability to run a single case in isolation.
  ```go
  func TestAdd(t *testing.T) {
      cases := []struct {
          name     string
          a, b     int
          want     int
      }{
          {"positive", 2, 3, 5},
          {"negative", -1, -1, -2},
          {"zero", 0, 0, 0},
      }
      for _, tc := range cases {
          t.Run(tc.name, func(t *testing.T) {
              if got := Add(tc.a, tc.b); got != tc.want {
                  t.Errorf("Add(%d, %d) = %d, want %d", tc.a, tc.b, got, tc.want)
              }
          })
      }
  }
  ```
- **DON'T:** Write a long sequence of copy-pasted assertions that repeat the same setup and comparison logic with slightly different literals. Table-driven tests consolidate that repetition into data and one code path, which is both easier to extend and easier to review.
- **DO:** Include both expected-success and expected-failure cases in the same table when a function returns an error, asserting `wantErr` alongside `want` rather than writing separate test functions for the error paths.
- **DON'T:** Use `t.Fatal`/`t.Fatalf` inside a goroutine spawned by a test. `FailNow` (which `Fatal` calls) must run on the test's own goroutine; calling it elsewhere doesn't stop the test correctly and can corrupt test output.
- **DO:** Use `t.Parallel()` for independent subtests to speed up the suite, but only after confirming the test cases don't share mutable state (including loop variables captured incorrectly pre-Go-1.22).
- **DO:** Prefer `testify/assert`/`require` or plain `if got != want { t.Errorf(...) }` consistently within a codebase rather than mixing several assertion styles across files — pick one convention and apply it uniformly.
- **DON'T:** Assert on exact error string contents (`if err.Error() != "..."`.) Compare with `errors.Is`/`errors.As` against sentinel or typed errors, or check `wantErr bool`/error type, so the test doesn't break every time a message is reworded.
- **DO:** Use `go test -cover` (and `-coverprofile` for a detailed HTML report) to find untested branches, especially error-handling paths that are easy to forget when writing tests by hand.
- **DO:** Put test helper functions that call `t.Fatal`/`t.Errorf` behind `t.Helper()` so failure line numbers point at the calling test, not at the helper's internals.

## Tooling — go vet, golangci-lint, and Friends

- **DO:** Run `go vet ./...` as a baseline check in CI and locally — it catches real bugs (format string mismatches, mutex copies, unreachable code, struct tags with typos) with essentially zero false positives, so a failure should always be fixed, not suppressed.
- **DO:** Adopt `golangci-lint` (a meta-linter aggregating `staticcheck`, `errcheck`, `govet`, `unused`, and others) with a project-level config file so the same rule set runs in CI as on a contributor's machine.
  ```yaml
  # .golangci.yml
  linters:
    enable:
      - errcheck
      - govet
      - staticcheck
      - unused
      - gosimple
  ```
- **DON'T:** Blanket-disable a linter category (e.g., turning off `errcheck` entirely) just to silence noisy findings. Fix real violations or add a narrowly scoped, justified `//nolint` comment on the specific line instead of losing the check everywhere.
- **DO:** Enable `errcheck` specifically, since it's the tool that catches ignored error returns — the single most common correctness gap in both hand-written and AI-generated Go code.
- **DO:** Run `go build ./...` and `go vet ./...` before claiming code compiles or is correct; never assert that generated Go code "should work" without actually running the toolchain against it.
- **DON'T:** Rely on an IDE's inline squiggles as the only verification step. CI should run the same `go vet`/`golangci-lint`/`go test` commands a contributor runs locally, so nothing passes review only because a particular editor configuration happened to catch it.
- **DO:** Use `staticcheck` for deeper static analysis beyond `go vet` — it flags dead code, deprecated API usage, inefficient string concatenation, and more subtle correctness issues.
- **DO:** Pin linter and Go toolchain versions in CI configuration (and ideally in `go.mod`'s `go` directive / a `tools.go` file for linter binaries) so lint results are reproducible across machines and over time.

## Common AI-Assistant Mistakes in Go

- **DON'T:** Discard an error return with `_` by default when generating example or "quick" code. AI assistants frequently do this to keep examples short, but it trains a habit that ships unchecked errors into real codebases; always check `err` or explicitly justify why not.
- **DON'T:** Invent standard library functions or package paths that sound plausible but don't exist (a hallucinated `strings.Reverse`, a nonexistent `slices.Filter` before it actually shipped, or an imagined `http.NewRequestWithTimeout`). Verify every stdlib call against real documentation or by compiling, especially for less commonly used packages.
- **DON'T:** Launch a goroutine to "make it concurrent" without adding the synchronization the new concurrency requires. A common AI-generated bug is converting a sequential loop into `go func(){...}()` calls that mutate a shared slice or map with no mutex and no channel — introducing a data race that wasn't there before.
  ```go
  // DON'T — data race: concurrent map writes with no synchronization
  results := map[string]int{}
  for _, url := range urls {
      go func(u string) {
          results[u] = fetch(u) // concurrent write to a plain map
      }(url)
  }
  ```
- **DON'T:** Write to a plain Go map from multiple goroutines under any circumstance — even for "just adding a key," this panics with "concurrent map writes" or corrupts memory. Use a `sync.Mutex`-protected map, `sync.Map` for specific access patterns, or channel-based aggregation instead.
- **DON'T:** Add a `context.Context` parameter that is accepted but never actually used for cancellation (never passed to the underlying I/O call, never checked in a `select`). This gives a false impression of cancellation support without providing it.
- **DON'T:** Generate a `panic(err)` inside library code as a stand-in for proper error propagation, especially in code paths handling external or user-supplied input. This is a frequent shortcut in generated code that turns a recoverable failure into a process crash for the caller.
- **DON'T:** Fabricate a third-party package name or API method that resembles a real one (e.g., a plausible-sounding function on a popular HTTP router or ORM that isn't actually in its API) without verifying against the library's actual documentation or source.
- **DON'T:** Claim code "handles errors properly" or "is safe for concurrent use" without having actually traced every returned error and every shared variable access. State these guarantees only after checking them, and flag genuine uncertainty rather than asserting correctness by default.
- **DON'T:** Default to `interface{}`/`any` and runtime type assertions to work around a generic or interface design problem in older-style Go code, when a properly scoped interface, a type parameter (Go 1.18+ generics), or a small set of concrete overloads would give compile-time safety instead.
- **DON'T:** Forget `defer resource.Close()` (or equivalent cleanup) immediately after a resource is successfully acquired — files, DB connections, HTTP response bodies, locks. Generated code that opens a resource and processes it in several branches, but only closes it on one path, leaks resources on every other path.
  ```go
  // DON'T — leaks the response body on early return
  resp, err := http.Get(url)
  if err != nil {
      return err
  }
  if resp.StatusCode != 200 {
      return fmt.Errorf("bad status: %d", resp.StatusCode) // body never closed
  }
  defer resp.Body.Close()

  // DO — defer immediately after the successful acquire
  resp, err := http.Get(url)
  if err != nil {
      return err
  }
  defer resp.Body.Close()
  if resp.StatusCode != 200 {
      return fmt.Errorf("bad status: %d", resp.StatusCode)
  }
  ```

## Quick Checklist
- [ ] Every file passes `gofmt`/`goimports` with no manual alignment.
- [ ] Identifiers use `MixedCaps`; packages are short, lowercase, and named for what they provide.
- [ ] No package stutters its own name in exported identifiers (`user.Profile`, not `user.UserProfile`).
- [ ] `internal/` is used to enforce real API boundaries.
- [ ] `main` stays thin; business logic lives in testable packages.
- [ ] Every returned error is checked or its omission is explicitly justified.
- [ ] Errors are wrapped with `%w`, not `%v`, when context is added.
- [ ] Sentinel/typed errors are compared with `errors.Is`/`errors.As`, never string matching.
- [ ] Error strings are lowercase and unpunctuated.
- [ ] `panic` is reserved for unrecoverable programmer errors, not expected failure paths.
- [ ] `recover` is used at defined boundaries (middleware, worker wrappers), always with logging.
- [ ] Every goroutine has an explicit, reachable exit condition.
- [ ] Shared mutable state is protected by a mutex, atomic op, or channel — never accessed unsynchronized.
- [ ] `sync.Mutex` and similar types are never copied by value.
- [ ] `context.Context` is the first parameter of I/O-bound functions and is actually honored via `Done()`.
- [ ] Tests and CI run with `-race` whenever goroutines touch shared state.
- [ ] Interfaces are small, defined at the point of consumption, and not designed "just in case."
- [ ] Struct embedding is used for real delegation, not simulated inheritance.
- [ ] Tests are table-driven with named subtests via `t.Run`.
- [ ] Test assertions check error type/sentinel, not exact error strings.
- [ ] `go vet ./...` and `golangci-lint run` pass cleanly in CI, with no blanket-disabled linters.
- [ ] Resources (files, connections, bodies, locks) are closed via `defer` immediately after successful acquisition.
- [ ] No fabricated stdlib or third-party API calls — every call is verified against real documentation or compiled code.
- [ ] Plain maps are never written to concurrently without synchronization.
- [ ] No `Get`-prefixed simple accessors; no ALL_CAPS constants.
