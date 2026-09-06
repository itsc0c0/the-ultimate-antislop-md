# JavaScript, TypeScript & Node.js

## Naming & Style

- **DO:** Use `camelCase` for variables and functions, `PascalCase` for classes, types, interfaces, and enums, and `SCREAMING_SNAKE_CASE` only for true module-level constants that are never reassigned. Consistent casing lets readers infer what a symbol is before they find its declaration, and mismatched casing is one of the fastest "this was AI-generated" tells.
```ts
// Bad
let User_Name = "koray";
function GetUser() {}
class userAccount {}

// Good
let userName = "koray";
function getUser() {}
class UserAccount {}
```
- **DO:** Name booleans with a predicate prefix (`is`, `has`, `can`, `should`, `did`) so call sites read like English. `if (userActive)` is ambiguous about whether it's a query or an assignment target; `if (isUserActive)` is not.
- **DON'T:** Name variables `data`, `result`, `temp`, `obj`, `item`, `value`, `res`, or `thing` when a domain-specific name is available. These generic names are the single most common signature of low-effort or AI-generated code and force every reader to re-derive what the variable holds from its usage.
```js
// Bad
const data = await fetchUsers();
const result = data.filter((item) => item.active);

// Good
const users = await fetchUsers();
const activeUsers = users.filter((user) => user.active);
```
- **DO:** Prefer full words over abbreviations except for extremely well-known ones (`id`, `url`, `html`, `config`, `req`/`res` inside an Express handler, `err` in a callback). `usrCnt` and `calcTtlPrc` save four keystrokes and cost the next reader ten seconds of decoding, every single time they read the line.
- **DON'T:** Use single-letter names outside of tiny, conventional scopes (loop indices `i`/`j`/`k`, coordinate pairs `x`/`y`, or a short arrow-function parameter whose meaning is obvious from context like `.map((n) => n * 2)`). A single-letter name in a function spanning more than a handful of lines forces readers to scroll back to remember what it is.
- **DO:** Name functions as verbs or verb phrases describing what they do (`calculateTotal`, `fetchOrder`, `isValidEmail`), and name values as nouns. A function named like a noun (`total`, `userValidation`) reads ambiguously at the call site as either an action or a stored value.
- **DON'T:** Pick a name that lies about the type or shape of what it holds — `userList` for a `Map`, `userArray` for a `Set`, `count` for a boolean. Prefer names that describe the contents' role rather than baking in a container type that might change, and never let the name contradict the actual type.
- **DO:** Keep naming consistent across a codebase for the same concept — pick one of `fetchX`/`getX`/`loadX` for retrieval and one of `remove`/`delete` for removal, then use it everywhere. Inconsistent synonyms for the same operation force readers to grep for three different verbs to find all call sites.
- **DON'T:** Prefix every helper, type, or file with the project or company name (`AcmeUserService`, `acme-utils.ts`) unless there's a real namespace collision to avoid. It adds visual noise to every reference without adding information, since the surrounding package/import path already establishes the origin.
- **DO:** Match file names to their primary export — `UserCard.tsx` exports `UserCard`, `formatCurrency.ts` exports `formatCurrency`. This makes imports and file trees predictable, and lets `grep`/"go to file" work by intuition instead of by memorized mapping.
- **DO:** Use `kebab-case` for file and directory names in Node/JS projects (`user-service.ts`, `date-utils.ts`) unless the framework mandates otherwise (e.g. React component files as `PascalCase.tsx` is a common, acceptable exception). Case-sensitive filesystems (Linux CI) and case-insensitive ones (macOS/Windows dev machines) diverge silently when naming is inconsistent, producing "works on my machine" import failures.
- **DON'T:** Mix naming conventions for the same category of file within one repository (`userService.ts` next to `order-service.ts` next to `PaymentService.ts`). Pick one convention per category and enforce it with a lint rule or a pre-commit check, not tribal memory.
- **DO:** Keep line length reasonable (most style guides converge on 80–100 characters) and let a formatter enforce it automatically rather than manually wrapping. Manually-wrapped lines rot the moment anyone edits the line without re-wrapping by hand.
- **DON'T:** Write dense one-liners that chain more than two or three operations without intermediate names, especially when they mix side effects with expressions. A wall of chained `.then().map().filter().reduce()` with no line breaks is quicker to write than to debug.
```js
// Bad
const r = users.filter(u=>u.active).map(u=>({...u,score:u.score*2})).sort((a,b)=>b.score-a.score).slice(0,10);

// Good
const activeUsers = users.filter((user) => user.active);
const scoredUsers = activeUsers.map((user) => ({ ...user, score: user.score * 2 }));
const topTenByScore = scoredUsers
  .sort((a, b) => b.score - a.score)
  .slice(0, 10);
```
- **DO:** Order class members and object properties consistently — typically: static members, fields, constructor, public methods, then private methods — and keep that order the same across every class in the codebase. A predictable layout means readers can jump straight to the section they need instead of scanning the whole file.
- **DON'T:** Leave commented-out code, `console.log` debugging statements, or `// TODO` markers without an owner/ticket reference in code that's presented as finished. Dead code and untracked TODOs are noise that erodes trust in the rest of the file — either delete it, wire it up, or file a ticket and reference it.
- **DO:** Write comments that explain *why*, not *what* — the code already says what it does; a comment should explain a non-obvious constraint, a workaround for a bug, or a business rule that isn't visible in the code itself.
```js
// Bad
// increment i by 1
i++;

// Good
// Stripe webhooks can arrive out of order, so we bump the cursor
// only after confirming this event is newer than the last one seen.
if (event.created > lastSeenTimestamp) cursor++;
```
- **DON'T:** Add a JSDoc block that restates the function signature in prose without adding new information (`/** Gets the user. @param id The id. @returns The user. */`). This is a classic AI-slop pattern — a comment that costs reading time and gives back nothing beyond what the signature already says.
- **DO:** Keep functions short enough to read on one screen and doing one identifiable thing. If you can't summarize what a function does in a single sentence without using "and", it's very likely doing too much and should be split.
- **DON'T:** Deeply nest conditionals when an early return or guard clause would flatten the logic. Nesting more than two or three levels deep makes the reader hold multiple conditions in their head simultaneously to understand any single line.
```js
// Bad
function getDiscount(user) {
  if (user) {
    if (user.isActive) {
      if (user.plan === "pro") {
        return 0.2;
      }
    }
  }
  return 0;
}

// Good
function getDiscount(user) {
  if (!user?.isActive) return 0;
  if (user.plan !== "pro") return 0;
  return 0.2;
}
```
- **DO:** Keep a single, consistent quote style (single or double) and semicolon usage across the whole project, delegated to Prettier or a similar formatter rather than manual discipline. Style debates are a waste of review time when a tool can settle them deterministically on every save/commit.
- **DON'T:** Use magic numbers or magic strings inline in logic — extract them into named constants, especially when the same value is repeated more than once. A named constant documents intent and gives you one place to change the value later.
```js
// Bad
if (retries > 3) throw new Error("too many retries");
setTimeout(poll, 5000);

// Good
const MAX_RETRIES = 3;
const POLL_INTERVAL_MS = 5000;
if (retries > MAX_RETRIES) throw new Error("too many retries");
setTimeout(poll, POLL_INTERVAL_MS);
```
- **DO:** Group related constants and configuration into a single well-named module (`config.ts`, `constants.ts`) rather than scattering literals across many files. This gives future maintainers one place to look when a value needs tuning.
- **DON'T:** Invent your own ad hoc formatting or linting conventions when an established, widely-adopted config (Airbnb, Standard, the TypeScript ESLint recommended set) already covers 95% of cases. Bespoke rulesets require onboarding every contributor manually and drift out of sync with tooling updates.

## Language Fundamentals & Idioms

### Variables & Scoping

- **DO:** Default to `const`; use `let` only for bindings that are genuinely reassigned; never use `var`. `var` is function-scoped (not block-scoped), is hoisted with a confusing "declared but undefined" window, and silently allows redeclaration — all classes of bugs that `const`/`let` eliminate by construction.
```js
// Bad
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 10); // logs 3, 3, 3
}

// Good
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 10); // logs 0, 1, 2
}
```
- **DON'T:** Rely on `var`'s function-scoping or hoisting behavior as a "clever" way to share state across a block. If state genuinely needs to live above a block, declare it explicitly above the block with `let`, not implicitly via hoisting.
- **DO:** Declare variables as close as possible to their first use, in the narrowest scope that works. This keeps the "live range" of a variable short, which reduces the chance it's read before being properly initialized or mutated unexpectedly far from its declaration.
- **DON'T:** Shadow outer-scope variable names with inner-scope ones of the same name (`const user = ...` inside a block that already has an outer `user` in scope), even though `let`/`const` make this technically safe. Shadowing is a frequent source of "I thought I was mutating the outer one" bugs during refactors, and linters (`no-shadow`) exist specifically to catch it.
- **DO:** Understand and rely on the temporal dead zone: referencing a `let`/`const` before its declaration throws a `ReferenceError` rather than silently returning `undefined` as `var` would. Treat that error as a signal of a genuine ordering bug, not something to "fix" by hoisting the declaration blindly.
- **DON'T:** Use `==` for comparisons; always use `===`/`!==` unless deliberately checking for both `null` and `undefined` in one shot (`x == null`), which is the one broadly accepted exception. Loose equality's coercion rules (`"" == 0`, `[] == false`, `null == undefined` but `null == 0` is `false`) are inconsistent enough that even experienced developers can't reliably predict every case.
```js
// Bad
if (count == "0") { /* ... */ }

// Good
if (count === 0) { /* ... */ }
// or, the one accepted == use:
if (value == null) { /* covers null and undefined */ }
```
- **DO:** Use `Object.is`-aware reasoning (or a proper deep-equality utility) when comparing values that might be `NaN` or `-0`, since `===` treats `NaN !== NaN` and `-0 === 0`. Rely on `Number.isNaN(x)` to check for `NaN`, never `x === NaN`, which is always `false`.
- **DON'T:** Rely on truthy/falsy checks for values that can legitimately be `0`, `""`, or `NaN` — the falsy check silently accepts a value the code didn't intend.
```js
// Bad — breaks when count is legitimately 0
if (!options.count) options.count = DEFAULT_COUNT;

// Good
if (options.count === undefined) options.count = DEFAULT_COUNT;
// or
options.count ??= DEFAULT_COUNT;
```
- **DO:** Use the nullish coalescing operator `??` for "use this default only when the left side is `null`/`undefined`," and reserve `||` for genuine "use this fallback for any falsy value" cases. Conflating the two is a very common source of bugs when a valid `0`, `""`, or `false` value gets silently replaced by a default.
- **DO:** Use optional chaining (`?.`) to safely access properties on potentially-missing objects instead of manual chained checks. It's shorter, and unlike a truthy check on the parent, it correctly short-circuits without hiding a legitimately falsy intermediate value.
```js
// Bad
const city = user && user.address && user.address.city;

// Good
const city = user?.address?.city;
```
- **DON'T:** Chain optional chaining so far that a `TypeError` a few properties deep gets silently swallowed into `undefined`, hiding a real bug (e.g., a typo'd property name) behind "it just returns undefined." If a property is expected to always exist once you're past a certain point, access it directly so a missing value throws loudly instead of propagating silently.
- **DO:** Use template literals for any string built from more than one piece, instead of `+` concatenation. Template literals are more readable, avoid accidental `+` operator type coercion bugs, and support multi-line strings natively.
```js
// Bad
const message = "Hello, " + user.firstName + " " + user.lastName + "!";

// Good
const message = `Hello, ${user.firstName} ${user.lastName}!`;
```
- **DON'T:** Use `+` to concatenate numbers and strings interchangeably and rely on implicit coercion — `1 + "1"` is `"11"` but `"1" - 1` is `0`. Convert explicitly with `String(x)`/`Number(x)` (or `Number.parseInt`/`Number.parseFloat` with an explicit radix) whenever type mixing is unavoidable.
- **DO:** Use `Number.parseInt(value, 10)` with an explicit radix (or `Number()` for a full numeric parse) rather than bare `parseInt(value)`. Omitting the radix historically caused strings like `"08"` to be parsed as octal in older engines, and even though modern engines default to base 10, an explicit radix documents intent and silences linters.
- **DO:** Freeze objects that represent constants using `Object.freeze()` when you want to prevent accidental mutation, and prefer `readonly` (TypeScript) / immutable data patterns generally over defensive copying scattered through the codebase.
```ts
const HTTP_STATUS = Object.freeze({
  OK: 200,
  NOT_FOUND: 404,
  SERVER_ERROR: 500,
} as const);
```
- **DON'T:** Mutate function arguments, especially objects and arrays passed in from a caller, unless the function is explicitly documented (and named) as a mutator. Silent mutation of inputs is a classic source of "why did this unrelated object change" bugs, especially once the same object is passed through several layers.
```js
// Bad — mutates the caller's array
function addTax(items) {
  items.forEach((item) => (item.price *= 1.08));
  return items;
}

// Good — returns a new array, leaves input untouched
function withTax(items) {
  return items.map((item) => ({ ...item, price: item.price * 1.08 }));
}
```
- **DO:** Use `Array.isArray(x)` to check whether a value is an array, never `typeof x === "object"` (which is also true for plain objects, `null`-typed edge cases aside) or `x instanceof Array` (which fails across realms/iframes).

### Functions

- **DO:** Prefer arrow functions for callbacks and short, non-method functions, and named function declarations for top-level, reusable, or recursive functions. Arrow functions lexically bind `this`, which avoids an entire category of `this`-binding bugs in callbacks, while named function declarations are hoisted and show up with a real name in stack traces.
- **DON'T:** Use arrow functions as object methods that need their own `this` (e.g., class-like object methods, or Express/Vue/Node event handlers that rely on the calling context). An arrow function captures `this` from its enclosing scope at definition time, so binding it as a method silently breaks any code that expects `this` to refer to the object it's called on.
```js
// Bad — `this` inside the arrow is not `counter`
const counter = {
  count: 0,
  increment: () => { this.count++; },
};

// Good
const counter = {
  count: 0,
  increment() { this.count++; },
};
```
- **DO:** Give functions default parameter values instead of manually checking for `undefined` inside the body. Default parameters are evaluated lazily (only when the argument is omitted), are visible in the signature, and are picked up by editor tooling and TypeScript alike.
```js
// Bad
function createUser(name, role) {
  role = role || "member";
  // ...
}

// Good
function createUser(name, role = "member") {
  // ...
}
```
- **DON'T:** Write functions that accept more than three or four positional parameters, especially several of the same type (multiple booleans or strings in a row) — callers can't tell what each position means without checking the signature, and it's trivial to pass arguments in the wrong order. Use a single options object with named properties instead.
```ts
// Bad — what do these three booleans mean at the call site?
function createReport(title: string, includeCharts: boolean, sendEmail: boolean, archive: boolean) {}
createReport("Q3", true, false, true); // unreadable at the call site

// Good
function createReport(title: string, options: { includeCharts?: boolean; sendEmail?: boolean; archive?: boolean }) {}
createReport("Q3", { includeCharts: true, archive: true });
```
- **DO:** Use rest parameters (`...args`) instead of the legacy `arguments` object when a function needs a variable number of arguments. `arguments` is not a real array (no `.map`/`.filter` without conversion), doesn't exist in arrow functions, and rest parameters are strictly more capable.
- **DO:** Return early from a function once its result is determined, rather than accumulating the result in a variable that's returned once at the end through nested branches. Early returns reduce nesting and make the set of exit conditions explicit and scannable.
- **DON'T:** Write functions with side effects hidden behind an innocuous-looking name (`getUser()` that also writes to a cache, logs analytics, and mutates a global). A function's name is a contract; if it does more than the name implies, either rename it or split the side effect into its own explicit call.
- **DO:** Make pure functions the default wherever practical — same input always produces same output, no reliance on or mutation of external state. Pure functions are trivial to unit test, safe to memoize, and safe to run in any order, none of which is true once hidden state or side effects are involved.
- **DON'T:** Use `Function` constructor strings or dynamically constructed function bodies to work around a language limitation. If you find yourself building a function from a string, there is almost always a data-driven or higher-order-function approach that avoids the security and tooling costs (see the Security section below).
- **DO:** Prefer composition of small, focused functions over one large function with many internal branches. `pipe`/`compose`-style chains of single-purpose functions are individually testable and often self-documenting through their names.
- **DON'T:** Use recursive functions without a clear, provably-terminating base case and without considering the input size against JavaScript's call stack limits (roughly a few thousand to ~15,000 frames depending on engine and frame size). For anything that might process large or attacker-controlled input depth, prefer an iterative approach or an explicit stack/queue.
```js
// Bad — stack overflow on large arrays, no tail-call optimization in V8
function sum(arr) {
  if (arr.length === 0) return 0;
  return arr[0] + sum(arr.slice(1));
}

// Good
function sum(arr) {
  return arr.reduce((total, n) => total + n, 0);
}
```

### Objects & Arrays

- **DO:** Use object and array destructuring to extract multiple properties/elements at once instead of repeated dot/bracket access. It's more concise and it documents, right at the top of a function, exactly which fields the function depends on.
```js
// Bad
function greet(user) {
  const name = user.name;
  const role = user.role;
  return `${name} (${role})`;
}

// Good
function greet({ name, role }) {
  return `${name} (${role})`;
}
```
- **DO:** Use the spread operator (`...`) for shallow copies and merges of objects/arrays instead of `Object.assign` for simple cases, and be explicit that spread/`Object.assign` both only produce a *shallow* copy — nested objects are still shared by reference.
```js
// Bad — mutates original, and callers may not expect that
function updateStatus(order, status) {
  order.status = status;
  return order;
}

// Good — new object, original untouched
function withStatus(order, status) {
  return { ...order, status };
}
```
- **DON'T:** Assume spread or `{ ...obj }` performs a deep clone. Nested objects/arrays are copied by reference, so mutating a nested field on the "copy" still mutates the original. Use `structuredClone(obj)` (available natively in modern Node and browsers) for a real deep clone, or a well-tested utility, not a hand-rolled recursive copier or `JSON.parse(JSON.stringify(obj))` (which silently drops `undefined`, functions, `Date` objects become strings, etc.).
```js
const original = { user: { name: "Ada" } };
const shallow = { ...original };
shallow.user.name = "Grace";
console.log(original.user.name); // "Grace" — the nested object was shared

const deep = structuredClone(original);
deep.user.name = "Ada";
console.log(original.user.name); // unaffected
```
- **DO:** Use `Object.entries()`, `Object.keys()`, and `Object.values()` with `for...of` or array methods to iterate objects, rather than `for...in`, which also walks inherited enumerable properties and requires a `hasOwnProperty` guard to use safely.
```js
// Bad
for (const key in user) {
  console.log(key, user[key]); // may include inherited props
}

// Good
for (const [key, value] of Object.entries(user)) {
  console.log(key, value);
}
```
- **DO:** Prefer array methods (`.map`, `.filter`, `.reduce`, `.find`, `.some`, `.every`, `.flatMap`) over manual `for` loops when transforming or querying collections — they're declarative, less error-prone (no off-by-one index bugs), and communicate intent through the method name itself.
- **DON'T:** Use `.forEach` (or a `for` loop) when a `.map`/`.filter`/`.find` would do — reaching for imperative iteration with manual array pushing when a direct transform exists is a common tell of unpolished code.
```js
// Bad
const names = [];
users.forEach((user) => {
  if (user.active) {
    names.push(user.name);
  }
});

// Good
const names = users.filter((user) => user.active).map((user) => user.name);
```
- **DON'T:** Use array methods where a plain `for`/`for...of` loop is actually clearer or more performant — e.g., needing to `break` early (array methods can't short-circuit except `.some`/`.every`/`.find`), or needing the loop index alongside complex control flow. Reach for the readable tool for the specific job rather than dogmatically preferring one style everywhere.
- **DO:** Use `Array.prototype.includes()` to check membership instead of `indexOf(x) !== -1`, and use a `Set` instead of an array when membership checks happen frequently in a hot path — array `includes`/`indexOf` are O(n) per call, `Set.has()` is O(1) on average.
```js
// Bad — O(n) lookup per call, called in a loop, so O(n*m) overall
function dedupeActive(users, activeIds) {
  return users.filter((u) => activeIds.includes(u.id));
}

// Good — O(1) average lookup
function dedupeActive(users, activeIds) {
  const activeIdSet = new Set(activeIds);
  return users.filter((u) => activeIdSet.has(u.id));
}
```
- **DO:** Use `Map` instead of a plain object when keys are dynamic, non-string, or when insertion order and a real `.size` matter. Plain objects coerce keys to strings, inherit prototype properties that can collide with user-supplied keys (see prototype pollution in Security), and don't have a native size.
- **DON'T:** Use a plain object as a hash map for untrusted or dynamic keys without safeguards — keys like `"__proto__"`, `"constructor"`, or `"toString"` can collide with the prototype chain. Prefer `Map`, or `Object.create(null)` for a prototype-less object, when keys come from user input.
```js
// Bad — vulnerable to key collision / prototype pollution vectors
const cache = {};
cache[userSuppliedKey] = value;

// Good
const cache = new Map();
cache.set(userSuppliedKey, value);
```
- **DO:** Use `Set` for uniqueness and dedup logic instead of manual "have I seen this before" array tracking. `[...new Set(array)]` is the idiomatic one-liner for deduping an array of primitives.
- **DO:** Sort arrays with an explicit comparator when order matters for anything but default string sorting — `Array.prototype.sort()` without a comparator converts elements to strings, so `[10, 2, 1].sort()` yields `[1, 10, 2]`, not numeric order.
```js
// Bad
[10, 2, 1].sort(); // [1, 10, 2] — lexicographic, not numeric

// Good
[10, 2, 1].sort((a, b) => a - b); // [1, 2, 10]
```
- **DON'T:** Forget that `.sort()` mutates the original array in place. If the caller doesn't expect that, sort a copy: `[...array].sort(comparator)`, or use `Array.prototype.toSorted()` (ES2023+) which returns a new array.

### Deep Equality vs. Reference Equality

- **DON'T:** Compare two objects or arrays with `===` expecting it to check whether their *contents* match — `===` on non-primitive values checks reference identity (are these two variables pointing at the literal same object in memory), so two structurally identical objects created separately are never `===` to each other, which surprises developers coming from languages with value-type comparison semantics for aggregate types.
```js
console.log({ a: 1 } === { a: 1 }); // false — different object references
console.log([1, 2] === [1, 2]);     // false — same reason

const same = { a: 1 };
console.log(same === same);          // true — actually the same reference
```
- **DO:** Use a real deep-equality function — `assert.deepStrictEqual` (Node's built-in, useful outside test files too), a test framework's `toEqual`/`toStrictEqual` matcher, or a small library like `fast-deep-equal` — when you actually need to compare the contents of two objects/arrays for structural equality, rather than writing an ad hoc recursive comparison by hand, which is easy to get subtly wrong for edge cases (`NaN`, `-0`, `Date` objects, `Map`/`Set`, circular references, property order).
```js
import assert from "node:assert/strict";
assert.deepStrictEqual({ a: 1, b: [1, 2] }, { a: 1, b: [1, 2] }); // passes — content matches
```
- **DON'T:** Use `JSON.stringify(a) === JSON.stringify(b)` as a deep-equality check — beyond the general `JSON.stringify` pitfalls covered in the JSON & Serialization section (dropped `undefined`, converted `Date`s, thrown-on-circular-reference, `BigInt` incompatibility), this approach is also sensitive to key *order*, so two objects with identical key-value pairs in a different insertion order can incorrectly compare as unequal.
- **DO:** Reach for `Object.is()` specifically when you need `===`-like comparison but with correct handling of the two cases where `===` behaves surprisingly: `Object.is(NaN, NaN)` is `true` (unlike `NaN === NaN`, which is always `false`), and `Object.is(0, -0)` is `false` (unlike `0 === -0`, which is `true`) — a niche but real distinction relevant to numerical code that needs to distinguish positive and negative zero.
- **DON'T:** Assume two `Map`s or `Set`s with the same entries are `===` to each other, or that a deep-equality library handles them correctly by default without checking — some general-purpose deep-equal implementations historically had incomplete support for `Map`/`Set`/typed-array comparison; verify the specific library/matcher you're using actually supports the collection types your comparison needs before trusting its result for those types.

### Array Method Edge Cases Worth Knowing

- **DON'T:** Call `.reduce()` without an initial value on an array that might be empty — `[].reduce((a, b) => a + b)` throws a `TypeError: Reduce of empty array with no initial value` rather than returning some sensible default, which is a common source of a rare-but-real production crash on whatever edge case first produces an empty array for a code path that was always tested against non-empty data.
```js
// Bad — throws if orders is empty
const total = orders.reduce((sum, order) => sum + order.amount);

// Good — always provide an initial value, which also fixes the type/starting-value ambiguity
const total = orders.reduce((sum, order) => sum + order.amount, 0);
```
- **DO:** Provide an explicit initial value to `.reduce()` even when the array is known to be non-empty in practice — beyond avoiding the empty-array crash, an explicit initial value also removes any ambiguity about the accumulator's starting type, which matters especially when reducing into a shape different from the array's element type (e.g., reducing an array of items into a single object or `Map`).
- **DON'T:** Assume `Array.prototype.sort()`, `.filter()`, `.map()`, or `.reduce()` skip over "holes" in a sparse array the same way `.forEach()` does, or that they all behave identically regarding sparse arrays — sparse-array edge-case behavior differs subtly across array methods, which is one more reason to avoid creating sparse arrays in the first place (e.g., avoid `new Array(5)` followed by only setting some indices) rather than relying on correctly remembering each method's specific hole-handling behavior.
- **DO:** Remember that `.find()`, `.some()`, and `.every()` short-circuit (stop iterating as soon as the result is determined), while `.map()`, `.filter()`, and `.forEach()` always visit every element — this matters both for performance (prefer `.some()` over `.filter(...).length > 0` for an existence check, since `.filter()` is forced to scan the entire array even after finding a match) and for correctness when the callback has side effects whose exact call count matters.
```js
// Less efficient — scans the entire array even after finding a match
const hasAdmin = users.filter((u) => u.role === "admin").length > 0;

// More efficient — stops at the first match
const hasAdmin = users.some((u) => u.role === "admin");
```
- **DON'T:** Mutate the array you're currently iterating over inside a `.forEach()`/`for...of`/`for` loop (pushing to it, removing elements from it) — the iteration behavior when the underlying array changes mid-loop is confusing and easy to get wrong (elements can be skipped or visited twice depending on exactly how the array was mutated and which method is iterating). Build a new array/list of changes during iteration and apply them after the loop finishes, or iterate over a copy (`[...array]`) if in-place mutation of the original during iteration is unavoidable.

### Equality, Coercion & Numbers

- **DO:** Be deliberate about floating-point arithmetic — JavaScript numbers are IEEE-754 doubles, so `0.1 + 0.2 !== 0.3`. For money or anything requiring exact decimal precision, work in integer minor units (cents) or use a dedicated decimal library (`decimal.js`, `big.js`) rather than raw floats.
```js
// Bad
const total = 0.1 + 0.2; // 0.30000000000000004
if (price === 19.99) { /* may never match after arithmetic */ }

// Good — track money in integer cents
const totalCents = 10 + 20; // exact
```
- **DO:** Use `BigInt` (the `123n` literal or `BigInt(x)`) for integers that may exceed `Number.MAX_SAFE_INTEGER` (2^53 - 1), such as large IDs from external systems (e.g., some database or snowflake-style IDs). Silent precision loss on large integers is a subtle correctness bug that's easy to miss in code review.
- **DON'T:** Mix `BigInt` and `Number` in arithmetic — `1n + 1` throws a `TypeError`. Convert explicitly at the boundary and keep a consistent type through a given calculation.
- **DO:** Use `Number.isInteger()` and `Number.isFinite()` (not the global `isFinite`, which coerces its argument first) when validating numeric input, since both correctly reject `NaN`, `Infinity`, and non-numeric coerced values.

### Symbols, Proxies & Immutability Patterns

- **DO:** Use `Symbol()` for object property keys that must never collide with a string key from another source (a library-internal marker property, a well-known protocol hook like `Symbol.iterator`), since every `Symbol()` call produces a guaranteed-unique value even when two symbols are created with the identical description string.
- **DON'T:** Use a `Symbol` where a plain string constant would do just as well and be easier to serialize, log, and debug — symbols can't be represented in JSON, are awkward to log meaningfully (they print as `Symbol(description)`), and add a layer of indirection that's only worth it when true uniqueness/collision-avoidance is the actual requirement.
- **DO:** Reach for `Proxy`/`Reflect` for genuinely dynamic, meta-programming needs — validating property writes generically across many properties, creating a reactive object wrapper (the mechanism many reactive-state libraries build on), or implementing a virtualized/lazy object — but recognize this is an advanced tool with real costs: every property access on a proxied object goes through the trap handlers, which is slower than plain property access and can be harder for readers and tooling (including some optimizing compilers) to follow.
```js
// A validating proxy — write access is intercepted and checked generically
function createValidated(target, validators) {
  return new Proxy(target, {
    set(obj, prop, value) {
      const validate = validators[prop];
      if (validate && !validate(value)) {
        throw new TypeError(`invalid value for ${String(prop)}: ${value}`);
      }
      obj[prop] = value;
      return true;
    },
  });
}
const user = createValidated({ age: 0 }, { age: (v) => typeof v === "number" && v >= 0 });
user.age = -5; // throws TypeError
```
- **DON'T:** Reach for a `Proxy` to solve a problem a simple getter/setter, a plain validation function, or a small wrapper class would solve just as well with far less indirection — `Proxy` is powerful but its behavior is invisible at the call site (`obj.foo = 1` looks like an ordinary assignment even when it's secretly running arbitrary trap logic), which makes it easy to misuse into "clever but impossible to debug."
- **DO:** Use `Object.freeze()` for shallow immutability on configuration objects and constants, and understand its limit — it only prevents reassignment/addition/deletion of the object's own direct properties; nested objects inside a frozen object are not themselves frozen and remain fully mutable unless you deep-freeze recursively.
```js
const config = Object.freeze({
  retries: 3,
  limits: { maxUsers: 100 }, // NOT frozen — this nested object is still mutable
});
config.retries = 5; // silently no-ops in non-strict mode, throws in strict mode/modules
config.limits.maxUsers = 999999; // succeeds — freeze() is shallow
```
- **DO:** Use a small recursive `deepFreeze` utility (or a library) when genuine deep immutability is required, and prefer immutable-by-convention data patterns (returning new objects instead of mutating, as covered earlier) as the primary strategy — `Object.freeze` is a runtime safety net, not a substitute for actually writing code that doesn't mutate its inputs in the first place.
- **DON'T:** Rely on `Object.freeze()` as a security boundary against a determined adversary running code in the same JavaScript realm — it prevents accidental mutation from well-behaved code, not deliberate circumvention (e.g., via `Object.isFrozen` checks being bypassed by working with a different, non-frozen reference to the same underlying data, or via prototype-chain tricks in more exotic cases).

### Array & Object Destructuring Patterns

- **DO:** Provide default values directly in a destructuring pattern for properties/elements that might be missing, instead of a separate follow-up line checking for `undefined`. This keeps the "what does this parameter default to" information colocated with the destructuring itself, where a reader will naturally look for it.
```js
// Bad — default handling separated from the destructuring
function createServer({ port, host }) {
  port = port === undefined ? 3000 : port;
  host = host === undefined ? "localhost" : host;
}

// Good
function createServer({ port = 3000, host = "localhost" } = {}) {
  // the `= {}` default lets createServer() be called with no argument at all
}
```
- **DO:** Rename a destructured property when the source object's field name would collide with an existing variable in scope, or when a more descriptive local name improves readability — `const { id: userId } = user;` is often clearer than a bare `id` once there's more than one kind of `id` in the surrounding function.
- **DON'T:** Destructure more than roughly four or five properties out of a single object in one pattern, or nest destructuring more than two levels deep, without a strong readability reason — beyond that point, a destructuring pattern becomes harder to scan than simply naming the object and accessing the handful of fields you actually need via dot notation.
```js
// Hard to scan — which fields actually matter here?
const { id, name, email, address: { city, zip }, preferences: { theme, locale } } = user;

// Often clearer for a function that only needs a couple of fields
function getDisplayName(user) {
  return `${user.name} (${user.address.city})`;
}
```
- **DO:** Use array destructuring with a placeholder (a skipped comma) to ignore elements you don't need, and use it for the common "swap two variables without a temp" idiom, which is one of the few places array destructuring is unambiguously the clearest option available.
```js
const [, second, , fourth] = ["a", "b", "c", "d"]; // skips indices 0 and 2
let a = 1, b = 2;
[a, b] = [b, a]; // swap without a temporary variable
```
- **DON'T:** Destructure a value that might be `null`/`undefined` at the top level without a guard or a default — destructuring `null`/`undefined` throws a `TypeError` immediately (`const { x } = null` throws), which is sometimes the desired fail-fast behavior but is often an accidental crash when the source value was only expected to sometimes be present.
- **DO:** Use the rest pattern (`const { password, ...publicUser } = user;`) to derive a new object with specific properties excluded, which is a common and readable idiom for stripping sensitive or internal fields before sending an object back to a client — but remember it only performs a shallow copy of the remaining properties, same as object spread.

### Modern String & Array Methods Worth Knowing

- **DO:** Use `String.prototype.replaceAll()` when the intent is genuinely "replace every occurrence" of a literal substring — before it existed, this required a global-flag regex (`str.replace(/foo/g, "bar")`) even for a plain literal string, which meant remembering to escape regex-special characters in what was conceptually just a plain string replacement. `replaceAll` with a string argument does a literal, non-regex replacement directly.
```js
// Before replaceAll existed, a literal string replacement needed regex + escaping
const escaped = literal.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
str.replace(new RegExp(escaped, "g"), replacement);

// Now, direct and unambiguous for literal-string replacement
str.replaceAll(literal, replacement);
```
- **DO:** Use `Array.prototype.at(-1)` to access the last element of an array (or any negative-index-from-the-end position) instead of `arr[arr.length - 1]` — it's shorter, less error-prone (no off-by-one risk in the length arithmetic), and works identically on strings (`str.at(-1)`) for the same "last character" access pattern.
- **DO:** Use `String.prototype.padStart()`/`padEnd()` for fixed-width formatting needs (zero-padding a number, aligning output columns) instead of a hand-rolled loop or string-repetition trick — these were added specifically to replace exactly that category of manual, easy-to-get-wrong string-building code.
```js
const invoiceNumber = String(42).padStart(6, "0"); // "000042"
```
- **DO:** Use `Array.prototype.flat(depth)` and `flatMap()` for flattening nested arrays and for the common "map then flatten one level" pattern, instead of `[].concat(...arr)` or a manual reduce-based flattening implementation — both are natively supported, more readable, and communicate intent directly through the method name.
- **DO:** Use the non-mutating array methods added in ES2023 (`toSorted()`, `toReversed()`, `toSpliced()`, `with()`) when a project's target runtime supports them, instead of the classic "spread then mutate" workaround (`[...arr].sort(...)`) — they express the same "give me a new array, don't touch the original" intent directly, without relying on the reader recognizing the spread-then-mutate idiom as intentional rather than an accidental omission of a copy step.
- **DON'T:** Assume every one of these newer methods is available without checking the project's actual Node.js/browser target — `at()`, `replaceAll()`, and `flat()`/`flatMap()` are broadly available in any reasonably current environment, but the ES2023 non-mutating array methods (`toSorted()` and friends) are recent enough that they genuinely require a modern runtime; verify against the project's declared engine/browser support before relying on them.

### Classes vs. Functions & OOP Patterns

- **DON'T:** Wrap stateless logic in a class whose only members are `static` methods, or a class that's instantiated once and never holds any meaningful mutable state. This is one of the clearest AI-assistant tells in JavaScript/TypeScript output — a class adds a constructor, a prototype chain, and ceremony (`new MyService().doThing()`) around what is functionally just a plain function, with zero benefit.
```ts
// Bad — no state, no reason to be a class
class MathUtils {
  static add(a: number, b: number) {
    return a + b;
  }
  static multiply(a: number, b: number) {
    return a * b;
  }
}
MathUtils.add(2, 3);

// Good — plain functions, simpler to import, test, and tree-shake
export function add(a: number, b: number) {
  return a + b;
}
export function multiply(a: number, b: number) {
  return a * b;
}
```
- **DO:** Reach for a class when you genuinely need encapsulated *mutable state* paired with behavior that operates on it, multiple independent instances of that state to exist at once, or a documented contract (an interface/abstract class) that several concrete implementations must satisfy (e.g., multiple payment-provider adapters implementing a shared `PaymentGateway` interface).
```ts
// Good use of a class — real encapsulated state + invariants to protect
class RateLimiter {
  #hits = new Map<string, number[]>();
  constructor(private readonly limit: number, private readonly windowMs: number) {}

  allow(key: string): boolean {
    const now = Date.now();
    const recent = (this.#hits.get(key) ?? []).filter((t) => now - t < this.windowMs);
    if (recent.length >= this.limit) return false;
    recent.push(now);
    this.#hits.set(key, recent);
    return true;
  }
}
```
- **DON'T:** Create a class purely to "namespace" a group of unrelated helper functions (`class StringUtils { static capitalize() {}; static slugify() {} }`). A module (a single file with named exports) already provides namespacing through its file path and import statement — a class adds nothing on top of that except boilerplate.
- **DO:** Prefer plain object literals or factory functions over classes for simple, stateless-or-simply-shaped object creation, especially when consumers never need `instanceof` checks, inheritance, or private fields. A factory function (`function createUser(name) { return { name, createdAt: new Date() }; }`) is simpler to read, requires no `new`, and can't be accidentally called without `new` (a classic legacy-JS foot-gun with constructor functions).
- **DON'T:** Reach for class inheritance (`extends`) as the default way to share behavior between two related types. Inheritance creates tight coupling between parent and child and tends to produce fragile hierarchies as requirements evolve (the classic "fragile base class" problem) — prefer composition (passing in collaborators, combining small independent pieces of behavior) unless there's a genuine, stable "is-a" relationship.
```ts
// Bad — inheritance forces every payment method through one rigid hierarchy
class Payment { protected fee = 0; process() { /* base logic */ } }
class CardPayment extends Payment { /* override for card specifics */ }
class PaypalPayment extends Payment { /* override for paypal specifics */ }

// Good — composition: each strategy is independent, easy to add/swap/test
interface PaymentStrategy { process(amount: number): Promise<void>; }
class PaymentProcessor {
  constructor(private readonly strategy: PaymentStrategy) {}
  process(amount: number) { return this.strategy.process(amount); }
}
```
- **DO:** Keep any class hierarchy that does exist shallow — one level of inheritance is usually fine, two is a warning sign, three or more is very likely a design that should be refactored toward composition. Deep hierarchies force readers to trace behavior across many files to understand what a single method actually does once overrides and `super` calls are accounted for.
- **DON'T:** Scatter `.bind(this)` calls through a constructor, or write every method as an arrow-function class field purely to work around `this` losing its binding when methods are passed as callbacks. If a class needs this much manual `this`-management to be usable, it's a signal the behavior doesn't actually need a class at all — a set of plain functions (optionally closing over shared state via a factory function) sidesteps the entire problem.
- **DO:** Use real private class fields (`#field`) for genuine encapsulation in modern JS/TS, instead of the `_field` underscore-prefix convention (which is purely cosmetic — `_field` is still fully public and accessible) or closure-based privacy tricks that complicate the class's shape.
```ts
// Bad — `_balance` is still publicly accessible; the underscore is only a convention
class Account {
  _balance = 0;
  withdraw(amount: number) { this._balance -= amount; }
}
account._balance = 999999; // nothing stops this

// Good — genuinely private, enforced by the language
class Account {
  #balance = 0;
  withdraw(amount: number) {
    if (amount > this.#balance) throw new Error("insufficient funds");
    this.#balance -= amount;
  }
  get balance() { return this.#balance; }
}
```
- **DON'T:** Mix `static` and instance state carelessly, especially in a server process handling concurrent requests — a `static` field is shared across every instance and across the entire lifetime of the module, so using one to hold per-request or per-user data leaks state between unrelated requests handled by the same running process.
- **DO:** Remember that in Node.js, a module is evaluated once and its result cached by the module system — so exporting a plain configured object/instance directly from a module already gives you singleton behavior for free (`export const db = createConnection(...)`), with no need for a hand-rolled Singleton-pattern class (a private static instance field plus a static `getInstance()` method) that's solving a problem the module system already solved.
- **DON'T:** Model pure data-transfer shapes (a database row, an API response payload) as class instances when a plain object typed with an `interface`/`type` is sufficient. Classes add constructor-call overhead at every creation site and create friction at serialization boundaries — a class instance doesn't round-trip through `JSON.stringify`/`JSON.parse` back into the same class without extra work, unlike a plain object that already matches its own type.
- **DO:** Reach for a class specifically when modeling a genuine domain entity whose *invariants* need active protection — e.g., a `Money` type that must always carry a valid, consistent currency and non-negative amount, where every mutation needs to go through validated methods rather than being freely settable on a plain object. This is exactly the scenario encapsulation exists for.

### `this`, Binding & Common Context Pitfalls

- **DON'T:** Pass a class method or an object method directly as a callback (`element.addEventListener("click", user.handleClick)`, `array.forEach(logger.log)`) without binding it — doing so detaches the method from the object it was defined on, so any use of `this` inside the method resolves to `undefined` (in strict-mode modules, which is the default for both ESM and TypeScript output) or the wrong object entirely, rather than throwing a helpful error at the point of the mistake.
```js
class Logger {
  prefix = "[app]";
  log(message) {
    console.log(`${this.prefix} ${message}`); // `this` depends on how log() is called
  }
}
const logger = new Logger();
["a", "b"].forEach(logger.log); // TypeError: Cannot read properties of undefined (reading 'prefix')

// Fixed: bind, or use an arrow function wrapper, or convert log to an arrow class field
["a", "b"].forEach((msg) => logger.log(msg));
```
- **DO:** Understand the four ways `this` gets determined in JavaScript — implicit binding (called as `obj.method()`), explicit binding (`.call()`/`.apply()`/`.bind()`), `new` binding (constructor calls), and lexical binding (arrow functions inherit `this` from their enclosing scope) — since a bug caused by the wrong one of these is otherwise very hard to diagnose without knowing which rule actually applies at the specific call site in question.
- **DO:** Use `Function.prototype.call()`/`.apply()` when you need to invoke a function with an explicitly chosen `this` and either individual arguments (`call`) or an array of arguments (`apply`), and use `.bind()` when you need to produce a new function permanently bound to a specific `this` for later use (e.g., as a callback) rather than invoking it immediately.
```js
function introduce(greeting) {
  return `${greeting}, I'm ${this.name}`;
}
const person = { name: "Ada" };
introduce.call(person, "Hi");           // "Hi, I'm Ada" — invoke now, explicit this
const boundIntroduce = introduce.bind(person);
boundIntroduce("Hello");                // "Hello, I'm Ada" — invoke later, this locked in
```
- **DON'T:** Rely on `this` inside a regular (non-arrow) standalone function expecting it to refer to some enclosing object "because that's where the function is defined" — a regular function's `this` is determined entirely by *how it's called*, not where it's defined; only arrow functions inherit `this` lexically from their surrounding scope at definition time.
- **DO:** Prefer arrow-function class fields (`handleClick = () => { ... }`) over binding in a constructor (`this.handleClick = this.handleClick.bind(this)`) when a class method genuinely needs to be passed around detached from its instance as a callback — the class-field form is shorter and keeps the binding declaration next to the method itself instead of in a separate constructor block that's easy to forget to update when a new method is added.
- **DON'T:** Use `.bind()`, `.call()`, or `.apply()` on an arrow function expecting it to change what `this` resolves to inside it — arrow functions ignore the `this` argument passed to all three of these; their `this` is permanently fixed to whatever it lexically captured at creation time, and none of the explicit-binding mechanisms can override that.

### Logical Assignment Operators in Practice

- **DO:** Use `??=` (logical nullish assignment) to assign a value only when a variable/property is currently `null` or `undefined`, `||=` (logical OR assignment) to assign only when it's currently any falsy value, and `&&=` (logical AND assignment) to reassign a variable only when it's currently truthy — each is a direct, more concise replacement for the equivalent `if` statement, and using the right one (rather than defaulting to `||=` out of habit) avoids the exact `0`/`""`/`false` footgun covered earlier in this document's discussion of `??` vs `||`.
```js
// Equivalent, longer forms shown for clarity:
config.timeout ??= 5000;              // if (config.timeout === undefined || config.timeout === null) config.timeout = 5000;
options.debug ||= false;              // if (!options.debug) options.debug = false;
cache.entry &&= transform(cache.entry); // if (cache.entry) cache.entry = transform(cache.entry);
```
- **DO:** Use `&&=` specifically for the "only touch this if it's already set to something truthy" pattern — a common use is conditionally transforming a value that might legitimately be absent, without introducing a separate `if` block purely to guard the reassignment.
- **DON'T:** Reach for `||=` on a property that can legitimately and validly hold `0`, `""`, or `false` as a real, intended value — exactly as with the plain `||` operator, `||=` will overwrite a deliberately-set falsy value with the right-hand side, which is very often not the intended behavior. Use `??=` in that case instead, unless overwriting every falsy value really is the desired behavior.
- **DO:** Recognize that all three logical assignment operators short-circuit — the right-hand side expression is only evaluated at all when the assignment would actually happen — which matters when the right-hand side has a side effect or is an expensive computation; `cache.value ??= computeExpensiveDefault()` only calls `computeExpensiveDefault()` when `cache.value` genuinely needs it, not on every single execution of that line.

### Dates, Time & Internationalization

- **DON'T:** Use the built-in `Date` object's mutable setter methods (`setDate`, `setMonth`, `setHours`) to derive a new date from an existing one without first cloning it — every `Date` instance is mutable, and mutating one in place is a classic source of "why did this unrelated date change" bugs when the same `Date` object is referenced elsewhere.
```js
// Bad — mutates the shared `order.createdAt` Date in place
function addDays(date, days) {
  date.setDate(date.getDate() + days);
  return date;
}

// Good — returns a new Date, leaves the input untouched
function addDays(date, days) {
  const result = new Date(date);
  result.setDate(result.getDate() + days);
  return result;
}
```
- **DO:** Use a well-maintained date library (`date-fns` for tree-shakeable, immutable pure functions, `Luxon` or `Day.js` for a fuller object-oriented API) for anything beyond trivial date arithmetic — timezone conversions, business-day calculations, human-friendly relative formatting, and recurring-date logic are all full of edge cases (DST transitions, month-length differences, leap years, leap seconds) that the native `Date` API handles poorly or not at all.
- **DON'T:** Store or compare dates as locale-formatted strings (`"01/02/2024"`) internally — this format is ambiguous (is that January 2nd or February 1st?) and doesn't sort correctly as a string. Store and pass dates as ISO 8601 strings (`"2024-01-02T00:00:00.000Z"`, which do sort correctly lexicographically) or as native `Date`/timestamp values, and only format to a locale-specific display string at the final point of rendering to a user.
- **DO:** Be explicit and deliberate about timezones — know whether a given `Date`/timestamp represents a moment in UTC or a specific local time, and convert only at display boundaries. A very common bug class is silently treating a UTC timestamp as if it were already in the server's or user's local time (or vice versa), which shifts displayed times by the timezone offset, sometimes shifting the displayed *date* itself across a day boundary.
- **DON'T:** Assume the server's local timezone matches any particular user's timezone, or that it will stay the same across deployments (many cloud platforms default containers to UTC, which is good practice, but code that assumes otherwise breaks when redeployed to a differently-configured environment). Store timestamps in UTC, and perform timezone conversion for display using the recipient's actual timezone (from user profile settings or the `Intl` API's detected timezone), not the server's.
- **DO:** Use the built-in `Intl` API (`Intl.DateTimeFormat`, `Intl.NumberFormat`, `Intl.RelativeTimeFormat`, `Intl.ListFormat`) for locale-aware formatting of dates, numbers, and currency instead of hand-rolling locale-specific formatting logic or pulling in a heavy formatting library for something the platform already provides natively and correctly.
```js
// Good — correct, locale-aware currency formatting with no extra dependency
const formatter = new Intl.NumberFormat("de-DE", { style: "currency", currency: "EUR" });
formatter.format(1234.5); // "1.234,50 €"
```
- **DON'T:** Compare two `Date` objects with `===`/`==` expecting value equality — `Date` objects are compared by reference, so two `Date` instances representing the identical moment in time are never `===` to each other. Compare their numeric value instead (`date1.getTime() === date2.getTime()`, or simply `+date1 === +date2`).
- **DO:** Validate that a `Date` constructed from a parsed/user-supplied string is actually valid before using it — an invalid date string passed to `new Date(...)` doesn't throw; it silently produces a `Date` object whose `.getTime()` is `NaN` ("Invalid Date"), which then propagates through further calculations as `NaN` until it surfaces as a confusing downstream failure. Check with `Number.isNaN(date.getTime())` (or a validation library) right after parsing.
```js
// Bad — no validation; a malformed date string silently becomes "Invalid Date"
const eventDate = new Date(req.body.date);
scheduleEvent(eventDate); // fails mysteriously downstream

// Good
const eventDate = new Date(req.body.date);
if (Number.isNaN(eventDate.getTime())) {
  throw new ValidationError(["date must be a valid ISO 8601 date string"]);
}
```
- **DO:** Store durations/intervals as an explicit unit (milliseconds is the JS-native convention for `setTimeout`/`Date` arithmetic) with the unit made obvious in the variable name (`timeoutMs`, `cacheTtlSeconds`) — an unnamed numeric duration is a frequent source of "was that seconds or milliseconds" bugs, especially when passing a value between a library that expects seconds and one that expects milliseconds.

### Working with Regular Expressions

- **DON'T:** Reuse a regular expression literal created with the `g` (global) or `y` (sticky) flag across multiple, separate `.test()`/`.exec()` calls without accounting for its stateful `lastIndex` property — a global/sticky regex remembers where its last match ended, so calling `.test()` repeatedly on different strings with the same regex instance can skip matches or return inconsistent results depending on call order.
```js
// Bad — the regex's lastIndex persists between calls, causing alternating true/false results
const hasDigit = /\d/g;
console.log(hasDigit.test("abc123")); // true, lastIndex now past the match
console.log(hasDigit.test("abc123")); // false! — resumes searching from the old lastIndex

// Good — reset lastIndex, or avoid the global flag when you only need a single test
const hasDigit = /\d/;
console.log(hasDigit.test("abc123")); // true
console.log(hasDigit.test("abc123")); // true, no shared state
```
- **DO:** Use named capture groups (`(?<year>\d{4})`) instead of positional capture groups when a regex has more than one or two captures, so the extraction code reads by meaning instead of by fragile numeric index that breaks silently if the pattern is later reordered.
```js
// Bad — fragile positional indices; reordering the pattern silently breaks this
const match = /(\d{4})-(\d{2})-(\d{2})/.exec(dateString);
const year = match[1];

// Good — self-documenting, resilient to pattern reordering
const match = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/.exec(dateString);
const { year, month, day } = match.groups;
```
- **DON'T:** Write a regex to parse or validate a well-known structured format (a full email address per the RFC, a URL, HTML) when a purpose-built parser exists — regex-based HTML/URL parsing in particular is a long-running source of subtle correctness bugs, since these formats have context-sensitive grammars that regular expressions fundamentally can't express correctly in the general case. Use `URL`/`URLSearchParams` for URLs, a proper HTML parser for HTML, and accept that "fully RFC-compliant email regex validation" is a well-known rabbit hole — validate email format loosely with a simple regex or a library and rely on a real deliverability check (e.g., a confirmation email) for the guarantee that actually matters.
- **DO:** Escape any user-supplied string before interpolating it into a `RegExp` constructed dynamically from a string (`new RegExp(userInput)`), since unescaped regex metacharacters in user input can change the pattern's meaning entirely — and combined with certain patterns, can also open the door to the ReDoS issues covered in the Security section.
```js
// Bad — a userInput of "a.*" or similar changes the search's actual meaning
const pattern = new RegExp(userInput);

// Good — escape regex metacharacters first
function escapeRegExp(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}
const pattern = new RegExp(escapeRegExp(userInput));
```
- **DO:** Use the `u` (Unicode) flag on regular expressions that process text which might contain characters outside the Basic Multilingual Plane (emoji, many non-Latin scripts) — without it, `.` and character classes operate on individual UTF-16 code units rather than full Unicode code points, which can incorrectly split a single visible character (like many emoji) into two separate, mismatched units.
- **DON'T:** Build a complex, unreadable single regex to validate an entire structured input (e.g., a full password-complexity rule set: length, uppercase, digit, symbol, all in one pattern) when several small, individually-testable checks would be clearer and produce better error messages for the specific rule that failed.
```js
// Bad — one opaque regex; on failure, you can't tell the user which rule they missed
const isValidPassword = /^(?=.*[A-Z])(?=.*[0-9])(?=.*[^A-Za-z0-9]).{8,}$/.test(password);

// Good — individually testable, individually reportable
const rules = [
  { test: (pw) => pw.length >= 8, message: "at least 8 characters" },
  { test: (pw) => /[A-Z]/.test(pw), message: "an uppercase letter" },
  { test: (pw) => /[0-9]/.test(pw), message: "a digit" },
];
const failedRules = rules.filter((rule) => !rule.test(password));
```

### Tagged Template Literals

- **DO:** Reach for tagged template literals when building a small domain-specific mini-language embedded in JS/TS — the tag function receives the literal string parts and the interpolated values *separately*, which is exactly the hook that lets a library safely process each independently (escaping/parameterizing the interpolated values while leaving the literal template structure untouched), which is what powers safe SQL-tag libraries and CSS-in-JS libraries alike.
```js
function sql(strings, ...values) {
  // strings and values arrive separately — never concatenated into one plain string —
  // which is exactly what lets a real implementation parameterize each value safely
  // instead of splicing it directly into the query text.
  const text = strings.reduce((acc, str, i) => acc + `$${i}` + str);
  return { text, values };
}
const userId = "abc123";
const query = sql`SELECT * FROM users WHERE id = ${userId}`;
// { text: "SELECT * FROM users WHERE id = $1", values: ["abc123"] } — safely parameterized
```
- **DO:** Prefer a real, well-tested tagged-template query library (`sql` from `postgres`/`slonik`, or your ORM's own tagged-template raw-query helper) over writing your own from scratch for anything production-facing — the safety of this pattern depends entirely on the tag function correctly parameterizing every interpolated value, and that's exactly the kind of security-critical code better delegated to a maintained library than reimplemented ad hoc.
- **DON'T:** Assume a tagged template automatically makes an operation "safe" just because it looks like it separates code from data — if the tag function itself just concatenates the strings and values back into one string internally (rather than genuinely keeping them separate for parameterized execution), the syntax provides no actual protection at all; the safety comes from the tag function's *implementation*, not from the template-literal syntax on its own.
- **DO:** Use tagged templates for other genuinely useful embedded-syntax cases beyond SQL — a `gql` tag for GraphQL query strings that enables editor syntax highlighting and static validation tooling, or a `styled` tag for CSS-in-JS — recognizing the common thread: the tag function gets to parse/validate/transform a literal template at build or runtime in ways a plain interpolated string never allows.

### Hoisting Regular Expressions Out of Hot Paths

- **DO:** Define a regular expression that's reused across many calls (inside a frequently-called function, or across many loop iterations) as a module-level or outer-scope constant, rather than as a literal re-created inside the function/loop body on every single invocation — per the JavaScript specification, evaluating a regex literal creates a new `RegExp` object each time, so a regex literal placed inside a hot function is allocated fresh on every call for no benefit when the pattern itself never changes.
```js
// Less efficient — a new RegExp object is allocated on every single call
function isValidSlug(str) {
  return /^[a-z0-9-]+$/.test(str);
}

// Better — compiled once, reused on every call
const SLUG_PATTERN = /^[a-z0-9-]+$/;
function isValidSlug(str) {
  return SLUG_PATTERN.test(str);
}
```
- **DON'T:** Hoist a regex with the `g` (global) or `y` (sticky) flag into a shared module-level constant without accounting for its statefulness (covered earlier in this section) — a shared, stateful regex reused via `.test()`/`.exec()` across multiple unrelated calls is exactly the `lastIndex` bug described above; hoisting a non-stateless regex for reuse needs either resetting `lastIndex` before each use or avoiding the global/sticky flags when only a single match/test is needed per call.
- **DO:** Recognize this as a minor, generally low-priority optimization relative to the algorithmic and I/O-related concerns covered throughout the Performance section — worth applying as a matter of habit for regexes in genuinely hot paths, but not something to go out of your way to chase in code where the actual bottleneck (per the "measure before optimizing" principle) lies elsewhere entirely.

### JSON & Data Serialization

- **DON'T:** Use `JSON.parse(JSON.stringify(obj))` as a general-purpose deep-clone utility. It silently drops properties with `undefined` values and functions, converts `Date` objects into plain ISO strings (losing the `Date` type), turns `NaN`/`Infinity` into `null`, throws on circular references, and loses `Map`/`Set`/`BigInt` entirely. Use `structuredClone(obj)` for a real, broadly-correct deep clone (available natively in modern Node.js and browsers) instead.
- **DO:** Be deliberate about how non-JSON-native types round-trip through `JSON.stringify`/`JSON.parse` — `Date` becomes a string (and needs to be explicitly re-parsed back into a `Date` on the way in), `undefined` object properties are omitted entirely, `Map`/`Set` serialize to `{}`/`[]`-shaped nonsense unless explicitly converted first, and `BigInt` throws a `TypeError` when stringified at all unless you supply a custom `replacer`.
```js
// BigInt throws by default when JSON.stringify'd
JSON.stringify({ id: 10n }); // TypeError: Do not know how to serialize a BigInt

// Handle it explicitly with a replacer
JSON.stringify({ id: 10n }, (_key, value) =>
  typeof value === "bigint" ? value.toString() : value
);
```
- **DO:** Use a `reviver` function with `JSON.parse` (or a schema library's `.transform()`) to convert known date-shaped string fields back into real `Date` objects immediately after parsing, at the boundary, rather than passing raw date strings deep into business logic and converting them ad hoc wherever they happen to be used.
- **DON'T:** Assume `JSON.stringify`'s key order is guaranteed to match the object's declared property order for all cases — while modern engines do preserve insertion order for string keys (per the spec, integer-like keys are reordered numerically first), relying on any specific serialized key order for anything beyond human readability (e.g., for a cache key or a signature) is fragile; sort keys explicitly if order-independence matters for equality/hashing.
- **DO:** Set a reasonable request body size limit on any endpoint that parses JSON from the network (`express.json({ limit: "1mb" })` or equivalent) — without an explicit limit, most frameworks either use a small unhelpful default or none at all, and an unbounded JSON body is both a memory-exhaustion denial-of-service vector and, for very large payloads, a source of that synchronous-parsing event-loop-blocking issue covered in the Node.js Server Patterns section.
- **DON'T:** Trust that `JSON.parse`'s output matches an expected TypeScript type just because you asserted it with `as`. As covered in the TypeScript Typing section, `JSON.parse` returns `any` at the type level with zero runtime guarantee behind it — validate the parsed shape with a real schema (Zod, etc.) at the boundary rather than asserting past the type checker.
- **DO:** Handle circular references explicitly (with a custom replacer that detects and short-circuits cycles, or a library like `flatted`/`circular-json`) if a data structure genuinely can contain them — the default `JSON.stringify` throws a `TypeError: Converting circular structure to JSON` on any circular reference, which is often the right failure mode, but if you need to serialize such a structure anyway (e.g., for debugging output), handle it deliberately rather than crashing.

## TypeScript Typing

### Incrementally Migrating JavaScript to TypeScript

- **DO:** Start a JS-to-TS migration by enabling `allowJs` and `checkJs` in `tsconfig.json` and adding `// @ts-check` to individual `.js` files (or project-wide via `checkJs`) before converting any files to `.ts` — this gets real type-checking value from the existing JSDoc annotations (or the types TypeScript can infer without any annotations at all) immediately, with zero file renames and zero risk of breaking the build.
```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "strict": false,      // start permissive; tighten incrementally, see below
    "outDir": "./dist"
  }
}
```
- **DO:** Convert files to `.ts` gradually, starting with leaf modules that have few or no internal dependencies (pure utility functions, type/constant definition files) before working up to files with many internal dependencies — converting a leaf module first means you're not blocked waiting on types from files you haven't converted yet, and each conversion is a small, independently reviewable, low-risk step rather than one enormous rewrite PR.
- **DON'T:** Attempt to enable `"strict": true` project-wide on day one of a migration for a large existing JavaScript codebase — this will surface hundreds or thousands of errors at once, most of them in files nobody's actively working on, producing a wall of noise that's demoralizing and hard to prioritize. Start with `strict: false` (or with only `noImplicitAny` enabled) and tighten the flags incrementally, ideally file-by-file via TypeScript's per-file strictness comments or a phased flag rollout.
- **DO:** Use `// @ts-nocheck` at the top of a specific, still-unconverted file to explicitly opt it out of the project's `checkJs`/`strict` checking during a migration, when that file isn't ready to be checked yet — this is a legitimate, scoped, migration-specific use of a suppression comment (unlike the general-purpose suppression-comment misuse warned against elsewhere in this document), precisely because it's temporary, visible, and removed file-by-file as the migration proceeds.
- **DON'T:** Let a migration stall indefinitely with a large, growing set of `@ts-nocheck`/`@ts-ignore`-suppressed files that nobody ever circles back to — track migration progress explicitly (a simple checklist, or a lint rule that flags any file using a suppression comment) so the eventual goal of full type coverage doesn't quietly become permanent partial coverage with no plan to finish.
- **DO:** Prioritize converting or type-checking the files most central to the application's actual bug history and most frequently touched by the team, rather than migrating in a purely mechanical order (e.g., alphabetical, or smallest-file-first) — the whole point of the migration is catching real bugs before they ship, so the files where that payoff is highest deserve to be converted first.

### `any` vs `unknown` vs Proper Types

- **DON'T:** Reach for `any` as a shortcut to make a type error go away. `any` disables type checking for that value and everything derived from it, silently propagating the loss of safety through the rest of the codebase — it is by far the most common way AI-generated TypeScript quietly stops being TypeScript.
```ts
// Bad
function processPayload(payload: any) {
  return payload.user.id; // no error even if payload has no `user`
}

// Good
interface Payload {
  user: { id: string };
}
function processPayload(payload: Payload) {
  return payload.user.id;
}
```
- **DO:** Use `unknown` instead of `any` for values whose type genuinely isn't known yet (e.g., JSON parsed from an API, a `catch` block's error). `unknown` forces a narrowing check (`typeof`, `instanceof`, a type guard, or a validation library) before the value can be used, which preserves safety while still allowing the value to flow through the system.
```ts
// Bad
function handleError(err: any) {
  console.log(err.message); // assumes shape, may throw at runtime
}

// Good
function handleError(err: unknown) {
  if (err instanceof Error) {
    console.log(err.message);
  } else {
    console.log(String(err));
  }
}
```
- **DO:** Enable `noImplicitAny` (bundled into `strict`) in `tsconfig.json` so an un-annotated parameter or variable that TypeScript can't infer becomes a compile error instead of a silent `any`. Implicit `any` is the same hazard as explicit `any`, just invisible in a code review that doesn't check the compiler flags.
- **DON'T:** Use `as any` or a double assertion (`as unknown as X`) to force a type past the compiler when it's complaining about a real mismatch. If the types genuinely don't line up, that's usually the compiler catching a real bug — fix the underlying type or data shape rather than silencing the checker.
- **DO:** Use type assertions (`as X`) only when you have information the compiler cannot infer (e.g., after a runtime check the compiler doesn't narrow on, or when working with a DOM API that returns a wider type than you know it will be), and keep the assertion as narrow and local as possible.
- **DON'T:** Sprinkle non-null assertions (`value!`) to silence "possibly undefined" errors without actually verifying the value is present. Each `!` is a promise to the compiler that can be wrong at runtime, and wrong `!` assertions are indistinguishable from real bugs until they crash in production.
```ts
// Bad
function getFirstUser(users: User[]) {
  return users[0]!.name; // throws at runtime if users is empty
}

// Good
function getFirstUser(users: User[]) {
  const first = users[0];
  if (!first) throw new Error("users array is empty");
  return first.name;
}
```
- **DO:** Prefer precise union types and literal types over broad primitives when the set of valid values is known and small. `type Status = "pending" | "active" | "archived"` catches typos and invalid values at compile time; `status: string` catches nothing.
```ts
// Bad
function setStatus(status: string) { /* ... */ }
setStatus("actve"); // typo compiles fine, fails at runtime

// Good
type Status = "pending" | "active" | "archived";
function setStatus(status: Status) { /* ... */ }
setStatus("actve"); // compile error
```

### Avoiding Overly-Wide Built-in Types

- **DON'T:** Type a parameter as the built-in `Function` type when a specific function signature is known — `Function` accepts literally any callable value with any number of arguments and any return type, which means TypeScript can't check that the function is actually called correctly or that its return value is used correctly; it's barely more useful than `any` for anything callable.
```ts
// Bad — Function tells the compiler almost nothing
function runCallback(cb: Function) {
  cb(1, 2, 3); // no arity or type checking at all
}

// Good — the actual expected signature is specified and checked
function runCallback(cb: (a: number, b: number) => void) {
  cb(1, 2); // compiler checks arity and argument types against the real signature
}
```
- **DON'T:** Type a value as the built-in `object` type expecting it to mean "a plain object with properties" — `object` in TypeScript means "any non-primitive value," which includes arrays, functions, class instances, and `Date` objects, not specifically plain key-value objects. If the intent is "some object with unknown properties," use `Record<string, unknown>`; if the intent is "any non-primitive value specifically" (a genuinely rare, advanced need), `object` is technically correct but should be a deliberate choice, not a guess at what "an object" means.
- **DON'T:** Type a value as `{}` expecting it to mean "an empty object" — the empty object type `{}` in TypeScript actually means "any value except `null` and `undefined`," including numbers, strings, functions, and populated objects, which is almost never what someone reaching for `{}` actually intends. Use `Record<string, never>` for a genuinely empty-object type, or `unknown`/a real interface for "any value"/"an object with these fields."
```ts
// Surprising: {} accepts almost anything, not just "empty objects"
function process(input: {}) { /* ... */ }
process(42);          // compiles — {} does not mean "empty object"
process("hello");     // compiles
process({ a: 1 });    // also compiles

// What was probably intended
function process(input: Record<string, unknown>) { /* ... */ }
```
- **DO:** Reach for `unknown` (a genuinely unknown value requiring narrowing before use), a specific interface/type (a known, specific shape), or `Record<string, unknown>` (an object with unknown property values but known-to-be-string keys) depending on which of those three distinct meanings is actually intended — the built-in wide types (`Function`, `object`, `{}`) are easy to reach for by pattern-matching on their short, generic-sounding names, but each one means something narrower and more surprising than its name suggests.

### Interfaces, Types & Object Shapes

- **DO:** Use `interface` for object shapes that represent entities, especially public API surfaces meant to be extended or implemented, and use `type` for unions, tuples, mapped types, and anything that isn't a plain extendable object shape. Both work for simple object shapes; the convention exists to signal intent — interfaces are extensible contracts, type aliases are exact descriptions.
- **DON'T:** Treat `interface` and `type` as interchangeable without a reason for the choice, especially mid-file, since it creates unnecessary inconsistency. Pick a team convention (many codebases: `interface` for object shapes, `type` for everything else) and apply it uniformly; let the linter (`@typescript-eslint/consistent-type-definitions`) enforce it.
- **DO:** Use `readonly` on interface/type properties and `ReadonlyArray<T>`/`readonly T[]` for arrays that should not be mutated by the consumer. This documents and enforces immutability contracts at compile time instead of relying on a code comment or convention nobody checks.
```ts
interface Order {
  readonly id: string;
  readonly items: readonly OrderItem[];
}
```
- **DON'T:** Model a value that can be one of several distinct shapes as one big interface with a pile of optional fields — this "one interface to rule them all" pattern loses the connection between related fields and allows invalid combinations to type-check. Use a discriminated union instead.
```ts
// Bad — nothing stops payload from having both card and bankAccount, or neither
interface Payment {
  method: string;
  cardNumber?: string;
  bankAccount?: string;
}

// Good — discriminated union enforces valid combinations
type Payment =
  | { method: "card"; cardNumber: string }
  | { method: "bank"; bankAccount: string };

function charge(payment: Payment) {
  if (payment.method === "card") {
    // payment.cardNumber is known to exist here
  }
}
```
- **DO:** Use discriminated unions (a shared literal "tag" field) plus `switch` with exhaustiveness checking for state machines, API responses, and any "one of several known shapes" data. Pair it with a `never`-typed default case so adding a new variant without handling it becomes a compile error, not a runtime gap.
```ts
type Result<T> =
  | { status: "success"; data: T }
  | { status: "error"; error: string };

function handle<T>(result: Result<T>) {
  switch (result.status) {
    case "success": return result.data;
    case "error": throw new Error(result.error);
    default: {
      const _exhaustive: never = result;
      return _exhaustive;
    }
  }
}
```
- **DON'T:** Model optional fields as `field: Type | undefined` when what you mean is `field?: Type` — they behave the same for reads but differ under `exactOptionalPropertyTypes` and, more importantly, differ in intent: an optional property can be omitted; `Type | undefined` typically implies it must be present but may hold `undefined`.
- **DO:** Extract shared, repeated inline object types into a named `interface`/`type` once they're used in more than one place. A duplicated inline shape that drifts between call sites is a correctness bug waiting to happen, and named types are self-documenting.
- **DON'T:** Reach immediately for `Record<string, any>` to describe "some object with unknown extra properties." If the object has some known required properties, type those explicitly and add an index signature for the rest with a real type (`Record<string, unknown>` at minimum), and consider `Partial<Record<KnownKey, Type>>` or a proper union instead.

### Generics

- **DO:** Use generics to preserve the relationship between input and output types, instead of widening to `any`/`unknown` and re-asserting the type at the call site. A generic function tells the compiler (and the reader) that whatever type goes in with `T`, the same `T` comes out — that relationship is exactly what generics exist to express.
```ts
// Bad — loses the input type
function firstItem(arr: any[]): any {
  return arr[0];
}

// Good — return type is inferred from the input array's element type
function firstItem<T>(arr: T[]): T | undefined {
  return arr[0];
}
```
- **DON'T:** Add a generic parameter that's only ever used once and adds no real constraint — this is over-engineering that makes the signature harder to read for no type-safety benefit. If a generic parameter doesn't appear in at least two places (typically an input and an output, or two related inputs), a concrete type is usually clearer.
- **DO:** Constrain generic parameters with `extends` when the function needs to access specific properties or call specific methods on the generic type, rather than casting inside the function body to work around an unconstrained generic.
```ts
// Bad
function getId<T>(entity: T) {
  return (entity as any).id; // unsafe, defeats the point of generics
}

// Good
function getId<T extends { id: string }>(entity: T) {
  return entity.id;
}
```
- **DO:** Give generic type parameters descriptive names once there is more than one in scope (`TInput`, `TOutput`, `TKey`, `TValue`) rather than leaving everything as bare `T`, `U`, `V` once the relationships between them aren't obvious from a single letter.
- **DON'T:** Default every generic parameter to `any` "to make it optional" — use a genuinely sensible default type, or make callers specify it, or infer it from usage. A generic that silently defaults to `any` reintroduces the exact type hole generics were meant to close.

### Strictness & Compiler Configuration

- **DO:** Enable `"strict": true` in `tsconfig.json` for any new TypeScript project. It bundles `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, and `alwaysStrict` — the combination that catches the overwhelming majority of type-related runtime bugs before they ship.
- **DON'T:** Disable `strict` (or individual strict flags) project-wide just to make a legacy migration or a stubborn third-party type error go away. Suppress the specific offending line with a narrowly-scoped `// @ts-expect-error` (with a comment explaining why) instead of weakening the compiler for the entire codebase.
```ts
// Bad — silences the check everywhere in tsconfig.json
{ "compilerOptions": { "strict": false } }

// Good — silences one known, explained line
// @ts-expect-error — legacy-sdk's types are wrong here; tracked in TICKET-123
const result = legacySdk.call(arg);
```
- **DO:** Prefer `// @ts-expect-error` over `// @ts-ignore` when suppressing a specific compiler error. `@ts-expect-error` fails the build if the line stops erroring (e.g., after an upstream type fix), which prevents suppression comments from silently becoming stale and pointless; `@ts-ignore` has no such safety net.
- **DO:** Turn on `noUncheckedIndexedAccess` for projects that index into arrays/objects by dynamic key or index. Without it, `arr[i]` is typed as `T`, not `T | undefined`, even though out-of-bounds access is trivially possible — this single flag catches a large class of "cannot read property of undefined" runtime errors at compile time.
- **DO:** Turn on `noUnusedLocals` and `noUnusedParameters` (or delegate the same check to ESLint) to catch dead variables and stale parameters left behind after refactors — exactly the kind of leftover an AI assistant's incremental edits tend to accumulate.
- **DON'T:** Target an old `lib`/`target` in `tsconfig.json` (e.g., `es5`) for a modern Node.js backend project without a real reason — this needlessly disables newer syntax and can pull in unnecessary polyfills. Match `target`/`lib` to the actual Node.js version(s) the code runs on.
- **DO:** Use project references (`tsconfig.json` `references`) or a proper monorepo TS setup for multi-package repositories instead of one giant `tsconfig.json` with relative path hacks (`../other-package/src/index.ts`) reaching across package boundaries. Path hacks break the package boundary that lets each package be built, versioned, and typed independently.

### Schema-Derived Discriminated Unions

- **DO:** Build discriminated-union runtime validation directly with a schema library's dedicated support for it (Zod's `z.discriminatedUnion()`) rather than a plain `z.union()` of several object schemas, when the union members share an actual discriminant field — `discriminatedUnion` gives Zod a fast, unambiguous way to pick the correct branch to validate against based on the discriminant alone, and it produces clearer validation error messages than a plain union, which has to try each member schema in turn and report a confusing combined failure when none match.
```ts
const PaymentSchema = z.discriminatedUnion("method", [
  z.object({ method: z.literal("card"), cardNumber: z.string() }),
  z.object({ method: z.literal("bank"), accountNumber: z.string() }),
]);

type Payment = z.infer<typeof PaymentSchema>; // exactly matches the hand-written union shown earlier
```
- **DO:** Derive the TypeScript type from the Zod (or equivalent) schema via `z.infer<typeof Schema>` rather than hand-writing a parallel discriminated-union type — this keeps the compile-time type and the runtime validator permanently in sync by construction, which directly addresses the general "types and runtime validation drift apart" risk raised in the TypeScript Typing section's discussion of schema validation.
- **DON'T:** Reach for a plain `z.union()` out of habit when the members actually share a clean discriminant field — beyond the error-message and performance benefits, `discriminatedUnion` also fails fast and clearly during schema *definition* if the discriminant field itself is missing or misconfigured on one of the branches, catching a schema-authoring mistake immediately rather than only at validation time against real data.
- **DO:** Keep the discriminant field's literal values consistent between the runtime schema and any hand-written parts of the codebase that also branch on the same field (a `switch` statement elsewhere handling the same union) — since both are ultimately meant to represent the exact same set of valid states, a drift between them (a value accepted by the schema but not handled by the switch, or vice versa) reintroduces exactly the kind of gap discriminated unions and exhaustiveness checking are meant to close.

### Type Narrowing & Guards

- **DO:** Use `typeof`, `instanceof`, `in`, discriminated-union tag checks, and user-defined type guards (`function isX(v): v is X`) to narrow `unknown`/union types before use, rather than asserting the type with `as`. Narrowing is checked by the compiler against the actual runtime logic; an assertion is not checked at all.
```ts
function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value
  );
}

function handle(value: unknown) {
  if (isUser(value)) {
    console.log(value.name); // safely narrowed to User
  }
}
```
- **DON'T:** Write a user-defined type guard (`v is X`) whose body doesn't actually verify enough of the shape to justify the claim — a type guard is a promise to the compiler, and an under-checked one (e.g., checking only that a value is an object without checking any properties) reintroduces the same runtime risk `unknown` was meant to prevent.
- **DO:** Prefer runtime schema validation (Zod, Valibot, io-ts, ArkType, or the built-in `JSON Schema` + ajv) at any trust boundary — parsing request bodies, environment variables, third-party API responses, config files — and derive the static TypeScript type from the schema. TypeScript types are erased at compile time and provide zero runtime protection; only actual runtime validation can catch malformed data from outside the program.
```ts
import { z } from "zod";

const UserSchema = z.object({
  id: z.string(),
  email: z.string().email(),
  age: z.number().int().nonnegative(),
});
type User = z.infer<typeof UserSchema>;

function parseUser(payload: unknown): User {
  return UserSchema.parse(payload); // throws on invalid data
}
```
- **DON'T:** Assert the type of `JSON.parse()`'s result, `req.body`, or any other external/network input directly (`const user = JSON.parse(json) as User`). This is a compile-time-only claim with no runtime check behind it — a malformed payload will pass the type checker and then crash or behave incorrectly deep inside the application.

### Assertion Functions & Advanced Narrowing

- **DO:** Use TypeScript assertion function signatures (`function assertIsUser(v: unknown): asserts v is User`) for validation helpers meant to throw on failure and narrow the type in the surrounding scope on success, rather than a boolean-returning type guard the caller has to wrap in an `if` at every call site — an assertion function lets the compiler narrow the type for the rest of the enclosing block immediately after the call, with no `if` needed.
```ts
function assertIsUser(value: unknown): asserts value is User {
  if (typeof value !== "object" || value === null || !("id" in value)) {
    throw new TypeError("expected a User");
  }
}

function handle(value: unknown) {
  assertIsUser(value);
  console.log(value.id); // narrowed to User immediately after the assertion call, no `if` needed
}
```
- **DO:** Use the simpler `asserts value` form (without a type predicate) for a generic "assert this is truthy/defined" helper (`function assertDefined<T>(value: T | undefined): asserts value is T`), which is a useful, reusable narrowing helper for the common "I know this isn't undefined here, but the compiler can't prove it" situation, as a safer alternative to a non-null assertion (`!`).
```ts
function assertDefined<T>(value: T | undefined, message = "value was undefined"): asserts value is T {
  if (value === undefined) throw new Error(message);
}

const found = items.find((i) => i.id === targetId);
assertDefined(found, `item ${targetId} not found`);
found.name; // narrowed to the defined type, and the failure mode is a clear thrown error
```
- **DON'T:** Write an assertion function whose runtime check doesn't actually verify what its type predicate claims — exactly as with a regular type guard, an assertion function is a promise to the compiler about what happens after it returns without throwing, and an under-checked implementation reintroduces the exact runtime risk the assertion was meant to close off.
- **DO:** Combine `in` operator narrowing with discriminated unions for narrowing plain objects that don't have an explicit discriminant tag field but do have structurally distinguishing properties — `if ("cardNumber" in payment)` narrows `payment` to the union member(s) that actually declare a `cardNumber` property, which is useful when integrating with external data that wasn't designed with a clean discriminant field in the first place.

### Utility Types & Advanced Patterns

- **DO:** Reach for TypeScript's built-in utility types (`Partial`, `Required`, `Pick`, `Omit`, `Record`, `Readonly`, `ReturnType`, `Parameters`, `Awaited`, `Exclude`, `Extract`, `NonNullable`) to derive related types from a single source of truth instead of hand-writing near-duplicate interfaces that drift apart over time.
```ts
interface User {
  id: string;
  name: string;
  email: string;
  passwordHash: string;
}

// Bad — a second, hand-maintained interface that will drift
interface PublicUser {
  id: string;
  name: string;
  email: string;
}

// Good — derived, stays in sync automatically
type PublicUser = Omit<User, "passwordHash">;
```
- **DO:** Use `Awaited<ReturnType<typeof fn>>` (or, better, name the resolved type explicitly) when you need the resolved type of an async function's return value, rather than duplicating the type by hand.
- **DON'T:** Build deeply nested, clever mapped/conditional types purely to avoid writing two straightforward interfaces, when the "clever" version costs the next reader ten minutes to decode. Type-level metaprogramming is a tool for genuine cases (library authors building generic APIs), not a default style for application code.
- **DO:** Use template literal types for strings with known structure (route paths, CSS units, event names) when the added compile-time safety is worth the complexity — e.g., `type EventName = \`on${Capitalize<string>}\`` for a plugin system's event names.
- **DON'T:** Overuse conditional types and infer clauses to reimplement logic that a runtime function (with a corresponding, simpler static type) would express more clearly. If a type-level computation takes more lines than the equivalent runtime function, reconsider whether it needs to exist at the type level at all.

### Template Literal Types & Readonly Tuples in Practice

- **DO:** Use template literal types to give string-based identifiers real compile-time structure when the structure is meaningful — extracting route parameters from a path pattern, constraining a CSS-unit string, or validating an event-name naming convention — since this catches a whole category of "typo in a string that should have a fixed shape" bugs that a bare `string` type lets through silently.
```ts
type RouteParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof RouteParams<`/${Rest}`>]: string }
    : Path extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

type Params = RouteParams<"/users/:userId/orders/:orderId">;
// { userId: string; orderId: string } — derived automatically from the route string itself
```
- **DON'T:** Reach for a template-literal-type computation like the one above as a default for ordinary route handling — most real applications are better served by a router library's own runtime param-typing support (many modern routers already infer this for you) than by hand-rolling type-level string parsing; reserve hand-written template-literal-type machinery for cases where no existing tool already provides the guarantee you need.
- **DO:** Use readonly tuple types (`readonly [x: number, y: number]`) to give a fixed-length, fixed-shape array real per-position types and names, instead of `number[]`, when a value is genuinely always exactly that shape (a 2D coordinate pair, an RGB triple) — a tuple catches "wrong number of elements" and "wrong type at this specific position" errors that a plain array type cannot.
```ts
// Bad — nothing stops a 4-element array, or numbers in the wrong meaning at each position
function movePoint(point: number[], dx: number, dy: number): number[] {
  return [point[0] + dx, point[1] + dy];
}

// Good — the shape and meaning of each position is explicit and checked
function movePoint(point: readonly [x: number, y: number], dx: number, dy: number): [number, number] {
  return [point[0] + dx, point[1] + dy];
}
```
- **DO:** Use variadic tuple types (`[first: T, ...rest: U[]]`) when typing a function whose behavior genuinely depends on having at least one argument of a specific type followed by a variable number of others — this is a real, if fairly advanced, tool for expressing "non-empty array with a distinguished first element" precisely, which a plain `T[]` type can't express (a plain array type can't rule out the empty-array case at compile time).

### Enums vs. Union Literals vs. `as const`

- **DON'T:** Default to TypeScript's `enum` without considering the alternatives — numeric enums compile to reverse-mapped objects with runtime footprint, and enums don't structurally match plain string/number literals from JSON or external systems the way a union type does. This causes friction at every API boundary that isn't TypeScript-aware.
- **DO:** Prefer union types of string literals, or an object literal marked `as const`, for most "fixed set of values" use cases — they have zero runtime cost when used as pure types, and match plain strings coming from JSON, databases, or query parameters without a conversion step.
```ts
// Often preferred over `enum Status { Pending, Active, Archived }`
const Status = {
  Pending: "pending",
  Active: "active",
  Archived: "archived",
} as const;
type Status = (typeof Status)[keyof typeof Status];
```
- **DO:** Use `as const` on literal arrays and objects to get the narrowest, most precise inferred type (literal types and readonly tuples) instead of the widened type TypeScript infers by default. This is especially useful for defining a fixed set of route names, permission strings, or configuration keys that should be checked against a literal union.
- **DON'T:** Use numeric enums for values that cross a serialization boundary (stored in a database, sent over an API, written to a log) unless you've deliberately pinned the numeric values and documented them — reordering enum members silently renumbers every value after the reordered one, which corrupts any already-persisted data that was mapped by number.
- **DO:** If `enum` is used (e.g., team convention, or genuine need for the reverse-mapping/namespacing it provides), prefer `const enum`-free string enums (`enum Status { Pending = "pending" }`) over numeric ones for readability in logs and debuggers, and be aware `const enum` requires special compiler support (isolatedModules-incompatible) that breaks under some bundlers/transpilers — avoid it unless you've verified your toolchain supports it.

### Branded Types, Function Overloads & Nominal Typing

- **DO:** Use branded (nominal) types when two values share the same underlying primitive type but represent semantically distinct concepts that should never be interchangeable — a `UserId` and an `OrderId` are both structurally just `string`, but passing one where the other is expected is a real bug that structural typing alone won't catch.
```ts
type UserId = string & { readonly __brand: "UserId" };
type OrderId = string & { readonly __brand: "OrderId" };

function toUserId(id: string): UserId {
  return id as UserId; // the one deliberate, boundary-only assertion
}

function getUser(id: UserId) { /* ... */ }

const orderId = "ord_123" as OrderId;
getUser(orderId); // compile error — OrderId is not assignable to UserId, even though both are strings
```
- **DON'T:** Reach for branded types for every primitive in the codebase reflexively — the ceremony (a brand type plus a constructor function at every boundary) is worth it specifically for identifiers and values where mixing them up is a plausible, costly bug (IDs across different entities, currency amounts in different units); it's overkill for values with no realistic confusion risk.
- **DO:** Use function overload signatures when a function's return type genuinely varies based on the *shape* of its input in a way a single generic signature can't cleanly express, and keep the overloads' implementation signature (the one actual function body) compatible with all of the declared overload signatures.
```ts
function parseConfig(input: string): Config;
function parseConfig(input: Buffer): Config;
function parseConfig(input: string | Buffer): Config {
  const text = typeof input === "string" ? input : input.toString("utf8");
  return JSON.parse(text);
}
```
- **DON'T:** Reach for overloads when a single well-designed generic function signature, or a union parameter type, already expresses the same contract more simply — overloads are a genuinely useful escape hatch for cases plain generics can't express, not a default style, and each additional overload signature is another thing a reader has to check when trying to understand what a call will resolve to.
- **DO:** Order function overload signatures from most-specific to least-specific — TypeScript resolves a call against the first overload signature that matches, so a broader signature listed before a narrower one can silently "shadow" the narrower one and make it unreachable at every call site.

### Typing Event Emitters & Callback-Based APIs

- **DO:** Type a Node `EventEmitter` subclass's events explicitly using a generic event-map interface (or a library like `tseep`/`emittery` with built-in typed events, or Node's own newer typed-emitter support patterns) so that `.on()`/`.emit()` calls are checked against the actual set of valid event names and their corresponding payload types, instead of every event name and payload being effectively `any`.
```ts
import { EventEmitter } from "node:events";

interface OrderEvents {
  created: [order: Order];
  shipped: [order: Order, trackingNumber: string];
  cancelled: [orderId: string, reason: string];
}

class OrderEmitter extends EventEmitter {
  override on<K extends keyof OrderEvents>(event: K, listener: (...args: OrderEvents[K]) => void): this {
    return super.on(event, listener as (...args: unknown[]) => void);
  }
  override emit<K extends keyof OrderEvents>(event: K, ...args: OrderEvents[K]): boolean {
    return super.emit(event, ...args);
  }
}

const orders = new OrderEmitter();
orders.on("shipped", (order, trackingNumber) => { /* both fully typed */ });
orders.emit("shipped", order); // compile error — missing the required trackingNumber argument
```
- **DON'T:** Leave a custom `EventEmitter` subclass's `.on()`/`.emit()` calls untyped (accepting a bare `string` event name and `...args: any[]`) in a TypeScript codebase — this is one of the easiest places for a typo'd event name (`"shiped"` instead of `"shipped"`) or a mismatched payload shape to silently compile and then fail at runtime as a listener that simply never fires, with no error anywhere to point at the mistake.
- **DO:** Wrap a legacy Node-style error-first callback API in a typed Promise-returning function at the boundary when the library itself has no typed Promise variant, so the rest of the codebase gets full type inference for the resolved value instead of manually annotating callback parameters at every call site.
```ts
function readConfigFile(path: string): Promise<string> {
  return new Promise((resolve, reject) => {
    fs.readFile(path, "utf8", (err, data) => {
      if (err) reject(err);
      else resolve(data);
    });
  });
}
```
- **DO:** Consider `AbortController`/`AbortSignal`-based cancellation over ad hoc "cancel" event patterns for new callback-based or streaming APIs — it's the standard, broadly-supported mechanism across modern Node.js and browser APIs alike, and using it consistently means one cancellation idiom works uniformly across `fetch`, streams, timers, and increasingly across Node's own core APIs, rather than a bespoke, one-off cancellation mechanism for each individual API.

### Type Complexity & Compiler Performance

- **DON'T:** Build deeply recursive conditional/mapped types (a type-level parser, a deeply nested string-manipulation type chain) without bounding their recursion depth — TypeScript's type checker has real, finite limits on recursion depth and computation, and an overly complex type-level computation can measurably slow down `tsc`, slow down editor responsiveness (autocomplete/hover latency across the whole project, not just the file with the complex type), or in extreme cases hit the compiler's own recursion limit and fail to compile at all.
- **DO:** Test genuinely complex type-level utilities against realistic input sizes (not just small textbook examples) before committing them, and watch for a noticeably slower `tsc --noEmit` run or laggy editor experience after adding one — these are concrete, measurable warning signs that a specific type is too computationally expensive for the compiler, not just a subjective readability concern.
- **DON'T:** Let type complexity creep in in exchange for marginal type-safety gains that a simpler type (or a runtime check) would provide almost as well — the earlier guidance on avoiding "clever" type-level metaprogramming for application code applies doubly once compiler performance is a factor: a type that's both hard for humans to read *and* measurably slow for the compiler to check is a cost paid on every single build and every single keystroke of editor autocomplete, for benefit that's often achievable more cheaply another way.
- **DO:** Prefer a small number of well-named, moderately-general utility types over a single maximally-generic one attempting to cover every possible case through heavy conditional-type branching — several simpler, more specific types are usually both easier for the compiler to check quickly and easier for a human to understand than one type-level Swiss Army knife.
- **DON'T:** Ignore a large TypeScript project's overall `tsc` build time creeping upward over time as "just how big projects are" without periodically profiling it (`tsc --extendedDiagnostics`, or the `--generatetrace` flag for a detailed trace) to find out whether a small number of specific overly-complex types are disproportionately responsible — in practice, compiler slowdowns are very often concentrated in a handful of genuinely problematic type definitions rather than spread evenly across the codebase, which makes them findable and fixable once actually profiled.

### Path Aliases & Module Resolution

- **DO:** Configure path aliases (`"paths"` in `tsconfig.json`, e.g. `"@app/*": ["src/*"]`) to replace long, fragile relative import chains (`../../../../shared/utils`) with short, stable, absolute-feeling ones (`@app/shared/utils`) once a project's directory nesting gets deep enough that relative paths become hard to read and easy to break when a file moves.
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@app/*": ["src/*"]
    }
  }
}
```
- **DON'T:** Configure a `tsconfig.json` path alias without also configuring the runtime (Node.js itself, your bundler, or your test runner) to resolve the same alias — `tsconfig.json`'s `"paths"` only affects the TypeScript type checker, not how Node.js actually resolves `import`/`require` calls at runtime. Code that type-checks cleanly with an alias can still throw `MODULE_NOT_FOUND` when actually run, unless the runtime is separately configured (via `package.json` `"imports"`, a bundler's alias config, `tsc-alias`, or a loader) to resolve the same alias.
- **DO:** Prefer the standard `"imports"` field in `package.json` (Node's native subpath-imports feature, using a `#`-prefixed specifier like `#utils/date`) for internal path aliasing in modern Node.js/ESM projects when possible — unlike `tsconfig.json` `"paths"`, this is understood natively by Node.js itself at runtime, so there's no separate runtime-resolution step to keep in sync with the type checker's configuration.
- **DON'T:** Set `"moduleResolution"` in `tsconfig.json` to a value that doesn't match how the code will actually be bundled/run (e.g., using the legacy `"node"` resolution strategy for a project that ships conditional `"exports"` map entries the classic resolver doesn't understand) — use `"bundler"` (for code that goes through a bundler and never runs its raw output directly under Node) or `"node16"`/`"nodenext"` (for code that runs directly under Node.js and needs to correctly resolve modern package `"exports"` semantics), matching the project's actual execution path.
- **DO:** Keep import paths consistent in style project-wide — either always relative within a single package/feature folder and aliased only across feature boundaries, or a single alias convention throughout — and let `import/order`/`no-restricted-imports` ESLint rules enforce it, rather than leaving it to individual judgment call by call.

### Working with Third-Party & Ambient Types

- **DO:** Check whether a JavaScript-only package already has community-maintained types on DefinitelyTyped (`npm install -D @types/<package>`) before assuming it's untyped and reaching for `any` or a hand-written ambient shim. Most popular untyped packages already have a matching `@types` package; installing it is almost always less work and more accurate than writing your own.
- **DON'T:** Hand-write a `declare module "some-package"` ambient type that duplicates types the package (or its `@types` package) already ships — a duplicated, hand-maintained type declaration will silently drift out of sync with the real package as it's upgraded, since nothing connects the two.
- **DO:** Use `declare module` ambient declarations for their genuinely intended purposes: describing a package that truly has no types available anywhere, typing non-code asset imports your bundler supports (`declare module "*.svg"`, `declare module "*.css"`), or augmenting a third-party module's existing types with fields your app adds to them.
```ts
// Augmenting Express's Request type to add a field your auth middleware attaches
declare global {
  namespace Express {
    interface Request {
      user?: { id: string; role: string };
    }
  }
}
export {};
```
- **DON'T:** Let an `any`-typed shim for an untyped dependency leak untyped values deep into application code through every call site that imports it. Wrap the untyped import once, at the boundary, behind a small function with an explicit, accurate return type — the rest of the codebase then only ever sees the properly-typed wrapper, not the untyped underlying library.
```ts
// Bad — every call site gets `any`, and every call site could introduce a bug
import untypedParser from "some-untyped-lib";
const result = untypedParser.parse(input); // any

// Good — one narrow, honest boundary
import untypedParser from "some-untyped-lib";
interface ParsedResult { tokens: string[]; valid: boolean; }
function parseInput(input: string): ParsedResult {
  return untypedParser.parse(input) as ParsedResult; // one deliberate, documented assertion
}
```
- **DON'T:** Assume a package's shipped types are accurate simply because the code compiles without error — some packages (especially older ones, or ones with community-maintained types that lag behind releases) ship types that don't match actual runtime behavior. When the compiler and the documented/observed runtime behavior disagree, trust the runtime behavior, and consider submitting a fix upstream to the type definitions rather than silently coding around the mismatch everywhere it's used.
- **DO:** Keep an `@types/<package>` version aligned with its corresponding runtime package's version when the two are versioned independently (true for most DefinitelyTyped packages) — a large version gap between the runtime package and its types package is a common, easy-to-miss source of types that no longer match the actual API surface.

### Type-Only Imports, Declaration Files & JSDoc Typing

- **DO:** Use `import type { X } from "./module"` (or the `type` modifier on individual named imports: `import { type X, y } from "./module"`) when importing something used only as a type, never as a runtime value. This makes the import's purpose explicit, and lets the compiler/bundler safely elide it from the compiled output entirely, since a type-only import has no runtime representation to preserve.
```ts
// Bad — ambiguous whether User is used as a value or purely as a type
import { User } from "./user";

// Good — explicit, and safely erased at compile time
import type { User } from "./user";
```
- **DON'T:** Import a type using a regular value import when the project has `isolatedModules` or `verbatimModuleSyntax` enabled (common in projects using esbuild/SWC/Babel to strip types, since those tools transpile files independently and can't always tell whether an import is type-only without the explicit marker) — a plain `import { User }` for a type-only symbol can either break the build or, worse, leave a dangling runtime import for something that doesn't actually exist at runtime.
- **DO:** Write and ship a `.d.ts` declaration file (`"types"`/`"typings"` field in `package.json`, generated via `tsc --declaration`) for any TypeScript library published for other projects to consume. Without it, consumers either get no type information at all or have to write and maintain their own ambient shim — publishing accurate types is a core part of a TypeScript library's actual API contract.
- **DON'T:** Hand-maintain a `.d.ts` file that's meant to describe your own TypeScript source in parallel with the source itself — generate it from the source via the compiler (`"declaration": true` in `tsconfig.json`) so the two can never drift out of sync. Hand-written declaration files are appropriate for describing *other*, non-TypeScript code (an untyped JS dependency, a non-JS asset), not for restating your own already-typed source a second time.
- **DO:** Use JSDoc type annotations (`@param`, `@returns`, `@type`) with `// @ts-check` (or a project-wide `"checkJs": true` in `tsconfig.json`) to get real type-checking in plain JavaScript files when a full TypeScript migration isn't yet practical. This is a genuinely effective incremental path — VS Code and `tsc` both understand JSDoc types well enough to catch real bugs in `.js` files without introducing a build step.
```js
// @ts-check

/**
 * @param {string} name
 * @param {number} age
 * @returns {{ name: string, age: number }}
 */
function createUser(name, age) {
  return { name, age };
}

createUser("Ada", "thirty"); // flagged: argument of type 'string' is not assignable to type 'number'
```
- **DON'T:** Let JSDoc type comments in a checked JS file drift out of sync with the actual function signature (adding a parameter without updating the `@param` list) — an inaccurate JSDoc type is worse than no type, since it actively lies to both the type checker's suppressed-by-mismatch behavior in some configurations and to any human reading the comment as documentation.
- **DO:** Use `satisfies` (TypeScript 4.9+) when you want to validate that a literal value conforms to a type *without* widening the value's inferred type to that type — this preserves the narrowest, most specific inferred type (useful for downstream autocomplete/exhaustiveness) while still getting the safety check that the value is structurally valid.
```ts
// With `: Record<string, number>`, the specific keys are lost — TS only knows it's some Record<string, number>
const scores: Record<string, number> = { alice: 10, bob: 20 };
scores.alice.toFixed(2); // fine, but scores.charlie also type-checks (and is undefined at runtime)

// With `satisfies`, the literal's specific keys are preserved for autocomplete/exhaustiveness,
// while still checking it conforms to Record<string, number>
const scores2 = { alice: 10, bob: 20 } satisfies Record<string, number>;
```

## Async & Promises

### `async`/`await` Fundamentals

- **DO:** Use `async`/`await` as the default style for asynchronous code instead of raw `.then()`/`.catch()` chains. `async`/`await` reads top-to-bottom like synchronous code, integrates with `try`/`catch` for error handling, and avoids the visual nesting that `.then()` chains accumulate.
```js
// Bad
function loadProfile(id) {
  return fetchUser(id)
    .then((user) => fetchOrders(user.id)
      .then((orders) => ({ user, orders })));
}

// Good
async function loadProfile(id) {
  const user = await fetchUser(id);
  const orders = await fetchOrders(user.id);
  return { user, orders };
}
```
- **DON'T:** Mix `await` and raw `.then()` chains in the same function without a clear reason — pick one style per function so the control flow is easy to trace. A function that `await`s one call and `.then()`s the next forces the reader to track two different asynchronous idioms simultaneously.
- **DO:** Mark a function `async` only if it actually needs to `await` something inside it, or needs to consistently return a Promise for its callers. Marking every function `async` "just in case" adds a microtask-queue hop and an implicit Promise wrapper even to synchronous logic, and misleads readers about whether the function does real asynchronous work.
- **DON'T:** `await` a value inside a loop when the operations are independent and could run concurrently — this serializes work that has no ordering dependency and can turn an O(1)-round-trip operation into an O(n)-round-trip one.
```js
// Bad — serializes N independent network calls
async function loadAll(ids) {
  const results = [];
  for (const id of ids) {
    results.push(await fetchUser(id));
  }
  return results;
}

// Good — runs them concurrently
async function loadAll(ids) {
  return Promise.all(ids.map((id) => fetchUser(id)));
}
```
- **DO:** `await` sequentially, in a loop, only when each iteration genuinely depends on the previous one's result (e.g., paginated API calls that need the next page's cursor), and add a comment noting *why* it's sequential so a future reader doesn't "optimize" it into a broken `Promise.all`.
- **DON'T:** Forget to `await` a Promise-returning call when you actually need its result or need to know when it completes before proceeding — this "fire and forget by accident" is one of the most common async bugs, especially inside non-async contexts like `.forEach()` or `Array.map()` callbacks that aren't themselves awaited.
```js
// Bad — forEach doesn't wait for the async callbacks; function returns before writes finish
async function saveAll(records) {
  records.forEach(async (record) => {
    await db.save(record);
  });
}

// Good
async function saveAll(records) {
  await Promise.all(records.map((record) => db.save(record)));
}
```
- **DO:** Understand that `.forEach()`, `.map()` (without awaiting the result), and other synchronous higher-order functions do not wait for async callbacks — each callback's returned Promise is discarded (`.forEach`) or collected into an array of pending Promises (`.map`) that must itself be handled, typically with `Promise.all`.

### Promise Combinators & Concurrency Control

- **DO:** Use `Promise.all()` for a batch of independent operations where any single failure should abort the whole batch, `Promise.allSettled()` when you need every result regardless of individual failures, `Promise.race()` for "first to settle wins" (e.g., a timeout race), and `Promise.any()` for "first to *succeed* wins." Picking the wrong combinator is a common, subtle correctness bug.
```js
// Promise.all — one failure rejects the whole batch immediately
const [user, orders] = await Promise.all([fetchUser(id), fetchOrders(id)]);

// Promise.allSettled — always resolves; inspect each result's status
const results = await Promise.allSettled(userIds.map(fetchUser));
const succeeded = results.filter((r) => r.status === "fulfilled").map((r) => r.value);
const failed = results.filter((r) => r.status === "rejected");
```
- **DON'T:** Use `Promise.all()` when partial failure is acceptable and expected (e.g., sending notifications to 100 users where one bad email shouldn't cancel the other 99) — a single rejection anywhere in the array immediately rejects the whole `Promise.all`, discarding the results of everything else that may have already succeeded.
- **DO:** Implement a concurrency limiter (a small custom queue, or a library like `p-limit`) when firing off a large or unbounded number of concurrent async operations against an external resource (a database, an API with rate limits, the filesystem). Unbounded `Promise.all(items.map(...))` over a large array can exhaust connection pools, hit rate limits, or blow past memory limits all at once.
```js
// Bad — could open thousands of concurrent connections at once
await Promise.all(hugeIdList.map((id) => fetchAndProcess(id)));

// Good — bounded concurrency
import pLimit from "p-limit";
const limit = pLimit(10);
await Promise.all(hugeIdList.map((id) => limit(() => fetchAndProcess(id))));
```
- **DO:** Implement timeouts for any Promise that talks to an external system (HTTP requests, database queries) using `Promise.race()` against a timer, `AbortController` where the underlying API supports it (most modern `fetch` and many Node APIs do), or a library's built-in timeout option. An operation with no timeout can hang the calling code indefinitely if the remote system stalls.
```js
async function fetchWithTimeout(url, timeoutMs = 5000) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  try {
    return await fetch(url, { signal: controller.signal });
  } finally {
    clearTimeout(timer);
  }
}
```
- **DON'T:** Wrap a `Promise` executor with logic that can throw synchronously without handling it, and never wrap an already-async operation in `new Promise((resolve, reject) => { ... })` unnecessarily — the "Promise constructor antipattern." If you already have a Promise-returning function or an `async` function to call, just call it; don't re-wrap it.
```js
// Bad — needless Promise constructor around an already-async call
function getUser(id) {
  return new Promise((resolve, reject) => {
    fetchUser(id).then(resolve).catch(reject);
  });
}

// Good
function getUser(id) {
  return fetchUser(id);
}
```
- **DO:** Reach for `new Promise(...)` only to *promisify* a genuinely callback-based or event-based API that has no Promise-returning equivalent, and always call `resolve`/`reject` from within a `try`/`catch` (or ensure the callback itself can't throw past the executor) so a synchronous error doesn't crash the process instead of rejecting the Promise.
- **DO:** Use `util.promisify` (Node's built-in) to convert legacy Node-style `(err, result) => {}` callback APIs into Promise-returning ones, rather than hand-rolling the wrapper each time.
```js
import { promisify } from "node:util";
import { readFile } from "node:fs";
const readFileAsync = promisify(readFile);
const contents = await readFileAsync("./config.json", "utf8");
// or, simpler: import { readFile } from "node:fs/promises";
```

### Unhandled Rejections & Error Propagation in Async Code

- **DON'T:** Leave a Promise chain without a `.catch()` (or the enclosing `async` function without a `try`/`catch` at some level of the call stack) — an unhandled rejection crashes a Node.js process outright as of Node 15+ (it previously only logged a warning), and in a browser it surfaces as a silent, easy-to-miss console warning. Every Promise needs a plan for its rejection path.
```js
// Bad — unhandled rejection; on Node 15+, this crashes the process
fetchUser(id).then((user) => console.log(user));

// Good
fetchUser(id)
  .then((user) => console.log(user))
  .catch((err) => logger.error("failed to fetch user", err));
```
- **DO:** Register a process-level `unhandledRejection` (and `uncaughtException`) handler in Node.js server processes as a last-resort safety net for logging and graceful shutdown — never as the primary error-handling strategy. Its job is to catch what genuinely slipped through, log it with full context, and shut the process down cleanly (an app in an unknown state should not keep serving traffic), not to paper over unhandled errors indefinitely.
```js
process.on("unhandledRejection", (reason) => {
  logger.error("Unhandled rejection", reason);
  // Fail fast: let the process manager (PM2, k8s) restart a clean instance.
  process.exit(1);
});
process.on("uncaughtException", (err) => {
  logger.error("Uncaught exception", err);
  process.exit(1);
});
```
- **DON'T:** Use an empty `.catch(() => {})` just to silence an unhandled rejection warning without actually handling the error (logging it, retrying, or surfacing it to the caller). This hides real failures from monitoring and makes production incidents far harder to diagnose after the fact.
- **DO:** Let errors propagate out of `async` functions naturally (via `throw` or an unhandled rejected Promise) so they can be caught by the appropriate layer, instead of catching every error immediately at the lowest level and returning `null`/`undefined`/a sentinel value that the caller has to remember to check.
```js
// Bad — swallows the error, caller can't tell success from failure without checking for null
async function getUser(id) {
  try {
    return await db.users.findById(id);
  } catch {
    return null;
  }
}

// Good — let the caller decide how to handle the failure
async function getUser(id) {
  return db.users.findById(id); // throws naturally on failure
}
```
- **DO:** Await Promises inside a `try` block specifically when you intend to catch and handle their rejection in that scope — an `await` outside a `try` simply lets the rejection propagate, which is often correct, but is a decision to make deliberately, not by accident.
- **DON'T:** Return a Promise from inside a `try` block without `await`-ing it, when the intent is for that block's `catch` to handle its rejection — without `await`, the function returns the (still-pending) Promise immediately, the `try`/`catch` around it exits, and a later rejection escapes uncaught by that `catch`.
```js
// Bad — the catch here never sees the rejection
async function loadUser(id) {
  try {
    return fetchUser(id); // missing await
  } catch (err) {
    logger.error(err); // never runs on fetchUser's rejection
    throw err;
  }
}

// Good
async function loadUser(id) {
  try {
    return await fetchUser(id);
  } catch (err) {
    logger.error(err);
    throw err;
  }
}
```
- **DO:** Use `AbortController`/`AbortSignal` to cancel in-flight async work (fetch requests, timers, database queries where supported) when the operation is no longer needed — e.g., a user navigates away, a request is superseded by a newer one. Uncancelled async work continues consuming resources and can still resolve and mutate state after it's no longer relevant, which is a common source of "why did this stale response overwrite my new state" bugs.
- **DON'T:** Swallow the distinction between "this async operation failed" and "this async operation was intentionally cancelled." Check `signal.aborted` (or catch the specific `AbortError`) and handle it as a distinct, usually silent, code path rather than logging it as an application error.
- **DO:** Use top-level `await` (supported in ESM modules and the Node.js REPL) only at a module's true top level for genuine startup-time async work (e.g., loading config before the module finishes initializing), not as a substitute for structuring an application's async entry point properly. Overusing it can create surprising module-loading order dependencies across an app's import graph.

### Iterators, Generators & Async Iterators

- **DO:** Use generator functions (`function*`) to produce a lazy sequence of values on demand, especially for large or effectively-infinite sequences, instead of eagerly building a full array up front. A generator only computes the next value when it's actually requested, which keeps memory bounded regardless of how many values the sequence could theoretically produce.
```js
// Bad — materializes the entire (possibly huge) sequence in memory at once
function range(start, end) {
  const result = [];
  for (let i = start; i < end; i++) result.push(i);
  return result;
}

// Good — produces values lazily, one at a time, on demand
function* range(start, end) {
  for (let i = start; i < end; i++) yield i;
}
for (const n of range(0, 1_000_000)) {
  if (n > 5) break; // stops immediately; never computes the rest
}
```
- **DO:** Use async generators (`async function*`) and `for await...of` to model a stream of asynchronously-produced values — paginated API results, database cursor rows, chunks read from a stream — as a single, composable iterable, instead of manually managing a queue, an index, and a "has more" flag by hand.
```js
async function* paginatedUsers(apiClient) {
  let cursor = null;
  do {
    const page = await apiClient.listUsers({ cursor });
    yield* page.items;
    cursor = page.nextCursor;
  } while (cursor);
}

for await (const user of paginatedUsers(apiClient)) {
  console.log(user.name);
}
```
- **DON'T:** Manually implement the iterator protocol (`Symbol.iterator` returning an object with a hand-rolled `.next()` method tracking state via closures) when a generator function would express the exact same logic far more concisely and with fewer places to introduce a state-tracking bug. Generators exist specifically to make writing custom iterables easy — reach for one before hand-rolling the protocol.
- **DO:** Implement `Symbol.iterator` (or `Symbol.asyncIterator`) on a custom class/object when you want instances of it to work naturally with `for...of`, spread syntax, and destructuring — this is the correct extension point for making a custom data structure feel like a native, iterable JS value.
- **DON'T:** Call `.next()` on a generator or async generator without understanding that generators are stateful and single-use — once fully iterated (or once you stop pulling values), the same generator object can't be "restarted" from the beginning; you need to call the generator function again to get a fresh iterator.
- **DO:** Use `yield*` to delegate to another iterable/generator from within a generator, instead of manually looping over the inner iterable and re-yielding each value one at a time — `yield*` is shorter and correctly forwards return values and thrown errors between the delegating and delegated generators.

## Error Handling

### Throwing & Catching

- **DO:** Throw `Error` objects (or subclasses of `Error`), never plain strings, numbers, or objects. Only `Error` instances carry a stack trace, a consistent `.message`/`.name` shape, and are what every logging tool, error tracker (Sentry, Rollbar), and `instanceof` check expects.
```js
// Bad
throw "user not found";
throw { code: 404, message: "user not found" };

// Good
throw new Error("user not found");
```
- **DO:** Create custom error subclasses for distinct, meaningful error categories your application needs to distinguish programmatically (validation errors, not-found errors, auth errors), rather than parsing `.message` strings to figure out what kind of error occurred.
```ts
class NotFoundError extends Error {
  constructor(entity: string, id: string) {
    super(`${entity} with id "${id}" was not found`);
    this.name = "NotFoundError";
    Object.setPrototypeOf(this, NotFoundError.prototype);
  }
}

class ValidationError extends Error {
  constructor(public readonly issues: string[]) {
    super(`Validation failed: ${issues.join(", ")}`);
    this.name = "ValidationError";
    Object.setPrototypeOf(this, ValidationError.prototype);
  }
}
```
- **DO:** Call `Object.setPrototypeOf(this, NewErrorClass.prototype)` (or compile with a target/`tsconfig` setting that handles it) in a custom `Error` subclass's constructor when targeting older JS output — TypeScript compiling to ES5 (and some older bundler configs) breaks `instanceof` checks against `Error` subclasses without this fix, because `super(message)` doesn't correctly set up the subclass prototype chain in those compiled targets.
- **DON'T:** Catch an error only to immediately rethrow it unchanged with no added value (`catch (e) { throw e; }`). An empty passthrough catch block adds nothing and just obscures the stack trace's origin; either add real handling (logging, wrapping with more context, cleanup) or remove the `try`/`catch` entirely.
- **DO:** Catch an error and rethrow it wrapped with additional context when the error crosses a meaningful boundary (e.g., a low-level "connection refused" becomes a higher-level "failed to save order"), using the `cause` option (ES2022+) to preserve the original error rather than losing it.
```js
// Bad — loses the original stack/cause
try {
  await db.save(order);
} catch (err) {
  throw new Error("failed to save order");
}

// Good — preserves the original error as `cause`
try {
  await db.save(order);
} catch (err) {
  throw new Error("failed to save order", { cause: err });
}
```
- **DON'T:** Use exceptions for expected, routine control flow (e.g., "user not found" during a normal lookup that the caller will handle with an `if`). Reserve exceptions for genuinely exceptional conditions; for expected "might not find this" cases, prefer a return value like `null`/`undefined`, an explicit `Result`-style union, or a documented sentinel — and reserve throwing for cases the caller isn't expected to routinely branch on.
```ts
// Debatable — a very common, expected outcome modeled as an exception
async function findUser(id: string): Promise<User> {
  const user = await db.users.findById(id);
  if (!user) throw new NotFoundError("User", id);
  return user;
}

// Often preferable when "not found" is a normal, expected outcome
async function findUser(id: string): Promise<User | null> {
  return db.users.findById(id);
}
```
- **DO:** Keep `try` blocks as small as possible, wrapping only the specific call(s) that can actually throw the error you intend to handle — a large `try` block makes it unclear which line actually threw, and can accidentally catch and mishandle errors from unrelated code.
```js
// Bad — unclear which of these three calls actually failed
try {
  const user = await fetchUser(id);
  const formatted = formatUser(user);
  await logAccess(user);
  return formatted;
} catch (err) {
  return null;
}

// Good — narrowly scoped, clear what's being guarded against
const user = await fetchUser(id);
let formatted;
try {
  formatted = formatUser(user);
} catch (err) {
  logger.warn("failed to format user, using fallback", err);
  formatted = fallbackFormat(user);
}
await logAccess(user);
return formatted;
```
- **DON'T:** Write a `catch` block that only does `console.log(err)` (or nothing at all) in production code. Silent or console-only error handling means failures are invisible to monitoring/alerting — use a real logger, and consider whether the error needs to be re-thrown, reported to an error tracker, or turned into a user-facing message.
- **DO:** Use `finally` for cleanup that must run whether the `try` block succeeded or threw (closing a file handle, releasing a lock, stopping a loading spinner), rather than duplicating the cleanup call in both the success path and every `catch` branch.
```js
async function withLock(key, fn) {
  const lock = await acquireLock(key);
  try {
    return await fn();
  } finally {
    await lock.release();
  }
}
```
- **DON'T:** `return` from inside a `finally` block — it silently overrides any `return` or `throw` from the `try`/`catch` blocks, discarding the original result or error with no indication anything was suppressed. This is confusing enough that most linters flag it (`no-unsafe-finally`).

### `AggregateError` & Multi-Error Patterns

- **DO:** Use the built-in `AggregateError` (ES2021+) to represent a single failure that's actually the combination of several underlying errors — most notably, `Promise.any()` rejects with an `AggregateError` containing every individual rejection reason when all of the input Promises reject, and this is the correct native tool for representing "all of these N attempts failed" as one error rather than picking just one of the underlying errors arbitrarily.
```js
try {
  const result = await Promise.any([tryPrimaryProvider(), trySecondaryProvider()]);
} catch (err) {
  if (err instanceof AggregateError) {
    console.error("all providers failed:", err.errors.map((e) => e.message));
  }
}
```
- **DO:** Construct your own `AggregateError` when a function legitimately needs to report multiple independent failures from one call — e.g., a batch validation function that checks several independent rules and wants to report every rule that failed at once, rather than throwing on the first failure and forcing the caller to fix and resubmit one error at a time.
```js
function validateAll(rules, input) {
  const errors = rules
    .map((rule) => { try { rule(input); return null; } catch (e) { return e; } })
    .filter(Boolean);
  if (errors.length > 0) {
    throw new AggregateError(errors, `${errors.length} validation rule(s) failed`);
  }
}
```
- **DON'T:** Silently collapse multiple distinct failures into a single generic error message that only reports the first one encountered, when the caller would genuinely benefit from seeing all of them at once — this is especially relevant for form/input validation, where reporting only the first invalid field forces a frustrating one-error-at-a-time correction cycle for the end user.
- **DO:** Check `err instanceof AggregateError` (or your framework's equivalent multi-error type) explicitly in error-handling code that might receive one, and iterate its `.errors` array to log or report each underlying cause individually — treating an `AggregateError` like an ordinary single `Error` (just reading `.message`) discards all but the aggregate's own summary message and loses the individual failure details.

### Working with `catch` and `unknown`

- **DO:** Treat the value caught in a `catch (err)` block as `unknown` in TypeScript (which is the default under modern TS versions with `useUnknownInCatchVariables`) and narrow it with `instanceof Error` before accessing `.message`/`.stack` — JavaScript allows `throw` on any value, so a caught value is not guaranteed to be an `Error` instance, especially when the throwing code is third-party.
```ts
try {
  await riskyOperation();
} catch (err) {
  const message = err instanceof Error ? err.message : String(err);
  logger.error("riskyOperation failed", { message });
}
```
- **DON'T:** Assume every caught value has a `.message` property without checking — third-party libraries, especially older or poorly-typed ones, sometimes throw plain strings, objects, or non-`Error` values, and accessing `.message` on those either returns `undefined` silently or throws a second error inside the `catch` block itself.

### Error Handling at Application Boundaries

- **DO:** Centralize error-to-HTTP-response translation in one place — a single Express/Fastify/Koa error-handling middleware — rather than manually formatting error responses inside every route handler. A single choke point guarantees consistent error response shape, consistent status-code mapping, and a single place to add error logging/tracking.
```js
// Express example
app.use((err, req, res, next) => {
  if (err instanceof ValidationError) {
    return res.status(400).json({ error: err.message, issues: err.issues });
  }
  if (err instanceof NotFoundError) {
    return res.status(404).json({ error: err.message });
  }
  logger.error("unhandled error", err);
  res.status(500).json({ error: "internal server error" });
});
```
- **DON'T:** Leak internal error details (stack traces, database error messages, file paths, internal identifiers) to end users or API clients in production. Return a generic message to the client and log the full detail server-side; a raw stack trace in an HTTP response is both an information-disclosure risk and unhelpful to the caller.
- **DO:** Use `express-async-errors` (or, in Express 5+, rely on its native support) or explicit `try`/`catch` + `next(err)` in every async Express route handler — Express 4's built-in error handling does *not* automatically catch a rejected Promise thrown inside an `async` route handler, so an unhandled rejection there silently hangs the request or crashes the process instead of hitting your error middleware.
```js
// Bad on Express 4 — a thrown/rejected error here bypasses the error middleware
app.get("/users/:id", async (req, res) => {
  const user = await findUser(req.params.id); // if this rejects, request hangs
  res.json(user);
});

// Good — wrap so rejections reach next(), or use express-async-errors
app.get("/users/:id", async (req, res, next) => {
  try {
    const user = await findUser(req.params.id);
    res.json(user);
  } catch (err) {
    next(err);
  }
});
```
- **DO:** Validate and fail fast on malformed input at the boundary of a function/module (an API handler, a queue consumer, a CLI argument parser) rather than letting bad data propagate deep into business logic before it causes a confusing failure far from its actual source.
- **DON'T:** Retry a failed operation blindly and unconditionally without distinguishing retryable errors (network blips, 503s, timeouts) from non-retryable ones (validation errors, 401/403, 404). Retrying a request that will deterministically fail again just wastes time and can amplify load on a struggling downstream system.
- **DO:** Use exponential backoff with jitter for retries against external services, and cap the maximum number of attempts. A naive fixed-interval retry storm from many clients simultaneously can synchronize into a "thundering herd" that keeps a recovering service down.
```js
async function withRetry(fn, { maxAttempts = 5, baseDelayMs = 200 } = {}) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxAttempts || !isRetryable(err)) throw err;
      const delay = baseDelayMs * 2 ** (attempt - 1) * (0.5 + Math.random());
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}
```
- **DO:** Log errors with structured context (request ID, user ID, operation name) rather than a bare message, so failures can be correlated across a distributed system's logs. `logger.error("save failed", { orderId, userId, err })` is searchable and joinable; `console.log("save failed")` is not.

### Result Types & Functional Error Handling

- **DO:** Consider a discriminated-union `Result<T, E>` return type (`{ ok: true, value: T } | { ok: false, error: E }`, hand-rolled or via a small library like `neverthrow`) for functions where failure is a routine, expected outcome the caller must explicitly handle — this makes the possibility of failure visible in the function's type signature, and the compiler forces every call site to check `ok` before accessing `value`, unlike a thrown exception whose possibility is invisible in the signature.
```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

function parseAge(input: string): Result<number, string> {
  const n = Number(input);
  if (!Number.isInteger(n) || n < 0) {
    return { ok: false, error: `"${input}" is not a valid age` };
  }
  return { ok: true, value: n };
}

const result = parseAge(input);
if (!result.ok) {
  console.error(result.error);
} else {
  console.log(result.value + 1); // narrowed to number here
}
```
- **DON'T:** Mix `Result`-style return values and thrown exceptions inconsistently for the same category of failure within one module or API surface — pick one convention per boundary (e.g., "parsing functions return `Result`, everything else throws") and apply it consistently, since a caller who doesn't know which convention a given function uses will handle its failures incorrectly at least some of the time.
- **DO:** Reserve exceptions for genuinely unexpected, unrecoverable-at-this-layer conditions (a programming bug, a violated invariant, an unrecoverable I/O failure), and reserve `Result`-style returns for expected, routinely-handled failure modes (validation failures, "not found," business-rule rejections) — the distinction to keep in mind is whether the immediate caller is expected to have a normal, planned code path for the failure case.
- **DON'T:** Wrap an entire large function body in a `try`/`catch` purely to convert every possible thrown error into a `Result`'s error branch — this tends to produce an overly-broad catch that can't distinguish between genuinely different failure causes. Catch narrowly around the specific operation that can fail, and let unrelated bugs (a genuine `TypeError` from a coding mistake elsewhere in the function) continue to throw and surface loudly instead of being silently absorbed into a generic error value.
- **DO:** Use `Promise.allSettled` in combination with a `Result`-style pattern when aggregating multiple independent operations that can each fail independently, so the caller gets a clear, typed per-item success/failure breakdown instead of the whole batch being reduced to a single opaque failure via a rejected `Promise.all`.

### Logging & Observability

- **DO:** Use a structured logging library (`pino` for high-throughput server logging, `winston` for flexible multi-transport needs) that emits logs as structured JSON, rather than `console.log` with hand-formatted strings, for any server process running in production. Structured logs can be queried, filtered, and aggregated by a log platform (Datadog, CloudWatch, ELK); a free-text string requires fragile regex parsing to extract the same information back out.
```js
// Bad — unstructured, hard to query at scale
console.log(`User ${userId} placed order ${orderId} for $${amount}`);

// Good — structured, queryable by field
logger.info({ userId, orderId, amount }, "order placed");
```
- **DON'T:** Use `console.log` as the primary logging mechanism in a production server. It's synchronous and unbuffered in some environments (which can itself become a performance bottleneck under high log volume), has no concept of log levels, can't easily be routed to different destinations, and produces unstructured output that's hard to search across many instances.
- **DO:** Use consistent log levels (`debug`, `info`, `warn`, `error`, and optionally `fatal`/`trace`) with a clear, team-shared convention for what belongs at each level, and configure the minimum level per environment (verbose `debug` locally, `info`-and-above in production) rather than logging everything at the same level regardless of severity.
- **DO:** Include a correlation/request ID in every log line related to handling a single request, generated once at the edge (or propagated from an upstream service via a header like `x-request-id`) and threaded through every downstream log call for that request. Without a correlation ID, reconstructing the full sequence of events for one specific failing request out of millions of interleaved concurrent log lines is extremely difficult.
```js
app.use((req, res, next) => {
  req.id = req.headers["x-request-id"] || crypto.randomUUID();
  req.log = logger.child({ requestId: req.id });
  next();
});
// later, in any handler: req.log.info({ userId }, "processing order");
```
- **DON'T:** Log entire large objects (a full request body, a full user record) indiscriminately "just in case it's useful later" — beyond the sensitive-data risk covered in the Security section, oversized log entries increase storage cost, slow down log ingestion, and bury the genuinely useful fields in noise. Log the specific fields that are actually useful for debugging and auditing.
- **DO:** Log the full error object (including its stack trace), not just its `.message`, when logging a caught error — the stack trace is frequently the single most useful piece of information for diagnosing where and why a failure happened, and stripping it down to just the message string throws that information away.
- **DON'T:** Treat logging as a substitute for proper error monitoring/alerting (Sentry, Datadog APM, or similar). Logs are for retrospective investigation once you already know something's wrong; a dedicated error-tracking tool is what actually notices a new class of error occurring and alerts a human, aggregates duplicate occurrences, and tracks whether a given error is a regression.
- **DO:** Emit metrics (counters, histograms, gauges) for key operational signals — request rate, error rate, response-time percentiles, queue depth — separately from logs, using a metrics library/agent (Prometheus client, StatsD, an APM vendor's SDK). Metrics are cheap to store and query over long time windows for trends and alerting thresholds in a way that grepping through logs isn't.

### Log Redaction & Sensitive Field Scrubbing

- **DO:** Configure a structured logging library's built-in redaction support (`pino`'s `redact` option, or an equivalent) to automatically scrub known-sensitive field names (`password`, `token`, `authorization`, `ssn`, `creditCard`) from every log call, rather than relying on every individual `logger.info(...)` call site to manually remember to exclude sensitive fields from whatever object it happens to log.
```js
import pino from "pino";

const logger = pino({
  redact: {
    paths: ["req.headers.authorization", "*.password", "*.creditCard", "user.ssn"],
    censor: "[REDACTED]",
  },
});

logger.info({ user: { email: "a@b.com", password: "hunter2" } }, "user updated");
// logs: { "user": { "email": "a@b.com", "password": "[REDACTED]" }, "msg": "user updated" }
```
- **DON'T:** Log an entire request or response object indiscriminately (`logger.info(req)`, `logger.info(response.data)`) as a debugging convenience without first confirming what fields it actually contains — this is one of the most common accidental ways sensitive data (auth headers, full payment payloads, tokens) ends up in log storage, entirely by accident, from a debug log line nobody thought carefully about at the time.
- **DO:** Maintain redaction rules as data (a shared, centrally-defined list of sensitive field-name patterns) rather than as scattered, one-off manual omissions at each individual call site — a centrally-defined redaction policy is auditable, consistently applied by construction, and a new sensitive field only needs to be added to the list once, rather than remembered at every future call site that might log an object containing it.
- **DON'T:** Assume redaction based on field name alone catches every case — a sensitive value can appear inside a free-text field that wasn't anticipated (an error message that happens to include a raw token because an upstream library included it in its own thrown error's message). Treat log redaction as a strong mitigation, not an absolute guarantee, and keep the broader "don't log more than you need" discipline from the Logging & Observability section above as the primary defense.
- **DO:** Apply the same redaction discipline to error-tracking tools (Sentry, Rollbar) as to logs — these tools also often capture full request context (headers, body) by default when reporting an exception, and most provide their own configurable scrubbing/redaction hooks that need the same deliberate configuration as a logging library's redaction option.

## Module System (ESM vs. CommonJS)

- **DO:** Pick one module system — ES Modules (`import`/`export`) or CommonJS (`require`/`module.exports`) — for a given package and use it consistently. Modern Node.js and the broader ecosystem have converged on ESM as the forward-looking default; use it for new projects unless a specific, load-bearing dependency or deployment target forces CommonJS.
- **DON'T:** Mix `require()` and `import` syntax within the same file, or randomly switch between them across files in the same package without a clear boundary (e.g., a documented CJS compatibility shim). This is one of the most common signs of AI-generated code that copy-pasted snippets from sources using different module systems without reconciling them.
```js
// Bad — mixed module syntax in one file, will not run as either CJS or ESM as-is
import express from "express";
const { readFile } = require("fs");

// Good (ESM)
import express from "express";
import { readFile } from "node:fs/promises";
```
- **DO:** Set `"type": "module"` in `package.json` to declare a package as ESM (making `.js` files ES modules by default), or explicitly use the `.mjs`/`.cjs` extensions when a package needs to mix module systems for a documented reason (e.g., a config file a CJS-only tool needs to `require`).
- **DON'T:** Assume `require()` can synchronously load an ESM-only package — as of Node's stable dual-package support, CommonJS can use dynamic `import()` (which returns a Promise) to load ESM, but a synchronous top-level `require()` of an ESM-only package throws `ERR_REQUIRE_ESM`. Many popular packages (e.g., several versions of `chalk`, `node-fetch`, `execa`) have gone ESM-only, which breaks unmigrated CommonJS codebases that try to `require()` them — check the target package's supported import style before adding it as a dependency.
```js
// Bad — throws ERR_REQUIRE_ESM if chalk is ESM-only in the installed version
const chalk = require("chalk");

// Good, from CommonJS — dynamic import returns a Promise
const { default: chalk } = await import("chalk");
```
- **DO:** Use named exports for most module APIs (`export function formatDate() {}`) over a single default export, especially for utility modules with several related functions — named exports enable better tree-shaking, better auto-import tooling, and prevent the "what do I even call this on import" ambiguity default exports create.
- **DON'T:** Default-export an object that's just a bag of unrelated functions (`export default { formatDate, formatCurrency, parseUrl }`) when named exports would let bundlers eliminate unused ones and let editors auto-import by name. Reserve a default export for genuinely singular things — a single class, a single component, a single configuration object that *is* the module's whole purpose.
- **DO:** Use `import.meta.url` (ESM) instead of the CommonJS-only `__dirname`/`__filename` globals when you need the current module's path in an ES module — those two globals don't exist in ESM and must be reconstructed via `fileURLToPath(import.meta.url)` if genuinely needed.
```js
import { fileURLToPath } from "node:url";
import { dirname } from "node:path";
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```
- **DON'T:** Use dynamic `require()` calls with a runtime-computed path/string (`require(someVariable)`) except for narrowly justified plugin-loading scenarios — it defeats static analysis (bundlers, tree-shakers, and type checkers can't see what's being required), can be a code-injection vector if the string is influenced by user input, and makes dependency graphs impossible to audit statically.
- **DO:** Use dynamic `import()` for genuine code-splitting/lazy-loading needs (loading a large, rarely-used module only when a specific code path is hit) — unlike dynamic `require`, it's a standard, statically-analyzable (by bundlers) part of the ESM spec designed exactly for this.
```js
async function generatePdfReport(data) {
  // Only pulled into the bundle/loaded into memory when this path actually runs.
  const { renderPdf } = await import("./pdf-renderer.js");
  return renderPdf(data);
}
```
- **DO:** Understand the difference between ESM's live-binding exports (a change to an exported `let` in module A is visible to module B that imported it) and CommonJS's copy-by-value `module.exports` object — code relying on mutating an exported binding from outside its owning module behaves differently under each system, and is fragile either way; prefer exporting functions/accessor patterns over mutable exported state.
- **DON'T:** Reach for `require.cache` manipulation, monkey-patching `Module.prototype._compile`, or other CommonJS internals to hot-reload modules or hack dependency injection. These are undocumented implementation details that vary across Node versions and have no ESM equivalent at all — use a proper dependency-injection pattern or a maintained hot-reload tool instead.
- **DO:** Keep circular dependencies out of the module graph where possible, and where genuinely unavoidable (rare, usually a sign of a design problem), understand that ESM and CJS resolve them differently (ESM's live bindings often "just work" if the circular reference is only used after both modules finish loading; CJS returns a partially-populated `module.exports` at the time of the circular `require`, which can be `undefined` for not-yet-defined exports). Either way, a circular dependency is usually a sign two modules should be split differently or merged.
- **DON'T:** Bundle server-only code (secrets, filesystem access, database clients) into a module that's also imported by client-side/browser bundles, relying on tree-shaking to remove the unused parts — a bundler bug, dynamic import, or dead-code-elimination miss can leak server secrets into a client bundle. Keep server-only and client-safe code in clearly separate modules/packages with no import path between them.

### CommonJS/ESM Default Export Interop

- **DO:** Understand what `esModuleInterop` (and its prerequisite, `allowSyntheticDefaultImports`) actually does in `tsconfig.json` before relying on `import defaultExport from "some-cjs-package"` syntax against a CommonJS package — CommonJS has no real concept of a "default export" the way ESM does (a CJS module's entire `module.exports` value *is* what gets imported), so `esModuleInterop` inserts a compatibility shim that lets `import x from "cjs-pkg"` work sensibly, resolving to `module.exports` itself when there's no `.default` property, or to `.default` when there is one (a package explicitly built to interop with ESM tooling might set one).
```json
{
  "compilerOptions": {
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true
  }
}
```
- **DON'T:** Assume `import x from "cjs-package"` and `const x = require("cjs-package")` are always guaranteed to produce the identical value without `esModuleInterop` enabled — without it, TypeScript requires the more verbose `import * as x from "cjs-package"` form for many CommonJS packages, and code written assuming the default-import form works, without the compiler option that makes it behave correctly, can compile in one project configuration and produce a subtly wrong (or outright broken) import in another.
```js
// Without esModuleInterop, a CJS package with `module.exports = function() {}` needs:
import * as express from "express";
// With esModuleInterop enabled, the more familiar default-import form works correctly:
import express from "express";
```
- **DO:** Keep `esModuleInterop: true` enabled for essentially all modern TypeScript projects (it's included in commonly-recommended base configurations and has been the sensible default for years) — the main reason to know the mechanics behind it is to correctly diagnose an import that behaves unexpectedly (e.g., getting the whole module namespace object instead of the expected single function/class) when working across the CJS/ESM boundary, not to disable it.
- **DON'T:** Confuse a TypeScript-compile-time-only interop setting (`esModuleInterop`) with what actually happens at runtime under Node.js's own native CJS/ESM interop rules — `esModuleInterop` only affects how the TypeScript compiler translates `import` syntax into the emitted JavaScript; if that emitted JavaScript is genuinely running as native ESM under Node (rather than being compiled down to CommonJS `require()` calls), Node's own separate, real interop rules for importing a CJS package from ESM apply instead, and the two systems' behavior for edge cases (like which properties get exposed as named imports) doesn't always match exactly.

### Dual-Package Hazard & Package `exports`

- **DO:** Understand the "dual-package hazard" before shipping a library that supports both ESM and CommonJS consumers: if a package is loaded once via `require()` and once via `import` within the same process (common in a dependency tree where different packages use different module systems to reach the same shared dependency), Node.js can end up with two separate module instances — meaning `instanceof` checks, singletons, and shared module-level state silently stop matching across the two copies.
- **DO:** Use the `"exports"` field in `package.json` with explicit `"import"`/`"require"` conditions to declare exactly which file is served for each module system, rather than relying on file-extension conventions (`.mjs`/`.cjs`) alone or the older, less precise `"main"`/`"module"` fields.
```json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  }
}
```
- **DON'T:** Design a library's public API around mutable, shared module-level singleton state (a shared cache object, a shared event bus instance) if the library is meant to support dual ESM/CJS consumption — this is exactly the pattern the dual-package hazard breaks, since consumers on the "other" module system silently get a second, disconnected instance of that supposedly-shared state.
- **DO:** Test a published (or `npm pack`-simulated) library against both a CommonJS consumer project and an ESM consumer project before publishing, especially after any change to the package's build output or `"exports"` field — the failure mode here (silent divergent module instances, or an outright `ERR_REQUIRE_ESM`) often doesn't show up in the library's own internal test suite, only when a real external consumer imports it.
- **DON'T:** Assume that just because a package "has types" it correctly resolves those types for both `import` and `require` consumers — the `"types"` condition inside `"exports"` needs its own explicit entries per module-system condition (or a correctly generated combined declaration) or TypeScript can silently fall back to resolving the wrong `.d.ts` file for one of the two module systems.

## Dependencies & Package Management

### `package.json` Hygiene

- **DO:** Keep `package.json`'s `dependencies` limited to packages actually imported at runtime, and put build/test/lint-only tooling (TypeScript, ESLint, test runners, type packages like `@types/*`) in `devDependencies`. Bloating `dependencies` increases production install size, attack surface, and Docker image size for packages that never run in production.
- **DON'T:** Leave unused packages in `package.json` after refactoring them out of the code. Dead dependencies inflate `node_modules`, slow installs, widen the security-vulnerability surface (`npm audit` flags them even though nothing uses them), and confuse the next person who assumes every listed dependency is load-bearing. Run `depcheck` or an equivalent tool periodically and actually remove what it flags.
- **DO:** Add an accurate `"engines"` field (`{ "node": ">=20.0.0" }`) when the code relies on features from a specific Node.js version, and consider enforcing it in CI (`npm install --engine-strict` or a dedicated check) so a contributor on an old Node version gets a clear error instead of a confusing runtime failure.
- **DON'T:** Invent or guess an npm package name and add it to `package.json` without verifying it exists and does what you think it does. This is one of the most damaging AI-assistant failure modes — a hallucinated package name can be squatted by an attacker (a real, documented supply-chain attack technique called "slopsquatting"), and even a merely wrong package name breaks the install outright. Verify every package name and its actual API against its real, current documentation or registry page before writing an import for it.
```json
// Bad — "lodash-utils" and "node-fetch-promise" are not real, commonly-hallucinated package names
{
  "dependencies": {
    "lodash-utils": "^1.0.0",
    "node-fetch-promise": "^2.0.0"
  }
}
```
- **DO:** Pin exact versions (no `^`/`~` range) for critical infrastructure dependencies where an unreviewed automatic upgrade could be dangerous (e.g., a cryptography library, a database driver with breaking migration behavior), and rely on the lockfile plus a deliberate, reviewed upgrade process (Renovate/Dependabot PRs) for everything else rather than disabling ranges project-wide.
- **DO:** Fill in `"description"`, `"license"`, `"repository"`, and `"author"` fields in `package.json` for any package that will be published or open-sourced. An npm package with no license field is technically "all rights reserved" by default, which surprises consumers who assumed it was open source.
- **DON'T:** Use `"license": "UNLICENSED"` (or omit the field) for a package you intend others to freely use — verify the license field matches your actual intent, since the difference between "UNLICENSED", "MIT", and "proprietary" has real legal consequences for downstream consumers.
- **DO:** Define `"scripts"` for every common workflow action (`build`, `test`, `lint`, `dev`, `start`, `typecheck`) so contributors and CI can run `npm run <script>` uniformly rather than needing to remember bespoke direct tool invocations per project.
- **DON'T:** Put multi-line, complex logic directly inside a `package.json` script string. A `package.json` script should be a short, readable invocation; genuinely complex logic belongs in a real script file (`scripts/build.js`, a shell script) that the `package.json` entry simply calls, so it can be tested, linted, and version-controlled with proper diffs.
- **DO:** Set `"private": true` in `package.json` for any application (as opposed to a publishable library) to prevent an accidental `npm publish` of internal application code to the public registry.
- **DO:** Use `"exports"` in `package.json` to explicitly define a package's public entry points (and their CJS/ESM variants) when publishing a library, rather than relying on the implicit "any file in the package is importable" behavior of the older `"main"`-only convention. `"exports"` also lets you deliberately hide internal modules from consumers, which prevents them from depending on implementation details that might change without a major version bump.

### Semantic Versioning & Version Ranges

- **DO:** Understand and correctly apply semantic versioning (`MAJOR.MINOR.PATCH`) when publishing a package — increment `MAJOR` for breaking changes, `MINOR` for backward-compatible feature additions, `PATCH` for backward-compatible bug fixes. Consumers' `^`/`~` ranges rely on this contract being honored; a breaking change shipped as a patch release breaks every consumer who trusted semver.
- **DO:** Understand what the caret (`^`) and tilde (`~`) range operators actually allow before relying on them — `^1.2.3` allows any `1.x.x` release `>=1.2.3` (locks the major version), `~1.2.3` allows any `1.2.x` release `>=1.2.3` (locks major and minor), and a bare `1.2.3` locks the exact version. Picking the wrong operator either over-constrains (missing legitimate bug fixes) or under-constrains (accepting a version with unreviewed new behavior) the dependency.
```json
{
  "dependencies": {
    "express": "^4.19.2",   // any 4.x.x >= 4.19.2
    "some-internal-tool": "~1.4.0", // any 1.4.x >= 1.4.0
    "critical-crypto-lib": "3.2.1"  // pinned exactly
  }
}
```
- **DON'T:** Use `*`, `latest`, or `x` as a dependency version range in any project meant to be reproducible (which is almost every project). These accept literally any future version, including breaking major releases, and make builds non-deterministic across time even before a lockfile enters the picture.
- **DO:** Bump a dependency's major version deliberately, reading its changelog/migration guide first, rather than letting an unattended tool auto-merge a major-version bump. Automated dependency-update tools (Renovate, Dependabot) are excellent for opening the PR; a human (or a CI suite that's actually comprehensive) should still review a major bump before it merges.
- **DON'T:** Assume a package's types (its bundled `.d.ts` or a separate `@types/*` package) are automatically kept in sync with the package's runtime version — check that an `@types/*` package's version range actually corresponds to the runtime package's version when they're versioned independently (common for older, non-TypeScript-native packages).

### Lockfiles

- **DO:** Commit the lockfile (`package-lock.json`, `pnpm-lock.yaml`, or `yarn.lock`) to version control for every application and most publishable libraries. The lockfile is what makes `npm install` reproducible byte-for-byte across machines and over time — without it, `^`/`~` ranges can silently resolve to a different (and possibly broken or compromised) version tomorrow than they did today.
- **DON'T:** Manually hand-edit a lockfile. Lockfiles encode a resolved dependency graph with integrity hashes computed by the package manager; a hand edit desyncs the file from what the manager would actually produce and can silently break integrity verification or produce an unreproducible install.
- **DO:** Use `npm ci` (not `npm install`) in CI/CD pipelines and Docker builds. `npm ci` requires an existing, up-to-date lockfile, installs exactly what it specifies (deleting `node_modules` first for a clean, deterministic install), and fails outright if `package.json` and the lockfile have drifted apart — exactly the guarantees a CI environment needs and `npm install` does not provide.
```bash
# Bad in CI — can silently update the lockfile and resolve slightly different versions
npm install

# Good in CI — deterministic, fails loudly on drift
npm ci
```
- **DON'T:** Commit more than one package manager's lockfile in the same project (e.g., both `package-lock.json` and `yarn.lock` present). This is a common artifact of switching tools without cleaning up, and it causes confusion (and sometimes genuinely different resolved dependency trees) about which lockfile is authoritative — delete the ones you're not using and enforce a single package manager via `"packageManager"` in `package.json` or a preinstall check.
- **DO:** Regenerate and re-commit the lockfile whenever `package.json`'s dependencies change, and review the lockfile diff in code review for unexpected large-scale changes (a sign of a version range being wider than intended, or a full accidental reinstall).
- **DO:** Run `npm audit` (or the pnpm/yarn equivalent) regularly, and treat high/critical findings as real work items — not something to silently `--force` past. Understand that automatic `audit fix --force` can itself introduce breaking major-version bumps, so review what it changes rather than running it blindly on a production branch.

### npm / pnpm / Yarn Conventions

- **DO:** Pick a single package manager per project and enforce it — mixing `npm install` and `yarn add` (or `pnpm add`) on the same project produces conflicting lockfiles and can install genuinely different dependency trees for different contributors. Declare the intended manager via the `"packageManager"` field in `package.json` (Corepack-enforced) so tooling and CI can validate the right one is in use.
```json
{
  "packageManager": "pnpm@9.7.0"
}
```
- **DO:** Prefer `pnpm` for monorepos and disk-space-sensitive environments — it uses a content-addressable store with hard links, so shared dependency versions across many packages/projects are only stored once on disk, and it enforces stricter dependency isolation (a package can't accidentally import another package's transitive dependency that it never declared) than npm's flat `node_modules` by default.
- **DON'T:** Rely on "phantom dependencies" — importing a package that isn't listed in your own `package.json` but happens to be available because npm's flat `node_modules` hoisted it up from a sibling dependency. It works by accident until that sibling dependency changes or is removed, at which point the phantom import breaks with no obvious cause. Always declare every package you directly import.
- **DO:** Use npm/pnpm/Yarn workspaces for monorepos with multiple internal packages that depend on each other, instead of publishing internal packages to a registry (public or private) purely to consume them locally, or using manual `file:`/relative-path hacks outside a workspace setup.
- **DON'T:** Use `npm link` (or manual symlink hacks) as a long-term solution for local cross-package development outside of workspaces — it's useful for a quick manual test, but its state is local-machine-only, invisible to CI, and easy to forget you have active, leading to "works on my machine" confusion.
- **DO:** Understand the difference between `npx` (execute a package's binary, installing it temporarily if not already available) and a locally-installed dev dependency's binary run via an npm script or `pnpm exec`/`yarn dlx` — `npx` silently fetching and running an arbitrary, unpinned version of a tool from the registry on every invocation is a supply-chain risk for anything beyond ad hoc, interactive use. Pin and install tools as real dev dependencies for anything that runs in CI or is part of a reproducible workflow.
- **DON'T:** Run `npx <package>` (or any "download and execute" command) with a package name you haven't verified, especially in a CI pipeline or a script that runs unattended — this is a direct code-execution supply-chain vector, and it's exactly the kind of command an AI assistant can hallucinate or mistype into existence.
- **DO:** Use `overrides` (npm)/`resolutions` (Yarn)/`pnpm.overrides` (pnpm) to force a specific version of a deeply-nested transitive dependency when it has a known vulnerability or bug that its direct parent package hasn't yet updated to fix, and document *why* the override exists with a comment referencing the issue.
```json
{
  "overrides": {
    "vulnerable-transitive-pkg": "^2.1.4"
  }
}
```
- **DO:** Use `.npmignore` (or the `"files"` field in `package.json`, which is the more explicit, allow-list-based approach) to control exactly what gets published to the registry — publishing test files, `.env` files, source maps with embedded source, or internal documentation bloats the package and can leak information unintentionally.
- **DON'T:** Publish a package without first running `npm pack --dry-run` (or equivalent) to inspect exactly what file set will actually be published. It's common to accidentally include a `.env`, credentials fixture, or an entire `node_modules` because of a missing/misconfigured ignore rule.

### Monorepo Build Tooling & Task Caching

- **DO:** Use a dedicated monorepo build orchestrator (Turborepo, Nx, or a workspace-aware task runner) once a monorepo grows to more than a handful of interdependent packages — these tools understand the dependency graph between packages and can run builds/tests/lints only for packages actually affected by a given change, and cache task outputs (locally and, optionally, remotely/shared across a team or CI) so unrelated or unchanged packages don't get needlessly rebuilt on every run.
- **DON'T:** Run every package's full build/test/lint script unconditionally on every CI run in a large monorepo without any affected-package or caching strategy — as the monorepo grows, this makes CI time grow roughly linearly with total repo size rather than with the size of an individual change, which becomes a serious velocity tax long before it becomes an emergency worth fixing.
- **DO:** Declare each package's task dependencies explicitly in the build orchestrator's configuration (e.g., "this package's `build` depends on its dependencies' `build` outputs") so the tool can correctly determine execution order and cache invalidation — an incorrect or missing dependency declaration is a common source of "stale cache" bugs, where a downstream package's cached build output doesn't actually reflect a real, more recent change in one of its dependencies.
```json
{
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test": { "dependsOn": ["build"], "outputs": [] },
    "lint": { "outputs": [] }
  }
}
```
- **DON'T:** Let build-cache correctness silently rot — periodically verify that a cache hit actually reflects the current, correct output (e.g., cache keys correctly incorporate every input file, environment variable, and dependency version that affects the output), since a task runner blindly serving a stale cached result because an input wasn't properly tracked is a subtle bug that can ship genuinely broken code while every check reports green.
- **DO:** Keep genuinely shared code (shared types, shared utilities, shared config) in their own explicit internal workspace packages with clear, versioned dependency edges to the packages that consume them, rather than reaching across package boundaries with relative-path imports (`../../other-package/src/thing`) that bypass the workspace/build-tool's dependency graph entirely and defeat its ability to correctly determine what needs rebuilding.

### Evaluating a New Dependency

- **DO:** Check a candidate package's maintenance signals before adding it — recency of the last release, whether open issues and pull requests get any response, download trends, and whether it has open, unresolved critical security advisories. A package with no commits in several years and a pile of unanswered issues is a liability waiting to surface, even if it works today.
- **DON'T:** Add a dependency for something trivially implementable in a handful of lines of code (a basic `isEmpty` check, a simple string-padding helper, a one-line clamp function). Recall the real "left-pad" incident, where the removal of one tiny, widely-depended-upon package broke a large swath of the JavaScript ecosystem's builds overnight — every dependency, however small, is a real liability (supply chain, maintenance burden, install size) that should earn its place.
- **DO:** Check a package's install/bundle footprint (via a tool like Bundlephobia, or by inspecting its own `node_modules` after install) before adding it to client-facing code specifically, since an innocuous-looking package can pull in a surprisingly large transitive dependency tree that bloats a browser bundle.
- **DON'T:** Add a new dependency that duplicates functionality an existing dependency in the project already provides (adding `date-fns` when `dayjs` is already used elsewhere, or a second HTTP client library alongside one already in use). Consolidate on one choice per concern — duplicated libraries for the same job bloat the dependency tree and create inconsistent usage patterns across the codebase that confuse future contributors about which one is "the" way to do it.
- **DO:** Prefer dependencies that ship their own TypeScript types natively, or have well-maintained `@types` coverage, over otherwise-similar untyped alternatives, all else being roughly equal — native typing reduces the amount of boundary-wrapping and manual assertion needed elsewhere in the codebase.
- **DO:** Check a dependency's license for compatibility with your project's own license and commercial intentions before adding it — some licenses (certain AGPL variants, for example) impose obligations on how software that depends on them can itself be distributed, which can be a real legal problem for a proprietary product and is easy to overlook when just running `npm install`.
- **DON'T:** Add a package to `package.json` on an AI assistant's (or your own half-remembered) suggestion without first confirming, against the actual npm registry or the package's real repository, that it exists, is the package you think it is, and does what you expect. A plausible-sounding but nonexistent or wrong package name is not a hypothetical risk — see "Common AI-Assistant Mistakes" below for why this specifically matters.

### Peer Dependencies & Version Conflicts

- **DO:** Declare a package as a `peerDependency` (rather than a regular `dependency`) when publishing a library/plugin meant to be used alongside a specific host library it must share a single instance of — a React component library depending on `react`, an ESLint plugin depending on `eslint`, a database-ORM plugin depending on the ORM's core package. This tells consumers "bring your own compatible version of this" instead of the library silently installing and bundling its own separate copy.
```json
{
  "peerDependencies": {
    "react": ">=18.0.0"
  },
  "devDependencies": {
    "react": "^18.2.0"
  }
}
```
- **DON'T:** List a peer dependency's package as a regular `dependency` "to make installation easier" — if two different packages in the same dependency tree each bundle their own separate copy of what should be a single shared instance (a UI framework, a plugin-hosting library that relies on singleton internal state), you can end up with two different, incompatible instances of the "same" library coexisting in one app, which is a notoriously confusing class of bug (mismatched React contexts, duplicate framework state) that's hard to diagnose without knowing peer dependencies are the actual root cause.
- **DO:** Specify a peer dependency's version range as permissively as is actually safe (a broad major-version range like `>=18.0.0 <20.0.0` rather than an overly narrow pin) so the library doesn't force every consumer onto one exact version — an unnecessarily narrow peer dependency range is a common, avoidable source of the "conflicting peer dependency" installation errors that push developers toward the `--legacy-peer-deps`/`--force` reflex covered in the "Insecure Quick Fixes" section above.
- **DON'T:** Silence a peer-dependency warning/conflict with `--legacy-peer-deps`/`--force` without first reading what the actual conflict is — modern npm's strict peer-dependency resolution surfaces a real, specific incompatibility between two packages' declared requirements; understanding which two packages disagree, and on what, is usually necessary to actually fix the root problem (upgrading one of the conflicting packages, or confirming the conflict is genuinely benign in this specific case) rather than just suppressing the check.
- **DO:** Use `peerDependenciesMeta` to mark a peer dependency as `optional: true` when a library only needs it for one specific, optional feature (an optional integration, an optional adapter) — this avoids forcing every consumer of the library to install a dependency they may never actually use, while still declaring the compatibility relationship for consumers who do use that feature.

## Linting & Formatting

- **DO:** Use ESLint (or Biome as a fast, increasingly popular combined linter+formatter) with a well-established base configuration (`eslint:recommended` plus `@typescript-eslint/recommended` for TS projects, or a broader preset like Airbnb's) rather than hand-picking rules one at a time from scratch. A maintained preset already encodes years of community consensus about which patterns are genuinely bug-prone.
- **DO:** Use Prettier (or a formatter bundled into your linter, like Biome's) for all code formatting, and configure your editor and a pre-commit hook to run it automatically. Automated formatting removes an entire category of code-review nitpicking ("put a space here", "this should wrap") and guarantees consistent style regardless of who or what wrote the code.
- **DON'T:** Let ESLint and Prettier fight each other over formatting rules (indentation, line length, quote style) — use `eslint-config-prettier` to disable ESLint's own formatting-related rules and let Prettier own formatting exclusively, while ESLint focuses on catching actual bugs and code-quality issues (unused variables, unreachable code, incorrect Promise handling).
- **DO:** Run lint and format checks in CI (not just as an editor plugin), and fail the build on violations. A rule that only runs locally, opt-in, is a rule that gets skipped the moment someone is in a hurry — and an AI assistant working headlessly has no editor plugin to catch anything at all.
- **DO:** Set up lint-staged with a pre-commit hook (via Husky or a similar tool) to run linting/formatting only on staged files before each commit. This catches issues at the earliest, cheapest point — before they're even committed — without paying the cost of linting the entire repository on every commit.
```json
{
  "lint-staged": {
    "*.{js,ts,tsx}": ["eslint --fix", "prettier --write"]
  }
}
```
- **DON'T:** Disable a lint rule with an inline comment (`// eslint-disable-next-line`) without a comment explaining why, and never disable a whole file's linting (`/* eslint-disable */` at the top) as a substitute for actually fixing the flagged issues. A bare disable comment hides the reasoning from the next person, who then can't tell if it's still valid.
```js
// Bad — no explanation, and it's disabling everything in scope
/* eslint-disable */
function legacyHack() { /* ... */ }

// Good — narrow, explained
// eslint-disable-next-line no-eval -- legacy plugin sandbox requires dynamic eval; tracked in TICKET-456
eval(pluginCode);
```
- **DO:** Enable rules that catch genuine async bugs specifically — `no-floating-promises` and `no-misused-promises` (from `@typescript-eslint`), `require-await`, and `no-return-await` (context-dependent) — since these catch exactly the class of "forgot to await" and "unhandled rejection" bugs that are otherwise invisible until they fail at runtime.
- **DON'T:** Configure a linter with rule severity levels that are all `"warn"` and no `"error"` in CI — warnings that never fail a build get ignored indefinitely. Reserve `"warn"` for rules genuinely in a transition/adoption period, and default rules that indicate real bugs to `"error"`.
- **DO:** Use `import/order` (or an equivalent rule, e.g., from Biome or `eslint-plugin-simple-import-sort`) to keep import statements grouped and alphabetized consistently (external packages, then internal aliases, then relative imports), and let it auto-fix rather than manually maintaining import order by hand.
- **DO:** Enable `eslint-plugin-security` (or rely on a security-focused static analyzer like Semgrep) to catch common risky patterns automatically — use of `eval`, non-literal `RegExp` construction from user input, unsafe `child_process` calls with unsanitized input — as a baseline safety net layered on top of manual code review.
- **DON'T:** Treat a clean lint pass as proof of correctness. Linting catches style issues and a specific, known set of bug patterns; it says nothing about whether the business logic is right. Passing lint is necessary, not sufficient.

### Type-Aware Linting Configuration

- **DO:** Enable `@typescript-eslint`'s type-checked rule sets (`recommendedTypeChecked`/`strictTypeChecked` in the flat-config presets) for rules that need real type information to work at all — `no-floating-promises`, `no-misused-promises`, `await-thenable`, `no-unnecessary-condition` — since these catch genuine, otherwise-invisible async and type-correctness bugs that ESLint's purely syntactic rules structurally cannot detect.
```js
// eslint.config.js (flat config)
import tseslint from "typescript-eslint";

export default tseslint.config(
  ...tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,       // lets ESLint use real type info from your tsconfig
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },
);
```
- **DON'T:** Enable type-checked lint rules without correctly wiring `parserOptions.project`/`projectService` to the project's actual `tsconfig.json` — without it, type-aware rules either fail outright with a configuration error or silently fall back to behaving as if no type information were available, which quietly disables exactly the class of check you enabled them for in the first place.
- **DO:** Expect type-aware linting to run noticeably slower than purely syntactic linting, since it requires the TypeScript compiler to actually build a type-checking program in the background — for very large codebases, consider running the type-aware rule set on a separate, slightly slower CI job/pre-push hook rather than on every single keystroke-triggered editor lint pass, while keeping fast syntactic-only rules in the tight local feedback loop.
- **DON'T:** Let a lint configuration's type-aware rules silently stop applying to new files because they fall outside the `include` patterns of the `tsconfig.json` that `parserOptions` points at — a common, easy-to-miss gap where new files (a newly added `scripts/` directory, a new package in a monorepo) technically compile but never actually get linted with the stricter, type-aware rule set because the linter's type-checking program was never told those files exist.

## Testing

### General Testing Conventions

- **DO:** Use a modern test runner — Vitest for new projects (especially anything already using Vite, or wanting fast native ESM/TS support with minimal config) or Jest (the long-established, still widely-used default, especially in existing codebases and React Native/Next.js projects) — rather than hand-rolling a custom test harness or `assert`-and-`console.log` scripts.
- **DO:** Name test files consistently with the codebase's convention — typically `*.test.ts`/`*.test.js` colocated next to the source file, or mirrored under a top-level `__tests__`/`test` directory — and keep one convention throughout the whole repository so `test` script globs and editor "jump to test" tooling work predictably.
- **DO:** Structure tests with clear, descriptive `describe`/`it` (or `test`) blocks that read like a specification when scanned top to bottom. A test named `it("returns the correct discount for pro users with expired trials", ...)` documents behavior; `it("test 1", ...)` documents nothing.
```js
// Bad
test("test1", () => {
  expect(calculateDiscount(user)).toBe(0.2);
});

// Good
describe("calculateDiscount", () => {
  it("applies a 20% discount for active pro-plan users", () => {
    const user = { plan: "pro", isActive: true };
    expect(calculateDiscount(user)).toBe(0.2);
  });

  it("applies no discount for inactive users, regardless of plan", () => {
    const user = { plan: "pro", isActive: false };
    expect(calculateDiscount(user)).toBe(0);
  });
});
```
- **DO:** Follow the Arrange-Act-Assert (AAA) structure within each test — set up state, perform the action under test, then assert the outcome — as three visually distinguishable sections (blank lines between them, or comments), so each test is quick to scan and understand in isolation.
- **DON'T:** Write a test that asserts on implementation details (internal variable names, private method call counts, exact intermediate data structures) instead of observable behavior (return values, thrown errors, calls to external dependencies/mocks that represent real side effects). Implementation-detail tests break on every harmless refactor even when the actual behavior hasn't changed, which trains developers to ignore failing tests.
- **DO:** Keep each test independent and able to run in any order, in isolation, or in parallel — a test suite where test B only passes because test A happened to run first and left behind some shared state is fragile and impossible to safely parallelize or reorder.
- **DON'T:** Share mutable state (a shared object, a module-level variable, a database row) across tests without resetting it in a `beforeEach`/`afterEach`. Leaking state between tests produces flaky, order-dependent failures that are notoriously hard to reproduce and debug.
```js
// Bad — shared array leaks state between tests
const users = [];
test("adds a user", () => {
  users.push({ name: "Ada" });
  expect(users).toHaveLength(1);
});
test("adds another user", () => {
  users.push({ name: "Grace" });
  expect(users).toHaveLength(1); // fails — actually 2, because of leaked state
});

// Good
describe("user list", () => {
  let users;
  beforeEach(() => { users = []; });

  test("adds a user", () => {
    users.push({ name: "Ada" });
    expect(users).toHaveLength(1);
  });
});
```
- **DO:** Test behavior at the appropriate level for its value — fast, numerous unit tests for pure logic and edge cases; fewer, slower integration tests for the interactions between real modules (a route handler plus a real, ephemeral test database); a small number of end-to-end tests for critical user-facing flows. This is the classic "testing pyramid" shape, and it exists because unit tests are cheap to write and run, while end-to-end tests are valuable but slow and expensive to maintain.
- **DON'T:** Write only end-to-end tests (or only unit tests) as a blanket strategy. All-E2E suites are slow, flaky, and painful to debug when they fail since a failure could originate anywhere in the stack; all-unit suites can pass fully while the pieces still don't integrate correctly with each other.
- **DO:** Cover edge cases explicitly and by name — empty arrays/objects, `null`/`undefined` inputs, zero, negative numbers, maximum values, duplicate entries, malformed strings — rather than only testing the "happy path" with one realistic-looking example input. Most production bugs live in the edges, not the middle.
- **DON'T:** Write a test that can never fail — e.g., asserting `expect(true).toBe(true)`, wrapping the assertion in a `try`/`catch` that swallows failures, or asserting on a mock's return value instead of the function under test's actual output. A test that structurally cannot catch a regression provides false confidence, which is worse than no test at all because it hides the gap.

### Node's Built-in Test Runner (`node:test`)

- **DO:** Consider Node.js's built-in test runner (`node:test`, stable since Node 20) for projects that want zero test-framework dependency footprint — it supports `describe`/`it`, hooks (`beforeEach`/`afterEach`), built-in mocking (`node:test`'s `mock` module), coverage collection, and TAP-compatible output, all without installing anything beyond Node itself.
```js
import { test, describe } from "node:test";
import assert from "node:assert/strict";

describe("calculateDiscount", () => {
  test("applies 20% for active pro users", () => {
    assert.equal(calculateDiscount({ plan: "pro", isActive: true }), 0.2);
  });
});
```
- **DON'T:** Assume `node:test` is a drop-in replacement with full feature parity for a mature ecosystem like Jest or Vitest — as of recent Node versions it still has a narrower ecosystem of plugins/matchers, less mature watch-mode/UI tooling, and fewer built-in assertion styles (leaning on Node's own `assert` module rather than a rich `expect(...).toX()` matcher library) — evaluate it against the project's actual needs rather than assuming it's strictly equivalent.
- **DO:** Use `node:test`'s built-in `--test-concurrency`, snapshot testing support (in newer Node versions), and native code-coverage flag (`--experimental-test-coverage` / stabilized coverage support) as they mature, since running tests via the runtime itself (rather than a separate framework's own test process/transform pipeline) can mean less configuration and faster startup for straightforward projects, especially plain-JS ones without a build step.
- **DO:** Choose between `node:test`, Vitest, and Jest based on the project's actual needs — `node:test` for minimal-dependency simplicity in straightforward Node projects; Vitest for fast native ESM/TS support and tight integration with a Vite-based frontend/build setup; Jest for its mature, extremely widely-adopted ecosystem and broad framework integration (React Native, many existing large codebases) — rather than defaulting to whichever one happens to come up most often in general training data regardless of project fit.

### Mocking, Stubbing & Test Doubles

- **DO:** Mock genuinely external dependencies — network calls, the filesystem, the system clock, random number generation, third-party paid APIs — so tests are fast, deterministic, and don't depend on external service availability or cost real money/quota on every run.
```js
import { vi } from "vitest";

test("retries on network failure", async () => {
  const fetchSpy = vi
    .fn()
    .mockRejectedValueOnce(new Error("network error"))
    .mockResolvedValueOnce({ ok: true });

  const result = await fetchWithRetry(fetchSpy);
  expect(fetchSpy).toHaveBeenCalledTimes(2);
  expect(result).toEqual({ ok: true });
});
```
- **DON'T:** Mock the very thing you're supposed to be testing (e.g., mocking the function under test itself, or mocking so much of a module's internals that the test only verifies the mocks call each other correctly). Over-mocking produces tests that pass even when the real implementation is completely broken.
- **DO:** Prefer dependency injection (passing collaborators as parameters/constructor arguments) over module-level mocking (`vi.mock`/`jest.mock` reaching into another file's internals) where practical — injected dependencies are explicit in the function signature, easier to reason about, and don't require the test framework's module-mocking machinery at all.
- **DON'T:** Leave mocked timers, mocked `Date`, or intercepted network calls (`nock`, MSW) active after a test finishes — always restore/reset them in an `afterEach`. A leaked fake timer or unrestored global mock silently breaks unrelated tests that run afterward in the same process.
```js
afterEach(() => {
  vi.useRealTimers();
  vi.restoreAllMocks();
});
```
- **DO:** Use a real, ephemeral test database (a Dockerized Postgres, an in-memory SQLite, a per-test-run schema) for integration tests that touch persistence, rather than mocking the ORM/database layer entirely. Mocking the database layer means the test never verifies your actual SQL/queries are correct — a very common source of "tests pass, production query fails" gaps.
- **DON'T:** Assert against a network request library's mock in a way that only checks it was called, without checking it was called with the correct arguments (URL, method, body, headers). `expect(fetchMock).toHaveBeenCalled()` alone doesn't catch a request sent to the wrong endpoint or with a malformed payload.

### Coverage & Test Quality

- **DO:** Track code coverage as one useful signal among several, not as the goal itself. A high coverage percentage tells you which lines executed during the test run; it says nothing about whether the assertions on those lines actually verify correct behavior.
- **DON'T:** Chase 100% coverage by writing tests that execute a line without meaningfully asserting on its behavior, just to make the coverage number go up. This is a classic "gaming the metric" failure mode that produces a large, slow test suite with little real protective value.
- **DO:** Prioritize test coverage for business-critical logic, complex conditional branches, and code with a history of bugs, over exhaustively testing trivial pass-through code (a one-line getter, a simple re-export). Spend testing effort where the risk of a silent regression is actually high.
- **DO:** Add a regression test for every bug fix, reproducing the original failure first (confirm the test fails against the old code), then confirm it passes against the fix. This both proves the fix actually addresses the reported bug and prevents the same bug from silently reappearing in a future refactor.
- **DON'T:** Delete or skip (`.skip`/`xit`/`xdescribe`) a failing test to "get CI green" without first understanding whether it's failing because of a real regression or because the test itself is outdated/flaky. A skipped test with no tracked follow-up ticket is a coverage gap that silently persists indefinitely.
- **DO:** Use snapshot testing (`toMatchSnapshot()`) narrowly, for genuinely large, stable, structural outputs (a rendered component tree, a generated config file) where manually writing out the full expected value would be impractical — and always review a snapshot diff carefully before accepting it, rather than reflexively running `--updateSnapshot` to make a failing test pass.
- **DON'T:** Use snapshot testing for values with any volatility (timestamps, random IDs, floating-point-sensitive calculations) without normalizing them first — an unstable snapshot that changes on every run trains the team to blindly accept snapshot updates without reading the diff, which defeats the entire purpose of the check.
- **DO:** Write tests before or alongside the implementation (whether following strict TDD or a looser "test as you go" approach) for any logic complex enough that verifying it by hand/manual testing alone would be error-prone or slow to repeat.
- **DON'T:** Ship a pull request with all-new logic and zero new or updated tests, on the assumption that "it works, I tried it manually." Manual testing isn't repeatable, isn't run on every future change, and doesn't document the expected behavior for the next person who touches the code.

### Mutation Testing & Test Suite Quality

- **DO:** Use a mutation testing tool (`Stryker Mutator` for JS/TS) periodically on critical modules to verify that the existing test suite actually *fails* when the underlying code is deliberately broken — a mutation tester automatically introduces small changes to the source (flipping a `<` to `<=`, changing a `+` to `-`, negating a boolean condition) and reports which "mutants" survived, meaning no test caught the change.
- **DO:** Treat a high-surviving-mutant-rate module as a signal that its tests are checking the wrong things (asserting that a function returns *something*, rather than the *specific correct* something) even if its line-coverage percentage looks good — this is the concrete, tool-backed answer to the "coverage percentage doesn't mean the tests are good" problem raised in the Coverage & Test Quality section above.
```
// A surviving mutant example: this test would still pass even if `>` were mutated to `>=`
test("applies discount only when count exceeds 10", () => {
  expect(getDiscount(10)).toBe(0);
  expect(getDiscount(11)).toBe(0.1);
  // Missing: a case at exactly the boundary (count === 10) that would catch a >/>= mutation
});
```
- **DON'T:** Run mutation testing across an entire large codebase on every CI run by default — it's computationally expensive (effectively running the test suite many times over, once per mutant). Scope it to critical modules, run it on a slower cadence (nightly, or on-demand for a specific module under review) rather than as a blocking check on every pull request.
- **DO:** Use a mutation-testing run's surviving mutants as concrete, actionable prompts for new test cases — each surviving mutant points at a specific line and a specific kind of change the current tests don't catch, which is far more targeted guidance than a general "write more tests" instruction.

### Testing Async Code, HTTP Handlers & Time

- **DO:** Use `supertest` (or your framework's equivalent, e.g., Fastify's built-in `.inject()`) to test HTTP route handlers by making real requests against the app instance in-process, without binding an actual network port. This exercises routing, middleware, and serialization exactly as they'd run in production, while staying fast and avoiding port-conflict flakiness in CI.
```js
import request from "supertest";
import { app } from "../app.js";

test("GET /users/:id returns 404 for an unknown user", async () => {
  const res = await request(app).get("/users/does-not-exist");
  expect(res.status).toBe(404);
  expect(res.body.error).toBe("user not found");
});
```
- **DO:** Always `return` or `await` the Promise/assertion chain inside an async test function. A test framework can only know a test failed if the failure surfaces before the test function's returned Promise settles — an un-awaited rejected assertion inside an `async` test can silently be lost, and the test reports as passing.
```js
// Bad — the rejection from expect(...).rejects is never awaited; test can pass even if it should fail
test("throws on invalid input", async () => {
  expect(parseConfig("garbage")).rejects.toThrow(); // missing await
});

// Good
test("throws on invalid input", async () => {
  await expect(parseConfig("garbage")).rejects.toThrow();
});
```
- **DON'T:** Mix the legacy `done` callback style with returning a Promise from the same test function — most test runners treat this as an error or a hang (the runner waits for `done()` to be called and/or the Promise to settle, and the two mechanisms can interact confusingly). Pick one style per test — for new code, prefer `async`/`await` over `done` callbacks entirely.
- **DO:** Use fake timers (`vi.useFakeTimers()` in Vitest, `jest.useFakeTimers()` in Jest) to deterministically test time-dependent logic — debounce/throttle functions, retry backoff, cache TTL expiry — instead of using real `setTimeout` delays in the test itself, which both slows the test suite down and can flake under CI load when real timing gets close to a threshold.
```js
test("retries three times with exponential backoff", async () => {
  vi.useFakeTimers();
  const fn = vi.fn().mockRejectedValue(new Error("fail"));
  const promise = withRetry(fn, { maxAttempts: 3, baseDelayMs: 100 });

  await vi.advanceTimersByTimeAsync(100);
  await vi.advanceTimersByTimeAsync(200);
  await expect(promise).rejects.toThrow("fail");
  expect(fn).toHaveBeenCalledTimes(3);
});
```
- **DO:** Assert on rejected Promises using the framework's dedicated matcher (`expect(promise).rejects.toThrow(SpecificError)` in Jest/Vitest) rather than a manual `try`/`catch` with a `fail()`/`expect(true).toBe(false)` fallback in the `try` block — the dedicated matcher is shorter, and it can't accidentally pass silently the way a hand-rolled try/catch can if the error type check is written incorrectly.
- **DO:** Test error paths and rejection cases as thoroughly as success paths — assert that a function rejects with the specific expected error type/message for each class of invalid input, not only that it resolves correctly for one realistic-looking happy-path input.
- **DON'T:** Leave real network calls active in tests that are meant to be unit/integration tests against your own code — intercept them (with `nock`, MSW, or a mocked HTTP client) so the test suite doesn't depend on third-party service availability, doesn't leak real requests (and possibly real side effects) to external systems, and stays fast.

### Property-Based & Fuzz Testing

- **DO:** Reach for property-based testing (via a library like `fast-check`) for logic with a large, well-defined input space and a checkable invariant — a sort function should always produce an output the same length as its input, containing the same elements, in non-decreasing order, for *any* array — instead of relying solely on a handful of hand-picked example inputs, which can easily miss the specific edge case that breaks the implementation.
```js
import fc from "fast-check";
import { test } from "vitest";

test("sorting is idempotent and preserves length", () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const sorted = mySort(arr);
      return sorted.length === arr.length && JSON.stringify(mySort(sorted)) === JSON.stringify(sorted);
    }),
  );
});
```
- **DO:** Use property-based testing specifically for parsing/serialization round-trip invariants (`parse(serialize(x))` should equal `x` for any valid `x`) and for pure data-transformation functions where the correctness property is easier to state in general terms than to enumerate as specific examples.
- **DON'T:** Reach for property-based testing as a default replacement for ordinary example-based unit tests — it shines specifically where a general invariant is easy to state and the input space is large; for logic whose correctness is best expressed via a handful of specific, meaningful business scenarios (e.g., "a pro-plan user gets a 20% discount"), plain example-based tests communicate intent far more directly.
- **DO:** Let a property-based testing library's shrinking behavior work for you when a test fails — most implementations automatically reduce a failing random input down to the smallest/simplest case that still reproduces the failure, which is usually far more useful for debugging than the original large random input that happened to trigger it.
- **DON'T:** Skip fuzz-testing (feeding a function large volumes of randomized, malformed, or adversarial input) for any function that parses untrusted external input (a request body parser, a file-format parser, a protocol decoder) — this is exactly the kind of code where an untested edge case (an unexpectedly-shaped input, an extreme value) tends to surface as a production crash or, worse, a security vulnerability rather than a caught test failure.

### Test Organization, Flakiness & CI Integration

- **DO:** Run the full test suite in CI on every pull request before merge, and treat a red CI run as a hard block on merging, not a suggestion. A test suite that isn't actually enforced as a merge gate degrades quickly, since failures start getting ignored "just this once" until the suite is red so often nobody trusts it.
- **DON'T:** Let a known-flaky test (one that fails intermittently for reasons unrelated to the code being tested — timing sensitivity, test-order dependence, an unmocked real network call) sit unaddressed with everyone just re-running CI until it passes. A tolerated flaky test trains the team to treat *all* red CI runs as noise, which is exactly the failure mode that lets a genuine regression slip through unnoticed.
- **DO:** Run test suites in parallel (most modern runners — Vitest, Jest — do this by default across worker processes) to keep feedback loops fast, but make sure tests are actually safe to run in parallel first (no shared mutable external state, no hardcoded shared ports/file paths across test files) — parallelizing an unsafe suite just turns "sometimes flaky" into "flaky more often."
- **DON'T:** Hardcode a fixed port, file path, or database name across multiple test files/suites that might run concurrently — use a dynamically-assigned port (port `0` lets the OS pick a free one), a per-test-run-unique temp directory, or a per-worker-isolated test database/schema, so parallel test workers can't collide with each other.
- **DO:** Use test data builders/factories (a `buildUser({ overrides })` helper with sensible defaults for every field) instead of copy-pasting a large literal fixture object into every test file. When a shared entity shape gains a new required field, a builder needs updating in one place; copy-pasted literals need updating everywhere they were pasted, and it's easy to miss one.
```ts
function buildUser(overrides: Partial<User> = {}): User {
  return {
    id: crypto.randomUUID(),
    email: "test@example.com",
    role: "member",
    createdAt: new Date("2024-01-01"),
    ...overrides,
  };
}

// call sites stay focused on what's actually relevant to each specific test
const adminUser = buildUser({ role: "admin" });
```
- **DO:** Seed and tear down any test database/external state deterministically for integration tests — a `beforeEach`/`afterEach` (or a transaction rolled back after each test) that resets to a known baseline — rather than letting tests accumulate state across runs, which produces results that depend on what order tests happened to run in and what ran before them.
- **DON'T:** Point automated tests at a shared, persistent staging/development database that other people or processes are also actively using. Concurrent, unrelated activity on a shared database is a direct source of flaky, hard-to-reproduce test failures — use an isolated, ephemeral test database per CI run (or per test file/worker) instead.
- **DO:** Set an explicit, reasonable timeout on tests that involve real I/O (even mocked/local I/O), so a hung test fails fast with a clear "timed out" message instead of stalling the entire CI run until a much longer default/global timeout is hit.

## Node.js Server Patterns

### Serving Static Assets & Cache Headers

- **DO:** Set long `Cache-Control: public, max-age=31536000, immutable` headers on static assets that are content-hashed/fingerprinted in their filename (`app.a3f9c1.js`) — since the filename itself changes whenever the content changes, it's safe to tell browsers and CDNs to cache the file essentially forever, and any new deployment simply references a new, differently-named file rather than needing the old one invalidated.
```js
app.use("/assets", express.static("dist/assets", {
  maxAge: "1y",
  immutable: true,
  etag: false, // unnecessary alongside a long immutable max-age for hashed filenames
}));

app.use("/", express.static("dist", {
  maxAge: 0, // HTML entry points should generally not be cached long, since they reference the hashed assets
  etag: true,
}));
```
- **DON'T:** Apply the same long-lived, aggressive cache header to non-fingerprinted files (a plain `styles.css` with no content hash in its name, or the HTML entry point that references the hashed asset filenames) — caching these aggressively means a new deployment's changes may not reach users for as long as the cache header specifies, since browsers/CDNs won't know to re-fetch a same-named file that's actually changed.
- **DO:** Use `ETag`/`Last-Modified`-based conditional requests (`If-None-Match`/`If-Modified-Since`, returning `304 Not Modified` when unchanged) for assets that can't be content-hashed in their filename, so a client that already has the current version avoids re-downloading the full payload while still getting notified promptly once it actually changes.
- **DON'T:** Serve static assets directly from the same Node.js process handling dynamic application logic in a high-traffic production deployment without considering offloading them to a CDN or a dedicated static-file server/object storage — Node.js can serve static files adequately, but a CDN is typically far more efficient at it (edge caching close to the client, no application-server CPU/memory spent per static request) and frees the application process's capacity for the dynamic work only it can do.
- **DO:** Set `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, and correct `Content-Type` headers on served static assets, especially user-uploaded ones — this ties back to the Output Encoding & XSS Prevention guidance above, since static-file serving is a common place those protections get accidentally skipped because "it's just a file server."

### Streams

- **DO:** Use Node.js streams (`Readable`, `Writable`, `Transform`, `Duplex`) for processing large files, large HTTP payloads, or any data set too big to comfortably hold in memory all at once. Streams process data in chunks with backpressure built in, keeping memory usage bounded regardless of the total data size.
```js
// Bad — loads the entire (potentially huge) file into memory at once
import { readFileSync, writeFileSync } from "node:fs";
const data = readFileSync("huge-log.csv", "utf8");
writeFileSync("huge-log-uppercase.csv", data.toUpperCase());

// Good — processes the file in bounded chunks
import { createReadStream, createWriteStream } from "node:fs";
import { Transform } from "node:stream";
import { pipeline } from "node:stream/promises";

const upper = new Transform({
  transform(chunk, _enc, callback) {
    callback(null, chunk.toString().toUpperCase());
  },
});

await pipeline(
  createReadStream("huge-log.csv"),
  upper,
  createWriteStream("huge-log-uppercase.csv"),
);
```
- **DO:** Use `stream.pipeline()` (or its Promise-based version, `stream/promises`' `pipeline`) to connect streams together, rather than manually chaining `.pipe()` calls. `pipeline` correctly propagates errors from any stage to a single handler and guarantees every stream in the chain is properly closed/destroyed when any stage fails — manual `.pipe()` chains don't do either of these by default and are a common source of leaked file descriptors on error.
```js
// Bad — an error partway through leaves streams open and unhandled
readStream.pipe(transformStream).pipe(writeStream);

// Good
import { pipeline } from "node:stream/promises";
try {
  await pipeline(readStream, transformStream, writeStream);
} catch (err) {
  logger.error("pipeline failed", err);
}
```
- **DON'T:** Ignore backpressure signals when manually writing to a stream (`writable.write(chunk)` returning `false`) in a tight loop — writing faster than the destination can drain buffers the excess data entirely in memory, which defeats the entire purpose of streaming and can exhaust memory on a large enough input.
```js
// Bad — ignores backpressure; can buffer unbounded data in memory
for (const chunk of hugeChunkList) {
  writable.write(chunk);
}

// Good — respects backpressure by waiting for 'drain'
for (const chunk of hugeChunkList) {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    await new Promise((resolve) => writable.once("drain", resolve));
  }
}
```
- **DO:** Prefer async iteration (`for await (const chunk of readable)`) for consuming a readable stream when a simple sequential read is all you need — it's easier to reason about and to wrap in `try`/`catch` than manually attaching `data`/`end`/`error` event listeners.
- **DON'T:** Attach a `data` event listener to a stream without also handling the `error` event. An unhandled `error` event on a stream (unlike most EventEmitters) throws and can crash the process if there's no listener for it — every stream you consume manually needs an `error` handler.
- **DO:** Use `Readable.from()` to convert an async generator, iterable, or array into a proper Node stream when you need to feed programmatically-generated data into a stream pipeline (e.g., streaming rows from a database cursor into an HTTP response).

### Middleware & Request Handling Patterns

- **DO:** Validate request input (body, query parameters, path parameters, headers) against an explicit schema at the top of every route handler (or via a validation middleware applied per-route) before any business logic runs, rejecting malformed requests immediately with a clear 400-level error. Validating late, or not at all, lets malformed data reach deeper layers where a failure is harder to trace back to "the request was bad" versus "the code is broken."
```js
import { z } from "zod";

const CreateOrderSchema = z.object({
  items: z.array(z.object({ sku: z.string(), quantity: z.number().int().positive() })).min(1),
  shippingAddressId: z.string().uuid(),
});

app.post("/orders", async (req, res, next) => {
  const parseResult = CreateOrderSchema.safeParse(req.body);
  if (!parseResult.success) {
    return res.status(400).json({ error: "invalid request", issues: parseResult.error.issues });
  }
  const order = await createOrder(parseResult.data);
  res.status(201).json(order);
});
```
- **DO:** Order middleware deliberately and understand what each layer depends on running before it — body-parsing before anything that reads `req.body`, authentication before authorization, request-logging early enough to capture every request including ones that fail validation later. Middleware order bugs (a route handler reading `req.body` before the body-parser middleware has run) are a common source of "works for this route but not that one" confusion, since Express applies middleware in registration order per matched path.
- **DON'T:** Register a global body-parsing middleware with no size limit, and don't apply an expensive middleware (heavy validation, a synchronous parsing step) to routes that don't need it — apply middleware as narrowly as its actual purpose requires (per-route or per-router) rather than blanket-applying everything globally "to be safe."
- **DO:** Implement a dedicated health-check endpoint (`/healthz`, `/health`) that verifies the process can actually serve traffic — ideally checking that critical dependencies (the database connection, a required downstream service) are reachable, not just that the HTTP server itself is up — so an orchestrator (Kubernetes liveness/readiness probes, a load balancer's health check) can correctly detect and route around an unhealthy instance.
- **DON'T:** Make a health-check endpoint itself expensive or slow (running a full data-integrity check, hitting every downstream dependency on every single probe) — orchestrators typically poll health checks frequently, so an expensive check adds unnecessary sustained load and can itself become a source of false-negative health failures under load.
- **DO:** Set `app.set("trust proxy", ...)` correctly (with the actual number of trusted proxy hops, or a specific trusted IP range — not a blanket `true` in an environment where it isn't warranted) when running behind a reverse proxy/load balancer, so that `req.ip` and `req.protocol` reflect the real client rather than the proxy's own address — getting this wrong breaks both IP-based rate limiting and any logic that depends on knowing whether the original request was HTTPS.
- **DON'T:** Trust client-supplied headers like `X-Forwarded-For` for security-relevant decisions (rate limiting by IP, geolocation-based access control) without configuring the framework's trusted-proxy settings correctly — an untrusted-by-default `X-Forwarded-For` header can be freely spoofed by any client, letting an attacker claim to be any IP address they choose.
- **DO:** Apply consistent, versioned API route conventions (`/api/v1/...`) from the start of a project, and plan a deliberate deprecation/migration path before introducing a breaking change to a public or widely-consumed internal API, rather than mutating an existing endpoint's response shape in place and breaking every existing consumer with no warning.

### Synchronous Crypto & Hashing Blocking the Event Loop

- **DON'T:** Call a synchronous password-hashing function (`bcrypt.hashSync()`, `bcrypt.compareSync()`) inside a request handler on a server that needs to serve concurrent traffic. Password hashing algorithms are deliberately, intentionally slow (that's what makes them resistant to brute-forcing), and a synchronous call blocks the entire event loop for that duration — meaning one user's login request can measurably stall every other concurrent request being handled by the same process for tens to hundreds of milliseconds.
```js
// Bad — blocks the event loop for the full hashing duration on every login attempt
app.post("/login", (req, res) => {
  const valid = bcrypt.compareSync(req.body.password, user.passwordHash);
  res.json({ valid });
});

// Good — the async variant offloads the actual hashing work off the main event loop thread
app.post("/login", async (req, res) => {
  const valid = await bcrypt.compare(req.body.password, user.passwordHash);
  res.json({ valid });
});
```
- **DO:** Use the async variant of any CPU-intensive cryptographic operation a library provides (`bcrypt.hash`/`bcrypt.compare` rather than their `*Sync` counterparts, `crypto.pbkdf2` rather than `crypto.pbkdf2Sync`, `crypto.scrypt` rather than `crypto.scryptSync`) in any code path that runs inside a request-serving process — these async variants use Node's internal thread pool (`libuv`'s worker threads) to run the expensive computation off the main event loop thread, keeping the server responsive to other concurrent requests while the hash computes.
- **DON'T:** Assume "the function returns a Promise" automatically means "this doesn't block the event loop" for every crypto/hashing library — check the specific library's documentation for whether its async API genuinely offloads work to a worker thread versus just wrapping a synchronous computation in a `Promise.resolve()` for API convenience alone; the two look identical at the call site but have very different effects on server responsiveness under load.
- **DO:** Be aware that Node's default `libuv` thread pool size (4 threads by default) is a shared, finite resource used by several kinds of async work (async crypto, some `fs` operations, DNS lookups on some platforms) — under a sustained high volume of concurrent password-hashing operations specifically, requests can start queuing for a free thread-pool slot even though the main event loop itself stays responsive; the thread pool size is configurable (`UV_THREADPOOL_SIZE`) if this becomes a genuine, measured bottleneck.
- **DON'T:** Reflexively increase `UV_THREADPOOL_SIZE` as a first response to slow hashing throughput without first confirming, via actual profiling, that thread-pool contention (rather than raw CPU capacity, or an unrelated bottleneck elsewhere) is the actual limiting factor — the right hashing cost parameters (bcrypt's cost factor, argon2's memory/time parameters) are themselves a deliberate security-vs-performance tradeoff that should be tuned first, independently of thread-pool sizing.

### Readiness vs. Liveness Checks

- **DO:** Expose two distinct health-check endpoints when running under an orchestrator that supports the distinction (Kubernetes' liveness and readiness probes being the most common example) — a liveness check answers "is this process alive and not deadlocked" (restart the container if this fails), while a readiness check answers "is this instance currently able to serve traffic correctly" (stop routing new traffic here if this fails, but don't necessarily restart it).
```js
app.get("/healthz/live", (req, res) => res.status(200).send("ok"));

app.get("/healthz/ready", async (req, res) => {
  try {
    await db.query("SELECT 1"); // fails fast if the DB connection is actually down
    res.status(200).send("ready");
  } catch {
    res.status(503).send("not ready");
  }
});
```
- **DON'T:** Use a single, identical health-check endpoint for both liveness and readiness purposes when the orchestrator supports distinguishing them — collapsing the two into one means a transient, recoverable dependency outage (readiness concern — should stop receiving new traffic temporarily) gets treated the same as a genuinely deadlocked process (liveness concern — should be killed and restarted), which can cause unnecessary restarts for problems a restart won't actually fix, or fail to restart a process that genuinely is stuck.
- **DO:** Make the readiness check reflect the instance's actual current ability to serve traffic correctly (can it reach the database, is a critical cache warmed, has startup initialization completed) — a readiness check that always returns `200` regardless of actual internal state defeats its entire purpose, since it stops the orchestrator from routing traffic away from an instance that genuinely can't serve it correctly right now.
- **DON'T:** Make the liveness check depend on external dependencies (a database, a downstream API) — if the liveness check fails whenever an external dependency is briefly unavailable, the orchestrator restarts the process, which does nothing to fix an external dependency outage and just adds unnecessary restart churn (and potential connection-pool warmup cost) on top of an already-degraded situation. Liveness should check only whether *this process itself* is still functioning, not whether everything it depends on is currently healthy.

### Event Loop Pitfalls

- **DO:** Understand that Node.js runs JavaScript on a single thread, and that any long-running synchronous computation blocks the entire event loop — no other request, timer, or I/O callback can run until it finishes. A single expensive `JSON.parse` of a huge payload, a large synchronous loop, or a heavy regex on untrusted input can stall an entire server process for every concurrent user at once.
```js
// Bad — a CPU-heavy synchronous loop blocks all other requests for its duration
app.get("/report", (req, res) => {
  let total = 0;
  for (let i = 0; i < 10_000_000_000; i++) total += i; // blocks everything
  res.json({ total });
});

// Good — offload genuinely CPU-heavy work to a worker thread
app.get("/report", async (req, res) => {
  const total = await runInWorker("./sum-worker.js", { limit: 10_000_000_000 });
  res.json({ total });
});
```
- **DO:** Move CPU-bound work (image processing, large-scale data transformation, cryptographic hashing of large inputs, complex regex on large/untrusted strings) into Worker Threads (`node:worker_threads`) or a separate process (a queue-backed job worker), instead of running it inline on the main event loop of a request-serving process.
- **DON'T:** Confuse "asynchronous" with "non-blocking automatically." `await`-ing a Promise correctly yields the event loop to other work while waiting for genuine I/O, but the synchronous code *between* each `await` still runs to completion on the main thread and blocks everything else during that time — wrapping a CPU-heavy synchronous function in an `async` wrapper does not make it non-blocking.
- **DO:** Break up large synchronous loops that must run on the main thread using `setImmediate()` (or, for very fine-grained yielding, chunking with periodic `await new Promise(resolve => setImmediate(resolve))`) so the event loop gets a chance to process other pending I/O and timers between chunks, if moving the work to a worker thread genuinely isn't feasible.
- **DON'T:** Use `setTimeout(fn, 0)` when you actually want to yield to I/O as soon as possible — prefer `setImmediate(fn)` for that purpose in Node.js; `setImmediate` callbacks run after the current poll phase completes, generally sooner and more predictably than a zero-delay timer, which is subject to timer-phase scheduling and a minimum clamped delay.
- **DO:** Be aware of `process.nextTick()`'s special, higher-priority queue — it runs before Promise microtasks and before the event loop continues to the next phase. Recursively/repeatedly scheduling work via `process.nextTick()` can starve I/O entirely (Node.js will keep draining the nextTick queue before doing anything else), which is a real, documented way to accidentally hang a server that looks alive but serves nothing.
- **DON'T:** Perform synchronous filesystem operations (`fs.readFileSync`, `fs.writeFileSync`, `fs.existsSync`) inside a request handler on a server that needs to serve concurrent requests. Sync fs calls block the event loop for the duration of the disk I/O; use the Promise-based (`fs/promises`) or callback-based async variants instead, reserving sync fs calls for one-time startup/CLI-script code where blocking briefly is genuinely fine.
```js
// Bad — blocks the event loop for every request while reading from disk
app.get("/config", (req, res) => {
  const config = fs.readFileSync("./config.json", "utf8");
  res.json(JSON.parse(config));
});

// Good
import { readFile } from "node:fs/promises";
app.get("/config", async (req, res) => {
  const config = await readFile("./config.json", "utf8");
  res.json(JSON.parse(config));
});
```
- **DO:** Watch for accidental synchronous work hiding inside seemingly-async APIs — some third-party libraries expose a Promise-returning wrapper around fundamentally synchronous, CPU-heavy internals (certain synchronous crypto operations, some templating engines, some image libraries without native async bindings). "Returns a Promise" is not the same guarantee as "doesn't block the event loop" — check what the library actually does internally for expensive operations.

### Guaranteed Resource Cleanup Patterns

- **DO:** Release every acquired resource — a database client checked out from a pool, a file handle, an acquired lock — in a `finally` block (or the equivalent structured pattern for the resource type), so cleanup runs whether the code in between succeeded, threw, or returned early. A resource release that only happens on the success path silently leaks that resource on every error path, which is exactly the kind of leak that only becomes visible after enough accumulated failures exhaust the pool.
```js
async function withDbClient(fn) {
  const client = await pool.connect(); // check out a connection from the pool
  try {
    return await fn(client);
  } finally {
    client.release(); // always returned to the pool, success or failure
  }
}
```
- **DON'T:** Release a resource manually at both "the end of the success path" and separately inside a `catch` block as two different lines of near-duplicate cleanup code — this duplication is exactly the kind of thing that's easy to update in one place and forget in the other during a later edit; a single `finally` block (or an equivalent guaranteed-cleanup construct) is both shorter and structurally guaranteed to run in every case, removing the duplication entirely.
- **DO:** Reach for the `using`/`await using` declarations (TC39's explicit resource management proposal, landing in modern TypeScript/JS runtimes) where available, for resources that implement the `Symbol.dispose`/`Symbol.asyncDispose` protocol — this gives automatic, guaranteed cleanup tied to the enclosing block's scope without needing to manually write the `try`/`finally` wrapper at every call site.
- **DON'T:** Assume a resource is cleaned up just because the function that acquired it returned — verify (via the specific library/API's documentation) whether closing/releasing is actually automatic or whether it genuinely requires an explicit call; assuming automatic cleanup that isn't real is a common way slow resource leaks get introduced by code that "looks fine" and passes casual testing, since a leak from unclosed resources usually only becomes visible under sustained load, not in a quick manual check.
- **DO:** Write a specific test (or at minimum a manual verification under load) confirming that a resource-acquiring code path actually releases what it acquires on both success and failure — for a database connection pool specifically, a load test that repeatedly exercises an error path is a direct, concrete way to catch an unreleased-connection leak before it reaches production, where it would otherwise only surface once the pool is fully exhausted.

### Memory Leaks

- **DO:** Remove event listeners when the object that registered them is done using the emitter, especially for long-lived emitters (a shared `EventEmitter`, `process`, a persistent WebSocket connection) that outlive the individual consumers attaching listeners to them. Each unremoved listener holds a reference to its closure, which holds references to everything that closure captured — a slow, classic Node.js memory leak.
```js
// Bad — every request adds a new listener that's never removed
app.get("/subscribe", (req, res) => {
  eventBus.on("update", (data) => res.write(JSON.stringify(data)));
});

// Good — listener is cleaned up when the connection closes
app.get("/subscribe", (req, res) => {
  const onUpdate = (data) => res.write(JSON.stringify(data));
  eventBus.on("update", onUpdate);
  req.on("close", () => eventBus.off("update", onUpdate));
});
```
- **DON'T:** Ignore Node's `MaxListenersExceededWarning`. It's Node's own leak detector telling you more than the default 10 listeners have accumulated on a single `EventEmitter` — almost always because listeners are being added repeatedly (e.g., inside a loop or a per-request handler) without ever being removed, rather than a case that genuinely needs `emitter.setMaxListeners()` raised.
- **DO:** Clear every `setInterval`/`setTimeout` you no longer need with `clearInterval`/`clearTimeout`, especially timers tied to a specific request, connection, or component lifecycle. An uncleared `setInterval` keeps running (and keeps the process alive, and keeps its closure's captured variables alive) forever, or until the process exits.
- **DON'T:** Let an in-memory cache (a plain `Map`/object used as a cache) grow without a bound, an eviction policy (LRU, TTL), or a maximum size. An unbounded cache is a memory leak by another name — it degrades gracefully until it doesn't, typically failing under exactly the sustained production traffic that makes it hardest to reproduce and debug.
```js
// Bad — grows forever, one entry per unique key ever seen
const cache = new Map();
function getCached(key, compute) {
  if (!cache.has(key)) cache.set(key, compute());
  return cache.get(key);
}

// Good — bounded with an LRU eviction policy
import { LRUCache } from "lru-cache";
const cache = new LRUCache({ max: 500, ttl: 1000 * 60 * 5 });
```
- **DO:** Use `WeakMap`/`WeakSet` instead of `Map`/`Set` when associating extra data with an object without preventing that object from being garbage-collected once nothing else references it — e.g., caching metadata keyed by a DOM node or a request object. A regular `Map` keyed by object reference keeps every key object alive for as long as the map exists, even if nothing else in the program still needs it.
- **DON'T:** Hold references to large objects (a full request/response object, a large parsed buffer) longer than necessary by capturing them in a closure that outlives their useful life (e.g., stashing `req` in a module-level array "just in case" for debugging). Trim what a long-lived closure captures to only what it actually needs going forward.
- **DO:** Use Node's `--inspect`/Chrome DevTools memory profiler, or a heap-snapshot comparison across two points in time under load, to actually diagnose a suspected leak rather than guessing. Compare retained-size growth of the same object type across snapshots taken minutes apart under steady load — genuine leaks show a specific type of object count growing without bound.
- **DON'T:** Restart a leaking process on a timer (a cron-scheduled restart, a "restart every 6 hours" hack) as a substitute for finding and fixing the actual leak. It's a legitimate short-term mitigation to buy time, but it masks the underlying bug, and the leak will eventually outpace whatever restart interval was chosen as traffic grows.

### Process Management, Clustering & Graceful Shutdown

- **DO:** Use a process manager (PM2, systemd, or your orchestrator's native restart policy — Kubernetes, ECS) to automatically restart a Node.js process if it crashes, rather than relying on the process staying alive indefinitely with no supervision. A single uncaught exception should not mean permanent downtime until a human notices.
- **DO:** Implement graceful shutdown — on `SIGTERM`/`SIGINT`, stop accepting new connections, let in-flight requests finish (within a bounded timeout), close database connections and other resources cleanly, then exit. Orchestrators like Kubernetes send `SIGTERM` before forcibly killing a container; a process that ignores it drops in-flight requests and can leave connections/transactions in a bad state.
```js
const server = app.listen(PORT);

async function shutdown(signal) {
  logger.info(`${signal} received, shutting down gracefully`);
  server.close(() => logger.info("HTTP server closed"));
  await db.close();
  await redisClient.quit();
  process.exit(0);
}

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```
- **DON'T:** Call `process.exit()` immediately inside a signal handler without first closing open connections/servers — this abruptly kills in-flight requests and can leave the database or message queue in an inconsistent state (a half-written transaction, an unacknowledged queue message that then gets redelivered and double-processed).
- **DO:** Use the built-in `cluster` module or, more commonly today, a process manager/orchestrator's horizontal scaling (multiple container replicas) to use more than one CPU core, since a single Node.js process only uses one core for JavaScript execution. A single-instance Node server leaves the rest of a multi-core machine's compute capacity completely idle for CPU-bound work.
- **DON'T:** Assume clustering/horizontal scaling automatically shares in-memory state (caches, in-process rate limiters, WebSocket connection registries) across instances — each process/replica has its own separate memory. Use a shared external store (Redis, a database) for any state that needs to be consistent across multiple instances.
- **DO:** Set appropriate resource limits (`--max-old-space-size` for the V8 heap, container memory/CPU limits) that match the actual expected workload, and monitor them — an unconfigured Node process left to a container orchestrator's default memory limit can be OOM-killed under load with a strategy generic enough to have no relationship to your app's actual footprint.

### File Uploads: Streaming, Validation & Storage

- **DO:** Stream file uploads directly to their final destination (disk, or an object storage service like S3 via a streaming upload) instead of buffering the entire uploaded file into memory first — a naive in-memory upload handler means the size of concurrent uploads the server can handle is bounded by available RAM, and a handful of large concurrent uploads can exhaust memory and take the whole process down.
- **DO:** Enforce a maximum upload size explicitly at the middleware/framework level (`multer`'s `limits.fileSize`, or equivalent) rather than relying on application logic to check the size only after the entire file has already been received — checking after the fact still lets an attacker force the server to receive (and thus spend bandwidth and time on) an arbitrarily large payload before it gets rejected.
- **DON'T:** Trust a file's declared MIME type (from its `Content-Type` header or its extension) as proof of its actual content — validate the real file type by inspecting its content (magic-byte/file-signature checking, via a library like `file-type`) for any upload whose type matters for how it will later be processed or served, since a malicious upload can freely lie about its declared type.
```js
import { fileTypeFromBuffer } from "file-type";

const detected = await fileTypeFromBuffer(uploadedBuffer);
if (!detected || !["image/png", "image/jpeg"].includes(detected.mime)) {
  throw new ValidationError(["file must be a PNG or JPEG image"]);
}
```
- **DO:** Store uploaded files outside the web server's publicly-served static directory (or in dedicated object storage) and serve them back through an application-controlled endpoint (or signed, time-limited URLs from the object storage provider) rather than directly from a public path — this keeps the application in control of access checks, and avoids a class of vulnerability where an uploaded file with executable content ends up directly, publicly servable at a predictable URL.
- **DON'T:** Skip virus/malware scanning for uploads in any application that later serves those files back to other users (not just the uploader) — an uploaded file is untrusted content from the uploader's perspective just as much as any other user input, and a file-sharing feature with no scanning is a plausible malware-distribution vector through your own infrastructure.

### Server-Sent Events (SSE) for One-Way Streaming

- **DO:** Reach for Server-Sent Events (a plain long-lived HTTP response with `Content-Type: text/event-stream`) instead of WebSockets when the actual requirement is one-way, server-to-client streaming (live progress updates, a notification feed, streaming an LLM's response token by token) — SSE runs over plain HTTP (simpler to proxy/load-balance than WebSocket's protocol upgrade, works through more infrastructure without special configuration), reconnects automatically on the client side via the native `EventSource` API, and avoids the added complexity of a bidirectional protocol for a use case that never actually needed the client to push data back over the same connection.
```js
app.get("/events", (req, res) => {
  res.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    Connection: "keep-alive",
  });
  const send = (data) => res.write(`data: ${JSON.stringify(data)}\n\n`);
  const unsubscribe = eventBus.subscribe(send);
  req.on("close", unsubscribe); // clean up when the client disconnects, echoing the Streams guidance above
});
```
- **DON'T:** Reach for the added complexity of WebSockets by default for every real-time feature without first checking whether the actual data flow is genuinely bidirectional — a chat application or a collaborative editor genuinely needs two-way communication and is a good fit for WebSockets; a live dashboard, a progress indicator, or a streaming AI response is one-way and is usually better served by SSE's simplicity.
- **DO:** Set the same connection-timeout and keep-alive considerations covered in the Server Timeout Configuration section for any long-lived SSE connection, and clean up the associated event-bus subscription (as shown above) on client disconnect — an SSE connection is just as much a long-lived resource as a WebSocket connection, and the same leaked-listener/leaked-connection risks covered in the Memory Leaks section apply equally to it.
- **DON'T:** Forget that some intermediary infrastructure (certain proxies, older load balancers) buffers HTTP responses by default, which can prevent SSE events from actually reaching the client in a timely, incremental fashion — verify that anything sitting between the server and the client is configured (or bypassed) to support genuinely streamed, unbuffered responses, since an SSE implementation that works perfectly in local development can silently degrade into a delayed, batched delivery once real production infrastructure is involved.

### Buffers & Binary Data

- **DO:** Use `Buffer.from(data, encoding)` with an explicit encoding when converting between strings and binary data, and be deliberate about which encoding is correct for the data at hand (`"utf8"` for text, `"base64"` for base64-encoded payloads, `"hex"` for hex strings) — omitting the encoding silently defaults to `"utf8"`, which produces garbage output if the underlying data was actually base64 or binary.
```js
// Bad — assumes utf8; corrupts genuinely binary/base64 data
const buf = Buffer.from(uploadedImageBase64);

// Good — explicit about what encoding the input actually is
const buf = Buffer.from(uploadedImageBase64, "base64");
```
- **DON'T:** Concatenate `Buffer`s with `+` or by converting them to strings and back — this can corrupt binary data (especially multi-byte UTF-8 sequences split across buffer chunk boundaries) and is also needlessly slow. Use `Buffer.concat([buf1, buf2])` to join buffers directly, which operates on the raw bytes without any string-encoding round-trip.
- **DO:** Account for multi-byte character boundaries when processing a stream of text chunk-by-chunk — a chunk boundary can fall in the middle of a multi-byte UTF-8 character, and naively calling `.toString("utf8")` on each raw chunk independently can produce corrupted/replacement characters at chunk edges. Use a `StringDecoder` (`node:string_decoder`) or an equivalent stream transform that's aware of encoding boundaries when converting a streamed buffer to text incrementally.
- **DON'T:** Assume a `Buffer`'s `.length` equals the number of *characters* in the text it represents — `.length` is the byte length, which differs from character count for any non-ASCII text (a single emoji or accented character can be multiple bytes in UTF-8). Use `[...str].length` or `Intl.Segmenter` for actual grapheme/character counting, and `Buffer.byteLength(str)` specifically when you need the byte size of a string.
- **DO:** Use typed arrays (`Uint8Array`, `Float64Array`, etc.) and `ArrayBuffer`/`DataView` directly, rather than `Buffer`, for binary data code paths that need to run identically in both Node.js and browser environments — `Buffer` is Node-specific (though it's a `Uint8Array` subclass), while typed arrays are a shared, standard JS feature available in both runtimes.

### HTTP Clients: Fetch, Retries & Timeouts

- **DO:** Use the built-in global `fetch` (available natively from Node 18+ without any dependency) for new Node.js server code making outbound HTTP requests, rather than defaulting to a third-party HTTP client out of habit — it covers the common cases natively, and reduces one more dependency (and one more thing to keep patched) in the project.
- **DO:** Reach for a dedicated HTTP client library (`got`, `undici`'s lower-level API, or a well-maintained wrapper) when you need capabilities `fetch` doesn't provide out of the box — built-in automatic retries with backoff, HTTP/2 support with fine control, detailed request/response interceptors, or connection-pooling tuning beyond what the platform default offers.
- **DON'T:** Assume `fetch` throws on a non-2xx HTTP status code — it doesn't. `fetch`'s Promise only rejects on a genuine network-level failure (DNS failure, connection refused, aborted request); a `404` or `500` response is still a "successful" fetch from the Promise's point of view and must be checked explicitly via `response.ok` or `response.status`.
```js
// Bad — a 500 error response is silently treated as success
const response = await fetch(url);
const data = await response.json(); // may throw trying to parse an HTML error page as JSON

// Good
const response = await fetch(url);
if (!response.ok) {
  throw new Error(`request failed: ${response.status} ${response.statusText}`);
}
const data = await response.json();
```
- **DO:** Set an explicit timeout (via `AbortSignal.timeout(ms)`, available natively, or a manually wired `AbortController`) on every outbound HTTP call to another service — an HTTP client with no timeout configured will, by default, wait indefinitely for a stalled or unresponsive server, which can cascade into your own service hanging and exhausting its own concurrency/connection limits.
```js
const response = await fetch(url, { signal: AbortSignal.timeout(5000) });
```
- **DON'T:** Retry every failed HTTP request unconditionally — distinguish idempotent methods (`GET`, `PUT`, `DELETE`) which are generally safe to retry, from non-idempotent ones (`POST` creating a new resource) where a blind retry after an ambiguous failure (the request may have actually succeeded server-side before the response was lost) can create a duplicate resource. For non-idempotent operations that must be retried safely, use an idempotency key the server can deduplicate on.
- **DO:** Read and respect a `Retry-After` header when a downstream API returns a `429 Too Many Requests` or `503 Service Unavailable` response — it tells you exactly how long the server wants you to wait before retrying, which is more accurate and more considerate of the downstream service than a fixed or purely exponential backoff computed with no information from the server at all.

### WebSockets & Real-Time Connections

- **DO:** Implement a heartbeat/ping-pong mechanism on long-lived WebSocket connections and close connections that stop responding to pings within a reasonable timeout. TCP connections can silently die (a client's network drops without a clean close handshake) without either side receiving a close event, leaving a "zombie" connection that the server thinks is still open and keeps resources allocated for indefinitely.
```js
wss.on("connection", (ws) => {
  ws.isAlive = true;
  ws.on("pong", () => { ws.isAlive = true; });
});

const interval = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) return ws.terminate(); // no pong since last check — assume dead
    ws.isAlive = false;
    ws.ping();
  });
}, 30_000);
```
- **DON'T:** Store WebSocket connection objects (or any per-connection state) in a plain in-process `Map`/array as the sole source of truth in a horizontally-scaled deployment — a message meant for a specific connected client only reaches them if the request handling that triggers the message happens to run on the same server instance/process that client is connected to. Use a shared pub/sub layer (Redis Pub/Sub, a message broker) to fan messages out across all instances when any of them might need to reach a client connected to a different instance.
- **DO:** Apply the same input validation discipline to WebSocket messages as to HTTP request bodies — parse and validate every incoming message against an expected schema before acting on it; a WebSocket connection is just as much an untrusted-input boundary as an HTTP endpoint, and it's easy to forget that once the initial connection/auth handshake feels "trusted."
- **DON'T:** Skip authentication/authorization on WebSocket connections just because the initial HTTP upgrade request happened over an authenticated session — verify identity and permissions explicitly as part of the WebSocket connection handshake (or via a signed, short-lived connection token), and re-verify authorization for actions requested over the socket, the same as you would for any other authenticated action.
- **DO:** Apply backpressure-aware sending on WebSocket connections — check `ws.bufferedAmount` before sending more data to a slow client, and consider dropping or coalescing non-critical messages (like frequent state updates) rather than letting the send buffer grow unbounded when a client can't keep up, which is the WebSocket-layer equivalent of the stream backpressure issue covered earlier.

### Scheduled Jobs, Idempotency & Distributed Locks

- **DON'T:** Run a `setInterval`-based or `node-cron`-based scheduled job directly inside every instance of a horizontally-scaled service without coordination — if the service runs as multiple replicas, each replica's own in-process scheduler fires independently, so the "single" job actually runs once per replica simultaneously, which is rarely the intent for something like "send the daily digest email."
- **DO:** Use a dedicated job scheduler with built-in leader election or locking (a managed cron service, a queue-based job system like BullMQ backed by Redis, or a database-based distributed lock) when a scheduled task must run exactly once across a fleet of replicas, rather than relying on in-process timers and hoping only one replica happens to be doing the work.
- **DO:** Design every job handler (scheduled or queue-consumed) to be idempotent — safe to run more than once with the same input without causing duplicate side effects — since at-least-once delivery/execution semantics are the norm for most real-world job queues and schedulers; a "process order" job that gets accidentally retried should not charge the customer twice.
```js
// Idempotent: uses a unique key to guarantee a side effect happens at most once
async function processPayment(orderId, idempotencyKey) {
  const existing = await db.payments.findByIdempotencyKey(idempotencyKey);
  if (existing) return existing; // already processed; return the prior result, don't redo it
  return db.payments.create({ orderId, idempotencyKey, status: "completed" });
}
```
- **DON'T:** Assume a job "probably didn't run twice" without an explicit idempotency mechanism (a unique constraint in the database, a dedicated idempotency-key table, a distributed lock held for the job's duration) — "probably" is not a guarantee, and duplicate execution of jobs with real side effects (charging money, sending emails, provisioning resources) is a recurring, costly production incident category.
- **DO:** Use a proper distributed lock (Redis-based via a well-tested library implementing something like the Redlock algorithm, or a database advisory lock) — not a hand-rolled "check a flag, then set it" pattern — when a critical section genuinely must be executed by only one process at a time across a fleet, since a naive check-then-set has an unavoidable race window between the check and the set that a real lock implementation is specifically designed to close.

### Multi-Tenant Data Isolation

- **DO:** Scope every single database query in a multi-tenant application by tenant ID explicitly, and make it structurally difficult to forget — a repository-layer helper that automatically injects the current tenant's filter, a query-builder wrapper that requires a tenant context to construct at all, or (for the strongest guarantee) database-level row-level security tied to the current connection's tenant context — rather than relying on every individual query author to remember to add a `WHERE tenant_id = ?` clause by hand every single time.
```js
// Bad — relies on every developer remembering the tenant filter on every query, forever
const orders = await db.query("SELECT * FROM orders WHERE customer_id = $1", [customerId]);

// Better — the tenant scoping is structural, not optional
class TenantScopedDb {
  constructor(private readonly tenantId: string, private readonly pool: Pool) {}
  async findOrders(customerId: string) {
    return this.pool.query(
      "SELECT * FROM orders WHERE tenant_id = $1 AND customer_id = $2",
      [this.tenantId, customerId],
    );
  }
}
```
- **DON'T:** Trust a tenant identifier supplied directly by the client (a `tenantId` in the request body or a query parameter) as the source of truth for which tenant's data to access — derive the current tenant from the authenticated session/token instead, so a client can't simply pass a different tenant's ID and access their data. If a client-supplied tenant identifier is used at all, it must be cross-checked against what the authenticated user is actually authorized to access, every time.
- **DO:** Write and run automated tests specifically for cross-tenant isolation — a test that creates data under two different tenants and asserts that tenant A's authenticated requests never return tenant B's data — treating this as a first-class, explicitly-verified security property of the system, not an incidental consequence of "the queries look right."
- **DON'T:** Assume a shared cache (Redis, an in-process cache) is automatically tenant-safe just because the database queries are — cache keys need the same explicit tenant scoping (`tenant:${tenantId}:orders:${orderId}`, not just `orders:${orderId}`) or one tenant's cached data can be served to a different tenant's request that happens to compute the same unscoped cache key.
- **DO:** Consider database-level enforcement (Postgres row-level security policies tied to a session variable set per-connection/per-request, or fully separate databases/schemas per tenant for the strongest isolation guarantee) for applications where a single missed application-level tenant filter would be a severe incident — application-level discipline alone means every single query, forever, across every future contributor, has to get it right; a database-level enforcement layer fails closed even when application code gets it wrong.

### Application Structure & Layering

- **DO:** Separate a server application into distinct layers with clear responsibilities — route/controller (parses and validates the HTTP request, calls the appropriate service, formats the response), service/business-logic (implements the actual domain rules, has no knowledge of HTTP), and data-access/repository (talks to the database, has no knowledge of business rules) — so each layer can be tested, reasoned about, and changed independently of the others.
- **DON'T:** Write "fat" route handlers that mix request parsing, business logic, and direct database queries all in one function body. This is one of the most common structural problems in real-world (and AI-generated) Express/Fastify code — it works for a simple CRUD endpoint, but it means business logic can't be unit-tested without spinning up an HTTP request, and it can't be reused if the same logic is later needed from a queue consumer or a CLI script.
```js
// Bad — parsing, validation, business logic, and DB access all tangled in one handler
app.post("/orders", async (req, res) => {
  if (!req.body.items || req.body.items.length === 0) return res.status(400).send("no items");
  let total = 0;
  for (const item of req.body.items) total += item.price * item.quantity;
  const order = await db.query("INSERT INTO orders (total) VALUES ($1) RETURNING *", [total]);
  res.json(order.rows[0]);
});

// Good — thin handler delegates to a testable, HTTP-agnostic service function
app.post("/orders", async (req, res, next) => {
  try {
    const parsed = CreateOrderSchema.parse(req.body);
    const order = await orderService.createOrder(parsed);
    res.status(201).json(order);
  } catch (err) { next(err); }
});
```
- **DO:** Organize files primarily by feature/domain (an `orders/` directory containing that feature's routes, service, and repository together) rather than purely by technical layer (a top-level `controllers/`, `services/`, `models/` split across the whole app) once a project grows past a small size — feature-based organization keeps everything related to one change together, while a purely layer-based split spreads a single feature's code across several distant directories that all need touching for one change.
- **DON'T:** Let the data-access layer leak database-specific types/objects (a raw ORM model instance, a raw driver row object) up through the service layer and into HTTP responses unchanged — map database rows into plain domain objects/DTOs at the repository boundary, so the rest of the application isn't coupled to the specific database library's own object shapes, and so internal-only fields don't accidentally leak into an API response.
- **DO:** Keep a consistent, explicit dependency direction — routes depend on services, services depend on repositories, and never the reverse — so the codebase's dependency graph stays a predictable, one-directional layering rather than a tangle where any file might import from any other.

### Database Access Patterns: Transactions, Migrations & Query Safety

- **DO:** Wrap multiple related writes that must all succeed or all fail together in an explicit database transaction, rather than issuing them as separate, independent queries. Without a transaction, a failure partway through a multi-step write (e.g., debiting one account and crediting another) leaves the database in an inconsistent, partially-applied state that's hard to detect and even harder to safely repair after the fact.
```js
// Bad — if the second query fails, the first has already committed; money vanishes
await db.query("UPDATE accounts SET balance = balance - $1 WHERE id = $2", [amount, fromId]);
await db.query("UPDATE accounts SET balance = balance + $1 WHERE id = $2", [amount, toId]);

// Good — both succeed or both roll back together
await db.transaction(async (tx) => {
  await tx.query("UPDATE accounts SET balance = balance - $1 WHERE id = $2", [amount, fromId]);
  await tx.query("UPDATE accounts SET balance = balance + $1 WHERE id = $2", [amount, toId]);
});
```
- **DON'T:** Hold a database transaction open across a slow, unrelated operation (an outbound HTTP call to a third-party API, a long computation) — a long-held transaction holds locks and consumes a connection from the pool for its entire duration, which under load can exhaust the connection pool and cause unrelated requests to start timing out waiting for a free connection. Keep a transaction's scope limited to just the database operations that genuinely need atomicity.
- **DO:** Manage schema changes through versioned, checked-into-version-control migration files (via the ORM's migration tooling, or a dedicated tool like `node-pg-migrate`/`Knex` migrations/`Prisma Migrate`) rather than making ad hoc schema changes directly against a production database by hand. Migrations give you a reviewable, reproducible history of every schema change, and a way to apply the same sequence of changes consistently across every environment (local, staging, production).
- **DON'T:** Write a migration that's only safe to run against an empty or small table without considering its effect on a large, live production table — an `ALTER TABLE` that rewrites every row, or an index creation that locks the table for writes, can cause real production downtime on a large table even though the exact same migration runs instantly against a small development database. Consider `CREATE INDEX CONCURRENTLY`-style non-locking approaches (Postgres) and multi-step, backward-compatible migration patterns for large or high-traffic tables.
- **DO:** Choose an appropriate transaction isolation level deliberately for operations sensitive to concurrent modification (inventory decrements, double-booking prevention) rather than accepting the database's default without considering it — the default isolation level (often "read committed") does not prevent every race condition (e.g., the classic two concurrent reads-then-writes both succeeding and overselling the last unit of inventory), and some cases genuinely need row-level locking (`SELECT ... FOR UPDATE`) or a stricter isolation level.
- **DON'T:** Let an ORM's convenience features (auto-loading relations, implicit lazy-loading) hide how many actual queries a piece of code is issuing — the N+1 query problem covered in the Performance section is very often introduced exactly this way, through an ORM's lazy-loaded relation being accessed inside a loop without the developer realizing each access triggers its own query. Check your ORM's query-logging output against what you expect during development, not just against whether the code "works."

### Idempotency Keys for Public APIs

- **DO:** Accept an explicit, client-generated `Idempotency-Key` header on any public POST/PATCH endpoint that creates or mutates a resource with real-world side effects (charging a payment, sending a notification, creating an order) — the client generates a unique key once per logical operation and resends the same key on any retry of that same operation, letting the server recognize and safely respond to a retried request without repeating its side effects.
```js
app.post("/charges", async (req, res) => {
  const idempotencyKey = req.headers["idempotency-key"];
  if (!idempotencyKey) return res.status(400).json({ error: "Idempotency-Key header required" });

  const existing = await db.charges.findByIdempotencyKey(idempotencyKey);
  if (existing) return res.status(200).json(existing); // safely return the prior result

  const charge = await paymentProvider.charge(req.body.amount);
  await db.charges.create({ ...charge, idempotencyKey });
  res.status(201).json(charge);
});
```
- **DON'T:** Generate the idempotency key server-side, or derive it purely from the request body's contents — the key needs to represent "this specific logical attempt by the client," which only the client genuinely knows (it should generate a new key for each new, intentional operation, and reuse the same key only when retrying an operation that may not have completed) — a server-derived key from the payload alone can't distinguish "the client is retrying because the first response was lost" from "the client is deliberately submitting the exact same charge amount again on purpose."
- **DO:** Store idempotency keys with a reasonable expiration (they don't need to be kept forever — a window covering the realistic maximum client retry duration, often 24 hours, is typical) and scope them per-endpoint or per-operation-type, so the same key value used for two conceptually different operations doesn't collide.
- **DON'T:** Treat idempotency-key deduplication as covering every possible retry scenario automatically — this pattern specifically handles "the same logical request, retried," typically due to a network failure between client and server; it doesn't replace the broader race-condition and transactional-integrity guidance covered elsewhere in this document for concurrent requests that are genuinely different from each other.

### Pagination: Cursor vs. Offset

- **DON'T:** Use naive `OFFSET`/`LIMIT` pagination as the default for large or frequently-changing datasets — `OFFSET` forces the database to scan and discard every row before the offset on each request, which gets linearly slower as the offset grows (page 500 is dramatically slower to fetch than page 1), and rows can shift between pages if the underlying data changes between requests (a row inserted before the current offset shifts every subsequent row by one, causing a duplicated or skipped row on the next page fetched by the client).
```sql
-- Slow at high offsets: the database still has to walk through and discard 100,000 rows
SELECT * FROM orders ORDER BY created_at DESC OFFSET 100000 LIMIT 20;
```
- **DO:** Use cursor-based (a.k.a. keyset) pagination for large or actively-changing datasets — pass the last-seen row's sort-key value as an opaque cursor, and query for rows strictly beyond that value, which lets the database use an index seek instead of scanning and discarding, and remains stable even as rows are inserted/deleted during pagination.
```sql
-- Fast and stable regardless of how deep into the result set you are
SELECT * FROM orders WHERE created_at < $1 ORDER BY created_at DESC LIMIT 20;
```
```ts
interface Page<T> { items: T[]; nextCursor: string | null; }

async function listOrders(cursor?: string, limit = 20): Promise<Page<Order>> {
  const rows = await db.orders.findMany({
    where: cursor ? { createdAt: { lt: new Date(cursor) } } : {},
    orderBy: { createdAt: "desc" },
    take: limit + 1, // fetch one extra to know if there's a next page
  });
  const hasMore = rows.length > limit;
  const items = hasMore ? rows.slice(0, limit) : rows;
  return { items, nextCursor: hasMore ? items[items.length - 1].createdAt.toISOString() : null };
}
```
- **DON'T:** Use a non-unique column alone as the cursor's sort key when rows can share the same value (e.g., pure `created_at` timestamps with insufficient precision, or a value with many ties) — this can cause rows with duplicate key values to be skipped or repeated across pages. Combine the primary sort key with a unique tiebreaker (typically the primary key/id) in both the `ORDER BY` clause and the cursor comparison to guarantee a strictly consistent, gap-free ordering.
- **DO:** Reserve offset pagination for small, relatively static datasets, or specifically for UI patterns that genuinely need direct "jump to page N" access (which cursor-based pagination doesn't support as naturally) — it's a legitimate, simpler choice when its specific weaknesses (performance at scale, instability under concurrent writes) don't actually matter for the dataset in question.
- **DO:** Cap the maximum allowed page size server-side regardless of which pagination style is used (tying back to the Resource Exhaustion guidance in the Security section) — never let a client-supplied `limit`/`pageSize` parameter be unbounded, since an unbounded page size defeats the entire purpose of paginating in the first place.

### Sizing Database Connection Pools

- **DON'T:** Set a database connection pool's maximum size to an arbitrarily large number "to be safe" — each open connection consumes real memory and resources on the database server itself, and most databases have a hard maximum total connection limit; a handful of service instances each configured with an oversized pool can collectively exhaust the database's connection limit long before any individual instance is actually using anywhere near its own configured maximum.
```js
// Bad — with 20 service replicas, this alone could request up to 2000 connections
const pool = new Pool({ max: 100 });

// Better — sized with the actual fleet size and DB connection limit in mind
// (e.g., DB max_connections=200, 20 replicas → ~10 per instance, leaving headroom)
const pool = new Pool({ max: 10 });
```
- **DO:** Size a connection pool based on the actual number of concurrently-running service instances and the target database's real maximum connection limit, not based on a single instance's peak in-process concurrency in isolation — the right question is "if every replica is running at its configured pool maximum simultaneously, does the total still fit comfortably under the database's connection ceiling," not just "what's a generous-feeling number for one instance."
- **DON'T:** Assume a larger connection pool always improves throughput — beyond a certain point (often surprisingly low, and specific to the database and workload), additional concurrent connections just increase contention on the database's own internal locks/resources without adding real parallel throughput, so blindly increasing pool size can make performance worse, not better, past that point.
- **DO:** Use an external connection pooler (PgBouncer for Postgres, ProxySQL for MySQL) in front of the database when running many service instances/replicas, each with their own application-level pool — a pooler that multiplexes many logical application connections onto a smaller number of actual database connections lets you scale the number of service replicas without scaling the raw connection count the database itself has to manage linearly alongside them.
- **DO:** Monitor actual connection pool utilization (active vs. idle vs. waiting-for-a-connection) in production rather than picking a pool size once and never revisiting it — a pool that's frequently exhausted (requests queuing for a free connection) under real traffic is a concrete, measurable signal to either increase pool size (if the database has headroom) or investigate why connections are being held longer than expected (a slow query, an unreturned connection due to a bug).

### Message Queues & Background Job Processing

- **DO:** Move slow, non-request-critical work (sending an email, generating a report, processing an uploaded file) out of the synchronous request/response cycle and into a background job processed via a real queue (BullMQ, SQS-backed workers, RabbitMQ), so the original HTTP request can respond quickly and the work can be retried independently if it fails.
- **DON'T:** Fire off "background" work with a bare, un-awaited async function call inside a request handler as a substitute for a real queue (`doExpensiveThing(data); res.json({ ok: true });` with no `await`) — if the process crashes, redeploys, or the un-awaited Promise rejects, that work simply vanishes with no record it was ever supposed to happen and no automatic retry. A real queue persists the job durably before considering the request complete.
- **DO:** Configure a dead-letter queue (or equivalent) for jobs that repeatedly fail after their configured retry attempts are exhausted, so failed jobs are captured for inspection and manual/automated reprocessing rather than being silently dropped once retries run out.
- **DON'T:** Assume queue message delivery is exactly-once — most real-world queue systems (SQS, RabbitMQ with at-least-once acknowledgment, Redis-backed queues) provide at-least-once delivery, meaning a consumer must be prepared to receive and process the same message more than once. This is the same idempotency requirement covered in the Scheduled Jobs section above, and it applies equally to queue-driven job processing.
- **DO:** Set an appropriate visibility timeout / lock duration on queue jobs that's longer than the job's expected processing time, and extend it explicitly for jobs that can legitimately run long — a job whose visibility timeout expires while it's still genuinely processing gets redelivered to another consumer while the first one is still working on it, causing exactly the duplicate-processing problem idempotency is meant to guard against, but avoidable here by configuring the timeout correctly in the first place.

### CLI Tools: Argument Parsing, Exit Codes & Streams

- **DO:** Use a dedicated argument-parsing library (`commander`, `yargs`, or Node's built-in `util.parseArgs` for simple cases) for any CLI tool with more than one or two flags, rather than manually indexing into `process.argv` — a real parser handles `--flag=value` vs `--flag value` forms, short-flag combining, help-text generation, and validation consistently, all of which are easy to get subtly wrong by hand.
- **DO:** Exit with a non-zero exit code (`process.exitCode = 1`, preferring this over an immediate `process.exit(1)` so pending I/O like a final log write can flush first) whenever a CLI tool fails, and exit `0` only on genuine success — shell scripts, CI pipelines, and process orchestration all depend on exit codes to determine whether a command succeeded, and a tool that always exits `0` regardless of outcome breaks every automated caller silently.
- **DON'T:** Write a CLI tool's normal output to `stderr`, or its error/diagnostic output to `stdout` — the conventional split (`stdout` for the tool's actual output/result, meant to be piped/captured; `stderr` for logs, progress, and errors, meant to be seen but not captured) is what lets shell users reliably pipe a tool's real output (`mytool | jq .`) without diagnostic noise corrupting it.
- **DO:** Read from `stdin` when a CLI tool is designed to be usable in a Unix pipeline (`cat file.json | mytool`), and detect via `process.stdin.isTTY` whether input is actually being piped in versus an interactive terminal, so the tool can behave sensibly (e.g., print usage/help) when run interactively with no piped input rather than hanging waiting for input that will never come.
- **DON'T:** Leave a CLI tool's `async` main function's rejection unhandled at the top level — wrap the tool's entry point in a `try`/`catch` (or a `.catch()` on the top-level call) that prints a clean, user-facing error message and sets a non-zero exit code, rather than letting an unhandled rejection dump a raw, intimidating stack trace as the tool's default failure behavior for an end user who isn't a Node.js developer.

### Runtime & Environment Parity (Docker, Node Versions)

- **DO:** Pin the exact Node.js version a project targets via an `.nvmrc`/`.node-version` file (for local development tooling like `nvm`/`fnm` to pick up automatically) and a matching base image tag in the project's `Dockerfile` (e.g., `node:20.11.1-slim`, not a floating `node:20` or `node:latest`), so "works on my machine" doesn't silently become "fails in CI" or "fails in production" due to a Node version drift nobody noticed.
- **DON'T:** Use a floating/`latest` base image tag in a production Dockerfile — it makes builds non-reproducible over time (the exact same Dockerfile can produce a different, potentially-breaking Node.js version's image on a rebuild months later) and defeats the same reproducibility guarantees a committed lockfile is meant to provide for the application's own dependencies.
- **DO:** Use a multi-stage Docker build — a build stage that installs dev dependencies and compiles/bundles the app, and a separate, minimal final stage that copies over only the production build output and production dependencies — to keep the final production image small and free of build-time-only tooling (compilers, dev dependencies, test files) that has no reason to ship to production.
```dockerfile
FROM node:20.11.1-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20.11.1-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
CMD ["node", "dist/index.js"]
```
- **DON'T:** Run a containerized Node.js process as `root` inside the container by default — create and switch to a dedicated non-root user in the Dockerfile, since running as root inside a container is an unnecessary privilege escalation risk if the container is ever compromised, even though the container provides some isolation from the host.

### Server Timeout Configuration & Slow Request Handling

- **DO:** Configure Node's HTTP server timeouts deliberately rather than leaving every one of them at its default — `server.timeout` (overall socket inactivity timeout), `server.keepAliveTimeout` (how long an idle keep-alive connection is held open), `server.headersTimeout` (how long the server waits to receive the complete request headers), and `server.requestTimeout` (how long it waits for the full request) each protect against a different flavor of slow-client resource exhaustion, and their defaults are not automatically correct for every deployment topology.
```js
const server = app.listen(3000);
server.keepAliveTimeout = 65_000; // must exceed the load balancer's own idle timeout — see below
server.headersTimeout = 66_000;   // per Node's docs, should be slightly larger than keepAliveTimeout
server.requestTimeout = 60_000;
```
- **DON'T:** Deploy a Node.js server behind a load balancer/reverse proxy (an AWS ALB, Nginx) without checking that Node's `keepAliveTimeout` is set *longer* than the load balancer's own idle-connection timeout. This is a well-documented, genuinely common production bug: if Node closes a kept-alive connection first, there's a race window where the load balancer can forward a new request onto the socket right as Node is closing it, producing sporadic `502`s under load that are hard to reproduce locally and easy to misdiagnose as an application bug.
- **DO:** Set a reasonable `requestTimeout`/per-route timeout so a client that sends its request extremely slowly (deliberately or due to a bad connection) can't hold a connection — and the request-handling resources behind it — open indefinitely. A "slowloris"-style attack works by opening many connections and sending data across them just slowly enough to never trigger a naive/absent timeout, tying up the server's limited concurrent-connection capacity with almost no bandwidth spent by the attacker.
- **DON'T:** Set request/response timeouts so aggressively short that legitimate slow-but-valid operations (a large file upload over a slow client connection, a genuinely long-running report generation endpoint) get cut off — tune timeouts per-route where operations have meaningfully different expected durations, rather than applying one blanket value that's either too permissive for fast endpoints or too strict for slow ones.
- **DO:** Terminate TLS and handle basic request sanitization/rate-limiting at a reverse proxy or edge layer (Nginx, a CDN, a cloud load balancer) in front of the Node.js process where practical — these layers are typically more battle-tested and resource-efficient at handling large volumes of slow/malicious connections than relying on the application server alone to defend itself against every connection-level attack.

### Environment Variable String Coercion Pitfalls

- **DON'T:** Treat `process.env.SOME_FLAG` as if it were already a boolean — every value in `process.env` is a string, always, regardless of what it "looks like." The string `"false"` is truthy in a plain `if` check, so `if (process.env.DEBUG)` is `true` even when the environment variable is literally set to the text `"false"`, which is a genuinely common, easy-to-miss bug.
```js
// Bad — "false" is a non-empty string, which is truthy
if (process.env.DEBUG) {
  enableDebugLogging(); // runs even when DEBUG="false"
}

// Good — compare against the actual expected string value explicitly
const isDebug = process.env.DEBUG === "true";
if (isDebug) {
  enableDebugLogging();
}
```
- **DO:** Convert environment-variable strings to their intended type explicitly and centrally (as part of the schema-validated config module covered in the Environment & Configuration section) — `Number(process.env.PORT)`, an explicit `"true"`/`"1"` string comparison for booleans, `JSON.parse` for structured values — rather than relying on JavaScript's implicit truthy/falsy coercion, which does not distinguish "unset," `"false"`, `"0"`, and `""` the way most developers intuitively expect for a boolean flag.
- **DON'T:** Assume an unset environment variable and an explicitly-set-to-empty-string one behave identically everywhere in your code — `process.env.UNSET_VAR` is `undefined`, while `process.env.EMPTY_VAR` (set to `""` in the environment) is the empty string `""`; both are falsy, but a `=== undefined` check specifically for "was this ever set at all" behaves differently from a general falsy check, and the distinction occasionally matters (e.g., for whether a default should apply).
- **DO:** Use a schema-validation library's built-in type coercion (Zod's `z.coerce.number()`/`z.coerce.boolean()`, or `envalid`'s typed accessors) to handle this conversion consistently and with clear validation-failure errors, rather than hand-rolling the same string-to-type conversion logic in multiple places across the codebase with slightly different (and possibly inconsistent) edge-case handling each time.

### Resilience Patterns: Circuit Breakers, Bulkheads & Graceful Degradation

- **DO:** Wrap calls to an unreliable or slow downstream dependency in a circuit breaker (a library like `opossum`, or a hand-rolled state machine) that stops sending requests to a dependency once its failure rate crosses a threshold, and periodically allows a small number of test requests through to detect recovery — this prevents a struggling downstream service from being kept down (or your own service from piling up slow, doomed requests) by continued traffic that's statistically very likely to fail anyway.
```js
import CircuitBreaker from "opossum";

const breaker = new CircuitBreaker(callPaymentProvider, {
  timeout: 3000,          // fail fast if a call takes longer than this
  errorThresholdPercentage: 50, // open the circuit once 50% of recent calls fail
  resetTimeout: 30000,    // after 30s, allow a trial request through again
});
breaker.fallback(() => ({ status: "queued_for_retry" }));
```
- **DON'T:** Let a single slow or failing downstream dependency exhaust your own service's entire pool of worker capacity/connections — isolate dependencies into separate resource pools (the "bulkhead" pattern, named for a ship's compartmentalized hull) so that one dependency's failure degrades only the functionality that depends on it, rather than starving unrelated requests of resources they need to succeed.
- **DO:** Design a deliberate, specific fallback behavior for when a non-critical dependency is unavailable (show cached/stale data with a "may be outdated" notice, hide a recommendations widget instead of failing the whole page, skip a non-essential analytics call) rather than letting one failing dependency take down an entire response that didn't actually need it to succeed.
- **DON'T:** Treat every downstream failure as equally fatal to the current request — distinguish dependencies the current operation genuinely cannot proceed without (a payment provider during checkout) from ones that are enhancements the operation can gracefully proceed without (a "customers also bought" recommendations service). Fail the request hard only for the former.
- **DO:** Set a maximum request queue depth / concurrency limit in front of any resource-constrained downstream dependency (a limited connection pool, a rate-limited third-party API), and shed load (reject with a clear "try again later" response) once that limit is reached, rather than letting an unbounded queue of waiting requests build up and eventually take down the whole process via memory exhaustion or cascading timeouts.

### Rate Limiting & Throttling Requests

- **DO:** Rate-limit at the application/API-gateway layer using an algorithm appropriate to the goal — a sliding-window or token-bucket algorithm for smooth, sustained-rate limiting (allows brief bursts while still enforcing a long-run average), or a fixed-window counter for simplicity when exact burst behavior at window boundaries doesn't matter much — implemented via a maintained library (`express-rate-limit`, `rate-limiter-flexible`) rather than a hand-rolled counter with subtle correctness issues at window edges.
```js
import rateLimit from "express-rate-limit";
import RedisStore from "rate-limit-redis";

const limiter = rateLimit({
  windowMs: 60_000,
  max: 100,
  store: new RedisStore({ sendCommand: (...args) => redisClient.sendCommand(args) }),
});
app.use("/api/", limiter);
```
- **DON'T:** Implement rate limiting with an in-memory counter (a plain `Map` incremented per request) in a horizontally-scaled deployment — each replica tracks its own independent counter, so a client can effectively multiply their real allowed rate by the number of replicas simply by having requests load-balanced across them. Use a shared store (Redis) so the limit is enforced consistently across the whole fleet.
- **DO:** Return standard rate-limit response headers (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, or the still-common `X-RateLimit-*` variants) and a `Retry-After` header on a `429` response, so well-behaved clients can adapt their request rate automatically instead of hammering the endpoint blindly and repeatedly hitting the limit.
- **DO:** Apply different, appropriately-scoped rate limits for different endpoint sensitivities — a strict limit on authentication/password-reset endpoints (brute-force targets), a looser limit for general authenticated API usage, and consider per-user/per-API-key limits in addition to per-IP limits, since a shared corporate NAT/proxy can otherwise cause many legitimate users behind the same IP to compete for one shared limit.
- **DON'T:** Rely solely on rate limiting as your only defense against abusive traffic — pair it with authentication/API keys for identifying and blocking specific bad actors, and consider a dedicated DDoS-mitigation layer (a CDN/WAF) in front of the application for volumetric attacks that a per-request application-layer rate limiter isn't designed to absorb on its own.

### `AsyncLocalStorage` & Request-Scoped Context

- **DO:** Use `node:async_hooks`' `AsyncLocalStorage` to propagate request-scoped context (a request ID, the authenticated user, a per-request logger instance) implicitly through an entire async call chain, instead of manually threading an extra `context`/`req` parameter through every single function signature along the way, many of which have nothing to do with the request themselves.
```js
import { AsyncLocalStorage } from "node:async_hooks";

const requestContext = new AsyncLocalStorage();

app.use((req, res, next) => {
  requestContext.run({ requestId: crypto.randomUUID(), userId: req.user?.id }, next);
});

// deep inside some unrelated utility function, with no `req`/`context` parameter threaded in at all
function logWithContext(message) {
  const ctx = requestContext.getStore();
  logger.info({ requestId: ctx?.requestId, userId: ctx?.userId }, message);
}
```
- **DON'T:** Reach for a plain module-level or global variable to hold "the current request" as a shortcut to avoid parameter-threading — under Node's concurrent, interleaved async execution, a single mutable global is shared across every in-flight request simultaneously, so one request's context can leak into or overwrite another's mid-flight. `AsyncLocalStorage` exists specifically to provide isolated, correctly-scoped context per async execution chain, which a plain shared variable cannot do safely.
- **DO:** Understand that `AsyncLocalStorage`'s context automatically follows the async call chain — through `await`, Promise chains, `setTimeout`, event emitters — with no manual propagation code needed at each step, which is exactly what makes it superior to the long-deprecated, error-prone `domain` module it effectively replaces for this use case.
- **DON'T:** Assume `AsyncLocalStorage` context survives across a genuinely separate, independently-scheduled async operation that doesn't inherit from the original call chain — a message picked up by a completely separate queue consumer later, for instance, starts a new context and needs the relevant identifiers (request ID, user ID) explicitly included in the queued message payload, not implicitly recovered from a stale `AsyncLocalStorage` reference.
- **DO:** Be aware that `AsyncLocalStorage` has a small but real performance overhead per async operation — for extremely hot, high-throughput code paths, measure its actual impact rather than assuming it's negligible, though for the overwhelming majority of typical request-handling workloads the cost is well justified by what it removes: manual, error-prone context-threading through the entire call graph.

### Environment & Configuration

- **DO:** Validate every environment variable the application depends on against an explicit schema at process startup (using Zod, `envalid`, or similar), and fail fast with a clear, specific error message if a required variable is missing or malformed. Discovering a missing `DATABASE_URL` when the first request tries to use it — potentially hours after deploy, whenever that code path is first hit — is a far worse failure mode than crashing immediately on boot with a clear message.
```ts
import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
});

export const env = EnvSchema.parse(process.env); // throws immediately on boot if invalid
```
- **DON'T:** Scatter raw `process.env.SOME_VAR` reads throughout the codebase. Centralize every environment variable read and its validation/parsing/defaulting into one config module that the rest of the application imports from — this gives you one place to see every configuration input the app depends on, one place to validate them, and compile-time typed access everywhere else instead of untyped strings that might be `undefined`.
- **DO:** Provide safe, sensible defaults for genuinely optional, non-sensitive configuration (a default port, a default log level), but never give a secret-shaped variable (a database password, a signing key, an API key) a hardcoded fallback value — a "default" secret is a secret that's the same across every environment and every deployment of the code, defeating the point of it being a secret at all.
- **DON'T:** Overload `NODE_ENV` with custom, project-specific meanings beyond its conventional `development`/`test`/`production` values. Many tools and libraries (Express, React, various bundlers) key real optimization and behavior decisions off `NODE_ENV` specifically — introducing a fourth custom value, or using it to mean something app-specific, can silently break those tools' own environment-detection logic. Use a separate, explicitly-named flag for app-specific environment branching.
- **DO:** Keep secrets and environment-specific values (database URLs, API keys, feature flags per environment) injected by the deployment platform (container orchestrator secrets, a secrets manager, CI/CD environment configuration) rather than committed in environment-specific config files containing real values. A `config.production.json` with real credentials committed to the repository is a leaked-credentials incident waiting to be discovered.
- **DO:** Load `.env` files only in local development (via `dotenv` or similar) and never rely on `.env` file loading as part of how a real deployed environment receives its configuration — production environments should receive configuration through the platform's actual environment-variable/secret-injection mechanism, which is more auditable and doesn't depend on a file being present and correctly synced on disk.

## Security

### Prototype Pollution

- **DON'T:** Merge or recursively assign untrusted, user-controlled objects into another object without guarding against `__proto__`, `constructor`, and `prototype` keys. A naive deep-merge/recursive-assign function that blindly walks nested keys can let an attacker-supplied `{"__proto__": {"isAdmin": true}}` payload pollute `Object.prototype` itself, affecting every plain object in the entire running process — a well-documented, real-world class of vulnerability (CVE-listed in multiple popular npm packages over the years, including old versions of `lodash` and various deep-merge libraries).
```js
// Bad — vulnerable to prototype pollution via a crafted payload
function merge(target, source) {
  for (const key in source) {
    if (typeof source[key] === "object" && source[key] !== null) {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
// merge({}, JSON.parse('{"__proto__":{"polluted":true}}'))
// now EVERY object in the process has `.polluted === true`

// Good — reject dangerous keys explicitly
const DANGEROUS_KEYS = new Set(["__proto__", "constructor", "prototype"]);
function safeMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (DANGEROUS_KEYS.has(key)) continue;
    if (typeof source[key] === "object" && source[key] !== null) {
      target[key] = safeMerge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
```
- **DO:** Use `Object.create(null)` or a `Map` for any object that will hold keys derived from untrusted input, so there's no prototype chain available to pollute in the first place.
- **DO:** Keep dependencies that perform deep merging/cloning/object assignment (`lodash.merge`, `deepmerge`, config-loading libraries) up to date, since prototype pollution vulnerabilities in widely-used packages are discovered and patched periodically — an outdated pinned version can carry a known, published CVE indefinitely.
- **DON'T:** Use `JSON.parse` reviver functions or a custom deserializer that assigns parsed keys directly onto a live object/class instance without the same `__proto__`/`constructor` key filtering — deserialization is one of the most common places untrusted, attacker-shaped data first enters the object graph.

### ReDoS (Regular Expression Denial of Service)

- **DON'T:** Write or accept regular expressions with nested quantifiers or ambiguous overlapping repetition (e.g., `(a+)+`, `(a|a)*`, `([a-zA-Z]+)*`) applied to untrusted, attacker-controllable input. These patterns can exhibit catastrophic backtracking — for certain crafted inputs, matching time grows exponentially with input length, letting a small malicious string (sometimes under 100 characters) freeze the single-threaded event loop for seconds or minutes, effectively a denial-of-service with almost no attacker effort.
```js
// Bad — catastrophic backtracking on an input like "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!"
const EMAIL_LIKE = /^([a-zA-Z0-9]+)+@[a-zA-Z0-9]+\.[a-zA-Z]+$/;

// Good — no ambiguous nested repetition, and/or use a well-tested validation library
const EMAIL_LIKE = /^[a-zA-Z0-9]+@[a-zA-Z0-9]+\.[a-zA-Z]+$/;
```
- **DO:** Run regular expressions that process user input through a static ReDoS checker (`safe-regex`, `eslint-plugin-security`'s regex rules, or the `recheck` tool) as part of code review or CI, especially for anything reachable from an unauthenticated endpoint (login forms, search inputs, webhook payload parsing).
- **DO:** Set an explicit timeout or use a linear-time regex engine (Node's built-in engine doesn't guarantee this, but libraries exist for constrained matching, and Node 24+ ships an experimental way to abort long-running regex execution) when a regex must run against genuinely untrusted, unbounded-length input and its safety can't be fully proven statically.
- **DON'T:** Trust that a regex is "obviously fine" because it works correctly on every test input you tried by hand — catastrophic backtracking often only manifests on specific adversarial input shapes that look nothing like normal test data (e.g., a string of repeated characters immediately followed by one character that finally fails to match), which is exactly why an automated ReDoS scanner catches what manual testing misses.
- **DO:** Prefer built-in string methods (`.startsWith`, `.endsWith`, `.includes`) over a regular expression when a regex's full pattern-matching power isn't actually needed — a simple substring check has no backtracking risk at all and is typically faster.

### `eval`, Dynamic Code Execution & Injection

- **DON'T:** Use `eval()`, the `Function` constructor, or `new Function(...)` on any string that includes or is derived from user input, under any circumstances. This is arbitrary remote code execution in the same process as your server, full stop — there is essentially never a legitimate production reason to do this with untrusted input, and even with "trusted" input it's fragile and hard to secure over time as trust boundaries shift.
```js
// Extremely dangerous — never do this with any input that could originate from a user
app.post("/calculate", (req, res) => {
  const result = eval(req.body.expression);
  res.json({ result });
});

// Good — use a dedicated, sandboxed expression parser/evaluator library
import { evaluate } from "mathjs";
app.post("/calculate", (req, res) => {
  try {
    const result = evaluate(req.body.expression); // still validate/limit input further
    res.json({ result });
  } catch {
    res.status(400).json({ error: "invalid expression" });
  }
});
```
- **DON'T:** Use `setTimeout`/`setInterval` with a string as the first argument (`setTimeout("doSomething()", 1000)`) — this implicitly calls `eval()` internally on the string. Always pass a function reference.
- **DON'T:** Use `child_process.exec()` with a command string built via string concatenation/interpolation of user input — this is a direct shell-injection vector, since `exec` runs its argument through a real shell that interprets `;`, `&&`, `|`, backticks, and similar shell metacharacters.
```js
// Bad — shell injection: filename = "; rm -rf / #" is catastrophic
import { exec } from "node:child_process";
exec(`convert ${filename} output.png`);

// Good — execFile with an argument array bypasses the shell entirely
import { execFile } from "node:child_process";
execFile("convert", [filename, "output.png"]);
```
- **DO:** Use `child_process.execFile()` or `child_process.spawn()` with an explicit argument array (not a single interpolated command string) whenever a command needs to incorporate any external or user-influenced input — passing arguments as an array bypasses shell interpretation entirely, which closes off the injection vector at the root.
- **DON'T:** Use `vm.runInThisContext` or Node's built-in `vm` module as a "security sandbox" for genuinely untrusted code — Node's `vm` module is explicitly documented as *not* a security mechanism; code run through it can still access and manipulate the host process via well-known prototype-pollution and constructor-chain escape techniques. For genuinely sandboxing untrusted code, use a real isolate-based solution (`isolated-vm`, a separate worker/container with OS-level isolation, or a managed serverless sandbox).
- **DO:** Parameterize every database query — use your ORM/query builder's parameter binding, or a raw driver's placeholder syntax (`$1`, `?`) — instead of building SQL/NoSQL queries via string concatenation or template literals with interpolated user input. This is the standard defense against SQL/NoSQL injection, and virtually every mainstream Node database library supports it natively.
```js
// Bad — SQL injection: username = "' OR '1'='1" bypasses the check entirely
const query = `SELECT * FROM users WHERE username = '${username}'`;

// Good — parameterized, the driver handles escaping safely
const result = await pool.query("SELECT * FROM users WHERE username = $1", [username]);
```
- **DON'T:** Build MongoDB (or other NoSQL) queries directly from a raw, unvalidated `req.body`/`req.query` object — MongoDB's query operators (`$where`, `$gt`, `$ne`, etc.) can be smuggled into a query object via user-controlled JSON, letting an attacker bypass intended filters (e.g., `{"password": {"$ne": null}}` to match any password). Validate the shape of query input against a schema before it ever reaches the database layer, and consider disabling `$where` server-side entirely.

### TLS/HTTPS & Certificate Handling

- **DON'T:** Set `rejectUnauthorized: false` on an HTTPS request/agent, or set the `NODE_TLS_REJECT_UNAUTHORIZED=0` environment variable, to work around a certificate validation error. This completely disables TLS certificate verification for the affected connection(s) — including protection against man-in-the-middle attacks — and is one of the most common "quick fixes" for a confusing cert error that trades away the entire security guarantee TLS exists to provide, often for the entire process if set via the environment variable rather than scoped to one request.
```js
// Extremely dangerous — disables certificate verification for this request entirely
const response = await fetch(url, { agent: new https.Agent({ rejectUnauthorized: false }) });

// Better — actually fix the root cause: add the missing intermediate/root CA
const agent = new https.Agent({ ca: fs.readFileSync("./internal-ca.pem") });
const response = await fetch(url, { agent });
```
- **DO:** Diagnose the actual cause of a certificate validation failure — an expired certificate, a missing intermediate CA in the chain, a hostname mismatch, or a genuinely self-signed certificate for an internal service — and fix that specific cause (renew the cert, supply the correct CA bundle via the `ca` option, fix the hostname) rather than disabling verification wholesale. For internal services with legitimately self-signed certificates, supply the specific trusted CA rather than disabling verification for all connections.
- **DON'T:** Hardcode a specific certificate/public key pin without a documented rotation plan — certificate pinning can be a legitimate extra defense for high-security use cases, but a pinned certificate that expires or is rotated by the server operator with no corresponding update on the client side turns a security measure into a self-inflicted, hard-to-diagnose outage.
- **DO:** Keep TLS/cipher configuration on your own HTTPS servers up to date with current best practices (a maintained minimum TLS version, secure cipher suites) via your framework/runtime's defaults rather than a stale, manually-pinned configuration copied from years-old documentation — Node.js's own TLS defaults are actively maintained and are usually the right choice unless there's a specific, understood compliance requirement driving a different configuration.
- **DON'T:** Transmit sensitive data (credentials, tokens, personal data) over plain, unencrypted HTTP even for "internal" or "just testing locally" traffic that might later be promoted to production without someone remembering to add TLS — default to HTTPS everywhere, including internal service-to-service traffic where it's practical, rather than treating TLS as something added only at the very end before shipping.

### Race Conditions & Time-of-Check-to-Time-of-Use (TOCTOU) Bugs

- **DON'T:** Check a condition and then act on it in two separate steps against shared, concurrently-mutable state (a database row, a Redis key, an in-memory counter) without a mechanism preventing another concurrent request from changing that state in between the check and the act — this "time-of-check to time-of-use" gap is a real, exploitable race condition, not just a theoretical concern, whenever the operation has any value to an attacker (redeeming a coupon, claiming a limited-quantity item, withdrawing funds).
```js
// Bad — two concurrent requests can both read "3 remaining" before either one writes back,
// letting both succeed and oversell past the actual limit
async function claimSeat(eventId, userId) {
  const event = await db.events.findById(eventId);
  if (event.seatsRemaining > 0) {
    await db.events.update(eventId, { seatsRemaining: event.seatsRemaining - 1 });
    return true;
  }
  return false;
}

// Good — the check and the decrement happen atomically, in the database itself
async function claimSeat(eventId, userId) {
  const result = await db.query(
    "UPDATE events SET seats_remaining = seats_remaining - 1 WHERE id = $1 AND seats_remaining > 0 RETURNING id",
    [eventId],
  );
  return result.rowCount > 0;
}
```
- **DO:** Push check-and-act operations on shared counters/limited resources down into a single atomic database statement (a conditional `UPDATE ... WHERE` as shown above, an atomic increment/decrement operation, a unique constraint that naturally rejects a duplicate claim) rather than implementing the check and the write as two separate round-trips from application code, which can never be made safe against concurrent execution no matter how carefully the two steps are written.
- **DON'T:** Assume a race condition "probably won't happen in practice" because it requires precise timing — under real production load with many genuinely concurrent requests (a popular product's flash sale, a viral coupon code), the exact interleaving a race condition needs happens routinely, not rarely; "it's timing-dependent" is a reason to fix it properly, not a reason to deprioritize it.
- **DO:** Use a database-level unique constraint as a race-safe guard for "this action should happen at most once" (a unique constraint on a `(userId, couponId)` pair prevents the same coupon from being redeemed twice by the same user, even under concurrent requests, since the second concurrent insert simply fails the constraint) — this is often simpler and more robust than reasoning about explicit locking at the application level.
- **DON'T:** Rely on application-level in-memory locks (a mutex implemented as a JS object/flag) to prevent a race condition in a horizontally-scaled, multi-process/multi-instance deployment — an in-memory lock only prevents concurrent execution *within a single process*; two requests hitting two different replicas at the same instant are completely unaffected by it. This is the same fundamental limitation covered in the Scheduled Jobs section's discussion of distributed locks, and it applies equally here.

### Verifying Webhook Signatures

- **DO:** Verify the cryptographic signature on every incoming webhook (from Stripe, GitHub, or any other provider that signs its webhook payloads) before trusting or acting on its contents — a webhook endpoint is a public URL that anyone can send a POST request to, and without signature verification, an attacker can simply forge a fake "payment succeeded" or "deployment completed" event by guessing the endpoint and crafting their own payload.
```js
import crypto from "node:crypto";

function verifyWebhookSignature(rawBody, signatureHeader, secret) {
  const expected = crypto.createHmac("sha256", secret).update(rawBody).digest("hex");
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signatureHeader));
}

app.post("/webhooks/provider", express.raw({ type: "application/json" }), (req, res) => {
  const isValid = verifyWebhookSignature(req.body, req.headers["x-signature"], WEBHOOK_SECRET);
  if (!isValid) return res.status(401).json({ error: "invalid signature" });
  const event = JSON.parse(req.body);
  // ...handle the now-trusted event
});
```
- **DON'T:** Let a global JSON body-parsing middleware consume and re-serialize the request body before webhook signature verification runs — most providers compute their signature over the exact raw bytes of the request body, and re-serializing a parsed-then-`JSON.stringify`'d body will not byte-for-byte match the original payload, causing every signature check to fail even for genuinely legitimate webhooks. Configure the raw-body parser specifically for the webhook route, ahead of any general JSON-parsing middleware, so the exact original bytes are what gets hashed.
- **DO:** Use `crypto.timingSafeEqual()` (covered earlier in the Security section) when comparing the computed signature against the provided one, rather than a plain `===` string comparison, for the same timing-attack reasons that apply to any other secret comparison.
- **DON'T:** Trust a webhook's payload contents to determine identity or authorization on their own (e.g., trusting a `userId` field inside the payload without validating it against what your own system expects) — signature verification proves the request genuinely came from the provider you registered the webhook with, not that every field inside the payload is safe to act on unconditionally; still validate the payload's shape and business-logic sanity the same as any other external input.
- **DO:** Implement idempotent webhook handling (checking whether a given event ID has already been processed before acting on it) — most webhook providers explicitly document that the same event can be delivered more than once (network retries, at-least-once delivery guarantees), so a webhook handler needs the same idempotency discipline covered in the Scheduled Jobs & Message Queues sections above.

### Mass Assignment & Over-Posting

- **DON'T:** Pass a raw, unfiltered request body directly into a database update/create call (`Object.assign(user, req.body)`, `db.users.update(id, req.body)`, an ORM's `.update(req.body)`) — this "mass assignment" pattern lets a client set *any* field the underlying model has, including ones that were never meant to be client-writable (`isAdmin`, `role`, `accountBalance`, `verifiedAt`), simply by including that field in the request payload, regardless of whether the UI/API was ever designed to expose it.
```js
// Bad — a request body of { "name": "New Name", "role": "admin" } silently grants admin
app.patch("/profile", requireAuth, async (req, res) => {
  await db.users.update(req.user.id, req.body);
  res.json({ ok: true });
});

// Good — only the fields the client is actually allowed to set pass through
const UpdateProfileSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  bio: z.string().max(500).optional(),
}).strict(); // .strict() rejects unknown keys outright instead of silently dropping them

app.patch("/profile", requireAuth, async (req, res) => {
  const allowedUpdates = UpdateProfileSchema.parse(req.body);
  await db.users.update(req.user.id, allowedUpdates);
  res.json({ ok: true });
});
```
- **DO:** Define an explicit allow-list of client-writable fields for every create/update endpoint — via a validation schema, a dedicated DTO type, or an explicit field-picking step — so the set of fields a request can actually influence is a conscious, reviewed decision rather than "whatever fields happen to exist on the underlying database model."
- **DON'T:** Rely on a schema's default behavior of silently stripping unrecognized keys as your only defense against mass assignment, without also considering whether a `.strict()`/`additionalProperties: false`-style rejection is more appropriate — silent stripping is often fine for genuinely unexpected extra fields, but for a field that's sensitive specifically *because* a client might try to set it (a `role` or `isAdmin` field on a `User` update), an explicit rejection (and ideally a logged/alerted attempt) can be a more deliberate signal than quietly dropping it and moving on.
- **DO:** Apply the same allow-list discipline to bulk/batch update endpoints and to GraphQL mutations, not just single-record REST update endpoints — the same over-posting risk applies anywhere a client-supplied object is used to drive a write, regardless of the specific transport or API style.
- **DON'T:** Assume front-end form field restrictions (a form that "only shows" editable fields) provide any actual security — a client-side UI limitation is trivially bypassed by anyone sending a request directly (via a browser's dev tools, `curl`, or a modified client), so the server-side allow-list is the only enforcement that actually matters.

### Authentication Mechanisms & Common Pitfalls

- **DO:** Regenerate the session identifier immediately after a successful login (most session libraries expose a `regenerate()`/equivalent call), never continuing to use whatever session ID existed before authentication. Without this, an attacker who can fix a victim's pre-login session ID (session fixation — e.g., by tricking them into visiting a link containing a known session ID) can hijack the now-authenticated session once the victim logs in, since the session ID never changed across the authentication boundary.
```js
app.post("/login", async (req, res) => {
  const user = await authenticate(req.body.email, req.body.password);
  if (!user) return res.status(401).json({ error: "invalid credentials" });
  req.session.regenerate((err) => {
    if (err) return res.status(500).json({ error: "login failed" });
    req.session.userId = user.id; // set on the NEW, regenerated session
    res.json({ ok: true });
  });
});
```
- **DO:** Validate an OAuth `redirect_uri` against an exact, pre-registered allow-list on the authorization server side, and never accept a wildcard or loosely-pattern-matched redirect target — an OAuth flow that redirects the authorization code/token to an attacker-controlled URI (via an insufficiently validated `redirect_uri` parameter) is a well-documented way to steal authorization codes/tokens meant for a legitimate client.
- **DO:** Use PKCE (Proof Key for Code Exchange) for any OAuth client that can't securely hold a client secret — single-page apps, mobile apps, and any other "public" client — which is now the broadly recommended approach even for confidential clients. PKCE prevents an intercepted authorization code from being redeemed by anyone other than the party that initiated the original request, closing a real interception risk that exists for public clients unable to keep a secret truly secret.
- **DON'T:** Implement your own OAuth/OIDC client or authorization-server logic from scratch when a well-maintained, spec-compliant library is available — OAuth/OIDC has many subtle, security-critical details (state-parameter CSRF protection, nonce validation, token audience checks, redirect URI matching) that are easy to get wrong individually and have already been solved correctly by mature libraries; reinventing this is a much higher-risk undertaking than it might initially appear.
- **DO:** Validate the OAuth `state` parameter on the callback matches what was generated at the start of the flow, to prevent CSRF attacks against the OAuth login flow itself — an attacker who can trick a victim into completing an OAuth flow initiated by the attacker (rather than the victim) can potentially link the victim's session to the attacker's own third-party account, depending on the specific flow.
- **DON'T:** Conflate authentication (proving who a user is) with authorization (determining what that authenticated user is allowed to do) as a single check — a valid, successfully-authenticated session proves identity, not permission; every sensitive action still needs its own explicit authorization check against that identity, echoing the IDOR guidance covered earlier in this section.
- **DO:** Invalidate all of a user's active sessions/refresh tokens on a password change or a suspected-compromise event, not just issue a new one going forward — leaving old sessions/tokens valid after a password reset defeats much of the point of the reset if the account was compromised specifically because an old credential/session was leaked.

### Server-Side Request Forgery (SSRF)

- **DON'T:** Make an outbound HTTP request to a URL supplied (fully or partially) by an untrusted client without restricting what that URL is allowed to point at — a feature that fetches a URL on the server's behalf (an image-proxy endpoint, a "import from URL" feature, a webhook-URL-testing endpoint) can be abused to make the server issue requests to internal-only services (a cloud metadata endpoint, an internal admin panel, another service on the private network) that the attacker could never reach directly themselves, using your server as a proxy into your own private network.
```js
// Bad — the server will fetch literally any URL the client provides, including internal ones
app.post("/fetch-preview", async (req, res) => {
  const response = await fetch(req.body.url);
  res.json({ html: await response.text() });
});
// An attacker sends { "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/" }
// and the server obligingly fetches internal cloud credentials on the attacker's behalf.
```
- **DO:** Validate a user-supplied URL against an explicit allow-list of permitted hosts/schemes before fetching it server-side, and reject URLs pointing at private/internal IP ranges (`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, link-local addresses like `169.254.0.0/16` which includes most cloud providers' instance-metadata endpoints) and non-HTTP(S) schemes (`file://`, `gopher://`) — a real, well-tested SSRF-prevention library or an explicit resolved-IP check is preferable to a naive string-based hostname check alone.
- **DON'T:** Rely on checking only the URL's hostname string for SSRF protection without also validating the IP address it actually resolves to — a hostname can be crafted to pass a naive string check while its DNS resolution (which the attacker fully controls if they own the domain) points at an internal IP address, and some URL parsers can also be tricked with unusual formatting (decimal/octal IP notation, unexpected redirects) that bypasses a purely textual check.
- **DO:** Disable HTTP redirect-following (or cap it strictly and re-validate each redirect target) for server-side fetches of user-supplied URLs — an initially-valid, allow-listed URL can respond with a redirect to an internal address, and a client that blindly follows redirects defeats an allow-list check that was only ever applied to the original URL.
- **DON'T:** Forget that SSRF risk isn't limited to an obvious "fetch this URL" feature — anywhere the server makes an outbound network call whose destination is influenced by user input (a webhook URL a user registers, an image URL embedded in user-submitted content that a server-side renderer fetches to generate a thumbnail, an XML parser that resolves external entities from user-supplied XML) carries the same risk and needs the same validation discipline.

### Path Traversal & Filesystem Safety

- **DON'T:** Build a filesystem path by directly concatenating or joining a user-supplied string without validating it first — a value like `"../../etc/passwd"` or an absolute path passed where a relative one was expected can escape the intended directory entirely, letting an attacker read or write files far outside the directory the application meant to expose (a classic and still very common path traversal vulnerability).
```js
// Bad — req.params.filename could be "../../../../etc/passwd"
app.get("/files/:filename", (req, res) => {
  res.sendFile(path.join(UPLOAD_DIR, req.params.filename));
});

// Good — resolve the final path and verify it's still inside the intended directory
app.get("/files/:filename", (req, res) => {
  const requested = path.resolve(UPLOAD_DIR, req.params.filename);
  if (!requested.startsWith(path.resolve(UPLOAD_DIR) + path.sep)) {
    return res.status(400).json({ error: "invalid filename" });
  }
  res.sendFile(requested);
});
```
- **DO:** Use `path.resolve()` (which normalizes `..` segments and produces an absolute path) followed by an explicit check that the resolved path still falls within the intended base directory, rather than trying to blacklist specific dangerous substrings (`"../"`) — substring blacklists are notoriously easy to bypass with encoding tricks, extra slashes, or platform-specific path separator differences, while resolve-then-verify checks the actual final destination directly.
- **DON'T:** Trust a client-supplied filename for how a file is stored on disk or served back — generate your own internal filename/identifier (a UUID, a hash) for stored files, and keep the user-supplied original filename only as metadata (used for the `Content-Disposition` download filename, for instance), never as the actual path component used to read or write the file on disk.
- **DO:** Run filesystem access with the least privilege actually required — a process that only ever needs to read from one specific upload directory shouldn't run with broad filesystem read/write access, and file permissions/OS-level sandboxing (a dedicated non-root user, a restricted container filesystem) should back up application-level path validation as defense-in-depth, not replace it.
- **DON'T:** Follow symbolic links blindly when serving user-controlled paths — a symlink planted inside an otherwise-safe upload directory (if user-controlled content can create one) can point outside that directory entirely, sidestepping a path-prefix check that only validated the *link's* path, not its actual target. Validate the resolved real path (`fs.realpath`) when symlinks are a plausible concern for the specific storage mechanism in use.

### Other Common Node/JS Security Pitfalls

- **DO:** Store secrets (API keys, database credentials, signing keys) in environment variables or a dedicated secrets manager (AWS Secrets Manager, HashiCorp Vault, Doppler), never hardcoded in source code or committed config files. A hardcoded secret in a git repository is compromised the moment the repo is cloned, forked, or leaked, and remains in git history even after being "removed" in a later commit unless history is rewritten.
- **DON'T:** Commit `.env` files containing real secrets to version control — commit a `.env.example`/`.env.sample` template with placeholder values instead, and add `.env` to `.gitignore` from the very first commit of a project.
- **DO:** Validate and sanitize all user input at the boundary — request bodies, query parameters, headers, file uploads — using a schema validation library (Zod, Joi, class-validator), before that data reaches business logic, database queries, or is reflected back into any response.
- **DON'T:** Trust `Content-Type` headers, client-reported file extensions, or file names for file uploads without independently verifying the actual file content (magic-byte/content sniffing) when the file type matters for security (e.g., preventing an executable disguised as an image from being served back with an executable content type).
- **DO:** Set security-relevant HTTP headers using a maintained middleware like `helmet` (for Express — other frameworks have equivalents) rather than manually setting each header by hand — it covers `Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security`, and several other headers whose exact correct values are easy to get subtly wrong by hand.
- **DON'T:** Roll your own password hashing (naive SHA-256/MD5 of a password, even with a manually-added salt) — use a purpose-built, slow, salted hashing algorithm designed for passwords specifically: `bcrypt`, `argon2` (generally the current best-practice recommendation), or `scrypt`. General-purpose fast hash functions are specifically what makes brute-forcing cracked password databases cheap; slow, memory-hard algorithms exist precisely to make that expensive.
```js
// Bad — fast, unsalted-by-default general-purpose hash, trivially brute-forced offline
import { createHash } from "node:crypto";
const hash = createHash("sha256").update(password).digest("hex");

// Good
import argon2 from "argon2";
const hash = await argon2.hash(password);
const isValid = await argon2.verify(hash, password);
```
- **DO:** Use `crypto.randomBytes()`/`crypto.randomUUID()` (Node's cryptographically secure random source) for anything security-sensitive — session tokens, password reset tokens, API keys, CSRF tokens — never `Math.random()`, which is not cryptographically secure and is predictable enough in some engines to be reverse-engineered from observed outputs.
```js
// Bad — Math.random() is not cryptographically secure
const token = Math.random().toString(36).slice(2);

// Good
import { randomBytes } from "node:crypto";
const token = randomBytes(32).toString("hex");
```
- **DO:** Use `crypto.timingSafeEqual()` when comparing secret values (tokens, HMAC signatures, API keys) instead of `===`/`==` — a standard string comparison short-circuits on the first mismatched character, and the resulting timing difference can, in principle, be measured remotely to guess a secret one character at a time (a timing attack). Constant-time comparison closes that side channel.
- **DON'T:** Deserialize untrusted data with `node-serialize`, unsafe YAML loaders (`yaml.load` instead of `yaml.safeLoad`/the schema-restricted variant), or any deserialization mechanism that can reconstruct arbitrary object types/functions from the serialized payload — several of these have documented remote-code-execution vulnerabilities when fed attacker-controlled input, because reconstructing a function or a class instance can execute arbitrary code as a side effect of deserialization itself.
- **DO:** Rate-limit authentication endpoints, password-reset flows, and any other endpoint that's a plausible brute-force or enumeration target, using a proper rate-limiting middleware/library backed by a shared store (Redis) in multi-instance deployments — an in-memory rate limiter alone doesn't protect a horizontally-scaled app, since each instance tracks its own separate counters.
- **DON'T:** Return different error messages (or timing) for "user not found" versus "wrong password" on a login endpoint — this lets an attacker enumerate valid usernames/emails one request at a time. Return a single generic message ("invalid credentials") for both cases, at consistent response time.
- **DO:** Keep dependencies patched against known vulnerabilities via `npm audit`, Dependabot/Renovate security alerts, or a dedicated SCA (software composition analysis) tool, and treat a disclosed critical vulnerability in a direct or transitive dependency as a priority fix rather than something to address "eventually."
- **DON'T:** Log sensitive data — raw passwords, full credit card numbers, authentication tokens, personally identifiable information beyond what's operationally necessary — in application logs. Logs are often shipped to third-party aggregators, retained for long periods, and accessible to a wider set of people than the production database itself; treat log content with the same sensitivity as any other data store.

### Clickjacking & Frame Protection

- **DO:** Set `X-Frame-Options: DENY` (or `SAMEORIGIN` if legitimate same-origin framing is needed) and/or the modern CSP `frame-ancestors` directive on any response that shouldn't be embeddable inside another site's `<iframe>` — without one of these, an attacker can overlay your page inside an invisible or disguised iframe on their own site and trick users into clicking on what they believe is the attacker's page but is actually a real button/link on your page (a "clickjacking" attack), potentially triggering an authenticated action the user never intended.
```js
import helmet from "helmet";
app.use(helmet.frameguard({ action: "deny" }));
// or via CSP directly, which supersedes X-Frame-Options in modern browsers
app.use((req, res, next) => {
  res.setHeader("Content-Security-Policy", "frame-ancestors 'none'");
  next();
});
```
- **DO:** Apply this protection especially to any page that performs a sensitive, authenticated, one-click action (a "delete account" confirmation, a funds-transfer confirmation, an OAuth consent screen) — these are exactly the kind of pages clickjacking is used to exploit, since the attack relies on tricking an already-authenticated user into an unintended click on a real, functioning control.
- **DON'T:** Set `X-Frame-Options: ALLOWALL` or omit frame protection headers entirely "to be safe/flexible" without a specific, deliberate reason your application needs to be embeddable by arbitrary third-party sites — the default posture for most applications should be "not embeddable," with framing explicitly allowed only for the specific origins that genuinely need it (via `frame-ancestors https://trusted-partner.com`), not left wide open by default.
- **DO:** Combine frame protection with other standard security headers via a single, well-maintained middleware (`helmet`, mentioned earlier in this section) rather than hand-setting each header individually — this both reduces the chance of missing one and keeps the configuration up to date as new recommended headers/directives emerge over time.

### Output Encoding & XSS Prevention

- **DON'T:** Interpolate unescaped user-controlled data directly into server-rendered HTML — a raw template string, an unescaped templating-engine variable, or a manually-constructed HTML response — since it lets an attacker inject arbitrary `<script>` tags or event-handler attributes that execute in another user's browser (stored or reflected cross-site scripting).
```js
// Bad — comment.text renders unescaped; an attacker's comment can contain a <script> tag
res.send(`<div class="comment">${comment.text}</div>`);

// Good — use a templating engine's default auto-escaping, or escape explicitly
import { escapeHtml } from "./utils/html.js";
res.send(`<div class="comment">${escapeHtml(comment.text)}</div>`);
```
- **DO:** Rely on your templating engine's default auto-escaping behavior (most modern engines — EJS's `<%= %>`, Handlebars' `{{ }}`, JSX's default text interpolation — escape by default) rather than an engine's "raw"/"unescaped" output directive (`<%- %>` in EJS, triple-stash `{{{ }}}` in Handlebars, `dangerouslySetInnerHTML` in React) unless you have a specific, reviewed reason to render pre-trusted HTML, and sanitize that HTML explicitly when you do.
- **DO:** Sanitize any HTML that must be rendered from a source you don't fully control (user-submitted rich text, an admin-authored HTML snippet stored in a database that could still have been tampered with) using a real sanitization library (`DOMPurify`, `sanitize-html`) configured with an explicit allow-list of tags/attributes, rather than a hand-written regex-based "strip script tags" filter — regex-based HTML sanitization is a well-known losing battle against the many ways HTML parsers can be tricked into executing script content that doesn't look like a `<script>` tag.
- **DON'T:** Trust `Content-Type`-based assumptions for what a browser will do with a response — serving user-uploaded content (like an SVG, which can contain embedded `<script>`, or an HTML file) from the same origin as your main application, without the `X-Content-Type-Options: nosniff` header and a properly restrictive `Content-Security-Policy`, can let a browser execute it as if it were part of your site.
- **DO:** Set a real `Content-Security-Policy` header (via `helmet` or manually) that restricts script sources to your own trusted origins, as defense-in-depth against XSS even when output is properly escaped elsewhere — CSP doesn't replace escaping/sanitization, but it substantially limits what an attacker can do even if an escaping bug slips through.
- **DON'T:** Reflect user input back into an HTTP response's headers (a redirect `Location` header built from a query parameter, a custom header echoing back a client-supplied value) without validating/encoding it — unvalidated header injection can allow response-splitting or open-redirect style attacks depending on how the value is used downstream.

### Resource Exhaustion & Unbounded Input

- **DON'T:** Accept deeply nested JSON (or any recursively-parsed format) from an untrusted source without a depth or size limit — a maliciously crafted, deeply nested payload can cause a naive recursive parser or a recursive object-processing function (a deep clone, a recursive validator, a recursive object-to-string formatter) to blow the call stack or consume excessive CPU/memory, a denial-of-service vector sometimes called a "JSON bomb" or "billion laughs"-style attack (the latter more classically associated with XML entity expansion, but the same class of resource-exhaustion risk applies to any recursively-expanding untrusted input).
- **DO:** Set explicit limits on request body size, array length, string length, and object nesting depth at the point untrusted data enters the system (body-parser size limits, schema-level `.max()` constraints on arrays/strings in Zod or similar, a maximum recursion depth in any hand-written recursive processor) — treat "how big/deep can this input be" as a question that needs an explicit, deliberate answer for every untrusted input boundary, not an unbounded default.
```ts
const CommentSchema = z.object({
  text: z.string().max(5000),
  tags: z.array(z.string().max(50)).max(20),
});
```
- **DON'T:** Process user-uploaded archive files (zip, tar) without limits on decompressed size and file count — a small compressed file can expand to an enormous decompressed size (a "zip bomb"), exhausting disk or memory if extracted without a running size check that aborts once a sane limit is exceeded.
- **DO:** Apply the same resource-exhaustion thinking to expensive computed responses, not just parsed input — an API endpoint that lets a client request an arbitrarily large `limit`/page size, or a report-generation endpoint with no cap on date-range size, can be used to force the server to do an unbounded amount of work from a single, cheap-to-send request. Cap pagination sizes and computation-bounding parameters server-side, regardless of what a client requests.
- **DO:** Consider algorithmic-complexity attacks beyond ReDoS specifically — any user-controlled input that drives the size of an internal data structure or the number of iterations of a loop (a user-specified "generate N items" count, a user-controlled sort/group-by key cardinality) is a potential resource-exhaustion vector if left uncapped, even when the algorithm itself is a perfectly reasonable O(n log n) or similar.

### Dependency & Supply-Chain Security

- **DO:** Treat every third-party dependency as code that runs with the full privileges of your application — a compromised or malicious package can read environment variables (including secrets), make network requests, and access the filesystem exactly as your own code can. Apply real scrutiny to what gets added, especially transitive dependencies pulled in without a direct decision to add them.
- **DON'T:** Run `npm install`/`pnpm install`/`yarn install` on a project from an untrusted source without at least a cursory review — install scripts (`preinstall`, `install`, `postinstall` in a package's `package.json`) run arbitrary code automatically the moment a package is installed, before any of your own code even executes. This is a real, actively-exploited supply-chain vector, not a theoretical one.
- **DO:** Be aware that npm supports disabling arbitrary install-script execution (`npm install --ignore-scripts`, or a persistent config setting) for environments where it's viable, and consider it for CI/build environments that don't genuinely need any package's install scripts to run, reducing the blast radius of a compromised transitive dependency.
- **DON'T:** Confuse a similarly-named but different package for the one you intended (typosquatting: `expres` vs `express`, `reactt` vs `react`) — this is a well-documented, actively-exploited attack where malicious actors publish packages with names one keystroke away from popular ones, hoping for exactly this kind of typo. Double-check an unfamiliar package name character-by-character against the real, intended package before installing it, especially when it was suggested rather than independently verified.
- **DO:** Verify lockfile integrity is actually being checked, not bypassed — `npm ci` and its equivalents in other package managers validate the lockfile's integrity hashes against what actually gets downloaded, which is a real defense against a package registry serving different content than what was originally locked. Don't work around a lockfile-integrity failure by deleting and regenerating the lockfile without first understanding *why* it failed to match.
- **DON'T:** Store CI/CD secrets (deploy keys, cloud credentials, npm publish tokens) as plain environment variables visible to every step of a pipeline, including third-party actions/steps from outside your own organization, without scoping them down. A malicious or compromised third-party CI action can exfiltrate any secret exposed to its execution context — use your CI platform's secret-scoping and minimal-permission features (e.g., GitHub Actions' `permissions` block, environment-scoped secrets) rather than granting broad, ambient access by default.
- **DO:** Enable automated dependency-vulnerability scanning (GitHub's Dependabot alerts, `npm audit` in CI, Snyk, or an equivalent) so newly-disclosed vulnerabilities in existing dependencies surface automatically, rather than relying on someone manually re-running an audit periodically and remembering to do so.
- **DON'T:** Grant an npm publish token (or any CI deployment credential) broader scope or longer lifetime than necessary — prefer short-lived, narrowly-scoped tokens (and, where the registry supports it, hardware-key-backed 2FA for publishing) over a single long-lived token with full account access sitting in a CI secret store indefinitely.

### Preventing Enumeration via Predictable Identifiers

- **DON'T:** Expose sequential, auto-incrementing integer IDs (`/orders/1042`) in any URL or API response for resources that are sensitive or that shouldn't be easily discoverable/countable by outsiders — a predictable, sequential ID lets anyone trivially enumerate the entire resource space by incrementing the number, which both leaks business information (roughly how many orders/users/whatever exist, and their creation order) and, combined with even a small authorization gap, dramatically widens the practical impact of an IDOR vulnerability (covered earlier) since every ID is trivially guessable rather than needing to be discovered.
```js
// Bad — sequential IDs are trivially enumerable: /invoices/1001, /invoices/1002, ...
const invoice = await db.invoices.create({ id: nextSequentialId(), ...data });

// Good — a UUID/ULID gives no information about ordering or total count, and can't be enumerated
import { randomUUID } from "node:crypto";
const invoice = await db.invoices.create({ id: randomUUID(), ...data });
```
- **DO:** Use UUIDs, ULIDs, or another non-sequential, effectively-unguessable identifier as the externally-exposed ID for any resource where enumeration or count-leakage is a real concern, while still allowing an internal, sequential primary key for the database's own indexing/join efficiency if that's genuinely beneficial — the two don't need to be the same value; a sequential internal ID plus a separate, random external-facing identifier gets both properties at once.
- **DON'T:** Treat a non-sequential ID as a substitute for actual authorization checks — an unguessable ID makes casual enumeration impractical, but it is not itself an access-control mechanism; anyone who legitimately obtains a specific ID (through a shared link, a previous legitimate response, or by other means) still needs the request to be checked against real authorization logic, exactly as covered in the IDOR guidance above. Treat unguessable IDs as one added layer of defense-in-depth, not a replacement for authorization.
- **DO:** Consider ULIDs specifically (over plain random UUIDv4) when you want the enumeration resistance of a random identifier while retaining rough chronological sortability for internal/operational convenience (ULIDs encode a timestamp prefix) — a reasonable middle ground for use cases where "unguessable" matters more than "the ID reveals nothing at all" and some ordering convenience is genuinely useful.

### Web-Specific Security: CORS, CSRF, JWT & Authorization

- **DON'T:** Configure `Access-Control-Allow-Origin: *` together with `Access-Control-Allow-Credentials: true` — browsers themselves reject this specific combination, but more fundamentally, pairing a wildcard origin with credentialed requests defeats the entire purpose CORS exists for: letting a server explicitly control which origins may make authenticated cross-origin requests against it.
- **DO:** Configure CORS with an explicit allow-list of trusted origins, and validate an incoming `Origin` header against that list rather than reflecting whatever origin the request happened to send back as the allowed origin — reflecting the request's own origin unconditionally is functionally equivalent to a wildcard for any attacker-controlled site.
```js
// Bad — reflects any origin back, functionally equivalent to a wildcard
app.use(cors({ origin: (origin, cb) => cb(null, origin) }));

// Good — explicit allow-list
const ALLOWED_ORIGINS = new Set(["https://app.example.com", "https://admin.example.com"]);
app.use(cors({
  origin: (origin, cb) => cb(null, !origin || ALLOWED_ORIGINS.has(origin)),
  credentials: true,
}));
```
- **DON'T:** Store session tokens or JWTs in `localStorage`/`sessionStorage` for anything security-sensitive. Both are fully readable by any JavaScript running on the page, which means any successful XSS (cross-site scripting) vulnerability anywhere on the site — including in a third-party script — can read and exfiltrate the token directly.
- **DO:** Store session tokens in cookies with `httpOnly` (unreadable by JavaScript, closing off the XSS-token-theft vector), `Secure` (only sent over HTTPS), and an appropriate `SameSite` value (`Lax` or `Strict`, which limits the cookie being sent on cross-site requests and thereby reduces CSRF exposure).
- **DON'T:** Assume cookie-based authentication is automatically safe from CSRF just because it's a JSON API rather than an HTML form — any endpoint a browser will automatically attach cookies to on a request is a CSRF target unless explicitly protected. Use a CSRF token pattern (double-submit cookie or synchronizer token) or lean on a strict `SameSite` cookie policy plus server-side origin/referer checking for state-changing endpoints.
- **DO:** Verify JWT signatures with an explicitly specified, expected algorithm (`jwt.verify(token, secret, { algorithms: ["RS256"] })`) rather than trusting whatever algorithm the token's own header claims to use. This closes the well-documented "`alg: none`" and algorithm-confusion attack classes, where a forged token claims a different (or no) algorithm to bypass verification against the server's actual expected signing method.
```js
// Bad — trusts whatever algorithm the token itself declares
const payload = jwt.verify(token, secretOrPublicKey);

// Good — pins the accepted algorithm explicitly, server-side
const payload = jwt.verify(token, secretOrPublicKey, { algorithms: ["RS256"] });
```
- **DON'T:** Put sensitive data (a plaintext password, full payment card numbers, anything that shouldn't be readable by whoever holds the token) inside a JWT payload — a JWT's payload segment is only base64url-*encoded*, not encrypted, and is trivially decodable by anyone who has the token, including the end user themselves via their browser's dev tools.
- **DO:** Keep access tokens short-lived and pair them with a separate, revocable refresh-token mechanism, rather than issuing long-lived access tokens that remain valid until their natural expiry with no way to revoke them early if the token is leaked or the user's access needs to be immediately cut off.
- **DO:** Enforce authorization (does *this specific authenticated user* have permission to access *this specific resource*) on every endpoint that returns or mutates resource-scoped data, not just authentication (is there a valid logged-in session at all). Checking only authentication is exactly how IDOR (Insecure Direct Object Reference) vulnerabilities happen — a logged-in user requesting `/api/orders/12345` for an order that isn't theirs.
```js
// Bad — any authenticated user can fetch any order by guessing/incrementing IDs
app.get("/api/orders/:id", requireAuth, async (req, res) => {
  const order = await db.orders.findById(req.params.id);
  res.json(order);
});

// Good — checks that the authenticated user actually owns this resource
app.get("/api/orders/:id", requireAuth, async (req, res) => {
  const order = await db.orders.findById(req.params.id);
  if (!order || order.userId !== req.user.id) {
    return res.status(404).json({ error: "order not found" });
  }
  res.json(order);
});
```

## Performance

### Algorithmic & Data-Structure Choices

- **DO:** Choose the right data structure for the access pattern before reaching for micro-optimizations — an O(n) `.includes()` check inside a loop over n items is an O(n²) algorithm no matter how "optimized" the inner comparison is; switching the lookup to a `Set`/`Map` (O(1) average) is a bigger win than any low-level tuning of the O(n) version.
- **DON'T:** Optimize code before measuring where the actual bottleneck is. Profile first (Node's built-in `--prof`/`--cpu-prof`, Chrome DevTools' profiler attached via `--inspect`, or an APM tool in production) — intuition about "what's slow" is frequently wrong, and optimizing a function that accounts for 0.1% of total request time wastes effort while the real 40%-of-the-time bottleneck goes untouched.
- **DO:** Batch database queries instead of issuing one query per item in a loop — the classic N+1 query problem, where fetching N related records triggers N additional round-trip queries instead of one batched query (a `WHERE id IN (...)`, a proper JOIN, or an ORM's eager-loading/`include` feature).
```js
// Bad — N+1 queries: 1 to fetch orders, then N more, one per order, to fetch its user
const orders = await db.orders.findMany();
for (const order of orders) {
  order.user = await db.users.findById(order.userId);
}

// Good — 2 queries total, regardless of how many orders there are
const orders = await db.orders.findMany();
const userIds = [...new Set(orders.map((o) => o.userId))];
const users = await db.users.findMany({ where: { id: { in: userIds } } });
const usersById = new Map(users.map((u) => [u.id, u]));
orders.forEach((order) => { order.user = usersById.get(order.userId); });
```
- **DON'T:** Fetch entire tables/collections into application memory to filter, sort, or paginate them in JavaScript when the database can do the same filtering/sorting/pagination far more efficiently with an index. Push `WHERE`, `ORDER BY`, and `LIMIT`/`OFFSET` (or cursor-based pagination) down to the database query itself.

### Spread & Rest Operator Performance Pitfalls

- **DON'T:** Use the spread operator to push a very large number of elements onto an array/into a function call (`arr.push(...hugeArray)`, `Math.max(...hugeArray)`) — spreading an array into function arguments passes each element as an individual argument, and JavaScript engines impose a real, finite limit on the number of arguments a function call can take; a sufficiently large array can throw `RangeError: Maximum call stack size exceeded` at exactly this line, which is easy to miss until the array involved happens to be large enough in production.
```js
// Bad — throws RangeError once hugeArray is large enough (the exact threshold is engine-dependent)
const max = Math.max(...hugeArray);

// Good — no argument-count limit, since it's a single loop rather than a single call with N arguments
const max = hugeArray.reduce((a, b) => Math.max(a, b), -Infinity);
```
- **DON'T:** Use spread inside a loop to build up an array or object incrementally (`result = [...result, newItem]` or `result = {...result, [key]: value}` executed repeatedly inside a loop) — each spread creates an entirely new copy of the accumulated structure, so an operation that looks like "add one item" is actually "copy everything so far, then add one item," turning an intended O(n) loop into an O(n²) one as the accumulated structure grows.
```js
// Bad — O(n^2): each iteration re-copies the entire array built up so far
let result = [];
for (const item of items) {
  result = [...result, transform(item)];
}

// Good — O(n): mutate a local accumulator directly, only exposing the final result
const result = [];
for (const item of items) {
  result.push(transform(item));
}
```
- **DO:** Reserve the "build a new copy via spread" pattern for cases where it genuinely runs once per meaningful update (a single state update in response to one user action, for instance) rather than inside a tight loop accumulating many updates — the immutability benefit of spread-based updates is real and worth keeping for those cases; the performance problem is specifically about repeating the full-copy operation many times in a loop that could instead accumulate directly and only produce one final (optionally still immutable) result.
- **DO:** Recognize that this is a specific instance of the general "don't reflexively apply an idiom without considering its cost in a hot path" theme covered elsewhere in this document — spread/immutable-update patterns are good defaults for the vast majority of code, and the performance concern here applies specifically to loops accumulating a large number of updates, not to occasional, single-shot uses.

### Avoiding Nested-Loop Lookups

- **DON'T:** Nest a `.find()`, `.filter()`, or `.includes()` call inside another array iteration when matching two collections against each other — this pattern quietly produces an O(n × m) algorithm, and it's one of the most common accidental-quadratic-complexity bugs, since each individual line looks completely reasonable in isolation and the problem only shows up as the data volume grows in production.
```js
// Bad — O(orders.length * products.length): fine on 50 test records, slow on 50,000 real ones
const enriched = orders.map((order) => ({
  ...order,
  product: products.find((p) => p.id === order.productId),
}));

// Good — O(orders.length + products.length): build a lookup once, then use it
const productsById = new Map(products.map((p) => [p.id, p]));
const enriched = orders.map((order) => ({
  ...order,
  product: productsById.get(order.productId),
}));
```
- **DO:** Build a `Map` (or plain object, for simple string keys) once up front whenever a collection needs to be repeatedly looked up by some key inside a loop — the one-time O(n) cost of building the map is virtually always cheaper than paying an O(n) linear scan on every single iteration of an outer loop.
- **DO:** Use `Array.prototype.reduce()` (or a simple `for` loop building a `Map`) to group an array of items by a key in a single O(n) pass, instead of filtering the same source array once per distinct group value (which turns an O(n) grouping operation into an O(n × distinct-group-count) one).
```js
// Bad — filters the whole array once per distinct status value
const byStatus = {};
for (const status of ["pending", "shipped", "delivered"]) {
  byStatus[status] = orders.filter((o) => o.status === status);
}

// Good — single pass groups everything at once
const byStatus = orders.reduce((groups, order) => {
  (groups[order.status] ??= []).push(order);
  return groups;
}, {});
```
- **DON'T:** Assume a nested-loop pattern is "fine" just because it performs acceptably against the small dataset used in local development or in an example/test fixture — always sanity-check the actual expected production data volume for code with this shape, since the gap between "instant on 20 rows" and "the request now times out on 20,000 rows" is exactly the kind of gap that a nested O(n²) lookup produces and a hand-tested happy path won't reveal.

### Efficient Data Serialization Formats

- **DO:** Default to JSON for HTTP API payloads — it's universally supported, human-readable for debugging, and "fast enough" for the overwhelming majority of applications — and only reach for a binary serialization format (Protocol Buffers, MessagePack, Avro) when you have a measured, specific need it addresses that JSON genuinely doesn't meet (very high-throughput internal service-to-service calls where serialization overhead is a proven bottleneck, or a need for a strictly-enforced, versioned schema contract between services).
- **DON'T:** Introduce a binary serialization format and its associated tooling complexity (schema files, code generation, a schema registry) preemptively, before profiling has actually shown JSON serialization/parsing to be a measurable bottleneck for the specific workload — this is a specific instance of the general "don't optimize before measuring" principle, and binary formats carry real ongoing costs (debuggability, tooling, cross-language schema management) that need to be justified by an actual, demonstrated need.
- **DO:** Use Protocol Buffers (or a similar schema-driven format) specifically when strict, versioned, cross-language contract enforcement between services is a primary goal, not just raw speed — the schema-first workflow (defining a `.proto` file, generating typed client/server code from it) forces API contracts to be explicit and centrally versioned in a way that a loosely-typed JSON payload agreed upon by convention doesn't provide on its own.
```protobuf
message Order {
  string id = 1;
  repeated OrderItem items = 2;
  int64 total_cents = 3;
}
```
- **DO:** Consider `MessagePack` as a comparatively low-friction middle ground when the goal is simply a smaller, faster-to-parse wire format without adopting a full schema-driven toolchain — it's a binary format that maps very directly onto the same data model as JSON (objects, arrays, strings, numbers), so migrating an existing JSON-based API to it is usually a much smaller lift than adopting Protocol Buffers.
- **DON'T:** Mix serialization formats inconsistently across an API surface (some endpoints JSON, others a binary format) without a clear, documented reason and a `Content-Type`/`Accept`-header-driven negotiation mechanism — an API consumer needs to be able to reliably determine which format a given endpoint uses, and inconsistency without content negotiation forces every client to special-case each individual endpoint.

### Request Coalescing & Deduplication

- **DO:** Coalesce multiple concurrent, in-flight requests for the *same* underlying resource into a single actual fetch/query, sharing the resulting Promise across every caller that asked for it at roughly the same time — this avoids the "thundering herd" pattern where many nearly-simultaneous requests for the same currently-uncached data (a popular product page right after its cache entry expires, for instance) each independently trigger their own redundant, expensive downstream fetch at once.
```js
const inFlight = new Map();

async function getUserDeduped(id) {
  if (inFlight.has(id)) return inFlight.get(id); // join the already-running fetch
  const promise = fetchUserFromDb(id).finally(() => inFlight.delete(id));
  inFlight.set(id, promise);
  return promise;
}
// 50 concurrent calls to getUserDeduped(42) within the same tick trigger exactly one DB query
```
- **DO:** Use a batching/deduplication utility purpose-built for this pattern (the DataLoader library, originally from the GraphQL ecosystem but broadly applicable) when a system makes many small, individual lookups by key that could instead be coalesced into fewer batched calls within the same tick/request — this is effectively the request-level analog of the N+1 query problem covered in the Performance section's data-structure guidance, solved by batching at the call layer instead of restructuring the calling code's loop.
- **DON'T:** Cache and reuse an in-flight Promise across requests that need genuinely different data (different query parameters, different user-specific results) under the same shared cache key — deduplication is only correct when the concurrent callers are asking for *exactly* the same thing; conflating two different requests into one shared result because their cache key computation was too coarse is a data-correctness bug, not a performance win.
- **DON'T:** Let a deduplication/coalescing cache entry outlive the single in-flight request it's meant to cover — once the underlying fetch resolves (or rejects), remove it from the in-flight map (as shown via `.finally()` above) so the *next* logically-separate request for the same resource triggers its own fresh fetch rather than incorrectly reusing an already-completed one indefinitely, which would turn a request-deduplication mechanism into an unintentional, un-expiring cache.
- **DO:** Apply request coalescing specifically at points where redundant concurrent work is both likely and costly — a popular cache key right after expiry, a batch of related lookups issued in the same tick, an expensive computation multiple parts of a single request pipeline happen to need — rather than wrapping every function call in a deduplication layer regardless of whether concurrent duplicate calls are actually a realistic scenario for that specific call site.

### Caching

- **DO:** Cache the results of expensive, frequently-repeated, and slowly-changing computations or external calls (an expensive aggregation query, a third-party API response that doesn't change often), with an explicit, deliberately-chosen expiration/invalidation strategy (TTL, event-driven invalidation, or a versioned cache key).
- **DON'T:** Cache without a clear invalidation story. "Cache invalidation is one of the two hard problems in computer science" for a reason — a cache with no expiry and no invalidation trigger silently serves stale data indefinitely, which is often worse than the slow response the cache was meant to fix, because the staleness is invisible until someone notices incorrect data downstream.
- **DO:** Use a shared, external cache (Redis, Memcached) rather than an in-process cache for anything that needs to be consistent across multiple server instances — an in-process cache means each replica can serve a different, independently-stale answer for the same key.
- **DON'T:** Cache user-specific or permission-sensitive data under a cache key that doesn't include the user/tenant/permission context — a shared cache key across different users' requests can leak one user's data to another.

### Avoiding Redundant Serialization Round-Trips

- **DON'T:** Parse a request body to JSON, transform it, re-serialize it back to a string, and hand it off to another layer that immediately parses it again — each parse/stringify cycle is real, non-trivial CPU work for anything beyond a tiny payload, and a middleware chain or internal service boundary that repeats this cycle several times over the same data is paying that cost redundantly for no benefit, since nothing about the data actually needed to become a string and back again at each hop.
```js
// Bad — body is parsed, then needlessly re-stringified and re-parsed by the next layer
app.use((req, res, next) => {
  const parsed = JSON.parse(req.rawBody);
  req.processedBody = JSON.stringify({ ...parsed, receivedAt: Date.now() }); // stringified again...
  next();
});
app.post("/orders", (req, res) => {
  const body = JSON.parse(req.processedBody); // ...and immediately parsed right back
});

// Good — stays as a plain object across the whole in-process chain; only serialized at the actual boundary
app.use((req, res, next) => {
  req.processedBody = { ...JSON.parse(req.rawBody), receivedAt: Date.now() };
  next();
});
app.post("/orders", (req, res) => {
  const body = req.processedBody; // already a usable object, no re-parse needed
});
```
- **DO:** Keep data in its native in-memory representation (a plain object/array) as it passes between functions/middleware within the same process, and reserve actual serialization (`JSON.stringify`) for genuine boundaries — sending a response, publishing to a queue, writing to a cache — where the data truly needs to leave the process or cross a real I/O boundary.
- **DON'T:** Round-trip data through a cache's serialization layer more often than necessary — reading a cached value, deserializing it, immediately re-serializing an only-slightly-modified version, and writing it straight back is sometimes unavoidable, but when the same request handler needs to read and then update the same cached value multiple times in sequence, consider restructuring to do the read-modify-write as a single logical step rather than several redundant round-trips through the cache client's own (de)serialization.
- **DO:** Profile serialization/deserialization cost specifically for endpoints handling large payloads before assuming it's negligible — for genuinely large JSON payloads (multi-megabyte API responses, for instance), parse/stringify cost is measurable, which is one more reason (alongside the earlier N+1 and pagination guidance) to avoid sending more data than a client actually needs in the first place, rather than sending everything and relying on the client to filter it down.

### Bundling & Startup

- **DO:** Lazy-load large, rarely-used dependencies (a PDF generator, a heavyweight image-processing library) via dynamic `import()` rather than a static top-level import, when the code path that uses them is only hit occasionally — this keeps both bundle size (client-side) and cold-start memory/time (serverless functions) down for the common case.
- **DON'T:** Import an entire large utility library just to use one small function from it, when the library or a modern JS engine already gives you a native equivalent, or when the library supports named/subpath imports that avoid pulling in the whole thing. Many popular utility libraries have shrunk in relevance as native JS/Node APIs added equivalents (`Array.prototype.flat`, `Object.fromEntries`, `structuredClone`) — check whether you actually need the dependency before adding it.
```js
// Often unnecessary in modern Node/browsers
import _ from "lodash";
const flat = _.flatten(nestedArray);

// Native equivalent, no dependency
const flat = nestedArray.flat();
```
- **DO:** Measure and monitor cold-start time specifically for serverless/edge functions (AWS Lambda, Cloudflare Workers, Vercel Functions) — large dependency trees, heavy top-level initialization work, and large bundle sizes directly translate into slower cold starts, which matter far more in a serverless context than in a long-lived server process.
- **DON'T:** Do expensive, one-time setup work (loading a large ML model, establishing a database connection) inside a request handler when it could be done once at module load / outside the handler and reused across invocations. In serverless environments specifically, understand your platform's execution-context reuse model so genuinely reusable setup (like a database connection) is initialized once per warm container, not once per single request.

### Micro-Optimization Judgment

- **DON'T:** Rewrite clear, idiomatic code into a "clever," harder-to-read form purely on the belief it's faster, without first measuring that it matters. Modern JS engines (V8 especially) apply aggressive optimizations to idiomatic patterns, and a manually "optimized" version is often no faster in practice while being measurably harder to read and maintain — readability is the default; sacrifice it only with a benchmark to justify it.
- **DO:** Use `console.time`/`console.timeEnd`, Node's `perf_hooks` module, or a proper benchmarking library (`tinybench`, `benchmark.js`) to get real numbers before and after a performance change, on realistic data volumes — not a hand-wavy sense of "this feels faster."
- **DON'T:** Assume a synchronous operation is "fast enough" just because it's fast on a small local test dataset — always sanity-check performance-sensitive code against production-scale data volumes, since many O(n²)-or-worse algorithms are indistinguishable from O(n) on a 10-item test array and catastrophically different on a 100,000-item production dataset.
- **DO:** Set and monitor concrete performance budgets (P50/P95/P99 response time targets, bundle size limits, memory ceilings) as part of the definition of done for performance-sensitive features, rather than treating performance as something to worry about only after a user complains.

### Avoiding Excessive Promise Allocation in Hot Loops

- **DON'T:** Wrap every single iteration of a large, genuinely synchronous, CPU-bound loop in its own `async` call or `await` when there's no actual asynchronous work happening inside it — each `await`, even of an already-resolved value, schedules a microtask and adds real (small but non-zero) overhead; multiplied across a very large loop with no genuine I/O inside it, this overhead is pure waste with no corresponding benefit, since there was never anything to actually wait for.
```js
// Bad — awaiting a synchronous computation in a tight loop adds needless microtask overhead
async function processAll(items) {
  const results = [];
  for (const item of items) {
    results.push(await computeSync(item)); // computeSync has no actual async work inside it
  }
  return results;
}

// Good — no reason to involve Promises/await at all for purely synchronous work
function processAll(items) {
  return items.map(computeSync);
}
```
- **DO:** Reserve `async`/`await` for code paths that have genuine asynchronous work to perform, and keep purely synchronous, CPU-bound transformations as plain synchronous functions — mixing the two only where a real `await` is needed keeps both the performance characteristics and the code's intent (this specific step involves waiting on something external) clear and accurate.
- **DON'T:** Create large numbers of short-lived Promises in a hot path when a simpler synchronous or batched alternative would do the same job — for extremely hot, profiled code paths specifically, Promise allocation and microtask scheduling do have a measurable (if usually small) cost, and it's one more thing worth checking when a hot path has already been profiled and needs further optimization, though it's rarely the first thing worth reaching for before more impactful changes (algorithmic complexity, I/O batching) have been addressed.
- **DO:** Batch a large number of small async operations into fewer, larger ones where the underlying operation supports it (a single batched database `INSERT` of many rows instead of many individual single-row `INSERT`s each individually awaited in sequence) — this reduces both the number of Promises created and, far more importantly, the number of actual round-trips to the external system, which is almost always the dominant cost compared to any Promise-allocation overhead itself.

### Node.js-Specific Performance Patterns

- **DO:** Enable response compression (the `compression` middleware for Express, or gzip/Brotli at the reverse-proxy/CDN layer) for text-based HTTP responses — JSON, HTML, CSS, JS. It's typically a very high win-to-effort ratio: often a large reduction in transferred bytes for a few lines of middleware configuration.
- **DO:** Reuse a single database connection pool and a single HTTP keep-alive agent across the lifetime of the process, initialized once at startup, rather than opening a new connection or a new TCP handshake for every request. Connection setup (especially TLS handshakes) has real, measurable latency and resource cost that a pool amortizes away.
```js
// Bad — creates a brand-new pool (and its underlying connections) on every request
app.get("/orders", async (req, res) => {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  const result = await pool.query("SELECT * FROM orders");
  res.json(result.rows);
});

// Good — one pool, created once, reused across all requests
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
app.get("/orders", async (req, res) => {
  const result = await pool.query("SELECT * FROM orders");
  res.json(result.rows);
});
```
- **DON'T:** Instantiate a new HTTP client, database client, or connection pool inside a request handler, a Lambda handler body evaluated per-invocation without accounting for execution-context reuse, or any other per-call hot path. Each new client typically means new connection setup overhead paid on every single call instead of once.
- **DO:** Stream large HTTP response bodies (large file downloads, large export/report generation, big paginated dumps) directly to the response object as they're generated, instead of building the entire payload as an in-memory string/buffer/array and sending it all at once. Streaming keeps memory usage bounded and lets the client start receiving data immediately rather than waiting for the entire payload to be assembled server-side first.
- **DO:** Use `Buffer.from()`, `Buffer.alloc()`, or `Buffer.allocUnsafe()` (only when you will immediately overwrite every byte) to create buffers, never the deprecated `new Buffer()` constructor. The deprecated constructor's behavior was ambiguous and, in one historically significant misuse pattern, could allocate uninitialized memory that exposed leftover data from other parts of the process — a real, documented security issue in old code that still shows up in outdated tutorials and, occasionally, AI-generated snippets trained on them.
- **DON'T:** Synchronously `JSON.stringify()` a very large object (megabytes-scale) directly on the main thread in a request-serving hot path without considering the blocking cost — for genuinely large payloads, consider a streaming JSON serializer or moving the serialization to a worker thread, since synchronous stringification of a large object can measurably stall the event loop for other concurrent requests.
- **DO:** Use HTTP/2 or connection keep-alive appropriately for services making many outbound requests to the same host, and configure a reasonable `maxSockets`/agent pool size for outbound HTTP clients so the app doesn't either starve itself with too few concurrent connections or overwhelm a downstream service with too many.

### V8 & Garbage Collection Awareness

- **DO:** Understand, at a basic level, that V8 (Node's JavaScript engine) optimizes objects based on their "shape" (the set and order of properties an object has, internally called a hidden class) — creating many objects with the same consistent shape lets V8 optimize property access heavily, while objects that start with one shape and later have properties added/deleted/reordered dynamically force V8 to fall back to slower, more general property-access code paths.
```js
// Less optimal — objects end up with different shapes depending on branch taken
function createPoint(x, y, hasZ) {
  const point = { x, y };
  if (hasZ) point.z = 0; // sometimes adds a property after creation
  return point;
}

// More optimal — every object created has the same consistent shape from the start
function createPoint(x, y, hasZ) {
  return { x, y, z: hasZ ? 0 : null };
}
```
- **DON'T:** Add or delete properties from an object after its creation in a hot path (a function called extremely frequently, processing large volumes of data) — prefer initializing an object with its full, final set of properties at creation time, since this is one of the more impactful, well-documented V8-specific optimization patterns for genuinely hot code paths.
- **DO:** Understand that V8's garbage collector performs periodic pauses to reclaim memory, and that a very large heap with a lot of live (non-garbage) data can produce longer, more noticeable GC pauses — for latency-sensitive services, monitor GC pause time (via `--trace-gc` or a proper APM tool's GC metrics) as a real contributor to tail latency, not just average response time.
- **DON'T:** Hold onto large amounts of data in memory longer than necessary "for convenience" (caching more than is actually needed, keeping full request/response bodies around after they're done being used) — every retained byte is both a direct memory cost and makes each garbage-collection cycle marginally more expensive, since the collector has to trace through more live data on every pass.
- **DO:** Treat V8-level micro-optimization (hidden-class stability, avoiding megamorphic call sites, avoiding `arguments` object usage) as a tool reserved for code that's been profiled and confirmed to be a genuine, measurable hot path — for the vast majority of application code, this level of optimization is unnecessary complexity that trades readability for a performance gain nobody will ever notice, echoing the general Micro-Optimization Judgment guidance above.

### Memoization, Debouncing/Throttling & Worker Threads vs. Web Workers

- **DO:** Memoize the result of a genuinely expensive, pure, repeatedly-called-with-the-same-arguments function — cache the computed result keyed by its arguments, and reuse it on subsequent calls instead of recomputing. This is a real, measurable win specifically for expensive pure functions called repeatedly with a small set of distinct inputs (e.g., a recursive Fibonacci-style computation, an expensive parsing step run on the same input multiple times within a request).
```js
function memoize(fn) {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
```
- **DON'T:** Memoize a function whose inputs are rarely repeated, whose output depends on external mutable state (making the cached result stale/wrong on a later call), or whose own execution is already cheap — memoization trades memory (and cache-key computation cost) for CPU time, and applying it where the trade doesn't pay off just adds overhead and a stale-cache risk for no benefit.
- **DO:** Debounce handlers that respond to rapidly-repeating events where only the *last* occurrence in a burst actually matters (a search-as-you-type input, a window-resize handler that triggers an expensive relayout) so the expensive work runs once after the burst settles, not once per individual event.
- **DO:** Throttle handlers where you need a steady, rate-limited cadence of execution during a continuous stream of events (a scroll handler updating a progress indicator, a mousemove handler) rather than only the trailing edge — throttling guarantees the handler runs at most once per interval throughout the whole burst, not just once at the end.
```js
function debounce(fn, delayMs) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delayMs);
  };
}

function throttle(fn, intervalMs) {
  let lastCall = 0;
  return (...args) => {
    const now = Date.now();
    if (now - lastCall >= intervalMs) {
      lastCall = now;
      fn(...args);
    }
  };
}
```
- **DON'T:** Confuse debounce and throttle, or apply neither to a handler bound to a high-frequency event (`scroll`, `resize`, `mousemove`, `input`) that triggers non-trivial work — an unthrottled/undebounced handler on these events can fire dozens to hundreds of times per second, each invocation doing real work, which is a very common and very avoidable source of janky UI performance.
- **DO:** Understand the distinction between Node.js's `worker_threads` (for offloading CPU-bound work within a Node.js server process, sharing memory via `SharedArrayBuffer` where needed, and communicating via message passing) and browser Web Workers (the browser-environment equivalent, with its own separate API surface) — they are conceptually similar but are different APIs for different runtimes, and code/examples for one do not directly apply to the other.
- **DON'T:** Reach for a worker thread/process for every asynchronous task — workers have real overhead (thread/process creation cost, message-passing serialization cost for data crossing the boundary) that only pays off for genuinely CPU-intensive work; I/O-bound async work (network calls, file reads) is already non-blocking on the main thread via the event loop and gains nothing from being moved to a worker.

### Software Bill of Materials & Package Provenance

- **DO:** Generate a Software Bill of Materials (SBOM — a machine-readable manifest of every dependency and transitive dependency in a build, typically in CycloneDX or SPDX format) as part of the release/CI pipeline for any application with real compliance requirements or a security-conscious customer base — an SBOM turns "what exactly is running in production" from an implicit, hard-to-reconstruct question into an explicit, queryable artifact, which matters enormously the moment a new CVE is disclosed in some widely-used package and every team needs to quickly determine whether they're affected.
- **DO:** Enable npm's provenance attestation (`npm publish --provenance`, now well-supported when publishing from a supporting CI platform like GitHub Actions) when publishing a package — provenance cryptographically ties a published package version back to the exact source repository, commit, and build workflow that produced it, giving consumers a verifiable way to confirm a package actually came from its claimed source rather than, for instance, from a compromised maintainer account publishing a tampered version directly.
- **DON'T:** Treat "the package is on npm with a familiar name" as sufficient trust signal on its own for a dependency that will run with full application privileges — cross-reference a new or unfamiliar dependency's provenance, maintainer history, and download/usage patterns as part of the evaluation covered in the "Evaluating a New Dependency" section above, especially for anything pulled in as a direct (not just transitive) dependency of security-sensitive code.
- **DO:** Keep an SBOM (and any dependency-vulnerability scan results tied to it) as a living artifact regenerated on every release, not a one-time snapshot — a dependency tree that was clean at last quarter's audit can easily have a newly-disclosed vulnerability in it today, and the value of an SBOM is directly tied to how current it actually is.

## Common AI-Assistant Mistakes in JS/TS

This section names the specific failure patterns most characteristic of AI-generated JavaScript/TypeScript specifically — many overlap with rules already stated above, but are called out together here because they cluster in AI output often enough to warrant a dedicated, scannable checklist of their own.

### Insecure "Quick Fixes" AI Assistants Commonly Suggest

Several of the DON'Ts throughout this document share a common shape worth naming explicitly: a change that makes an immediate error or warning go away by disabling the safety mechanism that raised it, rather than by fixing what the mechanism was actually complaining about. This pattern is disproportionately common in AI-assisted debugging specifically, because "make the red error text disappear" is a tempting, easy-to-satisfy local objective that doesn't require understanding *why* the check existed in the first place — and it should be treated as a warning sign whenever it's the proposed fix, not applied reflexively.

- **DON'T:** Respond to a TypeScript type error by widening the type to `any`, adding a blanket `// @ts-ignore`, or adding a non-null assertion (`!`) without first understanding whether the compiler caught a real bug. The type error disappearing is not the same thing as the underlying problem being fixed — very often the compiler was correctly flagging a real, exploitable gap in the code's handling of missing/mismatched data.
- **DON'T:** Respond to a CORS error by setting `Access-Control-Allow-Origin: *` (especially alongside credentials), or by disabling CORS checks in a way that affects production traffic, when the actual fix is to add the specific legitimate origin to an allow-list. A CORS error in local development is very often the browser correctly protecting a user from a cross-origin request that shouldn't be allowed in production either.
- **DON'T:** Respond to a certificate/TLS error by disabling certificate verification (covered in detail above), respond to a "permission denied" filesystem error by recursively `chmod 777`-ing a directory instead of fixing the actual ownership/permission mismatch, or respond to an installation failure by reflexively adding `--legacy-peer-deps`/`--force` to an npm command without first understanding what dependency conflict is actually being papered over — each of these makes the specific error disappear while removing a real protection or masking a real, unresolved incompatibility.
```bash
# A quick-fix reflex that can hide a genuine, unresolved dependency conflict
npm install --legacy-peer-deps --force

# Better: understand what's actually conflicting, and resolve or explicitly override it
npm install # read the actual peer-dependency conflict error
npm ls some-conflicting-package # see what's requiring incompatible versions
```
- **DON'T:** Respond to a failing test by weakening its assertion, adding `.skip`, or deleting it, when the actual code has a bug the test correctly caught — recall the Testing section's guidance on regression tests: a test that starts failing after a change is evidence to investigate, not an obstacle to route around on the way to a green CI run.
- **DON'T:** Respond to an unhandled promise rejection or an uncaught exception by wrapping the offending code in a `try`/`catch` that silently swallows the error (an empty `catch` block, or one that only logs at `debug` level) instead of understanding what's actually failing and either fixing it or handling it meaningfully. The crash disappearing from view is not the same as the underlying failure being resolved — it usually just relocates the failure somewhere less visible, later, and harder to diagnose.
- **DO:** Treat every error, warning, or failing check as informative by default — something to understand before deciding how to respond to it — and reserve actually suppressing/disabling a check for the specific, rarer cases where the check is a genuine false positive, documenting *why* it's a false positive right at the suppression. The general habit to build (and to actively resist when it's tempting) is: fix what the check is protecting against, don't just silence the check.

### Silently Changing Public API Contracts

- **DON'T:** Change a function's parameter order, a shared type's shape, an exported module's public interface, or an HTTP endpoint's request/response shape while "improving" or "refactoring" it, without explicitly calling out that the change is breaking. Every one of these is a contract other code depends on, and a silent, unannounced change to any of them can compile cleanly in the file being edited while breaking every other caller elsewhere in the codebase (or, for a published package/public API, every external consumer) that was never touched or reviewed as part of the same change.
```ts
// Before
export function createInvoice(customerId: string, amount: number, currency: string): Invoice {}

// Silently "improved" to accept an options object — every existing call site now breaks
export function createInvoice(options: { customerId: string; amount: number; currency: string }): Invoice {}
```
- **DO:** Search the codebase for every existing call site of a function/type/endpoint before changing its signature or shape, and either update every call site as part of the same change, or — if that's not feasible in one pass — explicitly flag the change as breaking and list what else needs to be updated, rather than leaving other call sites silently broken for someone else to discover later via a compiler error or, worse, a runtime failure.
- **DON'T:** Assume a local change is "just an internal implementation detail" without verifying that the changed symbol is actually only used internally — an exported function, type, or class member is part of that module's public contract by definition, regardless of whether the specific change felt internal from the perspective of the file being edited.
- **DO:** When a breaking change to a public/shared interface is genuinely the right call, make it via a proper deprecation path where one is warranted (keep the old signature working alongside the new one, mark it `@deprecated`, migrate call sites over time) rather than an in-place breaking change with no transition period, especially for anything consumed outside the immediate codebase being edited (a published package, a public API, a widely-used internal library other teams depend on).
- **DON'T:** Describe a change that alters a function's behavior for existing valid inputs (not just its signature) as a pure refactor if the description implies no behavior change — echoing the "silently water down error handling" concern raised earlier in this section, any change to what a function *does* for a given input, not only what its signature *looks like*, needs to be called out explicitly when it wasn't asked for.

### Placeholder & Stub Code Presented as Complete

- **DON'T:** Deliver a function containing a `// TODO: implement this` comment, a `throw new Error("not implemented")` stub, or a hardcoded fake/mock return value, while presenting the overall task as finished. If a piece of requested functionality genuinely isn't implemented yet, say so explicitly and clearly — burying an incomplete implementation inside otherwise-plausible-looking code is far worse than leaving it out, because it looks done on a quick read and the gap is only discovered later, at the worst possible time.
```js
// Bad — looks like a real implementation; the mock data is easy to miss on a quick review
async function getExchangeRate(from, to) {
  // TODO: call the real exchange rate API
  return 1.1; // placeholder
}

// Better — impossible to mistake for a finished implementation
async function getExchangeRate(from, to) {
  throw new Error("getExchangeRate is not yet implemented — needs a real exchange-rate API integration");
}
```
- **DON'T:** Generate an error-handling branch, an edge case, or a validation rule that does nothing but silently pass through data unchanged, purely to make the code compile/run without actually addressing what that branch was supposed to handle — an empty or no-op handler for a case that was explicitly asked for is functionally the same as the placeholder-code problem above, just harder to spot because there's no comment flagging it.
- **DO:** Explicitly flag every genuine simplification, every skipped edge case, and every assumption made while implementing a request, in plain language, as part of the response — "I implemented the happy path; I did not add retry logic for network failures" is honest and useful; silently shipping only the happy path while implying full coverage is not.
- **DON'T:** Claim test coverage, error handling, or edge-case handling exists for something that was not actually verified to work — describing a stub as "handles errors gracefully" when the actual implementation is an empty `catch` block is a fabricated claim about the code's behavior, in the same category as the general fabrication concerns raised earlier in this section, and specifically dangerous because it actively discourages the reviewer from checking the part that most needs checking.
- **DO:** When a task is genuinely too large or ambiguous to complete fully in one pass, deliver a clearly-scoped, honestly-labeled subset that's fully real and working, along with an explicit list of what remains — this is far more useful than a full-surface-area response where some fraction of it is silently fake.

### Ignoring Existing Codebase Conventions

- **DON'T:** Introduce a new HTTP client, date library, testing utility, error class, or state-management pattern into a codebase that already has an established, working choice for that exact need, just because a different tool came more readily to mind for the specific snippet being written in isolation. A codebase with `axios` already used everywhere gaining one new file that uses raw `fetch` instead isn't "using a modern API" — it's a fork in the project's conventions that the next maintainer now has to notice, understand, and eventually reconcile.
- **DO:** Actually look at how similar, existing code in the same project handles the same category of problem (error handling shape, logging calls, validation approach, naming conventions, file organization) before writing new code that needs to do the same category of thing, and match that existing pattern unless there's a specific, stated reason to deviate from it. Consistency with the surrounding codebase is worth more than any individual file being written in someone's personal-preference "ideal" style.
```js
// The rest of the codebase consistently does this:
const result = await withServiceErrorHandling(() => orderService.create(input));

// Bad — a new file that quietly reinvents its own error-handling shape instead
try {
  const result = await orderService.create(input);
} catch (e) {
  return { success: false, msg: e.toString() };  // different shape than every other handler
}
```
- **DON'T:** Reimplement a utility function (a date formatter, a validation helper, a retry wrapper) that already exists elsewhere in the project, just because it wasn't immediately visible from the specific file being edited. A duplicate, slightly-different reimplementation of existing logic is a maintenance liability — two copies that inevitably drift apart as one gets updated and the other doesn't — and it signals the codebase wasn't actually searched before new code was written.
- **DO:** Match the existing codebase's formatting, naming, and structural conventions even when a different convention is, in isolation, arguably better — a pull request that introduces a new file using `camelCase` file naming into a project that consistently uses `kebab-case`, or that suddenly switches from named exports to a default export where every sibling file uses named exports, creates friction and inconsistency that outweighs whatever marginal individual-file benefit the different convention offered.
- **DON'T:** Assume a codebase's existing pattern is wrong or outdated without actually checking why it's there — an unusual-looking pattern (a seemingly redundant null check, an oddly-specific retry count, a `// eslint-disable` with no visible comment) sometimes exists because of a past incident or a constraint that isn't obvious from the code alone; when a change would remove or contradict such a pattern, that's a moment to ask/investigate rather than to "clean it up" unilaterally.

### Hallucinated & Incorrect APIs

- **DON'T:** Invent an npm package name that sounds plausible but doesn't exist, or isn't the package that actually does what's needed. This is the single most damaging AI-specific failure mode in this ecosystem — a hallucinated package name that gets committed to `package.json` either breaks the install outright, or, worse, can be pre-registered by an attacker under that exact name (a real, documented supply-chain attack technique sometimes called "slopsquatting") who ships malicious code under a name an AI assistant is statistically likely to hallucinate again for someone else. Always verify a package's existence, current maintenance status, and actual API against its real, current registry/documentation page before writing an import for it — never rely on a remembered or "this sounds right" package name.
- **DON'T:** Call a method or pass an option that doesn't exist on a library's current version, based on a pattern half-remembered from a different library, an older major version, or a similar-sounding API (e.g., mixing up Axios's config shape with `fetch`'s, or calling a Lodash-style method on a native array that doesn't have it). Confirm the method exists on the actual installed version's real API surface — don't pattern-match from a similar-sounding library and assume it transfers.
```js
// Bad — `.get()` with this options shape is Axios's API, not native fetch's
const response = await fetch("/api/users", { params: { active: true } }); // fetch ignores `params` entirely, silently

// Good — native fetch takes a full URL; query params must be built explicitly
const url = new URL("/api/users", window.location.origin);
url.searchParams.set("active", "true");
const response = await fetch(url);
```
- **DON'T:** Use a deprecated or removed Node.js API without checking whether it's still current — e.g., the legacy `new Buffer()` constructor, `url.parse()` (superseded by the WHATWG `URL` class), `fs.exists()` (removed; use `fs.access()` or check via `fs.stat()`/`fs.promises.access()`), or `crypto.createCipher`/`createDecipher` (deprecated in favor of `createCipheriv`/`createDecipheriv`, which don't have the older functions' weak key-derivation issues). Check a target API against the current, actively-maintained Node.js documentation for the project's actual Node version, not against training-data patterns that may reflect an old or removed API.
- **DON'T:** Reference a browser-only global (`window`, `document`, `localStorage`, `fetch` on older Node versions before it was built in) inside code that's meant to run in a plain Node.js server context, or reference a Node-only global (`process`, `Buffer`, `require`) inside code meant to run in a browser. Mixing runtime-specific globals across environments is a frequent cross-contamination bug when an assistant pattern-matches syntax from one context (a browser example) into another (a server file) without checking which globals are actually available where the code will run.
- **DON'T:** Assume a specific Node.js API is available without checking the project's actual minimum supported Node version — e.g., `fetch` and `structuredClone` are only global without a flag/polyfill from Node 18+/17+ respectively, top-level `await` requires ESM, and `Array.prototype.group`/`groupBy`-style methods are new enough to not be available in many still-common Node LTS versions. Check the API against the project's declared `"engines"` field and its actual deployment target, not against "this is available in Node in general."

### Structural & Style Mistakes

- **DON'T:** Mix `require()`/`module.exports` and `import`/`export` syntax within the same file or inconsistently across a project without a deliberate, documented reason. This happens when generated code stitches together snippets that assumed different module systems, and it's one of the fastest, most visible tells that code wasn't written (or wasn't reviewed) by someone tracking the whole file's context — see the Module System section above for the full treatment.
- **DON'T:** Default every function parameter, variable, and return type to `any` (or skip type annotations entirely in a way that lets them silently infer to `any`) as a way to "just make the TypeScript errors go away." This defeats the entire purpose of using TypeScript and is extremely common in AI-generated TypeScript specifically, because `any` is always the path of least resistance to a file that compiles — see the TypeScript Typing section above for the specific alternatives (`unknown`, proper unions, generics, schema validation).
- **DON'T:** Leave unhandled promise rejections uncaught — generating an `async` function or a `.then()` chain with no corresponding `.catch()`/`try`-`catch` anywhere in the call chain. This is easy to miss when generating code in isolated snippets/functions without tracing how each one will actually be invoked and whether its caller handles rejection — see the Async & Promises section above.
- **DON'T:** Over-nest callbacks into "callback hell" when `async`/`await` (or, at minimum, named intermediate functions and Promise chaining) would flatten the structure — this pattern shows up when an assistant defaults to the oldest, most broadly-applicable-looking async style (nested callbacks) rather than the idiomatic modern one for the target codebase's actual conventions.
```js
// Bad — classic callback-hell shape, easy for an assistant to fall back into
getUser(id, (err, user) => {
  if (err) return handleError(err);
  getOrders(user.id, (err, orders) => {
    if (err) return handleError(err);
    getOrderItems(orders[0].id, (err, items) => {
      if (err) return handleError(err);
      console.log(items);
    });
  });
});

// Good
try {
  const user = await getUser(id);
  const orders = await getOrders(user.id);
  const items = await getOrderItems(orders[0].id);
  console.log(items);
} catch (err) {
  handleError(err);
}
```
- **DON'T:** Wrap simple, stateless logic in an unnecessary class (a class with only static methods, or one instantiated exactly once with no meaningful state) — this is a recurring AI-generated-code pattern, likely because class-based examples are extremely common in training data for demonstrating "how to structure code," even where the actual logic has no state to encapsulate. See the "Classes vs. Functions & OOP Patterns" section above.
- **DON'T:** Add defensive code that guards against conditions that are already impossible given the code's own type signatures or preceding logic (e.g., a `typeof x === "function"` check on a parameter TypeScript already guarantees is a function, or a `null` check on a value the code two lines above just unconditionally assigned a non-null value to). Excess defensive checks that guard against impossible states are noise that makes readers doubt their own understanding of the code's actual guarantees, and are a common over-cautious AI-generated pattern.
- **DON'T:** Generate deeply nested conditional/ternary expressions in a single line as a substitute for clear, sequential logic — a nested ternary (`const x = a ? b ? c : d : e ? f : g;`) is a common "technically correct, practically unreadable" output pattern that should be refactored into an `if`/`else if` chain, a `switch`, or a lookup object/map.
```js
// Bad — technically works, genuinely hard to parse at a glance
const shippingCost = weight > 50 ? (express ? 45 : 30) : weight > 20 ? (express ? 25 : 15) : express ? 12 : 5;

// Good
function getShippingCost(weight, express) {
  if (weight > 50) return express ? 45 : 30;
  if (weight > 20) return express ? 25 : 15;
  return express ? 12 : 5;
}
```

### Copy-Paste Duplication vs. Parameterized Abstraction

- **DON'T:** Generate several near-identical functions/route handlers/schemas that differ only in a handful of literal values (an entity name, a field list, a status code) when a single, parameterized function or a small data-driven loop would express the exact same behavior once, correctly, and be maintained in one place going forward. Generating each case somewhat independently — one prompt/request at a time, or one file at a time — tends to produce exactly this kind of accidental duplication, where a human author working across the whole file at once would naturally have noticed the repetition and factored it out.
```js
// Bad — four near-identical handlers; a bug fixed in one is still present in the other three
app.get("/users/:id", async (req, res) => {
  const user = await db.users.findById(req.params.id);
  if (!user) return res.status(404).json({ error: "user not found" });
  res.json(user);
});
app.get("/products/:id", async (req, res) => {
  const product = await db.products.findById(req.params.id);
  if (!product) return res.status(404).json({ error: "product not found" });
  res.json(product);
});
// ...repeated again for /orders/:id and /invoices/:id

// Good — one parameterized factory, one place to fix a bug or add a cross-cutting concern
function makeGetByIdHandler(model, entityName) {
  return async (req, res) => {
    const record = await model.findById(req.params.id);
    if (!record) return res.status(404).json({ error: `${entityName} not found` });
    res.json(record);
  };
}
app.get("/users/:id", makeGetByIdHandler(db.users, "user"));
app.get("/products/:id", makeGetByIdHandler(db.products, "product"));
```
- **DON'T:** Duplicate a validation schema, a type definition, or a constant list of allowed values across multiple files instead of defining it once and importing it everywhere it's needed — each duplicate is a place the two copies can silently drift apart the next time only one of them gets updated, and the drift is often only discovered when it causes a confusing, hard-to-trace bug far later.
- **DO:** Recognize the specific trigger for this pattern — being asked to "add the same kind of thing for X" as something that already exists for Y — as a deliberate signal to go find and reuse/parameterize the existing implementation, rather than writing a fresh, independent copy of similar logic from scratch because the existing one wasn't in the immediate context being edited.
- **DON'T:** Over-correct into premature, forced abstraction after spotting only two similar-looking pieces of code, especially when they're similar today but plausibly need to diverge independently in the near future (two validation rules that happen to look alike now but govern genuinely different business concepts). The classic guidance of preferring duplication over the *wrong* abstraction still applies — the goal is recognizing genuine, stable repetition, not mechanically merging anything that looks superficially similar.
- **DO:** Apply the "rule of three" as a practical, non-dogmatic heuristic — two occurrences of similar logic might still be coincidental or likely to diverge, but a third occurrence is a strong signal the pattern is real and stable enough to be worth extracting into a shared function, schema, or constant.

### Outdated Patterns & Legacy API Usage

- **DON'T:** Reach for patterns and APIs that were once idiomatic but have since been superseded by better, now-standard alternatives, without checking whether the target codebase actually still needs the old approach. Training data spans many years of JavaScript's evolution, and an assistant that doesn't check the project's actual dependencies/Node version can default to outdated idioms that were the best available answer years ago but no longer are.
```js
// Outdated — the `request` package has been deprecated and unmaintained for years
const request = require("request");
request("https://api.example.com/data", (err, res, body) => { /* ... */ });

// Current — native fetch, no dependency needed at all on modern Node
const res = await fetch("https://api.example.com/data");
const body = await res.text();
```
```js
// Outdated — moment.js is in official maintenance mode and explicitly recommends against new usage
const moment = require("moment");
const formatted = moment().format("YYYY-MM-DD");

// Current — a lighter, actively-recommended alternative (or native Intl for simple cases)
import { format } from "date-fns";
const formatted = format(new Date(), "yyyy-MM-dd");
```
- **DON'T:** Use callback-style Node.js core APIs (`fs.readFile(path, (err, data) => {})`) by default in new code written for a modern Node.js target — every core module with a callback API also ships a Promise-based counterpart (`fs/promises`, `dns/promises`, `timers/promises`), which composes far better with `async`/`await` and avoids manually threading error-first callbacks through the rest of the function.
- **DON'T:** Generate code using `var`, the `arguments` object in new function definitions, prototype-based inheritance boilerplate (`Foo.prototype.bar = function () {}`) for new class-like structures, or CommonJS as the default module system for a brand-new project with no legacy constraint forcing it — these were the idiomatic patterns before ES2015 classes/modules/`let`/`const` existed, and generating them by default (rather than because the specific project genuinely still uses them) is a strong "this wasn't checked against the actual target codebase" signal.
- **DO:** Actually check a target codebase's existing conventions, its `package.json` `"type"` field, its `tsconfig.json`/Node engine target, and its already-installed dependencies before choosing which era of JavaScript idiom to generate — the correct pattern to use is "whatever this specific project already uses," not "whatever pattern appears most frequently across all training data regardless of recency."
- **DON'T:** Suggest a specific version-gated API or syntax feature (optional chaining, nullish coalescing, top-level await, a newly-added `Array` method) without confirming it's actually supported by the project's minimum target Node.js/TypeScript/browser version — using next year's syntax in a codebase pinned to an older runtime produces code that looks modern and correct but throws a syntax error the moment it's actually run.

### Over-Engineering & Under-Engineering

- **DON'T:** Introduce an interface, an abstract base class, or a dependency-injection framework for a piece of logic that has exactly one real implementation and no near-term plan for a second one. Speculative abstraction built for a flexibility need that doesn't yet exist ("YAGNI" — You Aren't Gonna Need It) adds indirection that costs every future reader time, in exchange for flexibility that may never be used, and that's easy to add later exactly when it's actually needed.
```ts
// Over-engineered for a single, stable implementation with no near-term second one
interface EmailSender { send(to: string, subject: string, body: string): Promise<void>; }
class SendgridEmailSender implements EmailSender { /* ... */ }
class EmailService {
  constructor(private sender: EmailSender) {}
}

// Right-sized for the actual current need
async function sendEmail(to: string, subject: string, body: string) {
  return sendgridClient.send({ to, subject, html: body });
}
```
- **DON'T:** Under-engineer the opposite way either — shipping a function with no input validation, no error handling, and no consideration of concurrent/edge-case behavior because the happy path "looks done." Both failure modes come from the same root cause: not actually thinking through what the code needs to handle, just generating something that superficially satisfies the immediate request.
- **DO:** Calibrate the amount of abstraction and defensive engineering to the actual, current requirements and the code's actual blast radius — a one-off internal migration script warrants far less defensive engineering than a public API endpoint handling untrusted payment data, and treating every piece of code with identical ceremony (either maximal or minimal) ignores that real difference in stakes.
- **DON'T:** Add configuration options, feature flags, or extensibility hooks to a function/module "in case they're needed later" when nothing in the current requirements calls for them. Unused flexibility is not free — it's more surface area to test, document, and reason about, for a need that's speculative rather than real.
- **DO:** Recognize that the right amount of engineering effort is the amount that solves the actual stated problem clearly and correctly, handles the edge cases that are plausible for the code's actual inputs, and is easy for the next person (human or AI) to extend when a genuinely new requirement shows up — not the maximum possible robustness, and not the minimum that merely compiles.

### Fabrication, Overconfidence & Missing Verification

- **DON'T:** Present generated code as tested or verified when it hasn't actually been run. If code hasn't been executed against real inputs (or at minimum against the project's actual type checker and linter), say so explicitly rather than implying confidence the code doesn't yet warrant — a claim of "this works" that's actually "this looks like it should work" is a trust-destroying gap once it's discovered.
- **DON'T:** Fabricate specific version numbers, benchmark results, or API behavior details that weren't actually looked up or measured, when a request calls for factual specificity (e.g., "this API was added in Node 18.4.0" stated with confidence but not actually verified against the changelog). State genuine uncertainty plainly rather than filling a gap with a plausible-sounding but unverified specific claim.
- **DON'T:** Silently drop, water down, or subtly change error handling, edge-case handling, or security checks present in original code while "refactoring for readability" or "simplifying." A refactor that changes behavior — even to code that looks cleaner — needs to be flagged explicitly, not slipped in as if it were behavior-preserving.
- **DON'T:** Claim a fix resolves an issue without a plausible mechanism connecting the change to the reported symptom, and without having reasoned through (or ideally tested) whether the change actually addresses the root cause rather than a coincidentally-related symptom. "I changed something near the error and the error message changed" is not the same as "I found and fixed the actual bug."
- **DON'T:** Generate a large volume of boilerplate/scaffolding code with no genuine logic difference across many near-identical files (near-duplicated CRUD handlers, near-duplicated validation schemas) when a shared, parameterized helper/generic function would express the same behavior once. Repetition at this scale is a common AI-output pattern — driven by generating each file somewhat independently — that a human author reviewing their own code would naturally consolidate.
- **DON'T:** Assume the newest-sounding syntax or API is definitely available/appropriate for the target project without checking its actual toolchain (TypeScript version, Node version, bundler, browser support targets). Using top-level `await`, a very recent `Array.prototype` method, or a bleeding-edge TC39 stage-2/3 proposal syntax in a codebase that doesn't support it produces code that looks modern and correct but simply doesn't run.
- **DO:** When genuinely uncertain whether a package, API, or language feature exists or behaves as expected, say so explicitly and suggest how to verify it (check the package registry, check the changelog, run a quick test) rather than presenting a guess with unwarranted confidence. An honest "I'm not certain this method exists on this version — worth double-checking" is far more useful than a fluent, wrong answer stated as fact.

## Quick Checklist
- Names are meaningful, consistently cased, and searchable — no cryptic single-letter vars outside tight loop scopes.
- `const` by default; `let` only for genuine reassignment; no `var`.
- No `any` used as an escape hatch — use `unknown` plus narrowing, or a real type.
- Every `async` function's rejections are handled — no unhandled-promise-rejection warnings.
- No mixing of `require`/`import` within the same module; match the project's module system.
- Errors carry enough context (what failed, with what input) — no bare `throw new Error("failed")`.
- No swallowed `catch` blocks that discard the error without logging, rethrowing, or handling it.
- `package.json` dependencies are pinned/lockfile-controlled; no unpinned `latest`-tag installs in production configs.
- ESLint/Prettier (or the project's actual configured tooling) run clean before code is presented as done.
- Tests actually exercise behavior, not just call the function and assert nothing.
- No blocking synchronous I/O (`fs.readFileSync` and friends) on a hot server request path.
- User input is validated/sanitized before use in a query, shell command, or HTML output.
- No `eval`, `new Function(...)`, or `dangerouslySetInnerHTML`-style sinks fed with untrusted input.
- Promises are awaited or explicitly returned — no silently-dropped floating promises.
- No `==` where `===` is meant; equality checks are intentional.
- Large lists render/paginate rather than loading everything into memory or the DOM at once.
- No invented npm package, method, or config flag presented without genuine confidence it exists.
- No claim that "tests pass" or "this is fixed" without having actually run something.
- No stub/mock/placeholder data left in code presented as the finished answer.
- Refactors that touch error handling or validation are flagged explicitly, not slipped in silently.
- Near-duplicated files/handlers are consolidated into one parameterized implementation where practical.
- New syntax/APIs are checked against the project's actual Node/TypeScript/bundler target before use.
