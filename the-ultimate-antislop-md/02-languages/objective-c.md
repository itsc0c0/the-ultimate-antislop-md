# Objective-C

## Naming Conventions

- **DO:** Prefix custom classes, protocols, functions, and global constants with a two-or-three-letter namespace prefix (`ACMEUserManager`, `ACMENetworkError`) tied to the project or organization. Objective-C has no module namespacing at the language level, so the prefix is the only thing preventing a name collision with another library or with Apple's own frameworks.
- **DO:** Name C functions declared at global scope (rather than as Objective-C methods) with the same project prefix as classes (`ACMEClampValue(...)`), since free functions are just as capable of colliding with another library's global symbol as a class name is.
- **DO:** Name a getter for a non-Boolean property to match the property name exactly with no `get` prefix (`- (NSString *)name;` for a `name` property, never `- (NSString *)getName;`), matching Cocoa's own convention — a `get`-prefixed method name in Objective-C specifically implies the older "returns via out-parameter" pattern (`- (void)getName:(NSString **)name)`), not a simple accessor.
- **DON'T:** Reuse Apple's reserved two-letter prefixes (`NS`, `UI`, `CA`, `CG`, and similar) for your own types. Apple explicitly reserves these, and a class like `NSHelper` risks silently colliding with a symbol Apple introduces in a future SDK.
- **DO:** Name methods so each parameter is labeled at the call site, the same descriptive way Swift's API guidelines describe — Objective-C pioneered this convention, and a method like `insertObject:atIndex:` reads correctly at its call site without needing to check the declaration.
  ```objectivec
  // GOOD — every argument is labeled, unambiguous at the call site
  - (void)insertObject:(id)object atIndex:(NSUInteger)index;
  [array insertObject:newItem atIndex:3];
  ```
- **DO:** Prefix private method names with nothing special (Objective-C has no true private methods) but keep them out of the public header, and consider a `p_` or `_private` convention only if the team already has one — don't invent a new private-method naming scheme mid-codebase.
- **DO:** Name Boolean accessor methods with an `is` prefix for the getter (`isEqual:`, `isFinished`) and follow standard `set`-prefixed setters for corresponding properties (`setFinished:`), matching Foundation's own conventions so KVC/KVO and property synthesis behave as expected.
- **DON'T:** Use ambiguous single-letter or heavily abbreviated variable and property names (`NSString *s`, `NSArray *arr2`) outside of the tightest loop-counter scope (`for (NSInteger i = 0; ...)`). Objective-C's verbosity is part of its idiom; fighting it with cryptic abbreviations makes code harder to read, not easier.
- **DO:** Name delegate protocol methods with the delegating class's name as a prefix and the triggering event as a suffix (`tableView:didSelectRowAtIndexPath:`, `connectionDidFinishLoading:`), matching Cocoa's own delegate method naming so the pattern stays recognizable to anyone familiar with the platform.
- **DO:** Name factory/convenience class methods starting with the type's own name in lowercase (`+ (instancetype)buttonWithType:`), not with a generic word like `create` or `make`, to match Foundation/UIKit's convention for class-side convenience constructors.
- **DO:** Use `instancetype` as the return type for class factory methods and `init` methods instead of the class's own explicit type name, so subclasses inherit a factory method that correctly returns the subclass type rather than being locked to the base class's type.
  ```objectivec
  // BAD — a subclass calling +user would get back a plain User,
  // not a Subclass, even when called on the subclass
  + (User *)user;

  // GOOD — instancetype resolves to whatever class it's actually
  // called on, working correctly for subclasses too
  + (instancetype)user;
  ```
- **DO:** Name constants with a scope-appropriate prefix and a `k`-free modern style (`static NSString * const ACMEAPIBaseURLKey = @"apiBaseURL";`) rather than the older Hungarian-notation `kConstantName` convention that predates `static const` support in Objective-C — the older style still appears in legacy code but is not the current recommended convention.
- **DO:** Name category methods with a project-specific prefix on the method name itself (`- (NSString *)acme_trimmedString;`) when extending a Foundation/UIKit class via a category, to reduce the chance of silently colliding with a method some other library's category adds to the same class.
  ```objectivec
  // A category on a class you don't own should prefix its new methods
  @interface NSString (ACMEAdditions)
  - (NSString *)acme_trimmedString;
  @end
  ```
- **DO:** Use modern Objective-C literal syntax for collections and boxed values (`@[a, b]`, `@{key: value}`, `@42`, `@YES`) instead of the older, verbose class-method equivalents (`[NSArray arrayWithObjects:a, b, nil]`, `[NSNumber numberWithInt:42]`) — the literal syntax is shorter, and unlike the `...WithObjects:` variadic form, it doesn't have the historical footgun of silently truncating a collection if a `nil` sneaks into the middle of the argument list.
  ```objectivec
  // BAD — verbose, and a stray nil mid-list silently truncates the array
  NSArray *names = [NSArray arrayWithObjects:@"Alice", @"Bob", nil];

  // GOOD — concise, and a nil literal is a compile error, not a silent bug
  NSArray *names = @[@"Alice", @"Bob"];
  ```
- **DO:** Use subscripting syntax (`array[0]`, `dict[key] = value`) for `NSArray`/`NSDictionary`/`NSMutableArray`/`NSMutableDictionary` access instead of the older `objectAtIndex:`/`objectForKey:`/`setObject:forKey:` method calls, since it reads closer to how the same operation looks in most other modern languages and is exactly equivalent under the hood.
- **DON'T:** Mix modern literal/subscripting syntax and legacy verbose method-call syntax inconsistently within the same file — pick the modern syntax as the default for new and touched code, reserving the verbose form only for cases the literal syntax genuinely can't express (like `NSMutableArray` methods with no literal equivalent, e.g. `insertObject:atIndex:`).
- **DO:** Declare block-typed properties and parameters with a `typedef` when the same block signature is used in more than one place (`typedef void (^CompletionBlock)(NSData * _Nullable, NSError * _Nullable);`), so the signature is defined once and reused, rather than repeated verbatim at every declaration site.
  ```objectivec
  typedef void (^ACMECompletionBlock)(NSData * _Nullable data, NSError * _Nullable error);

  @interface NetworkClient : NSObject
  - (void)fetchWithCompletion:(ACMECompletionBlock)completion;
  @end
  ```

## Memory Management Under ARC

- **DO:** Rely on Automatic Reference Counting for ordinary `strong`/`weak` object properties and locals — ARC inserts the retain/release calls at compile time, so manual `retain`/`release`/`autorelease` calls should not appear anywhere in ARC-enabled code.
- **DON'T:** Write manual `retain`, `release`, or `autorelease` calls, or override `dealloc` to call `[super dealloc]`, in a file compiled under ARC. These are pre-ARC (manual retain-count, "MRC") idioms; under ARC they are either compiler errors or redundant no-ops depending on context, and mixing the two mental models produces confused, incorrect code.
- **DO:** Declare delegate and other back-reference properties `weak` (or `assign` only when targeting a pre-weak-reference runtime, which is effectively never in current code) to avoid retain cycles between a delegating object and its delegate.
  ```objectivec
  // GOOD — delegate does not keep its owner alive
  @property (nonatomic, weak) id<MyViewDelegate> delegate;
  ```
- **DO:** Capture `self` weakly in blocks stored as a property or handed to an API that retains the block beyond the current scope, using the standard `__weak typeof(self) weakSelf = self;` dance, then re-strengthen inside the block body to avoid `self` disappearing mid-execution.
  ```objectivec
  __weak typeof(self) weakSelf = self;
  self.completionHandler = ^{
      __strong typeof(self) strongSelf = weakSelf;
      if (!strongSelf) { return; }
      [strongSelf finishUp];
  };
  ```
- **DON'T:** Capture `self` strongly by default inside a block assigned to a property of `self` (or passed to `NSNotificationCenter`, `dispatch_async` on a queue `self` outlives, etc.) — this is the direct Objective-C analog of a Swift closure retain cycle, and it is just as common a leak source.
- **DO:** Use `__unsafe_unretained` only on the rare pre-`weak`-runtime target or in genuinely performance-critical inner loops where the referenced object's lifetime is provably guaranteed — unlike `__weak`, it does not zero out automatically on deallocation and leaves a dangling pointer if the assumption is wrong.
- **DO:** Break retain cycles between parent/child object graphs by making the child's back-reference to its parent `weak`, exactly as with delegates — a common leak is a custom "child" model object holding a `strong` reference back to the "parent" collection that owns it.
- **DON'T:** Assume ARC eliminates all memory management concerns — retain cycles through strong reference cycles (not just blocks: two objects each holding a `strong` property pointing at the other) are just as possible under ARC as manual reference counting; ARC only automates the retain/release calls, it doesn't detect cycles.
- **DO:** Use Instruments (Leaks and Allocations templates) or Xcode's Memory Graph Debugger to verify suspected retain cycles in Objective-C code exactly as you would in Swift — the tooling is shared across both languages.
- **DO:** Declare `@property` attributes explicitly and correctly for the intended semantics — `strong` for ordinary owned references, `weak` for non-owning back-references, `copy` for `NSString`/`NSArray`/block properties that should snapshot their value rather than alias a mutable caller-owned instance, and `assign` only for primitive scalar types.
  ```objectivec
  // GOOD — copy protects against the caller mutating an NSMutableString
  // out from under you after assignment; strong would only retain
  // the reference, not snapshot its current contents.
  @property (nonatomic, copy) NSString *name;
  ```
- **DON'T:** Declare an `NSString`, `NSArray`, `NSDictionary`, or block property as `strong` when `copy` is what's actually intended. If a caller passes a mutable subclass instance (`NSMutableString`) and later mutates it, a `strong` property silently reflects that external mutation since it only retains a reference to the same object — `copy` avoids this entire class of bug for value-like Foundation types.
- **DO:** Use `@synthesize` only in the rare cases the compiler doesn't already auto-synthesize backing ivars and accessors (implementing a custom getter/setter for one property while needing default synthesis for others, or targeting a protocol-declared property) — in ordinary modern Objective-C, explicit `@synthesize` for every property is unnecessary boilerplate the compiler already handles.
- **DO:** Release resources like `dealloc`-time cleanup (removing observers, invalidating timers) even under ARC, since ARC only automates memory (retain/release) management, not other resource cleanup a class's lifecycle depends on.
  ```objectivec
  - (void)dealloc {
      [[NSNotificationCenter defaultCenter] removeObserver:self];
  }
  ```
- **DO:** Break a retain cycle between a block and the object that owns it using the same weak/strong dance for `self` even when the block is a short-lived animation or completion block passed as a method argument (not stored on a property) — if that specific API is documented to retain the block for any meaningful duration (many system animation and networking APIs do), the same capture discipline applies regardless of whether the block is literally assigned to a property.
- **DO:** Use `@weakify`/`@strongify`-style macros (from libextobjc, or a hand-rolled equivalent) consistently across a codebase that leans heavily on multi-level block nesting, so the same weak-self/strong-self boilerplate isn't hand-typed slightly differently at every call site.
  ```objectivec
  __weak typeof(self) weakSelf = self;
  [self.service fetchWithCompletion:^(NSData *data) {
      __strong typeof(self) strongSelf = weakSelf;
      if (!strongSelf) return;
      [strongSelf.service processMore:data completion:^{
          __strong typeof(self) innerSelf = weakSelf;  // a fresh weak->strong each level
          [innerSelf finish];
      }];
  }];
  ```
- **DON'T:** Assume a `weak` property is automatically thread-safe to read just because it "zeroes out" safely on deallocation — reading and using a `weak` property across multiple statements (`if (self.delegate) { [self.delegate doThing]; }`) has a race window where the object could deallocate between the check and the use on another thread; capture it into a strong local first (`id strongDelegate = self.delegate; if (strongDelegate) { [strongDelegate doThing]; }`) when thread-safety actually matters.
- **DO:** Use `@autoreleasepool` blocks explicitly around large batch-processing loops on platforms/contexts without an enclosing run-loop iteration to periodically drain autoreleased objects (command-line tools, background processing threads), since without a run loop cycling, autoreleased objects only get released when the pool is explicitly drained.
- **DO:** Always call `removeObserver:forKeyPath:` (or the block-based KVO API's corresponding teardown) for every `addObserver:forKeyPath:options:context:` registration, matching each registration one-to-one — an unbalanced KVO registration crashes the app when the observed object deallocates while an observer is still registered on it, which is a distinctly different (and often more confusing) failure mode than an ordinary retain cycle.
- **DO:** Use a private, unique `static void *` context pointer with each KVO registration and check it in the observer callback (`if (context == &MyContext)`) rather than switching purely on the key path string, so KVO notifications from a superclass's independent KVO registration on the same key path don't get misattributed to the wrong observer.
- **DON'T:** Use `objc_setAssociatedObject`/`objc_getAssociatedObject` (associated objects) as a general-purpose way to add stored properties to a class via a category, when the class can simply be subclassed or wrapped instead — associated objects add a layer of indirection that's easy to leak (an association with `OBJC_ASSOCIATION_RETAIN` keeps the associated object alive for the host object's lifetime) and harder to discover than an ordinary declared property.
- **DO:** Mark protocol methods that aren't required for every conformer as `@optional`, and check `respondsToSelector:` before calling an optional protocol method on a delegate, since calling an unimplemented `@optional` method directly raises `-[NSObject doesNotRecognizeSelector:]`.
  ```objectivec
  if ([self.delegate respondsToSelector:@selector(didUpdateProgress:)]) {
      [self.delegate didUpdateProgress:progress];
  }
  ```

## Header/Implementation Discipline

- **DO:** Expose only the minimal public API in the `.h` header — properties and methods other classes actually need to call — and keep everything else (implementation-detail methods, mutable versions of readonly properties, internal state) in a class extension (`@interface MyClass ()`) at the top of the `.m` file.
  ```objectivec
  // MyModel.h — public, read-only surface
  @interface MyModel : NSObject
  @property (nonatomic, copy, readonly) NSString *name;
  - (void)refresh;
  @end

  // MyModel.m — private extension, real (mutable) storage
  @interface MyModel ()
  @property (nonatomic, copy, readwrite) NSString *name;
  @end
  ```
- **DON'T:** Declare mutable, writable properties in a public header when only the class itself should be able to mutate them. Expose a `readonly` property publicly and redeclare it `readwrite` in a private class extension, so external code can read but not blindly overwrite internal state.
- **DO:** Keep `#import` statements in the header limited to what's needed for the header's own declarations (forward-declare classes with `@class` where only a pointer type is needed), and put implementation-only imports in the `.m` file. This keeps compile-time header dependencies minimal and avoids import cycles.
- **DO:** Group related methods under `#pragma mark -` sections (`#pragma mark - Lifecycle`, `#pragma mark - UITableViewDataSource`) in the implementation file so Xcode's jump bar gives a navigable outline of a large `.m` file.
- **DON'T:** Let a single `.m` file grow to contain unrelated protocol conformances and helper logic with no `#pragma mark` structure — split large classes into categories in separate files (`MyClass+Networking.m`) when a class's responsibilities genuinely span multiple concerns, rather than one sprawling implementation file.
- **DO:** Declare `NS_DESIGNATED_INITIALIZER` on the one initializer that fully sets up an object's state, and have every convenience initializer call through to it — this documents (and lets the compiler partially enforce) the correct initialization chain for subclasses.
- **DO:** Use `NS_UNAVAILABLE` on `init`/`new` when a class must be created only through a specific factory method, so misuse is a compile-time error instead of a runtime surprise.
  ```objectivec
  @interface Currency : NSObject
  + (instancetype)currencyWithCode:(NSString *)code amount:(NSDecimalNumber *)amount;
  - (instancetype)init NS_UNAVAILABLE;
  @end
  ```
- **DO:** Split a large `.m` file's protocol conformances into separate category files (`MyViewController+TableViewDataSource.m`) when a class implements several substantial delegate/data-source protocols, so each file stays focused on one responsibility and is independently reviewable.
- **DON'T:** Redeclare a public property with a different type or attribute in a private class extension in a way that's inconsistent with the public declaration (other than the readonly→readwrite widening pattern) — this can produce subtle compiler behavior differences and confuses anyone trying to understand the property's actual contract from the header alone.
- **DO:** Keep a consistent, predictable order for members within a class's public interface (properties first, then class methods, then instance methods; or grouped by feature) so a reader scanning any header in the project can find what they're looking for using the same mental model each time.
- **DO:** Declare a protocol in its own header when it's implemented or consumed by more than one class, so both sides can `#import` just the protocol definition without needing to import a full class header that may pull in unrelated dependencies.
  ```objectivec
  // ACMEDataSourceProtocol.h — a standalone protocol header
  @protocol ACMEDataSource <NSObject>
  - (NSInteger)numberOfItems;
  - (id)itemAtIndex:(NSInteger)index;
  @end
  ```
- **DON'T:** Put implementation-only helper functions or static variables at file scope in a header — `static` file-scope declarations belong in the `.m` file; a header is meant to declare the class's contract, not carry its private implementation details along with it into every file that imports it.
- **DO:** Use `NS_TYPED_ENUM`/`NS_TYPED_EXTENSIBLE_ENUM` on a group of related `NSString *` constants (common for notification names, dictionary keys, or configuration option names) so they import into Swift as a proper namespaced type instead of a set of loose global string constants.
- **DO:** Document non-obvious method contracts directly in the header with a comment — what a `nil` parameter means if it's allowed, what units a numeric parameter is measured in, whether a method is safe to call from a background thread — since the header is the interface other developers (and other classes' authors) actually read before calling into a class.

## Nullability Annotations

- **DO:** Wrap the entire content of a header (or at least every method/property whose nullability isn't the file-wide default) in `NS_ASSUME_NONNULL_BEGIN` / `NS_ASSUME_NONNULL_END`, then mark only the exceptions explicitly `nullable`. This makes "non-null by default, opt in to nullable" the norm, matching how Swift's imported-optional bridging actually reads these headers.
  ```objectivec
  NS_ASSUME_NONNULL_BEGIN

  @interface UserService : NSObject
  - (nullable NSUser *)cachedUserWithID:(NSString *)userID;
  - (void)fetchUser:(NSString *)userID
         completion:(void (^)(NSUser * _Nullable user, NSError * _Nullable error))completion;
  @end

  NS_ASSUME_NONNULL_END
  ```
- **DON'T:** Leave a public Objective-C header without any nullability annotations in a codebase that's bridged into Swift. Every unannotated pointer type imports into Swift as an implicitly unwrapped optional (`Type!`), which silently reintroduces force-unwrap-style crash risk on the Swift side for values that may genuinely be `nil`.
- **DO:** Annotate block/closure parameter and return types with their own nullability (`void (^ _Nullable completion)(NSData * _Nullable, NSError * _Nullable)`) since a block type's nullability doesn't automatically follow the enclosing method's — each pointer inside the block signature needs its own annotation.
- **DO:** Match nullability annotations to actual runtime behavior, not aspiration — if a completion handler's `error` parameter is genuinely always non-nil on failure and always nil on success, annotate it that way; a `nullable` annotation on a value that's actually always present pushes needless unwrap-and-check code onto every Swift caller.
- **DO:** Use `NS_SWIFT_NAME` to give a clearer or more Swift-idiomatic name to a method/type when its default Swift-imported name (mechanically derived from the Objective-C selector) reads awkwardly, rather than leaving Swift callers with a clunky auto-translated signature.
  ```objectivec
  - (void)fetchUserWithID:(NSString *)userID
                completion:(void (^)(User * _Nullable, NSError * _Nullable))completion
      NS_SWIFT_NAME(fetchUser(id:completion:));
  ```
- **DO:** Annotate an Objective-C method whose completion-handler shape follows the `(T?, Error?) -> Void` convention with `NS_SWIFT_ASYNC_NAME`/`NS_SWIFT_ASYNC`, when targeting a Swift version that can import it as a native `async` function automatically, so Swift call sites get `try await fetchUser(id:)` instead of a manually-written completion-handler wrapper.
- **DO:** Verify generated nullability and Swift-name annotations against the actual Swift interface Xcode generates (via "Jump to Definition" on the imported type from Swift, or the generated interface preview) rather than assuming the annotations produced the intended result — subtle mismatches between the annotation and the intended Swift API surface are easy to introduce and easy to miss without checking the generated output directly.
- **DO:** Annotate a method that only ever returns non-nil despite what the compiler might otherwise infer with `NS_ASSUME_NONNULL`'s default treatment, but reserve `_Nonnull` overrides for exceptions inside an otherwise-nullable region — don't scatter explicit `_Nonnull`/`_Nullable` on every single parameter once `NS_ASSUME_NONNULL_BEGIN` already establishes the sane default, since that just adds visual noise.
- **DON'T:** Annotate a method `nonnull` and then actually return `nil` from some code path under an edge-case condition (an error state, an unfound cache entry) — this violates the nullability contract Swift relies on, and the resulting force-unwrap crash on the Swift side will not point back to the Objective-C method that lied about its contract.

## Bridging with Swift

- **DO:** Expose only Objective-C APIs that are meant for Swift consumption through the module's umbrella/bridging header, and keep implementation-detail Objective-C classes out of it, so the generated Swift interface doesn't leak internal types Swift code shouldn't touch.
- **DO:** Use lightweight generics (`NSArray<NSUser *> *`) on Objective-C collection properties and method signatures so they import into Swift as properly typed `[User]` rather than `[Any]`, saving every Swift caller a manual cast.
- **DON'T:** Design new shared model/business-logic types in Objective-C when the project is primarily Swift going forward — write new code in Swift and only touch Objective-C when extending existing Objective-C-owned code or a component that must stay Objective-C for a specific interop reason (e.g., subclassing a legacy base class).
- **DO:** Annotate Objective-C `NS_ENUM`/`NS_OPTIONS` declarations properly (with `NS_ENUM(NSInteger, MyEnum)`, not a bare `enum`) so they import into Swift as a real `enum` with case-style members, instead of a set of loose global integer constants.
- **DO:** Mark Objective-C classes not intended for Swift subclassing with `NS_SWIFT_UNAVAILABLE` or keep them out of Swift's visibility entirely when subclassing them from Swift would break invariants the Objective-C side depends on (e.g., a class that assumes `init` is always called directly and never overridden).
- **DO:** When a Swift type must be visible to Objective-C (for KVO, target-action, or an Objective-C-based framework callback), mark it `@objc` and, if needed, `@objcMembers` at the class level — but do this deliberately for the specific types that need it, not blanket across the whole Swift codebase, since `@objc` exposure has real ABI and dynamic-dispatch implications.
  ```swift
  // Only expose what Objective-C actually needs to call
  @objc class LegacyBridgeAdapter: NSObject {
      @objc func handleNotification(_ note: Notification) { ... }
  }
  ```
- **DO:** Remember that Swift enums with associated values, Swift-only generics, and Swift structs/protocols with `Self` requirements do not bridge to Objective-C at all — design the Objective-C-facing surface of a mixed-language module around plain classes, `@objc`-compatible enums (`@objc enum Status: Int`), and simple method signatures, keeping richer Swift-only types on the Swift-internal side of the boundary.
- **DO:** Use a dedicated bridging/adapter type at the language boundary (a thin Swift class wrapping a richer Swift model, exposing only `@objc`-compatible members) rather than trying to force a complex Swift type's full API surface to be Objective-C-compatible, which usually means stripping away the very features (generics, associated values, protocol extensions) that made it worth writing in Swift in the first place.
- **DO:** Use a bridging header (`ProjectName-Bridging-Header.h`) to expose specific Objective-C headers to Swift in a mixed-language target, importing only the headers Swift code actually needs — importing the whole framework's umbrella header indiscriminately pulls unrelated internal types into every Swift file's global namespace.
- **DO:** Test that a value round-trips correctly across the bridge for edge cases specific to the type-system mismatch — `NSInteger`'s platform-dependent width versus Swift's `Int`, an Objective-C `nil`-tolerant collection method meeting Swift's non-optional array elements, an `NSNumber` boxing a value in a way Swift's `as?` cast to a concrete numeric type might unexpectedly fail on.
- **DON'T:** Assume a Swift `struct` passed into Objective-C-adjacent code (via an intermediate Swift wrapper) retains value semantics once it crosses into the Objective-C side — Objective-C has no concept of value types, so any bridging layer necessarily boxes the value in an object, and further mutation on the Objective-C side won't be reflected back unless the bridging code explicitly re-synchronizes it.
- **DO:** Keep the Objective-C-facing subset of a mixed codebase's API surface deliberately small and stable, since every `@objc`-exposed Swift API becomes part of what the slower-moving Objective-C side depends on — a large, frequently-changing bridged surface creates ongoing bridging-layer maintenance that a narrower, well-chosen surface avoids.

## Common AI-Assistant Mistakes

- **DON'T:** Generate manual-retain-count (MRC) code — explicit `retain`/`release`/`autorelease` calls, `dealloc` overrides that call `[super dealloc]` — in a modern codebase. Nearly all current Objective-C projects are ARC-only; verify the project isn't using `-fno-objc-arc` before writing MRC-style memory management, and default to ARC idioms.
- **DON'T:** Omit `NS_ASSUME_NONNULL_BEGIN`/`END` and per-parameter nullability annotations on new public headers, especially in a project that bridges into Swift — this is one of the most common gaps in AI-generated Objective-C, and it downgrades type safety for every Swift caller.
- **DON'T:** Capture `self` strongly by default in blocks assigned to properties or passed to APIs that retain them past the current scope. Default to the `__weak`/`__strong` dance shown above for any block that outlives the immediate call, mirroring the same discipline expected in Swift closures.
- **DON'T:** Invent Foundation/UIKit/AppKit selector names or method signatures that sound plausible (a hallucinated `-[NSString trimmedString]` that doesn't exist, an imagined `NSURLSession` convenience initializer) — verify against current Apple documentation or existing usage in the codebase rather than pattern-matching from a similar but different API.
- **DON'T:** Mix Swift-style API design (trailing closures, argument omission) into generated Objective-C — Objective-C has its own idiom (fully labeled selectors, explicit block syntax) and code that tries to look "Swift-like" in Objective-C usually reads as unidiomatic and sometimes doesn't compile.
- **DON'T:** Skip the class-extension pattern for private properties and instead put mutable, writable versions of properties directly in the public header just because it's less code to write. This exposes implementation details and mutability that calling code should not rely on.
- **DON'T:** Forget `#import` and `@class` forward declarations, or import an entire framework header (`#import <UIKit/UIKit.h>`) inside another header when a forward declaration would do — unnecessary header-level imports slow compilation across a large project and can create import cycles.
- **DON'T:** Declare `NSString`/`NSArray`/`NSDictionary`/block properties as `strong` by default instead of `copy` — verify the correct attribute for the property's actual value semantics rather than reaching for `strong` as a one-size-fits-all default.
- **DON'T:** Generate Objective-C code assuming an older, pre-ARC, pre-`instancetype`, pre-`NS_ENUM` style is still current best practice just because older training examples used it — modern Objective-C (roughly the last decade's worth of conventions) has settled idioms that should be the default output, with any older-style pattern used only where a specific legacy codebase constraint requires it.
- **DON'T:** Assume a category method addition is safe without a project-specific prefix. Generated code that adds a category method to a Foundation/UIKit class with a generic, unprefixed name risks silently colliding with a same-named method from a dependency, with unpredictable results about which implementation actually runs.
- **DON'T:** Generate verbose pre-literal-syntax collection construction (`[NSArray arrayWithObjects:...]`) or pre-subscripting element access (`objectAtIndex:`) by default when modern literal and subscripting syntax is available and idiomatic for the target deployment version — default to the modern syntax unless the project's existing code consistently uses the older style.
- **DON'T:** Generate a block parameter or property without a `typedef` when the same block signature recurs across multiple declarations in the generated code — repeating a multi-parameter block type inline at every declaration site is a readability and consistency smell worth fixing at generation time.

## Quick Checklist
- Prefix custom types/globals with a project namespace prefix; never reuse Apple's reserved prefixes (`NS`, `UI`, etc.).
- Label every method parameter at declaration and call site; avoid ambiguous unlabeled selectors.
- Never write manual `retain`/`release`/`autorelease`/`[super dealloc]` in ARC-enabled code.
- Declare delegate and back-reference properties `weak`, not `strong`, to avoid retain cycles.
- Use the `__weak`/`__strong` capture dance for blocks stored on `self` or handed to long-lived APIs.
- Confirm the target actually uses ARC before writing any MRC-style code.
- Expose only the minimal public API in `.h`; keep mutable/internal state in a class extension in the `.m`.
- Use `@class` forward declarations in headers instead of full framework imports where possible.
- Group implementation code with `#pragma mark -` sections; split sprawling classes into categories.
- Declare `NS_DESIGNATED_INITIALIZER` and route convenience initializers through it.
- Wrap headers in `NS_ASSUME_NONNULL_BEGIN`/`END`; mark only true exceptions `nullable`.
- Annotate block parameter/return types with their own nullability, not just the enclosing method's.
- Match nullability annotations to actual runtime behavior, not wishful thinking.
- Use lightweight generics on `NSArray`/`NSDictionary` properties so Swift imports them fully typed.
- Declare enums with `NS_ENUM`/`NS_OPTIONS`, not bare `enum`, for correct Swift bridging.
- Write new business logic in Swift where the project is Swift-forward; reserve Objective-C for legacy/interop needs.
- Use `NS_SWIFT_NAME`/`NS_SWIFT_UNAVAILABLE` deliberately to shape the generated Swift interface.
- Never invent Foundation/UIKit/AppKit selectors or signatures — verify against real documentation or codebase usage.
- Don't graft Swift-style API idioms (trailing closures, label omission) onto Objective-C code.
- Verify retain-cycle-prone patterns (delegates, stored blocks, parent/child references) with Instruments or the Memory Graph Debugger.
- Use `instancetype` for factory/`init` methods so subclasses inherit correctly-typed constructors.
- Declare `copy` (not `strong`) for `NSString`/`NSArray`/`NSDictionary`/block properties that should snapshot their value.
- Prefix category method names on classes you don't own to avoid silent collisions with another library's category.
- Remove `NotificationCenter` observers and invalidate other resources in `dealloc`, even though ARC handles memory alone.
- Never let a `nonnull`-annotated method actually return `nil` on any code path.
- Design a thin Swift-facing adapter type at the language boundary instead of forcing a rich Swift type to be fully Objective-C-compatible.
