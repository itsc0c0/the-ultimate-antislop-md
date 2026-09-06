# C

## Memory Management Discipline

- **DO:** Pair every `malloc`/`calloc`/`realloc` with exactly one corresponding `free`, and make ownership of each allocation clear from the code structure — ideally one function or module that unambiguously "owns" the pointer and is responsible for freeing it.
  ```c
  // DON'T — unclear who owns buf, easy to double-free or leak
  char *buf = malloc(256);
  process(buf); // does process() free it? unclear from the call site

  // DO — ownership documented and localized
  char *buf = malloc(256);
  if (!buf) return -1;
  process(buf);   // process() only reads/writes, never frees
  free(buf);
  buf = NULL;
  ```
- **DON'T:** Assume `malloc` always succeeds. Check the return value for `NULL` before dereferencing it — under memory pressure or with a corrupted allocator, `malloc` failing and going unchecked is a direct path to a null-pointer dereference crash or worse.
- **DO:** Set a pointer to `NULL` immediately after freeing it, especially for any pointer that might be checked, freed, or reused later in the same function. A dangling non-NULL pointer that outlives its allocation is the root cause of most use-after-free bugs.
- **DON'T:** Call `free()` on a pointer more than once. A double free corrupts the heap allocator's internal bookkeeping and is a well-known, actively exploited vulnerability class; always null the pointer after freeing so a stray second `free(ptr)` becomes a harmless no-op (`free(NULL)` is well-defined and does nothing).
- **DO:** Check the return value of `realloc` into a *different* variable than the one being resized, since `realloc` returns `NULL` on failure without freeing the original block — overwriting your only pointer to it with `NULL` leaks the original allocation.
  ```c
  // DON'T — leaks the original buffer if realloc fails
  buf = realloc(buf, new_size);

  // DO — preserve the original pointer on failure
  char *tmp = realloc(buf, new_size);
  if (!tmp) { free(buf); return NULL; }
  buf = tmp;
  ```
- **DO:** Use tools like Valgrind, AddressSanitizer (`-fsanitize=address`), or LeakSanitizer during development and in CI to catch leaks, use-after-free, and heap corruption automatically, rather than relying on manual code review alone to catch every allocation/free pair.
- **DON'T:** Mix allocation strategies for the same object across a codebase (e.g., sometimes stack-allocating a struct, sometimes heap-allocating it, with callers unsure which). Document and enforce a single, consistent ownership/lifetime convention per type, especially at API boundaries.
- **DO:** Prefer stack allocation (automatic storage) for data whose size is known and whose lifetime doesn't need to outlive the current scope. Reach for `malloc` only when the size is dynamic, the data must outlive the current function, or the object is too large for the stack.
- **DON'T:** Return a pointer to a local (stack-allocated) variable from a function. The moment the function returns, that stack frame is invalid, and the returned pointer is dangling — this compiles without error but produces undefined behavior on use.
  ```c
  // DON'T — returns a dangling pointer to a destroyed stack frame
  char *make_greeting(void) {
      char buf[32];
      snprintf(buf, sizeof(buf), "hello");
      return buf; // buf's storage no longer exists after return
  }

  // DO — heap-allocate (caller frees) or use an output buffer
  char *make_greeting(void) {
      char *buf = malloc(32);
      if (buf) snprintf(buf, 32, "hello");
      return buf;
  }
  ```

## Avoiding Leaks, Use-After-Free, and Double-Free

- **DO:** Design every function that allocates and returns a resource with a matching, clearly named "destroy"/"free" counterpart (`thing_create`/`thing_destroy`), and document who calls it and when as part of the type's contract.
- **DON'T:** Free a resource and continue using it on any code path, including error-handling paths added later during maintenance. A common source of use-after-free is a `goto cleanup`/early-return refactor that frees a resource but leaves a later line still referencing it.
- **DO:** Use a single, consistent cleanup pattern per function — the `goto cleanup` / `goto err` idiom in C is idiomatic and battle-tested for functions with multiple resources and multiple failure points, since it centralizes freeing logic instead of duplicating it at every early return.
  ```c
  int process_file(const char *path) {
      FILE *f = NULL;
      char *buf = NULL;
      int rc = -1;

      f = fopen(path, "r");
      if (!f) goto cleanup;

      buf = malloc(BUF_SIZE);
      if (!buf) goto cleanup;

      /* ... use f and buf ... */
      rc = 0;

  cleanup:
      free(buf);
      if (f) fclose(f);
      return rc;
  }
  ```
- **DON'T:** Free memory inside a loop or callback and then continue iterating over a structure that assumed the freed node was still linked (e.g., freeing a linked-list node before advancing to `node->next`). Save the next pointer before freeing the current node.
- **DO:** Track allocation and free counts (or use an arena/pool allocator with a single teardown point) for long-running programs where manual pairing across many call sites becomes error-prone — a bump allocator or arena that's freed all at once at a well-defined checkpoint eliminates an entire class of leak and use-after-free bugs for that data's lifetime.
- **DON'T:** Assume a leaked allocation "doesn't matter" because the process is short-lived. Short-lived-today code is frequently reused in a long-running context later (a library, a server loop), and even in genuinely short-lived programs, leaks make it harder to use leak-detection tools to find *other*, real bugs.
- **DO:** Run the full test suite under AddressSanitizer (`-fsanitize=address,undefined`) as a standard CI step for any C project handling untrusted input or doing significant manual memory management — it turns silent memory corruption into an immediate, loud test failure with a precise stack trace.
- **DON'T:** Free a pointer that was never heap-allocated (a stack array, a string literal, a pointer into the middle of another allocation). `free()` must be called only on a pointer previously returned by `malloc`/`calloc`/`realloc` (or `NULL`); calling it on anything else is undefined behavior, often crashing immediately or corrupting the heap silently.

## Buffer Overflows & Bounds Safety

- **DO:** Always know and check the size of a buffer before writing into it, and use the bounded variants of string/memory functions (`snprintf` instead of `sprintf`, `strncpy`/`strlcpy` instead of `strcpy`, `memcpy` with a verified length) everywhere user- or externally-derived data flows into a fixed-size buffer.
  ```c
  // DON'T — no bound; overflow if src is long
  char name[32];
  strcpy(name, src);

  // DO — explicit, checked bound
  char name[32];
  snprintf(name, sizeof(name), "%s", src);
  ```
- **DON'T:** Trust a length field or size argument that comes from external input (a network packet, a file header, a command-line argument) without validating it against the actual buffer capacity before using it in an allocation or copy. This is the root cause of a large fraction of real-world CVEs in C code.
- **DO:** Use `sizeof(buf)` (on real arrays, not decayed pointers) rather than a hardcoded numeric literal for a buffer's size wherever possible, so a later change to the buffer's declared size doesn't silently desynchronize from a copy/paste literal elsewhere.
- **DON'T:** Assume `sizeof` on a pointer parameter gives you the size of the array it points to. Once an array decays to a pointer (as a function parameter), `sizeof(ptr)` gives the pointer's size (commonly 8 on 64-bit), not the array's — always pass the actual buffer length as a separate parameter.
  ```c
  // DON'T — sizeof(buf) here is sizeof(char*), not the caller's array size
  void fill(char *buf) {
      memset(buf, 0, sizeof(buf)); // wrong! only zeroes 8 bytes
  }

  // DO — pass the length explicitly
  void fill(char *buf, size_t len) {
      memset(buf, 0, len);
  }
  ```
- **DO:** Validate array indices against both bounds (`0 <= i && i < n`) before indexing, especially when the index is computed from arithmetic on external input — an off-by-one or integer underflow in the index computation is enough to read or write out of bounds.
- **DON'T:** Perform pointer or size arithmetic that can overflow (e.g., `malloc(count * size)` where `count * size` overflows `size_t` and wraps to a small number, causing a subsequent loop to write far past the undersized allocation). Use `calloc(count, size)` (which checks for overflow internally per the C standard) instead of manual multiplication when allocating arrays.
  ```c
  // DON'T — count * size can overflow, undersizing the allocation
  int *arr = malloc(count * sizeof(int));

  // DO — calloc checks for multiplication overflow
  int *arr = calloc(count, sizeof(int));
  ```
- **DO:** Null-terminate every string buffer explicitly when using functions that don't guarantee termination on truncation (`strncpy` famously does not null-terminate if the source is exactly as long as or longer than the destination) — verify the exact truncation semantics of whichever bounded function you use.
- **DON'T:** Read or write past the end of an array "just this once" in a loop bound (`for (i = 0; i <= n; i++)` over an array of size `n`). This classic off-by-one is one of the most common bugs in both hand-written and generated C, and it's worth deliberately re-checking every loop bound against the actual array size.

## Pointer Safety

- **DO:** Check every pointer that can legitimately be `NULL` (a `malloc` result, a return value documented as possibly `NULL`, a lookup that can miss) before dereferencing it, at the point where the possibility is real — not everywhere defensively, which just adds noise, but everywhere it's actually possible.
- **DON'T:** Dereference a pointer received as a function parameter without either documenting (and enforcing via an assertion or explicit check) that it must be non-null, or handling the `NULL` case gracefully. A public API function that silently crashes on `NULL` input is a poor contract; either check or clearly document the precondition.
  ```c
  // DON'T — silent crash on NULL, no documented contract
  int str_len(const char *s) {
      int n = 0;
      while (s[n]) n++;
      return n;
  }

  // DO — explicit precondition check
  int str_len(const char *s) {
      if (!s) return -1;
      int n = 0;
      while (s[n]) n++;
      return n;
  }
  ```
- **DO:** Keep pointer arithmetic within the bounds of a single array object (including one-past-the-end, which is valid for comparison but not for dereferencing). Pointer arithmetic that walks outside the object it was derived from is undefined behavior even if the resulting address happens to be readable memory.
- **DON'T:** Cast a pointer to an unrelated type and dereference it, bypassing the type system's aliasing rules (strict aliasing), except through the explicitly sanctioned mechanisms (`memcpy`, a `union` in implementations that support it, or `unsigned char *` byte access). Violating strict aliasing lets the optimizer produce genuinely incorrect code, not just a style problem.
- **DO:** Use `const` on pointer parameters that a function only reads, both to document intent and to let the compiler catch accidental writes. `void process(const char *data, size_t len)` tells every caller and every future maintainer that `process` won't modify `data`.
- **DON'T:** Compare pointers from different allocations with `<`/`>`/`<=`/`>=` (only `==`/`!=` are well-defined across unrelated objects). Pointer relational comparison is only defined within the same array/object.
- **DO:** Initialize every pointer at declaration, even if only to `NULL`, rather than leaving it holding an indeterminate value. An uninitialized pointer that happens to look valid (a stray stack value that resembles a real address) is far more dangerous than one guaranteed to be `NULL`, since dereferencing `NULL` fails loudly while dereferencing garbage may not.
- **DON'T:** Assume two pointers are non-aliasing (or aliasing) without verifying it, when writing performance-sensitive code that the compiler could otherwise optimize aggressively (e.g., via `restrict`). If two pointer parameters genuinely never overlap, mark them `restrict` explicitly rather than relying on assumptions the compiler can't verify on its own.

## Undefined Behavior Pitfalls

- **DO:** Treat every occurrence of undefined behavior (signed integer overflow, out-of-bounds access, use of an uninitialized value, a data race, dereferencing a null/invalid pointer, violating strict aliasing) as a real bug to fix, even when the program "seems to work" — a compiler is free to assume UB never happens and can transform the surrounding code in ways that only manifest as a bug after an optimization level change or compiler upgrade.
- **DON'T:** Rely on signed integer overflow wrapping around predictably (like unsigned integers do). Signed overflow is undefined behavior in C, and optimizing compilers actively exploit that undefinedness (e.g., eliminating an overflow check that "can never be true" under the as-if rule) — use unsigned types for values expected to wrap, or check for overflow before it happens.
  ```c
  // DON'T — relies on UB; may be optimized away entirely
  if (a + b < a) { /* detect overflow */ }  // UB if a, b are signed and this overflows

  // DO — check before the operation, with unsigned/careful arithmetic
  if (a > INT_MAX - b) { /* overflow would occur */ }
  ```
- **DO:** Compile with `-Wall -Wextra -Werror` (GCC/Clang) as a baseline, and treat every warning as something to actually fix — warnings frequently point directly at latent undefined behavior (uninitialized variables, signed/unsigned comparison mismatches, implicit narrowing conversions).
- **DON'T:** Read an uninitialized local variable. Its value is indeterminate, and using it (even just reading it into another variable, in some cases) is undefined behavior — always initialize variables at declaration or guarantee they're assigned on every path before first use.
- **DO:** Use `-fsanitize=undefined` (UBSan) during development and in CI to catch undefined behavior at runtime that static warnings miss — signed overflow, misaligned access, invalid enum values, and more, each reported with a precise source location.
- **DON'T:** Assume the order of evaluation of function arguments or of operands around most operators is defined. `f(g(), h())` does not guarantee `g()` runs before `h()`, and modifying and reading the same variable in separate, unsequenced parts of the same expression (`i = i++ + 1;`) is undefined behavior, not merely "compiler-dependent."
- **DO:** Cast between numeric types explicitly and deliberately, especially between signed and unsigned, since implicit conversions can silently change a value's meaning (a negative `int` compared against an `unsigned` promotes to a huge positive value, breaking a bounds check that looked correct).
  ```c
  // DON'T — implicit signed/unsigned comparison bug
  int len = get_length(); // could be -1 on error
  if (len < buffer_size) { ... } // if buffer_size is size_t, len is promoted to huge unsigned value

  // DO — check the error case and compare compatible types explicitly
  int len = get_length();
  if (len < 0) { /* handle error */ }
  else if ((size_t)len < buffer_size) { ... }
  ```
- **DO:** Treat a function's declared return type and documented contract as binding, including for corner cases (what happens on empty input, on the maximum representable value, on `NULL`) — undefined behavior often hides specifically in the corner cases that look like they "obviously" should just work.

## Header & Build Hygiene

- **DO:** Guard every header against multiple inclusion, either with traditional `#ifndef`/`#define`/`#endif` include guards or `#pragma once` (widely supported, though technically non-standard) — a header included twice without a guard produces duplicate-definition compile errors or silent ODR violations.
  ```c
  #ifndef MYLIB_PARSER_H
  #define MYLIB_PARSER_H

  /* declarations */

  #endif /* MYLIB_PARSER_H */
  ```
- **DON'T:** Put function definitions (not just declarations) or non-`static`/non-`inline` variable definitions directly in a header included by multiple translation units. This causes multiple-definition linker errors, or silent, hard-to-debug One Definition Rule violations depending on the linkage.
- **DO:** Keep headers minimal — declare only what other translation units genuinely need, forward-declare structs/types where a full definition isn't required, and push implementation details into the `.c` file. This keeps compile times down and reduces unnecessary rebuilds when internals change.
- **DON'T:** `#include` a header just because some other header you need happens to include it transitively. Include exactly what a file directly uses ("include what you use"), so removing an unrelated header elsewhere doesn't silently break unrelated files that were relying on the transitive include.
- **DO:** Separate public API headers (in an `include/` directory) from internal, implementation-only headers (kept alongside their `.c` files or in a clearly marked internal directory), so consumers of a library can't accidentally depend on implementation details.
- **DON'T:** Define project-wide macros with generic, collision-prone names (`#define MAX`, `#define MIN`, `#define VERSION`) in a widely included header. Prefix macros with a project-specific namespace (`MYLIB_MAX`) or prefer `static inline` functions/enums where the semantics allow it, since macros don't respect scoping and can silently clash with system or third-party headers.
- **DO:** Use a real build system (Make with explicit dependency tracking, CMake, Meson, or similar) rather than a single hand-maintained compile command, once a project has more than a couple of source files — manual builds silently skip rebuilding files whose dependencies changed, producing stale binaries that "work" until they mysteriously don't.
- **DON'T:** Suppress compiler warnings project-wide with a blanket flag (`-w`) to make a noisy build "clean." Fix the warnings, or suppress a specific, understood one narrowly (a pragma around the specific line, with a comment) — a wall of suppressed warnings hides the next real bug among the noise.
- **DO:** Pin and document the minimum required C standard (`-std=c11`, `-std=c17`) explicitly in the build configuration rather than relying on the compiler's default, since defaults vary across compiler versions and platforms and silently change what's valid code.

## Error Handling Conventions

- **DO:** Follow the standard C convention of returning a status/error code from a function (commonly `0` for success, non-zero for failure, or a negative value alongside a positive "count" success value) and check that return value at every call site before trusting the function's output parameters.
  ```c
  int fd = open(path, O_RDONLY);
  if (fd < 0) {
      perror("open");
      return -1;
  }
  ```
- **DON'T:** Ignore the return value of a function that can fail — `write()`, `close()`, `fclose()`, `malloc()`, and even `scanf()` all have meaningful failure/partial-completion return values that are easy to drop silently, especially `close`/`fclose`, whose failure (e.g., a delayed write error on a network filesystem) is often the first indication data wasn't actually persisted.
- **DO:** Check `errno` immediately after a call that failed and is documented to set it, since `errno` is only meaningful right after such a call — any intervening call (even a seemingly unrelated one) can overwrite it before you read it.
  ```c
  // DON'T — errno may have been clobbered by an intervening call
  FILE *f = fopen(path, "r");
  log_message("attempting open"); // could reset errno
  if (!f) fprintf(stderr, "open failed: %s\n", strerror(errno));

  // DO — read errno immediately, before anything else runs
  FILE *f = fopen(path, "r");
  if (!f) {
      fprintf(stderr, "open failed: %s\n", strerror(errno));
      return -1;
  }
  ```
- **DON'T:** Use a function's return value as both a valid data result and an error sentinel without a documented, unambiguous convention (e.g., returning `-1` from a function whose valid results are all non-negative is fine; returning `0` as an error sentinel from a function where `0` is also a legitimate result is not). Where the value space is ambiguous, use an out-parameter plus a separate boolean/status return instead.
- **DO:** Define a small, consistent set of error codes (an `enum`) for a library's own API rather than overloading system `errno` values or returning inconsistent, per-function magic numbers, so callers can write one consistent error-handling pattern against your API.
- **DON'T:** Use `assert()` to validate conditions that depend on external input (a file's contents, a network message, user input). `assert` is compiled out entirely when `NDEBUG` is defined (a common release-build setting), silently removing the check exactly in the builds where untrusted input is most likely; use a real runtime check with proper error handling instead.
- **DO:** Document, for every public function, exactly what it returns on success and on each failure mode, and whether it sets `errno` — this contract is what lets callers write correct error handling instead of guessing.
- **DON'T:** Use `printf`, `scanf`, `syslog`, or any function accepting a format string with a runtime-derived (non-literal) string as the format argument. This is the classic format string vulnerability: if attacker-influenced data reaches the format parameter, it can read or write arbitrary memory via `%x`/`%n` specifiers.
  ```c
  // DON'T — format string vulnerability if user_input contains %s, %n, etc.
  printf(user_input);

  // DO — user data is a value, never the format string
  printf("%s", user_input);
  ```

## Common AI-Assistant Mistakes in C

- **DON'T:** Use `gets()`, `strcpy()`, `strcat()`, or `sprintf()` without a bound in generated code, even in "simple example" snippets. These functions have no way to prevent a buffer overflow when the source is longer than expected, and `gets()` has been removed from the C standard entirely because it cannot be used safely. Use `fgets`, `strncpy`/`snprintf`-based copying, `strncat`, and `snprintf` instead.
- **DON'T:** Generate a fixed-size buffer sized from an assumption ("names are usually under 64 characters") without validating the actual input length against that bound before copying into it. A generated function that declares `char buf[64]` and then copies unchecked external input into it is a textbook buffer overflow waiting to happen.
  ```c
  // DON'T — silent assumption about input length, no bound check
  char buf[64];
  strcpy(buf, argv[1]);

  // DO — bound the copy to the actual buffer size
  char buf[64];
  snprintf(buf, sizeof(buf), "%s", argv[1]);
  ```
- **DON'T:** Skip the `NULL` check after `malloc`/`calloc`/`realloc` in generated code "for brevity." Every allocation in code presented as correct or production-quality needs its failure path handled, even if that path is just a clean early return.
- **DON'T:** Write an off-by-one loop bound when translating "for each element" into C, particularly `<=` instead of `<` against an array's size, or an index computed as `n` instead of `n - 1` for the last valid element. Double-check every loop bound against the actual array size before presenting generated code as correct.
- **DON'T:** Forget to free every allocation on every code path, especially error paths added after the "happy path" was already written. A frequent generated-code pattern allocates a resource, then adds an early `return` for an error case discovered later, without going back to free the already-allocated resource on that new path.
- **DON'T:** Pass a runtime string (especially one built from user input or concatenation) directly as a `printf`-family format argument. Always generate `printf("%s", str)` rather than `printf(str)`, even when `str` "obviously" doesn't contain format specifiers in the example — the pattern itself is the mistake, and copying it forward into real code with real input is how format string vulnerabilities happen.
- **DON'T:** Assume a struct or array is zero-initialized by default in C the way it might be in some other languages. Automatic (stack) storage is not zero-initialized unless explicitly done (`= {0}`, `memset`, or `calloc` for heap memory) — generated code that reads a field before explicitly setting it on every path is reading indeterminate memory.
  ```c
  // DON'T — struct fields hold indeterminate values until explicitly set
  struct Point p;
  if (should_set_x) p.x = 5;
  use(p.y); // p.y may never have been initialized

  // DO — zero-initialize explicitly
  struct Point p = {0};
  if (should_set_x) p.x = 5;
  use(p.y); // well-defined: 0
  ```
- **DON'T:** Claim generated C code is "memory-safe" or "free of undefined behavior" without actually tracing every allocation, every buffer bound, and every pointer lifetime. State what was checked and what wasn't, rather than asserting safety as a default — C provides no automatic guarantees the way memory-safe languages do, so every claim of safety has to be earned by explicit reasoning, not asserted by default.

## Quick Checklist
- [ ] Every `malloc`/`calloc`/`realloc` has exactly one clearly owned matching `free`.
- [ ] Every allocation's return value is checked for `NULL` before use.
- [ ] Pointers are set to `NULL` immediately after being freed.
- [ ] No pointer is freed more than once; no pointer is freed that wasn't heap-allocated.
- [ ] `realloc`'s result is assigned to a temporary, never overwriting the only reference to the original block.
- [ ] No function returns a pointer to a local (stack) variable.
- [ ] Cleanup on multi-resource, multi-failure-path functions uses one consistent pattern (e.g., `goto cleanup`).
- [ ] Every buffer write uses a bounded function (`snprintf`, `strncpy`/`strlcpy`) with the real destination size.
- [ ] Buffer/array sizes from external input are validated against actual capacity before use.
- [ ] `sizeof` is never used on a decayed pointer parameter to infer array length.
- [ ] Array allocation multiplication uses `calloc(count, size)`, not `malloc(count * size)`.
- [ ] Every loop bound is checked against the real array size (no `<=` off-by-one).
- [ ] Every pointer parameter's nullability contract is documented and either checked or asserted.
- [ ] Pointer arithmetic stays within a single array object (plus one-past-the-end for comparisons only).
- [ ] `const` is applied to every pointer parameter a function only reads.
- [ ] No reliance on signed integer overflow, unsequenced side effects, or other undefined behavior.
- [ ] Build uses `-Wall -Wextra -Werror` (or equivalent) with warnings actually fixed, not suppressed.
- [ ] `-fsanitize=address,undefined` runs in CI or regularly in development.
- [ ] Every header has an include guard or `#pragma once`.
- [ ] Headers declare only what's needed; implementation details stay in `.c` files.
- [ ] Every fallible call's return value/status code is checked at the call site.
- [ ] `errno` is read immediately after the call that set it, never after intervening calls.
- [ ] `assert()` is never used to validate untrusted external input.
- [ ] No runtime/user-derived string is ever passed directly as a `printf`-family format argument.
- [ ] Structs/arrays are explicitly zero-initialized where any field might be read before being set.
- [ ] `gets()`/unbounded `strcpy`/`strcat`/`sprintf()` never appear, even in example code.
