# C++

## RAII (Resource Acquisition Is Initialization)

- **DO:** Bind every resource's lifetime (heap memory, file handles, sockets, locks, database connections) to an object's constructor/destructor pair, so the resource is automatically released when that object goes out of scope — this is the core discipline that makes C++ resource management exception-safe without manual cleanup code at every exit point.
  ```cpp
  // DON'T — manual acquire/release, leaks if an exception is thrown mid-function
  void writeLog(const char* path, const std::string& msg) {
      FILE* f = fopen(path, "a");
      fputs(msg.c_str(), f); // if this throws or the function returns early, f leaks
      fclose(f);
  }

  // DO — RAII wrapper releases automatically, even on exception or early return
  struct FileHandle {
      FILE* f;
      explicit FileHandle(const char* path, const char* mode) : f(fopen(path, mode)) {}
      ~FileHandle() { if (f) fclose(f); }
      FileHandle(const FileHandle&) = delete;
      FileHandle& operator=(const FileHandle&) = delete;
  };
  void writeLog(const char* path, const std::string& msg) {
      FileHandle f(path, "a");
      if (f.f) fputs(msg.c_str(), f.f);
  }
  ```
- **DON'T:** Write manual `acquire()`/`release()` pairs and rely on remembering to call `release()` on every exit path, including exception paths. Any function with more than one return statement (or any call that can throw) makes manual pairing fragile; an RAII type makes forgetting structurally impossible.
- **DO:** Use standard library RAII types (`std::lock_guard`, `std::unique_lock`, `std::fstream`, `std::unique_ptr`/`std::shared_ptr`, `std::vector`) instead of hand-rolling equivalents whenever the standard type already covers the need — they're well-tested, exception-safe, and instantly recognizable to any C++ reader.
- **DON'T:** Rely on a `try`/`catch`/`finally`-style pattern for cleanup — C++ has no `finally`. Model cleanup as a destructor instead; `std::scope_exit`-style wrappers (or a minimal hand-rolled equivalent) exist precisely for one-off cleanup that doesn't warrant a dedicated class.
- **DO:** Make RAII types non-copyable (or give them well-defined copy semantics, like deep-copying a `unique_ptr`-managed resource, if copying is genuinely meaningful) so accidental copies don't cause a double-release when both copies' destructors run.
- **DO:** Keep RAII object lifetimes as short and as scoped as possible — declare a `lock_guard` right where the critical section begins, not at the top of a long function, so the resource (a lock, in this case) is held for the minimum necessary duration.
- **DON'T:** Manage two independent resources with one RAII type unless they truly share a single lifetime. Prefer composing multiple small, single-responsibility RAII members over one class whose destructor has to carefully sequence several unrelated cleanups.

## Smart Pointers vs Raw Pointers

- **DO:** Default to `std::unique_ptr<T>` for exclusive ownership of a heap-allocated object — it has zero overhead compared to a raw pointer, automatically deletes its object on destruction, and its move-only semantics make single ownership explicit in the type system.
  ```cpp
  std::unique_ptr<Connection> conn = std::make_unique<Connection>(host, port);
  // conn is destroyed (and Connection's destructor runs) automatically at scope exit
  ```
- **DO:** Reach for `std::shared_ptr<T>` only when an object genuinely has multiple, independent owners whose lifetimes aren't statically nested — not as a default "safe" choice. `shared_ptr` carries real reference-counting overhead (atomic increments/decrements) and can create reference cycles that leak memory if not broken with `std::weak_ptr`.
- **DON'T:** Use raw `new`/`delete` in modern application code for ownership. If a raw pointer is `new`'d and someone has to remember to `delete` it later, that's exactly the bug class smart pointers exist to eliminate; use `make_unique`/`make_shared` (or a container) instead.
  ```cpp
  // DON'T — manual new/delete, leaks on any early return or exception
  Widget* w = new Widget();
  if (!w->init()) { delete w; return false; }
  process(w);
  delete w;

  // DO — ownership and cleanup handled automatically
  auto w = std::make_unique<Widget>();
  if (!w->init()) return false;
  process(w.get());
  ```
- **DO:** Use raw pointers (or references) for *non-owning* access — a function that merely observes or uses an object without participating in its lifetime should take a raw pointer or reference parameter, not a smart pointer. Passing `std::shared_ptr<T>` into every function that just needs to read `T` needlessly couples that function to a specific ownership model and adds refcounting overhead for no benefit.
- **DON'T:** Pass `shared_ptr` by value into functions that don't need to extend or share ownership. Passing by value bumps the reference count on every call (an atomic operation) for no purpose if the callee is only borrowing; pass `const T&`/`T*`/`const shared_ptr<T>&` instead, matching the actual ownership need.
- **DO:** Use `std::weak_ptr` to break reference cycles between `shared_ptr`s that reference each other (a classic parent/child or observer relationship) — a cycle of `shared_ptr`s never reaches a zero refcount and leaks for the life of the program.
- **DON'T:** Call `.get()` on a smart pointer and store the resulting raw pointer somewhere that can outlive the smart pointer itself. The raw pointer carries no ownership information, so nothing stops it from dangling once the smart pointer's object is destroyed.
- **DO:** Use `make_unique`/`make_shared` instead of a bare `new` expression passed into the smart pointer's constructor — beyond being shorter, `make_shared` allocates the control block and object together in one allocation (a real performance win), and both forms avoid a subtle exception-safety gap that can occur when a `new` expression is one of several arguments in the same function call.

## Modern C++ Idioms vs Legacy Patterns

- **DO:** Prefer `auto` for local variable declarations where the type is obvious from the right-hand side or is a long, unwieldy iterator/template type, improving readability without sacrificing static typing — the type is still fixed at compile time, just inferred rather than spelled out.
  ```cpp
  // Verbose legacy style
  std::map<std::string, std::vector<int>>::iterator it = m.find(key);

  // Modern, equally type-safe
  auto it = m.find(key);
  ```
- **DON'T:** Use `auto` where it obscures a meaningful type distinction the reader actually needs (e.g., `auto x = getValue();` when whether `x` is a reference, a pointer, or hides an expensive implicit conversion matters to correctness). Prefer an explicit type when it materially aids understanding or correctness.
- **DO:** Use range-based `for` loops over index-based loops when the index itself isn't needed, eliminating an entire class of off-by-one and iterator-invalidation bugs.
  ```cpp
  // DON'T — legacy index-based loop, unnecessary indexing risk
  for (size_t i = 0; i < items.size(); ++i) { process(items[i]); }

  // DO — range-based for
  for (const auto& item : items) { process(item); }
  ```
- **DO:** Use `nullptr` instead of `NULL` or `0` for pointer literals. `nullptr` has a real, distinct type (`std::nullptr_t`) that can't accidentally be treated as an integer in overload resolution, unlike the macro `NULL`.
- **DO:** Use scoped enums (`enum class`) instead of legacy unscoped `enum` — they don't leak their enumerator names into the surrounding scope, don't implicitly convert to `int`, and prevent comparing enumerators from unrelated enums.
  ```cpp
  // DON'T — legacy unscoped enum pollutes the namespace, implicitly converts to int
  enum Color { Red, Green, Blue };

  // DO — scoped, strongly typed
  enum class Color { Red, Green, Blue };
  Color c = Color::Red;
  ```
- **DO:** Use `constexpr` for values and simple functions that can be computed at compile time, over `#define` macros or plain `const` for numeric constants — `constexpr` gives real type-checking and scoping that macros lack, and can move work from runtime to compile time.
- **DON'T:** Use C-style casts (`(int)x`, `(Widget*)ptr`) in C++ code. They silently pick whichever of `static_cast`, `const_cast`, or `reinterpret_cast` "works," hiding what kind of conversion is actually happening; use the named cast that matches your intent so the compiler (and reader) can flag a mismatch.
  ```cpp
  // DON'T — ambiguous about intent, can silently strip const or reinterpret bits
  Derived* d = (Derived*)basePtr;

  // DO — explicit about what kind of conversion is intended
  Derived* d = dynamic_cast<Derived*>(basePtr); // checked, or static_cast if certain
  ```
- **DO:** Reach for structured bindings (`auto [key, value] = *it;`), `if`/`switch` with an init-statement, and `std::optional`/`std::variant`/`std::string_view` (C++17) or concepts and ranges (C++20) where they genuinely simplify code, rather than sticking to pre-C++11 idioms out of habit.
- **DON'T:** Use `std::endl` by default when `'\n'` suffices. `std::endl` forces a stream flush on every call, which is a real, measurable performance cost in output-heavy code; reserve it for the specific cases where an immediate flush is actually required.

## Const-Correctness

- **DO:** Mark every member function that doesn't modify observable object state as `const`, and every parameter/reference/pointer that a function doesn't modify as `const` — this documents intent, lets the compiler catch accidental mutation, and is required for calling those functions on `const` objects/references at all.
  ```cpp
  class Rectangle {
      double w, h;
  public:
      double area() const { return w * h; }        // doesn't modify state
      void resize(double nw, double nh) { w = nw; h = nh; } // modifies state, not const
  };
  ```
- **DON'T:** Use `mutable` to work around a `const` method that "needs" to modify a field, unless the field is genuinely incidental to the object's observable state (a cache, a lazily computed value, a mutex protecting logically-const access). Using `mutable` to bypass const-correctness for a field that *is* part of the object's real state defeats the purpose of marking the method `const` at all.
- **DO:** Pass non-trivially-copyable parameters (strings, vectors, other class types) by `const&` when the function only reads them, avoiding an unnecessary copy on every call while still preventing the callee from mutating the caller's argument.
- **DON'T:** Pass a large object by value "to keep it simple" when the function doesn't need its own independent copy. Passing by value forces a full copy (or a move, if the caller passes an rvalue) at the call site; `const&` communicates read-only intent with no copy at all in the common case.
- **DO:** Use `std::string_view` (C++17) for read-only string parameters that don't need to outlive the call, avoiding both a copy and an implicit `std::string` construction when the caller already has a `const char*` or substring — but never store a `string_view` past the lifetime of the data it views.
- **DO:** Return `const` references or values from accessors that expose internal state read-only, and prefer returning by value for small types where a copy is cheap and lifetime concerns (dangling references) outweigh the copy's cost.
- **DON'T:** Const-cast away constness (`const_cast`) to call a non-const function on a `const` object. If a `const` method needs to be called, the object shouldn't have been made `const` in that context in the first place — `const_cast` here is a sign the design's constness is wrong, not a legitimate workaround; the rare legitimate use is interfacing with a poorly-const-annotated legacy API.

## Move Semantics & Rule of 0/3/5

- **DO:** Follow the Rule of Zero by default — design classes so they own no raw resources directly (composing them instead from `unique_ptr`, `vector`, `string`, and other RAII members) and let the compiler-generated copy/move constructors, assignment operators, and destructor do the right thing automatically, with no manual special member functions at all.
  ```cpp
  // Rule of Zero: no manual destructor/copy/move needed —
  // each member already manages its own resource correctly.
  class Session {
      std::unique_ptr<Connection> conn;
      std::vector<std::string> log;
      std::string name;
      // no ~Session(), no copy/move ctor/assignment written by hand
  };
  ```
- **DO:** Follow the Rule of Five whenever a class *does* directly manage a raw resource (a raw pointer, a file descriptor, a handle) — define the destructor, copy constructor, copy assignment, move constructor, and move assignment together, since the compiler-generated defaults for a class holding a raw resource are almost always wrong (typically causing a double-free or shallow copy).
- **DON'T:** Define only a destructor for a class that owns a raw resource and leave the copy constructor/assignment operator as compiler-generated defaults. The default copy constructor does a shallow, member-wise copy — for a raw pointer, both the original and the copy now think they own the same resource, and both destructors will eventually free it, producing a double free.
  ```cpp
  // DON'T — Rule of Five violated: only the destructor is custom
  class Buffer {
      int* data;
  public:
      Buffer(size_t n) : data(new int[n]) {}
      ~Buffer() { delete[] data; }
      // no copy ctor/assignment defined — compiler generates a shallow copy!
  };
  // Buffer a(10); Buffer b = a; // both a.data and b.data point to the same array
  // when a and b are both destroyed, delete[] runs twice on the same pointer
  ```
- **DO:** Mark move constructors and move assignment operators `noexcept` whenever they genuinely cannot throw (which is nearly always true for a move that just transfers ownership of a pointer/handle). Standard containers like `std::vector` check for a `noexcept` move constructor to decide whether to move or copy elements during reallocation — a throwing move silently downgrades to copies everywhere the container is used.
- **DON'T:** Forget to leave a moved-from object in a valid, destructible state. A move constructor/assignment should null out or reset the source object's resource handle so its destructor doesn't also try to release the resource that was just transferred to the new owner.
  ```cpp
  Buffer(Buffer&& other) noexcept : data(other.data) {
      other.data = nullptr; // required — otherwise other's destructor double-frees
  }
  ```
- **DO:** Use `std::move` to explicitly signal that a value's resources should be transferred rather than copied, but only on an object you're finished using — `std::move` doesn't itself move anything, it just casts to an rvalue reference, enabling the move-constructor/assignment overload to be selected.
- **DON'T:** Use an object after `std::move`-ing it, other than to assign it a new value or destroy it. A moved-from standard-library object is left in a valid but unspecified state — reading its old value afterward is a common, subtle bug, not a hard error the compiler will catch for you.
- **DO:** Accept parameters by value plus `std::move` internally (the "sink" parameter idiom) when a function unconditionally takes ownership of an argument — this lets the compiler elide the copy for rvalue arguments while still working correctly for lvalue arguments (which get copied once, at the call site, same as a `const&` + internal copy would cost).
  ```cpp
  class Logger {
      std::string prefix;
  public:
      explicit Logger(std::string p) : prefix(std::move(p)) {} // one copy for lvalues, zero for rvalues
  };
  ```

## Avoiding Undefined Behavior

- **DO:** Treat every sanitizer/warning finding (AddressSanitizer, UndefinedBehaviorSanitizer, `-Wall -Wextra`) as a genuine bug to fix, for the same reason as in C — undefined behavior in C++ gives the optimizer license to produce surprising results, and "it works in this build" is not evidence of correctness.
- **DON'T:** Read from or write to a `std::vector`/`std::array`/C array with `operator[]` past its bounds. Unlike `.at()`, `operator[]` performs no bounds checking by default — use `.at()` when the index isn't already provably in range, and reserve `operator[]` for hot paths where the bound has already been established.
  ```cpp
  // DON'T — operator[] does not bounds-check; UB if i is out of range
  int get(const std::vector<int>& v, size_t i) { return v[i]; }

  // DO — .at() throws std::out_of_range instead of invoking UB
  int get(const std::vector<int>& v, size_t i) { return v.at(i); }
  ```
- **DO:** Assume any iterator can be invalidated by a container mutation (insertion, erasure, reallocation) unless you've checked that specific container's invalidation guarantees, and never dereference an iterator after an operation that might have invalidated it.
  ```cpp
  // DON'T — erasing invalidates it; using it afterward is UB
  for (auto it = v.begin(); it != v.end(); ++it) {
      if (*it == target) v.erase(it); // it is now invalid, and ++it reads from it
  }

  // DO — erase returns a valid next iterator
  for (auto it = v.begin(); it != v.end(); ) {
      it = (*it == target) ? v.erase(it) : std::next(it);
  }
  ```
- **DON'T:** Access a `std::optional`, `std::variant`, or `std::any` without checking it holds a value/the expected alternative first, when using the unchecked accessors (`operator*` on `optional`, `std::get` without a prior `holds_alternative`/`try` check on `variant`). Prefer `.value()` (which throws) or an explicit check when correctness matters more than raw speed.
- **DO:** Initialize every member variable, either via a default member initializer or in every constructor's initializer list, rather than leaving primitive-typed members (`int`, pointers, `bool`) to their indeterminate default value. Unlike class-type members (which have their own default constructors run automatically), built-in typed members are uninitialized by default.
- **DON'T:** Call a virtual function from a constructor or destructor and expect it to dispatch to a derived class's override. During construction/destruction, the dynamic type is the class currently being constructed/destructed, not the most-derived type, so this doesn't do what it looks like it does — it's legal C++, but it's a well-known trap, not the polymorphic call the code appears to intend.
- **DO:** Use `std::span` (C++20) or an explicit `(pointer, length)` pair instead of a bare pointer when passing a contiguous range to a function that needs to know its extent, so the function can bounds-check instead of trusting an implicit, undocumented length.

## STL Usage

- **DO:** Reach for the standard containers (`std::vector`, `std::unordered_map`, `std::string`, `std::array`) as the default choice over a hand-rolled data structure or raw array — they're well-tested, have well-understood complexity guarantees, and are instantly recognizable to any C++ reader, which a bespoke reimplementation is not.
- **DO:** Default to `std::vector` for sequential storage unless a specific access pattern justifies something else (`std::deque` for efficient front/back insertion, `std::list` only when frequent mid-sequence insertion/deletion with stable iterators genuinely dominates, which is rarer in practice than it first appears due to `vector`'s cache locality advantages).
- **DON'T:** Reach for `std::list` by default "because insertion is O(1)." In practice, `std::vector`'s contiguous memory layout usually outperforms `std::list` even for workloads with insertions/deletions, due to cache locality — benchmark before choosing a linked structure over a contiguous one.
- **DO:** Use the `<algorithm>` header's standard algorithms (`std::sort`, `std::find`, `std::accumulate`, `std::transform`, `std::copy_if`, ranges algorithms in C++20) instead of hand-written loops for common operations — they express intent directly, are well-tested, and are often more optimized than an equivalent hand-written loop.
  ```cpp
  // DON'T — hand-rolled loop reimplementing a standard algorithm
  bool found = false;
  for (const auto& x : v) { if (x == target) { found = true; break; } }

  // DO — expresses intent directly
  bool found = std::find(v.begin(), v.end(), target) != v.end();
  ```
- **DON'T:** Call `.size()` on a container and compare it against a signed integer without considering the mismatch. `.size()` returns an unsigned type (`size_type`, typically `size_t`); comparing it against a negative signed value promotes the signed value to unsigned, silently producing an unexpected huge comparison result rather than the intended negative-is-less-than-zero check.
- **DO:** Reserve capacity with `.reserve(n)` on a `std::vector` (or similar) before a loop that will push many known-or-estimated-count elements, to avoid repeated reallocation and copying as the vector grows incrementally.
- **DON'T:** Use `std::map`/`std::set` (ordered, tree-based, `O(log n)`) when `std::unordered_map`/`std::unordered_set` (hash-based, average `O(1)`) would do and ordering isn't actually needed. Conversely, don't reach for `unordered_map` when deterministic iteration order or range queries are actually required — pick based on the real access pattern, not habit.
- **DO:** Prefer `emplace_back`/`emplace` over `push_back`/`insert` when constructing the element in place (passing constructor arguments directly) avoids a temporary object and a subsequent move/copy — but understand it constructs in place from the given arguments, so it isn't a drop-in replacement when you already have a constructed object to insert (use `push_back(std::move(obj))` for that case).

## Build Systems — CMake

- **DO:** Use `target_`-scoped commands (`target_include_directories`, `target_link_libraries`, `target_compile_definitions`, `target_compile_options`) with explicit `PUBLIC`/`PRIVATE`/`INTERFACE` visibility instead of the legacy global commands (`include_directories`, `link_libraries`) — modern, target-based CMake keeps each target's requirements scoped and correctly propagated to its consumers, rather than leaking into every target in the whole build.
  ```cmake
  # DON'T — legacy global commands affect every target in the directory
  include_directories(include)
  link_libraries(mylib)

  # DO — scoped to the specific target, with explicit propagation
  add_library(mylib src/mylib.cpp)
  target_include_directories(mylib PUBLIC include)

  add_executable(myapp src/main.cpp)
  target_link_libraries(myapp PRIVATE mylib)
  ```
- **DO:** Set the minimum required CMake version (`cmake_minimum_required(VERSION ...)`) and the project's required C++ standard explicitly via `target_compile_features` or `set(CMAKE_CXX_STANDARD 20)` plus `CMAKE_CXX_STANDARD_REQUIRED ON`, so the build is reproducible instead of depending on whatever default the local toolchain happens to pick.
- **DON'T:** Glob source files with `file(GLOB ...)` for a target's source list. CMake doesn't automatically detect new files matching the glob until you manually re-run configuration, so a newly added source file can silently fail to be compiled until someone remembers to reconfigure — list source files explicitly, or accept the tradeoff deliberately and document it.
- **DO:** Express dependencies through `find_package` (for system/installed libraries) or `FetchContent`/a package manager (Conan, vcpkg) for third-party libraries, rather than hardcoding absolute include/library paths that only work on the original author's machine.
- **DON'T:** Set compiler flags globally with `CMAKE_CXX_FLAGS` when a `target_compile_options` on the specific target would do. Global flags apply to every target in the project, including third-party dependencies pulled in via `add_subdirectory`/`FetchContent`, which may not tolerate your project's warning level or sanitizer flags.
- **DO:** Keep debug and release configurations properly separated (`CMAKE_BUILD_TYPE` or, for multi-config generators, the appropriate per-configuration flags) so debug builds retain assertions, debug symbols, and no optimization, while release builds get full optimization — conflating the two produces a build that's either too slow to iterate on or too stripped-down to debug.
- **DO:** Use CMake's `enable_testing()`/`add_test()` (with CTest) or an integrated test framework's CMake support (GoogleTest's `gtest_discover_tests`, Catch2's CMake integration) so tests run through the same build system and CI invocation as everything else, rather than a separate, hand-maintained test-running script.
- **DON'T:** Check generated build artifacts (the `build/` directory, `CMakeCache.txt`, compiled binaries) into version control. Keep the build directory out-of-tree and gitignored; only the `CMakeLists.txt` and source files should be tracked, since the build output is regenerable and machine-specific.

## Common AI-Assistant Mistakes in C++

- **DON'T:** Generate raw `new`/`delete` for ownership in code presented as modern or production-quality. This is one of the most common tells of outdated or careless generated C++ — reach for `std::make_unique`/`std::make_shared` (or a container) by default, and only use raw `new` when implementing a low-level allocator or smart pointer itself.
- **DON'T:** Define a custom destructor for a class managing a raw resource without also defining (or explicitly `= delete`-ing) the copy constructor and copy assignment operator. Generated code that adds "just a destructor" to free a resource, without addressing the Rule of Five, produces a class that double-frees the moment it's copied — a bug that often doesn't surface until later, unrelated code happens to copy an instance.
- **DON'T:** Return a reference or pointer to a local variable, or to a temporary, from a function. This is directly analogous to the C mistake of returning a pointer to a stack variable, and it's just as easy to generate accidentally in C++ when refactoring a function to "avoid a copy" by returning `const T&` from something that constructs `T` locally.
  ```cpp
  // DON'T — returns a reference to a destroyed temporary/local
  const std::string& firstWord(const std::string& s) {
      std::string result = s.substr(0, s.find(' '));
      return result; // result is destroyed when the function returns
  }
  ```
- **DON'T:** Pass large objects (strings, vectors, other class types) by value when a `const&` would avoid an unnecessary copy, especially in generated code that defaults to pass-by-value "for simplicity" without considering the performance and correctness implications for large or non-trivially-copyable types.
- **DON'T:** Use `std::endl`, `using namespace std;` at file/header scope, or other patterns widely taught as bad practice but common in older tutorials that appear frequently in training data. `using namespace std;` in a header pollutes every translation unit that includes it, risking name collisions in code the header's author never anticipated.
- **DON'T:** Fabricate a standard library or well-known third-party API (an STL function that doesn't exist, an incorrect signature for a real one, a Boost/Qt method that sounds plausible but isn't real) without verifying it against actual documentation or a compiler. Confidently generating `std::vector::find()` (which doesn't exist — that's `std::find(v.begin(), v.end(), x)`) is a recognizable and common failure mode.
- **DON'T:** Claim generated code "has no undefined behavior," "is exception-safe," or "has no memory leaks" without actually tracing ownership, bounds, and exception paths through the code. State these properties only after real verification (compiling, running sanitizers, working through the Rule of Five for each resource-owning type), not as a default claim attached to any code that compiles.
- **DON'T:** Mix owning and non-owning pointer conventions inconsistently within the same generated codebase — some functions taking `unique_ptr<T>` by value to signal ownership transfer, others taking a raw `T*` for the exact same kind of ownership transfer elsewhere. Pick one convention (owning transfer via smart pointer, non-owning access via raw pointer/reference) and apply it consistently so a reader can tell ownership semantics from the signature alone.

## Quick Checklist
- [ ] Every resource's lifetime is bound to an object via RAII — no manual acquire/release pairing.
- [ ] Standard RAII types (`lock_guard`, `fstream`, smart pointers) are used before hand-rolling one.
- [ ] `std::unique_ptr` is the default for owned heap allocations; `shared_ptr` is used only for genuine shared ownership.
- [ ] No raw `new`/`delete` appears for ownership in application code.
- [ ] Non-owning access uses raw pointers/references, not smart pointers passed unnecessarily.
- [ ] `weak_ptr` breaks any `shared_ptr` reference cycle.
- [ ] `auto`, range-based `for`, `nullptr`, and `enum class` are used over their legacy equivalents where they aid clarity.
- [ ] No C-style casts; every cast is a named cast (`static_cast`, `dynamic_cast`, etc.) matching real intent.
- [ ] Every non-mutating member function and every read-only parameter is marked `const`.
- [ ] `mutable` is reserved for genuinely incidental state (caches, mutexes), not to bypass real const-correctness.
- [ ] Large/non-trivial parameters are passed by `const&` unless the function needs its own copy.
- [ ] Classes follow the Rule of Zero by default, or the full Rule of Five when they own a raw resource.
- [ ] Move constructors/assignment are `noexcept` where truly non-throwing, and leave the source valid but empty.
- [ ] No use of a moved-from object except to reassign or destroy it.
- [ ] Container/array access is bounds-checked (`.at()`) unless the index is already provably in range.
- [ ] Iterators are never used after an operation that may have invalidated them.
- [ ] Every member variable is initialized, either via default member initializer or every constructor.
- [ ] No virtual call from a constructor/destructor expecting derived-class dispatch.
- [ ] STL containers and `<algorithm>` are used before a hand-rolled equivalent.
- [ ] Container choice (`vector` vs `list` vs `map` vs `unordered_map`) matches the real access pattern, not habit.
- [ ] `.reserve()` is used before loops appending many known-or-estimated elements.
- [ ] CMake uses `target_`-scoped commands with explicit `PUBLIC`/`PRIVATE`/`INTERFACE`, not legacy global commands.
- [ ] C++ standard and minimum CMake version are set explicitly, not left to toolchain defaults.
- [ ] Debug and release configurations are kept properly separated.
- [ ] Build artifacts are gitignored, never committed.
- [ ] No fabricated STL/library API calls — every call is verified against real documentation or a compile.
- [ ] Every "memory-safe"/"exception-safe"/"no leaks" claim is backed by actual tracing or sanitizer runs, not asserted by default.
