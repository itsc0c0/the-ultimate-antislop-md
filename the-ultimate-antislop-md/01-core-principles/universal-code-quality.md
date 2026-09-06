# Universal Code-Quality Principles

This part covers the principles that hold regardless of which language, framework, or runtime you're writing for. Language-specific idiom and syntax live in later parts of this document; what follows is the reasoning that should survive a rewrite from one language into another. Treat it as the substrate every other part builds on: a codebase can follow every framework convention in this document to the letter and still be slop if the names lie, the functions do six things each, the errors vanish silently, and every change balloons because nobody trusted the code enough to keep it simple.

## Naming

Names are the primary interface between the author's intent and every future reader's understanding, including the author's own understanding six months later. A codebase is read far more often than it is written, and a name is read every single time the thing it labels is used — so a few extra seconds spent choosing a name pays for itself hundreds of times over. Treat naming as a design activity, not an afterthought to be fixed in review.

### General Naming Quality

- **DO:** Choose names that reveal intent without needing a comment to explain them. If you feel the urge to add a comment next to a variable or function explaining what it actually holds or does, that explanation belongs in the name itself.
```
// Bad: needs a comment to be understood
int d; // elapsed time in days

// Good: self-explanatory
int elapsedTimeInDays;
```

- **DO:** Make names searchable and greppable. Single-letter names and magic numbers are painful to search for across a codebase; a name like `MAX_RETRY_COUNT` can be found instantly, while `5` cannot be distinguished from every other `5` in the file.

- **DO:** Prefer names that are easy to pronounce and discuss out loud. Code is discussed in meetings, code reviews, and pairing sessions; a name like `genymdhms` forces every conversation about it into spelling it out letter by letter instead of just saying it.

- **DON'T:** Use names that differ by only a hard-to-notice character or by a similar-looking word. `userInfo` and `userInfor`, or `accountsList` and `accountList` sitting side by side in the same scope, invite typo-driven bugs and force readers to double- and triple-check which one they're looking at. Use names that are visually and semantically distinct.

- **DON'T:** Encode the data type or container type into a variable name (Hungarian-notation-style prefixes such as `strName`, `arrUsers`, `iCount`). Modern languages and editors already surface type information; baking it into the name adds noise and becomes actively wrong the moment the underlying type changes but nobody updates the name.
```
// Bad: name lies the moment the type changes
List<String> strUserNames = fetchUserNames(); // later refactored to a Set — name still says List

// Good: name describes what it is, not how it's implemented
Collection<String> userNames = fetchUserNames();
```

- **DON'T:** Reuse the same name for two different concepts in overlapping scopes (shadowing) unless the language idiom explicitly expects it. A reader who sees `result` in an outer scope and a different `result` in a nested block has to actively track which one is in play at every line.

- **DO:** Keep a consistent vocabulary for a given concept across the whole codebase. If you call it `fetch` in one module, don't call the equivalent operation `get`, `retrieve`, and `load` in three other modules for no reason — pick one verb per concept and use it everywhere that concept appears.

- **DON'T:** Use the same word for two unrelated concepts (false cognates). If `add` means "insert into a collection" in one class, don't also use `add` to mean "sum two numbers" in a neighboring class — a reader who has learned the meaning in one place will misapply it in the other.

- **DON'T:** Reach for a numeric suffix (`data1`, `data2`, `userTemp`, `resultNew`) as a substitute for actually explaining how two similarly-purposed things differ. A numbered name tells the reader that two things are related but hides the one piece of information they actually need — what distinguishes them.
```
// Bad: the numbering hides what actually differs between the two
function mergeData(data1, data2) { ... }

// Good: the names say what each one actually is
function mergeData(existingRecord, incomingUpdate) { ... }
```

### Casing and Formatting Conventions

- **DO:** Pick one casing convention per identifier category (variables, functions, types, constants, files) and apply it uniformly across the codebase, following whatever the language's ecosystem already treats as standard. Consistent casing turns naming style into something a reader stops noticing, which is exactly the goal — it should fade into the background.

- **DON'T:** Mix casing styles for the same kind of identifier within one file or module (e.g., `user_name` next to `userAge` next to `UserEmail` all as local variables). Inconsistent casing is a small thing individually but adds up to a codebase that constantly signals "different rules apply here," which erodes trust in every other convention too.

- **DO:** Reserve visually distinct casing (such as all-caps) for the category of thing your ecosystem conventionally marks that way — typically true constants — and nothing else, so that casing itself carries information instead of being decorative.

### Abbreviations vs. Verbosity

- **DON'T:** Invent cryptic, project-specific abbreviations that aren't already standard in the domain or the ecosystem. `usrCfgMgr` saves a few keystrokes for the author and costs every future reader a moment of decoding; that trade is almost never worth it.
```
// Bad: cryptic and ambiguous
def calc_disc_amt(ord, cust_typ):
    ...

// Good: readable without extra context
def calculate_discount_amount(order, customer_type):
    ...
```

- **DO:** Allow abbreviations that are genuinely universal in the domain (`id`, `URL`, `HTML`, `min`/`max`, `config` when it's the established shorthand in that ecosystem) — the test is whether a new team member would already know the term, not whether the author finds it obvious.

- **DON'T:** Swing to the opposite extreme and produce needlessly long, over-qualified names that repeat context already given by the surrounding scope, type, or namespace. A method on an `Account` class called `getAccountBalance` is redundant with its own receiver; `getBalance` says the same thing with less noise.
```
// Bad: redundant with the enclosing type
class Invoice {
    getInvoiceTotal() { ... }
    setInvoiceTotal(value) { ... }
}

// Good: the type already provides the context
class Invoice {
    getTotal() { ... }
    setTotal(value) { ... }
}
```

- **DO:** Match name length to scope size and lifetime. A loop index used for two lines inside a tight, obvious loop can be a short conventional name; a value that lives across a long function, is a parameter, or is exported needs a fuller, descriptive name because its context isn't immediately visible at every point of use.

### Naming by Role, Not by Type or Implementation

- **DO:** Name things after what they represent or what they do in the domain, not after how they happen to be implemented right now. A collection of active users should be named for that fact (`activeUsers`), not for its current data structure (`activeUserSet` when a set is just today's implementation detail).

- **DON'T:** Bake a specific algorithm, data structure, or storage mechanism into a name unless that detail is genuinely and permanently part of the contract. `usersHashMap` forces a rename the day someone swaps in a different structure for legitimate performance reasons, even though nothing about the concept changed.

- **DO:** Name functions after the action they perform and, where relevant, what they return, using a verb-first pattern for actions and a noun/adjective pattern for values and predicates. A function named `processData` tells the reader nothing about what "processing" actually means; `normalizeWhitespace` or `deduplicateRecords` tells them exactly what to expect.
```
// Bad: says nothing about the actual behavior
function process(items) { ... }

// Good: name is the specification
function removeExpiredItems(items) { ... }
```

- **DON'T:** Give a function a name that describes only part of what it does, especially when the omitted part is a side effect. If a function named `getUser` also writes an audit-log entry and updates a last-seen timestamp, the name is actively hiding behavior the caller needs to know about.

- **DO:** Name a predicate function so that it reads naturally in an `if` statement or a boolean context. `canUserEdit(user, document)` reads as a sentence at its call site; a name like `userEditCheck(user, document)` forces the reader to mentally translate it into a question before they can understand the branch it controls.

### Boolean Naming Conventions

- **DO:** Prefix boolean variables, fields, and functions with a form that reads as a yes/no question — `is`, `has`, `can`, `should`, `did` — so that every use site reads like a sentence and the polarity is unambiguous.
```
// Bad: ambiguous polarity, reads awkwardly at the call site
if (userStatus) { grantAccess(); }

// Good: reads as a question with an obvious answer
if (user.isActive) { grantAccess(); }
```

- **DON'T:** Name a boolean as a bare noun or adjective without a predicate prefix (`visible`, `enabled`, `status`) when a clearer question-form name is available — a bare adjective is fine only when the surrounding type already makes it read unambiguously (e.g., a `visible` field directly on a `Widget`), and even then a prefix rarely hurts.

- **DON'T:** Use negative boolean names (`isNotValid`, `isDisabled` when the more common check is really "is this enabled") if it can be avoided, because double negatives at the call site (`if (!isNotValid)`) are a frequent source of logic errors and always cost the reader an extra beat of mental inversion.
```
// Bad: forces double negation at call sites
if (!user.isNotVerified) { proceed(); }

// Good: positive phrasing composes cleanly with `!`
if (user.isVerified) { proceed(); }
```

- **DO:** Keep boolean names binary in meaning. If a concept actually has three or more states ("pending," "approved," "rejected"), model it as an enumeration or status value instead of stringing together multiple booleans (`isPending`, `isApproved`, `isRejected`) that can accidentally end up in an impossible combined state.

### Names That Lie

- **DON'T:** Leave a name unchanged after the behavior it describes has changed. A function called `validateInput` that was later modified to also mutate or normalize the input is now lying to every caller who trusts the name; either rename it or split the mutation out.
```
// Bad: name promises read-only validation, but it also mutates
function validateEmail(user) {
    user.email = user.email.trim().toLowerCase(); // silent side effect
    return EMAIL_REGEX.test(user.email);
}

// Good: name and behavior match, or the behaviors are split
function normalizeEmail(user) { user.email = user.email.trim().toLowerCase(); }
function isValidEmail(email) { return EMAIL_REGEX.test(email); }
```

- **DON'T:** Use a plural name for something that holds a single item, or a singular name for a collection. `userList` that actually holds one `User` object, or `item` that actually holds an array, forces every reader to override what the name told them by re-reading the actual usage.

- **DON'T:** Give two things that behave differently deceptively similar names (`processOrder` vs. `processOrders` where one validates and the other charges a payment). If two operations aren't the same operation at different cardinalities, don't name them as if they were.

- **DO:** When you must rename something because its old name became misleading, rename it everywhere in the same change — a partial rename (new name in the declaration, old name still lingering in a comment, log message, or a sibling variable) recreates the exact confusion the rename was meant to fix.

### Naming Files, Modules, and Namespaces

- **DO:** Name files and modules after the single primary thing they contain or the single responsibility they group, so a directory listing alone gives a reader a rough map of the codebase without opening anything. A file called `helpers.js` or `utils.py` that accumulates unrelated functions over time stops being a name at all — it's just a place things get dumped.

- **DON'T:** Let a catch-all module (`utils`, `common`, `misc`, `helpers`) become the default destination for anything that doesn't obviously belong elsewhere. These modules tend to grow without bound precisely because their name imposes no constraint on what can go in them; a name that means "everything" communicates nothing about any specific thing inside it.

- **DO:** Keep a consistent naming pattern for parallel files across a codebase (test files, configuration files, type-definition files, style files) so that a reader who has learned the pattern in one part of the codebase can predict where to find the corresponding file anywhere else, without needing to search.

### Avoiding Noise Words and Redundant Context

- **DON'T:** Pad names with content-free filler words that add length without adding meaning — `data`, `info`, `object`, `manager`, `handler`, `helper` used as the entire meaningful part of a name rather than as a genuine, specific role. `userData` for something that's just a `User` says nothing that `User` didn't already say.
```
// Bad: noise words that carry no real information
class UserInfoManagerHelper { ... }
function processUserData(userData) { ... }

// Good: names carry only the information that's actually meaningful
class UserRepository { ... }
function normalizeUser(user) { ... }
```

- **DON'T:** Repeat context in a name that the surrounding scope, namespace, or module already provides. Inside a module already named `orders`, a function called `createNewOrderRecord` is carrying two redundant words (`Order`, and arguably `New`/`Record`) that the module name and the domain already imply.

- **DO:** Let the smallest correct name win. Before finalizing a name, try removing each word from it one at a time and ask whether the name still reads unambiguously in its actual context — any word whose removal doesn't cost clarity should usually be cut.

### Domain Vocabulary

- **DO:** Name things using the same terms the domain experts and end users of the system already use for those concepts, rather than inventing new technical-sounding synonyms. If the business calls something a "hold," code that instead calls it a `PendingLock` forces every conversation between engineers and domain experts to include a mental translation step, and that translation gap is where misunderstandings and bugs creep in.
```
// Bad: invents technical jargon that doesn't match how the business talks
class TransactionSuspensionRecord { ... }

// Good: matches the term the domain (and support tickets, and the product
// spec) actually uses for this concept
class Hold { ... }
```

- **DON'T:** Let the same domain concept acquire multiple different names across different parts of a codebase because different engineers, at different times, each independently translated the domain term into their own preferred technical vocabulary. A single concept should have a single name that both engineers and domain experts recognize, end to end through the system.

- **DO:** Push back, respectfully, when the domain's own term for something is itself genuinely ambiguous or overloaded to mean two different things in different contexts. Adopting a business's vocabulary verbatim is the default, not an absolute rule — occasionally the code needs to draw a distinction the business's informal language doesn't bother to draw, and that's worth resolving collaboratively rather than baking the ambiguity into the code.

- **DO:** Update code vocabulary when the business's own vocabulary for a concept changes. Code that still says `client` after the product and the rest of the company have moved to calling that concept `member` is a growing source of confusion for anyone trying to relate the code to current conversations, tickets, or specs about the system.

### Symmetry in Paired Operations

- **DO:** Use conventional, symmetric verb pairs for operations that are natural opposites — `open`/`close`, `start`/`stop`, `add`/`remove`, `connect`/`disconnect`, `lock`/`unlock` — rather than inventing a mismatched pair for one side (`open`/`terminate`, `add`/`clear`). A reader who has found one half of a pair should be able to guess the name of the other half correctly on the first try.
```
// Bad: mismatched, unpredictable pairing
function beginSession() { ... }
function terminateSession() { ... }

// Good: a conventional, predictable pair
function startSession() { ... }
function stopSession() { ... }
```

- **DON'T:** Give the two ends of a resource's lifecycle (acquire/release, subscribe/unsubscribe, register/deregister) unrelated names that don't visually or lexically pair with each other. Mismatched pairs make it harder to grep for "everywhere this resource gets released" and harder to visually audit that every acquire has a matching release nearby.

### Encoding Units and Quantities

- **DO:** Include the unit in the name of any variable or parameter representing a quantity where more than one unit is plausible — time durations, sizes, distances, currency amounts. `timeoutMs` versus a bare `timeout` removes an entire, very common class of bug where one part of the codebase assumes milliseconds and another assumes seconds.
```
// Bad: unit is ambiguous — is this seconds, milliseconds, or a whole Duration object?
function setTimeout(timeout) { ... }
retryConnection(timeout: 5000); // five milliseconds? five seconds?

// Good: the unit travels with the name, impossible to misread
function setTimeoutMs(timeoutMs) { ... }
retryConnection(timeoutMs: 5000);
```

- **DON'T:** Mix units for the same conceptual quantity across a codebase without making the unit explicit at every boundary where values move between them (one module storing durations in milliseconds, another in seconds, with no naming or type distinction between the two). This is one of the most common and most expensive categories of real-world bugs, and it is almost entirely preventable through naming discipline alone.

- **DO:** Prefer a language or library's dedicated type for a quantity (a duration type, a money type, a distance type) over a bare number whenever one is available, since a dedicated type can enforce unit-correctness at compile time or construction time instead of relying purely on naming convention to prevent mixing units.

## Functions & Methods

Functions are the unit of reasoning in almost every codebase: a reader typically has to hold one function's worth of logic in their head at a time, decide whether they trust it, and move on. Every property in this section exists to keep that unit small enough, honest enough, and predictable enough that the reader can actually do that.

### Single Responsibility and Function Size

- **DO:** Give every function one reason to change. A function that both computes a value and writes it to a database and sends a notification has three unrelated reasons to be modified in the future, and a change to any one of those concerns risks breaking the other two.

- **DO:** Keep functions small enough to read in one sitting without scrolling and without losing track of the branches. There's no universal line-count rule that fits every language, but if you can't summarize what a function does in a single short sentence without using the word "and" more than once, it's doing more than one job.
```
// Bad: one function juggling validation, computation, and I/O
function submitOrder(order) {
    if (!order.items.length) throw new Error("empty order");
    let total = 0;
    for (const item of order.items) total += item.price * item.qty;
    total = applyDiscounts(total, order.customer);
    db.save(order);
    emailService.send(order.customer.email, "Order confirmed");
    return total;
}

// Good: each function has a single, nameable job
function validateOrder(order) { ... }
function calculateTotal(order) { ... }
function persistOrder(order) { ... }
function notifyCustomer(order) { ... }
function submitOrder(order) {
    validateOrder(order);
    const total = calculateTotal(order);
    persistOrder(order);
    notifyCustomer(order);
    return total;
}
```

- **DO:** Extract a well-named function whenever you notice a block of code that needs a comment to explain what it's doing as a unit. The extracted function's name replaces the comment and becomes independently testable.

- **DON'T:** Let a function grow simply because "it's already here" and adding one more branch feels cheaper than extracting a new function. Functions accrete responsibilities gradually, not all at once, so each addition should be evaluated against what the function is supposed to do, not just against how easy it is to bolt on.

- **DO:** Keep the operations inside a function at a single level of abstraction. Mixing high-level orchestration ("process the payment") with low-level detail (manually formatting a request payload byte by byte) in the same function forces the reader to context-switch between "what" and "how" line by line.

- **DON'T:** Treat a name containing "And" (`validateAndSave`, `fetchAndTransform`) as an acceptable permanent shape for a function, even though it accurately describes what the function does. The conjunction in the name is a direct symptom of the function itself doing more than one job; either it's fine as a small, deliberate orchestration step that calls two well-named single-purpose functions, or it's a sign those two responsibilities should be split apart and called separately by whatever needs both.

### Minimizing Side Effects

- **DO:** Make a function's side effects (writes to disk, network calls, mutation of shared or passed-in state, global state changes) obvious from its name and its position in the codebase. A caller should never be surprised that calling something named `calculateTax` also mutated the input object it was given.

- **DON'T:** Mutate arguments passed into a function unless mutation is the function's clearly documented and expected purpose. A function that silently mutates an object it was handed creates action-at-a-distance bugs where a caller three call-frames away is affected by a change they can't see at the call site.
```
// Bad: silently mutates the caller's object
function applyDiscount(cart) {
    cart.total = cart.total * 0.9; // caller has no idea this happened
    return cart.total;
}

// Good: returns a new value, leaves the input untouched
function calculateDiscountedTotal(cart) {
    return cart.total * 0.9;
}
```

- **DO:** Isolate side-effecting code (I/O, external calls, global mutation) at the edges of the system, and keep the core logic that makes decisions as side-effect-free as possible. This makes the decision-making logic trivially testable without mocks or fakes, and confines the harder-to-test parts to a thin, well-understood boundary layer.

- **DON'T:** Have a function silently depend on or modify state that isn't visible in its signature (a module-level variable, a singleton, an ambient "current user" context) when a normal parameter would do. Hidden inputs and outputs are the single biggest reason a function that "worked yesterday" mysteriously breaks after an unrelated change elsewhere in the file.

### Argument Count and Parameter Design

- **DO:** Keep the number of parameters small. A function needing zero, one, or two parameters is easy to call correctly from memory; beyond three or four, callers routinely have to look up the signature every time, and the odds of passing arguments in the wrong order increase.

- **DO:** Group related parameters that are always passed together into a single object or structured value. If four out of six parameters always travel together, they're really describing one concept (an address, a date range, a set of options) and deserve to be named as one.
```
// Bad: long, order-dependent parameter list
function createUser(firstName, lastName, email, street, city, zip, country) { ... }

// Good: related parameters grouped into meaningful structures
function createUser(name, email, address) { ... }
```

- **DON'T:** Add an output parameter (a parameter the function writes into instead of returning a value) when the language and context comfortably support returning a value, including multiple values via a structured return type. Output parameters obscure what the function actually produces and make the call site read like an input when it's actually receiving a result.

- **DO:** Give parameters names precise enough that a reader skimming a call site can guess what's being passed without jumping to the definition, especially in languages without mandatory named arguments — this is what keeps a three-argument call from turning into an unreadable wall of positional literals.

### Boolean Flags and Hidden Branching

- **DON'T:** Pass a boolean flag into a function purely to make it silently take one of two different code paths internally. A flag parameter is a strong signal that the function is actually two functions wearing one name, and the call site (`sendEmail(user, true)`) gives the reader no idea what `true` means without checking the definition.
```
// Bad: an opaque flag silently switches behavior
function renderReport(data, asPdf) {
    if (asPdf) { return renderPdf(data); }
    return renderHtml(data);
}
renderReport(data, true); // true... meaning what, exactly?

// Good: split into two clearly named functions
function renderReportAsPdf(data) { return renderPdf(data); }
function renderReportAsHtml(data) { return renderHtml(data); }
```

- **DO:** Split a function with a behavior-switching flag into two separate, clearly named functions, or replace multiple related flags with a single enumerated option value when the branches genuinely share most of their logic. Either approach makes the call site self-documenting.

- **DON'T:** Stack multiple boolean parameters on the same function signature (`createWidget(visible, enabled, bordered)`); at the call site, `createWidget(true, false, true)` is unreadable and easy to get wrong, and the combinatorial explosion of flag states is rarely all valid anyway.

### Command-Query Separation

- **DO:** Keep functions that return information (queries) separate from functions that change state (commands). A function should either answer a question or cause an effect, not both — mixing them means a caller can't glance at a call site and know whether it's safe to call repeatedly or safe to ignore the return value.
```
// Bad: the query mutates state as a side effect
function getNextId() {
    return currentId++; // looks like a read, is actually a write
}

// Good: separate the read from the write, or name it to reflect the mutation
function peekNextId() { return currentId; }
function incrementAndGetNextId() { return ++currentId; }
```

- **DON'T:** Give a command (a state-changing operation) a return value that callers are expected to rely on for meaningful data, beyond a simple success/failure or the entity just acted upon — if callers need to query afterward, that's a hint the command and the query should stay separate.

- **DO:** Name command functions with imperative verbs (`save`, `send`, `delete`) and query functions with descriptive nouns or `get`/`is`/`has` forms, so the grammatical shape of the name itself signals which category a function falls into.

### Temporal Coupling

- **DON'T:** Design an API where calling functions in the wrong order silently produces incorrect results instead of an error. If `initialize()` must be called before `process()`, and calling `process()` first just corrupts data instead of failing loudly, the dependency between them is invisible until it breaks in production.
```
// Bad: works only if called in a specific, undocumented order
report.setData(data);
report.calculate();
report.render(); // renders garbage if calculate() was skipped

// Good: the object enforces or eliminates the ordering requirement
const report = Report.fromData(data); // calculation happens internally
report.render();
```

- **DO:** Make required call ordering explicit in the API shape itself — through constructors that require the needed data up front, through return values that only the next legitimate call can produce, or through a state machine that rejects out-of-order calls with a clear error — rather than relying on documentation or convention alone.

- **DON'T:** Split a single logical operation into multiple calls that must happen consecutively with no other calls in between, if the sequence can instead be expressed as one function or one atomic unit of work. Every call boundary you introduce is a place where another developer can (and eventually will) insert something that breaks the hidden assumption.

### Consistent Return Behavior

- **DO:** Keep a function's return type and return semantics consistent across every one of its exit paths. A function that returns an entity in one branch, `null` in another, and throws in a third forces every caller to defensively handle three different response shapes instead of one predictable contract.
```
// Bad: three different, inconsistent ways of signaling "no result"
function findUser(id) {
    if (!id) return undefined;
    if (id < 0) throw new Error("bad id");
    const user = db.lookup(id);
    return user || false;
}

// Good: one consistent shape for the "not found" case
function findUser(id) {
    if (!id || id < 0) throw new Error(`Invalid user id: ${id}`);
    return db.lookup(id); // returns the user or null — always one of these two
}
```

- **DON'T:** Return a special sentinel value (`-1`, an empty string, `null`) to mean "failure" or "not applicable" when that sentinel is also a legitimate, ordinary value the function could otherwise return. A caller can't distinguish "the count is genuinely zero" from "the count could not be computed" if both are represented as `0`.

- **DO:** Make it obvious from a function's name, type signature, or documentation whether it can return an absent/empty result as part of normal operation, so callers know up front whether they need to handle that case instead of discovering it the first time it happens in production.

### Functions as a Unit of Testability

- **DO:** Write functions with their eventual testability in mind — clear inputs, a clear output or effect, and as few hidden dependencies as possible. A function that's awkward to test in isolation (it needs a database, a live network call, or a fully constructed application context just to exercise one branch of logic) is usually a function that's doing too much or depending on too much.

- **DON'T:** Treat "this is hard to unit test" as a problem to route around with heavier test infrastructure before first asking whether the function's design is what's actually making it hard to test. Often, separating the pure decision logic from the I/O it triggers resolves the testability problem and improves the function's design at the same time.

### Handling the Whole Input Domain

- **DO:** Design a function to explicitly account for every input it can actually receive, not just the inputs exercised by the cases that happened to come to mind first. If a function can be called with an empty list, a negative number, or a missing optional field, decide deliberately what it does in each case rather than leaving that behavior to whatever the implementation happens to do by accident.
```
// Bad: only the "normal" case was considered; edge cases fall through
// to whatever the underlying operations happen to do
function averageOf(numbers) {
    return numbers.reduce((a, b) => a + b) / numbers.length; // throws on empty, unclear on non-numeric
}

// Good: the edge cases are explicit, deliberate decisions
function averageOf(numbers) {
    if (numbers.length === 0) {
        throw new Error("Cannot average an empty list");
    }
    return numbers.reduce((a, b) => a + b, 0) / numbers.length;
}
```

- **DON'T:** Leave a function's behavior on boundary and edge-case inputs (empty collections, zero, negative numbers, null/absent values, maximum-size inputs) as an accident of whatever the underlying operations happen to produce. An undefined edge case discovered by a caller in production is a bug even if the "normal" path was always correct.

- **DO:** Explicitly reject inputs outside a function's intended domain rather than silently accepting and mishandling them. A function that's only meant to operate on positive numbers should say so and fail clearly on a negative one, rather than quietly returning a nonsensical result that the caller has to notice is wrong on their own.

### Consistent Parameter Ordering

- **DO:** Keep parameter order consistent across a family of related functions — if one function takes `(id, data)`, a sibling function operating on the same kind of entity should also take `(id, data)`, not `(data, id)`. Predictable ordering lets a caller who has learned one function in the family correctly guess the signature of the others without checking.

- **DON'T:** Vary the position of a conceptually similar parameter (a callback, an options object, an identifier) from one function to the next within the same codebase or library. Inconsistent ordering is a common source of subtle bugs when a caller, working from memory or from a similar nearby function, passes arguments in the wrong position.

### Default Values and Optional Parameters

- **DO:** Reserve default parameter values for genuinely optional settings where the default is the overwhelmingly common, safe choice for almost every caller. A default that's wrong for a meaningful fraction of call sites just means those call sites silently get incorrect behavior from every caller who didn't know they needed to override it.
```
// Risky: a default that's silently wrong for a common, security-sensitive case
function createSession(user, secure = false) { ... }
createSession(user); // silently insecure, and it's not obvious from the call site

// Safer: force a conscious choice for anything consequential
function createSession(user, secure) { ... }
createSession(user, true); // explicit at the call site
```

- **DON'T:** Give a default value to a parameter whose "wrong" setting is dangerous, expensive, or hard to detect after the fact (e.g., an unencrypted connection, a bypass of validation, a much slower code path). For consequential choices, forcing every caller to decide explicitly is worth the small extra verbosity, because it guarantees the choice was actually made rather than silently defaulted.

- **DO:** Make sure a parameter's default value is visible at the call site or easily discoverable, rather than requiring a reader to open the function's definition to know what will happen when an optional argument is omitted. Named/keyword arguments, where the language supports them, help make omitted values and their defaults easier to reason about directly from the call.

## Comments & Documentation

A comment is a place where the code has failed to explain itself, with one important exception: explaining *why* a decision was made, which the code can never express on its own no matter how well it's written. Judge every comment by whether it earns its keep against that standard.

### Why, Not What

- **DO:** Reserve comments for the reasoning the code cannot express on its own — why a non-obvious approach was chosen, why an apparently better alternative was rejected, what external constraint (a bug in a dependency, a regulatory requirement, a business rule, a performance measurement) forced this particular shape.
```
// Bad: restates what the next line already says
// increment i by one
i = i + 1;

// Good: explains a non-obvious reason
// Retry once before failing: the upstream API returns spurious 502s
// under load roughly 1% of the time (see incident-4231).
retryOnce(callUpstreamApi);
```

- **DON'T:** Write a comment that merely restates the code in prose. A comment that says what the very next line already says in code adds a second thing to keep in sync and zero information; when the code changes and the comment doesn't, it becomes actively misleading.

- **DO:** Treat the need for a "what does this do" comment as a signal to improve the code instead — extract a well-named function, rename a variable, or restructure the logic — so the code communicates its own purpose and the comment becomes unnecessary.

- **DO:** Use comments to flag genuinely non-obvious edge cases, workarounds for external bugs, or subtle invariants that a future editor could easily break without realizing it (e.g., "this must run before X initializes, see ticket #123").

### Dead Code and Stale Comments

- **DON'T:** Leave commented-out code in the codebase. Version control already remembers every previous version of every line; a block of commented-out code just adds visual noise and leaves every reader wondering whether it's safe to delete or secretly still needed.
```
// Bad: dead code left "just in case"
// function oldCalculateTotal(items) {
//     return items.reduce((a, b) => a + b.price, 0);
// }
function calculateTotal(items) {
    return items.reduce((a, b) => a + b.price * b.quantity, 0);
}

// Good: just delete it — the old version is in the version-control history
function calculateTotal(items) {
    return items.reduce((a, b) => a + b.price * b.quantity, 0);
}
```

- **DON'T:** Write a comment that duplicates information likely to drift, such as restating a parameter's type, a default value, or a piece of logic in words, when that information is already expressed in the code itself and will be trivially outdated the next time the code changes without the comment being touched.

- **DO:** Delete or update a comment in the same change that invalidates it. A stale comment is worse than no comment, because it actively misleads a reader who reasonably assumes a comment reflects current behavior; treat an outdated comment as a bug, not a cleanup nicety.

- **DON'T:** Leave a `TODO` or `FIXME` comment without enough context to act on later (no ticket reference, no explanation of what's missing or why it wasn't done now). An untraceable TODO tends to fossilize in the codebase for years, since no one but the original author knows whether it's still relevant, safe to ignore, or urgent.

### When a Comment Is the Wrong Fix

- **DON'T:** Reach for a comment to explain confusing code as a substitute for making the code itself less confusing. If a function needs a paragraph-long comment to explain what it does, that's usually a sign the function should be broken into smaller, well-named pieces, not that it needs more prose wrapped around it.

- **DO:** Ask, before writing an explanatory comment, whether a rename, an extracted function, or a restructured conditional would make the comment unnecessary. Reserve the comment itself for cases where no amount of renaming or restructuring can convey the "why."

- **DON'T:** Use a comment to apologize for or excuse bad code ("sorry, this is a hack, will fix later") without either fixing it now, filing a tracked follow-up, or explaining the real constraint that forced the shortcut. An apology comment with no actionable trail just documents guilt, not information.

### Commit Messages, PR Descriptions, and Docs as Documentation

- **DO:** Treat a commit message as documentation for future readers of the history, not as a note to yourself in the moment. A good commit message explains what changed and, more importantly, why — the motivation, the problem being solved, or the tradeoff being made — information that's often invisible from the diff alone.
```
// Bad commit message
fix bug

// Good commit message
Fix race condition in session refresh

Two concurrent requests could both see an expired token and both
trigger a refresh, corrupting the token store. Serialize refreshes
with a per-user lock.
```

- **DON'T:** Write a commit message or PR description that only restates the diff in prose ("changed function X", "updated file Y") without saying why the change was made or what problem it solves. The diff already shows *what* changed; the message's whole job is to add what the diff can't show.

- **DO:** Write PR descriptions that give a reviewer enough context to evaluate the change without reconstructing it from scratch: the problem being solved, the approach taken and why (especially if alternatives were considered and rejected), how it was tested, and any follow-up work intentionally left out of scope.

- **DO:** Keep a README and any public API documentation in sync with the behavior it describes, and treat a documentation update as part of the definition of "done" for a change that alters user-facing or public-API behavior — documentation that describes a previous version of the software is often worse than no documentation, because it actively misleads.

- **DON'T:** Let API documentation describe the intended or aspirational behavior instead of the actual, current behavior. If a documented parameter, endpoint, or return value doesn't match reality, every consumer who trusts the docs will build on a false premise and discover the mismatch the hard way.

### Self-Documenting Code Through Structure

- **DO:** Let types, structured return values, and well-named intermediate variables carry information that a comment would otherwise have to state in prose. A well-named, well-typed structure documents its own shape permanently and gets checked by tooling; a comment describing the same shape has to be trusted and never gets checked by anything.
```
// Relies on a comment to describe the shape of what's returned:
// returns { status: "ok" | "error", data: array of items or null on error }
function fetchItems() { ... }

// The structure documents itself, and mismatches are caught automatically:
type FetchResult = { status: "ok"; data: Item[] } | { status: "error"; message: string };
function fetchItems(): FetchResult { ... }
```

- **DO:** Break a long, unnamed expression into intermediate variables with descriptive names as a lighter-weight alternative to a comment explaining what the expression computes. A well-named intermediate value explains itself at the point of use and can't drift out of sync with the logic the way a comment can.

### Documentation for Public APIs

- **DO:** Document the public surface of anything meant to be used by other people or other teams — its parameters, its return behavior, its failure modes, and any preconditions or invariants a caller must uphold — at a level of detail proportional to how far removed the consumer is from the author. A teammate sitting next to you needs less written documentation than a public package used by strangers.

- **DO:** Include realistic usage examples in documentation for non-trivial public APIs. An example that shows a typical call and its typical result answers most of the questions a new consumer actually has, faster than a parameter-by-parameter description would.

- **DON'T:** Let internal implementation comments leak into public-facing documentation, or vice versa. Documentation aimed at a consumer of an API should describe the contract (what it does, what it promises); documentation aimed at a future maintainer of the implementation should describe the internals (how and why it currently works that way) — conflating the two audiences confuses both.

### Recording Significant Design Decisions

- **DO:** Record the reasoning behind a significant, non-obvious architectural or design decision somewhere durable and discoverable — a design document, a decision record, a well-placed comment at the relevant point in the code — at the time the decision is made, while the tradeoffs and rejected alternatives are still fresh. Months later, nobody but the original decision-maker reliably remembers why the road not taken was rejected, and that reasoning is exactly what's needed when someone later wonders whether it's safe to revisit the decision.

- **DON'T:** Let a significant design decision live only in a chat message, a meeting nobody wrote down, or a single person's memory. Decisions recorded nowhere durable might as well not have been explained at all, from the point of view of anyone who joins the project later or wasn't in the room.

### Keeping Documentation Examples Honest

- **DO:** Prefer documentation examples that are actually executed as part of the test suite over examples that are just prose someone typed and hoped stayed accurate. An example that runs and is checked against real output cannot silently drift out of sync with the code the way a hand-written, unverified example inevitably will as the underlying behavior evolves.

- **DON'T:** Let a documentation example depend on a code path significantly different from what a real caller would actually exercise. A contrived example that doesn't reflect realistic usage teaches a pattern nobody should actually follow, and defeats the purpose of showing an example in the first place.

## Error Handling Philosophy

How a system fails is as much a design decision as how it succeeds. Sloppy error handling doesn't just produce bad error messages — it produces systems that corrupt data quietly, that are impossible to debug in production, and that erode trust because nobody believes the "success" path actually succeeded.

### Fail Fast vs. Graceful Degradation

- **DO:** Fail fast and loudly for programming errors and violated invariants — situations where continuing execution would be operating on data or state the code was never designed to handle. A loud, immediate failure at the point something goes wrong is far cheaper to debug than a corrupted result discovered three layers downstream.
```
// Bad: silently continues with an invalid value, corrupting downstream results
function calculateDiscount(percent) {
    if (percent < 0 || percent > 100) percent = 0; // silently "fixed"
    return price * (1 - percent / 100);
}

// Good: fails immediately at the point of the actual problem
function calculateDiscount(percent) {
    if (percent < 0 || percent > 100) {
        throw new Error(`Invalid discount percent: ${percent}`);
    }
    return price * (1 - percent / 100);
}
```

- **DO:** Reserve graceful degradation for situations where a partial or fallback result is genuinely acceptable to the system's users and is explicitly designed as such — not as a way to avoid dealing with an error. A search feature that falls back to a cached result when the live index is down is a deliberate design choice; a database write that "degrades" by silently not writing is data loss wearing a nicer name.

- **DON'T:** Choose graceful degradation by default just because it avoids an unhandled crash in the moment. Ask explicitly, for each failure point, whether continuing with a degraded or default outcome is actually safe for that specific operation, or whether it's quietly hiding a problem the caller needed to know about.

### Never Silently Swallow Errors

- **DON'T:** Catch an exception or check an error return value and then do nothing with it (an empty catch block, an ignored error return). A swallowed error is a bug reported to nobody — the failure still happened, but now there's no trace of it when something downstream breaks in a confusing way.
```
// Bad: the error vanishes without a trace
try {
    saveToDatabase(record);
} catch (e) {
    // ignore, probably fine
}

// Good: at minimum, log with context; ideally, handle or propagate
try {
    saveToDatabase(record);
} catch (e) {
    logger.error("Failed to save record", { recordId: record.id, error: e });
    throw e;
}
```

- **DON'T:** Catch an error only to convert it into a generic, contextless fallback value (returning `null`, `false`, or `0` on any failure) unless that fallback is a deliberate, documented part of the function's contract. A caller receiving `null` can't tell whether that means "not found" or "the operation failed for an unrelated reason," and will handle the two very differently if it knew.

- **DO:** When you catch an error you don't fully handle, either re-throw it, wrap it with additional context and re-throw, or explicitly and visibly log it at a severity that will actually be noticed. Catching should always mean "I am doing something meaningful with this," never "I am making this go away."

- **DO:** Make it a deliberate, reviewed decision whenever an error truly should be ignored (e.g., a best-effort cleanup operation that shouldn't block the main flow), and say so explicitly in the code — a one-line comment stating why it's safe to ignore this specific error turns an invisible judgment call into a visible, reviewable one.

### Recoverable vs. Unrecoverable Errors

- **DO:** Distinguish, in how errors are modeled and handled, between conditions the caller can reasonably recover from (a validation failure, a not-found lookup, a network timeout worth retrying) and conditions that indicate a bug or an unrecoverable environment problem (a violated internal invariant, an out-of-memory condition, corrupted configuration). They deserve different handling strategies — retry, fallback, or user feedback for the former; fail loudly and stop for the latter.

- **DON'T:** Use the same generic error type or the same catch-all handling strategy for both categories. Treating a validation error and a critical system fault identically means callers either overreact to routine, expected failures or, more dangerously, silently swallow serious ones because the handling code was written for the routine case.

- **DO:** Make recoverable, expected failure conditions part of a function's explicit contract (a specific error type, a result type, a documented set of failure modes) rather than relying on callers to guess which exceptions might occur or to wrap every call in a broad catch just in case.

### Giving Errors Enough Context to Debug

- **DON'T:** Raise or log an error with a generic, contextless message like "something went wrong" or "operation failed." By the time an error is visible to a developer, the original context (which input, which user, which record, which step) is often already gone from the call stack or the process; the error message is frequently the only surviving evidence.
```
// Bad: no way to know what failed or why
throw new Error("failed");

// Good: enough context to act on without reproducing the bug
throw new Error(
    `Failed to charge payment for order ${order.id}: ` +
    `gateway returned ${response.status} (${response.body})`
);
```

- **DO:** Include the relevant identifiers, inputs, and state in every error message and log entry — what operation was being attempted, on what entity, with what key parameters — so that whoever reads the error later can act on it without having to reproduce the failure from scratch.

- **DO:** Preserve the original underlying error (its message, type, and stack trace) when wrapping it in a higher-level error for additional context, rather than replacing it outright. The original error is frequently the only clue to the actual root cause; a wrapper that discards it trades a small amount of noise for a permanent loss of information.

- **DON'T:** Log the same error at multiple layers of the call stack as it propagates upward, each time as if it were new. Duplicate logging for a single failure clutters logs, makes it hard to tell how many distinct failures actually occurred, and often loses the original context by the time it reaches the top.

### Avoiding Exceptions (or Error Paths) as Control Flow

- **DON'T:** Use exceptions, or an equivalent error-signaling mechanism, to implement expected, routine branching logic (e.g., throwing to break out of a loop, or using a "not found" exception for what is really just a normal, expected outcome of a lookup). Exception-based control flow is typically slower, obscures the normal logical structure of the code, and blurs the line between "this is exceptional" and "this is Tuesday."
```
// Bad: exceptions used for an expected, routine outcome
function getUser(id) {
    try {
        return db.findUserOrThrow(id);
    } catch (NotFoundError) {
        return null; // "not found" is a completely normal outcome here
    }
}

// Good: model the expected outcome directly, no exception involved
function getUser(id) {
    return db.findUser(id); // returns null/None if not found — a normal case
}
```

- **DO:** Reserve exceptions (or your language's equivalent mechanism) for genuinely exceptional conditions — situations the immediate caller isn't expected to handle as part of normal logic — and model expected alternate outcomes (not found, validation failed, empty result) as ordinary return values, optional types, or result types instead.

- **DO:** Match the error-signaling mechanism to how likely and how routine the failure is. A condition that happens on a meaningful fraction of calls (e.g., "user input was invalid") is a normal outcome that belongs in the type system or return value; a condition that should essentially never happen (e.g., "the database connection pool object is null") is a fair candidate for an exception.

### User-Facing vs. Internal Error Detail

- **DO:** Separate the detailed, technical error information meant for logs and developers from the simplified, safe message meant for end users. A user needs to know their payment didn't go through and what to do next; they don't need — and shouldn't see — a raw stack trace, an internal file path, or a database error string.
```
// Bad: leaks internal detail straight to the end user
catch (e) {
    showToUser(`Error: ${e.stack}`); // exposes internals, means nothing to a user
}

// Good: log the detail internally, show something actionable externally
catch (e) {
    logger.error("Payment charge failed", { orderId: order.id, error: e });
    showToUser("We couldn't process your payment. Please try again.");
}
```

- **DON'T:** Expose internal implementation details — stack traces, raw exception messages, internal identifiers, database schema details — in errors shown to end users or returned across a public API boundary. Beyond being confusing and unhelpful to the person seeing them, this can leak information useful to an attacker probing the system.

- **DO:** Give end-user-facing error messages enough specificity to be actionable ("your session expired, please log in again") rather than a generic, unhelpful catch-all ("an error occurred"), while still keeping the deep technical detail in the logs where developers can find it.

### Logging Discipline Around Errors

- **DO:** Choose a log severity that reflects actual operational significance — routine, expected conditions logged at a low severity; genuine problems that need attention logged at a level that will actually trigger notice. Logging everything at the same high severity trains everyone to ignore that severity level entirely, which defeats its purpose the one time it matters.

- **DON'T:** Log sensitive data (passwords, tokens, full payment details, other regulated personal information) as part of error context, even when it would make debugging more convenient. Logs are often retained, searched, and accessed more broadly than the systems they describe, and logged secrets have a way of leaking far past their original purpose.

### Validating Input at the Boundary

- **DO:** Validate untrusted or external input as early as possible — at the point it enters the system (a request handler, a file parser, a public function's entry point) — so that everything past that boundary can safely assume the data is well-formed. Pushing validation deep into the system means every internal function has to defensively re-check assumptions that should have already been guaranteed.
```
// Bad: invalid data can travel deep into the system before anything notices
function handleSignup(request) {
    createAccount(request.body); // no validation — bad data flows straight through
}

// Good: validated once, at the boundary; everything downstream can trust it
function handleSignup(request) {
    const data = validateSignupInput(request.body); // throws or returns a typed, valid result
    createAccount(data);
}
```

- **DON'T:** Trust that data is valid just because it came from "inside" the system (a database, an internal service, a previous processing step) without a boundary check at some point having actually verified it. Data that was valid when it was written can become stale or inconsistent by the time it's read, and internal boundaries between services or components deserve the same skepticism as external ones when they're not under the same direct control.

### Retry Behavior

- **DO:** Bound the number of retry attempts and back off between them (rather than retrying instantly and repeatedly) for any operation that might legitimately be retried after a transient failure. An unbounded retry loop against a struggling downstream system doesn't just fail to help — it actively worsens the problem by adding more load to something that's already failing.

- **DON'T:** Retry an operation that isn't idempotent as though retrying is always safe. Blindly retrying a non-idempotent operation on a failure whose actual outcome is unknown (did the write actually happen before the connection dropped?) risks duplicating the effect; know which operations are safe to retry and which need a different recovery strategy.

### Partial Failure in Bulk Operations

- **DO:** Decide explicitly, for any operation acting on a batch or collection of items, what happens when some items succeed and others fail — abort everything, continue and report per-item results, or roll back everything already done. Leaving this undecided means the actual behavior is whatever the implementation happens to do, which is rarely the behavior a caller actually wants or expects.
```
// Bad: unclear what happened on partial failure — some emails sent, some not,
// caller has no way to know which
function sendBulkEmails(recipients) {
    for (const r of recipients) { emailService.send(r); } // stops silently on first failure
}

// Good: explicit per-item outcome, caller can act on exactly what happened
function sendBulkEmails(recipients) {
    return recipients.map(r => {
        try { emailService.send(r); return { recipient: r, status: "sent" }; }
        catch (e) { return { recipient: r, status: "failed", error: e.message }; }
    });
}
```

- **DON'T:** Report a bulk operation as a single success or failure when it actually processed multiple independent items with independent outcomes. A caller who receives only "3 of 10 failed, operation returned an error" cannot tell which 3, and cannot safely retry without risking re-processing the 7 that already succeeded.

### Timeouts and Bounded Waiting

- **DO:** Put an explicit, reasonable timeout on any operation that waits on something outside the current process's direct control — a network call, a lock, a queue, another service's response. Without a bound, a single slow or hung dependency can stall the calling code indefinitely, and that stall frequently cascades into a much larger outage as callers waiting on the caller pile up behind it.
```
// Bad: no bound on how long this can hang, and hangs cascade upstream
const response = await httpClient.get(url);

// Good: fails predictably instead of hanging forever
const response = await httpClient.get(url, { timeoutMs: 5000 });
```

- **DON'T:** Choose a timeout value arbitrarily without considering what happens to the caller, and the caller's caller, when it's hit. A timeout that's too short causes spurious failures on legitimately slow-but-successful operations; a timeout that's too long defeats the purpose of having one at all — both deserve a deliberate choice grounded in the operation's real, observed latency.

### Avoiding Overly Broad Exception Handling

- **DON'T:** Catch the broadest possible error category (a base exception type, a bare `except`, a catch-all handler) around a large block of code when only a specific, narrow failure is actually anticipated. A broad catch silently absorbs unrelated bugs — a typo causing a name error, a null reference somewhere unexpected — alongside the one failure mode it was actually written to handle, and treats all of them identically.
```
// Bad: a broad catch hides bugs unrelated to the one thing being handled
try {
    const price = parsePrice(input);
    const total = price * quantity * getDiscountMultiplier(); // a real bug here
    return total;
} catch (e) {
    return 0; // masks the parse failure AND any unrelated bug in the same block
}

// Good: narrow the try block and the caught type to what's actually expected
let price;
try {
    price = parsePrice(input);
} catch (e) {
    if (e instanceof ParseError) return 0;
    throw e; // anything else is unexpected — let it surface
}
return price * quantity * getDiscountMultiplier();
```

- **DO:** Keep the code inside a `try` block (or equivalent) as narrow as possible — just the operation that can actually raise the specific error being handled — so that a broad catch can't accidentally absorb an unrelated failure from a neighboring line that happens to sit inside the same block.

## Abstraction & Complexity

Abstraction is a tool for managing complexity, not a virtue to pursue for its own sake. The right amount of abstraction makes a codebase easier to change; too little duplicates decisions all over the place, and too much buries simple logic under indirection that has to be unwound before anyone can understand what's actually happening.

### YAGNI and Premature Generalization

- **DON'T:** Build a generic, configurable, or pluggable mechanism for a need you anticipate but don't yet have — an extra layer of configuration options, a plugin system, a generalized data model — before a concrete second use case actually exists. Speculative generality is guessed at, and guesses about future requirements are wrong often enough that the abstraction usually has to be reworked anyway once real requirements show up, at which point it's often harder to change than if it had never been built.
```
// Bad: generalized for hypothetical future formats nobody has asked for
function exportData(data, format, options = {}) {
    if (format === "csv") { return exportCsv(data, options); }
    // dozens of unused option flags "for future formats"...
}

// Good: build exactly what's needed now
function exportAsCsv(data) { ... }
// Generalize later, once a second real format is actually required.
```

- **DO:** Build the simplest thing that satisfies the actual, current requirement, and let real, observed needs — not hypothetical ones — drive when and how you generalize. It is almost always cheaper to add an abstraction later, once you know its real shape, than to maintain and eventually unwind a wrong guess.

- **DON'T:** Justify extra complexity with "we might need this later" as the sole reason. If "later" arrives, you'll have far better information about the actual shape of the need than you do now; building for an imagined future need usually means building the wrong abstraction for the real one when it eventually shows up.

### The Rule of Three

- **DO:** Tolerate duplication the first time a piece of logic is needed in a second place, and treat the appearance of a third, genuinely similar occurrence as the trigger to extract a shared abstraction. Two occurrences often aren't enough to reveal which parts of the duplicated logic are coincidentally similar versus fundamentally the same thing; a third data point makes the real shape of the abstraction much clearer.

- **DON'T:** Abstract away duplication the moment it appears a second time, especially when the two occurrences might just coincidentally look alike right now but are conceptually unrelated. Merging them prematurely creates a false coupling that has to be un-merged later when their behaviors diverge for legitimate, unrelated reasons.

- **DO:** When you do extract a shared abstraction after seeing the pattern repeat, name it after the actual shared concept the repetitions have in common — not just "the code that used to be copy-pasted in three places" — so the abstraction reads as a deliberate design decision rather than an accident of deduplication.

### Avoiding Unnecessary Indirection

- **DON'T:** Introduce a layer of indirection (an interface with exactly one implementation and no planned second one, a factory for something that's never actually varied, a configuration system for values that never change) purely on the theoretical principle that it might someday be useful. Every layer adds a hop a reader has to follow to understand what actually happens, and a layer that exists "just in case" charges that cost to every single reader without ever paying it back.
```
// Bad: an interface and factory for something with exactly one implementation
interface PaymentProcessor { charge(amount); }
class StripePaymentProcessor implements PaymentProcessor { charge(amount) { ... } }
class PaymentProcessorFactory { create() { return new StripePaymentProcessor(); } }
// ...three files and two indirections to call one method.

// Good: call it directly until a second real implementation exists
class StripePaymentProcessor { charge(amount) { ... } }
```

- **DO:** Introduce an interface, an abstraction boundary, or a dependency-injection seam when there is a real, current reason for it — multiple genuine implementations, a hard requirement to swap implementations (such as for testing against a real external dependency), or a well-understood, likely-to-change boundary — not preemptively.

- **DON'T:** Split logic across many small layers or micro-modules where the pattern buys no real flexibility and only forces the reader to jump between many files to trace a single operation end to end. Depth for its own sake ("architecture astronaut" layering) is complexity with no corresponding benefit.

- **DO:** Judge the right number of layers by how easy the system is to change and to trace, not by how closely it matches an idealized architecture diagram. A flatter design that's easy to follow beats a deeply layered one that technically demonstrates more design patterns but takes ten minutes to trace a single request through.

### Duplication vs. the Wrong Abstraction

- **DO:** Recognize that a small amount of duplicated code is frequently cheaper, in total cost over time, than a shared abstraction that doesn't actually fit all its call sites. Duplicated code is easy to read locally and easy to change independently; a bad abstraction has to be understood globally and, when one caller's needs diverge, has to be pried apart under time pressure.
```
// A shared abstraction stretched to cover a case it doesn't really fit,
// with a flag threading a special case through unrelated callers:
function saveEntity(entity, opts = {}) {
    if (opts.skipValidationForLegacyImport) { ... } // now every caller
    // must reason about this flag, even though only one caller needs it
}

// Sometimes two small, independent, slightly duplicated functions
// are the actually simpler and safer choice:
function saveEntity(entity) { validate(entity); persist(entity); }
function importLegacyEntity(entity) { persist(entity); } // no shared flag needed
```

- **DON'T:** Force a new, slightly different use case into an existing abstraction by adding conditional branches, flags, or special-case parameters to it. An abstraction that has grown several "except when" branches to accommodate cases it wasn't designed for has usually become harder to understand than the duplication it was meant to replace.

- **DO:** When an existing abstraction stops fitting a new case cleanly, treat that as a legitimate trigger to split it back apart, redesign it, or let the new case duplicate a portion of the logic independently — reverting a bad abstraction is a normal, healthy outcome, not a failure.

### Cyclomatic Complexity and Nested Conditionals

- **DO:** Stay aware of how many independent paths a function has (roughly, how many `if`/`else`/`case`/loop branches it contains) and treat a high branch count as a signal to simplify — split the function, extract conditions into well-named predicates, or restructure the logic — rather than as an inevitable cost of "just how this function has to be."

- **DON'T:** Nest conditionals many levels deep. Each additional level of nesting multiplies the number of paths a reader has to hold in their head simultaneously to understand any single line inside the innermost block.
```
// Bad: deep nesting makes the actual logic hard to follow
function getDiscount(user) {
    if (user) {
        if (user.isActive) {
            if (user.orders.length > 0) {
                if (user.totalSpent > 1000) {
                    return 0.2;
                }
            }
        }
    }
    return 0;
}

// Good: guard clauses flatten the structure to the essential path
function getDiscount(user) {
    if (!user || !user.isActive) return 0;
    if (user.orders.length === 0) return 0;
    if (user.totalSpent <= 1000) return 0;
    return 0.2;
}
```

- **DO:** Use guard clauses and early returns to handle edge cases and exit conditions at the top of a function, so the main body of the function only has to express the primary, successful path without being wrapped in nested conditionals.

- **DO:** Extract a complex boolean condition into a well-named predicate function or variable instead of inlining it, especially when the condition combines several checks with `&&`/`||`. A name like `isEligibleForDiscount` communicates intent instantly; the raw boolean expression it replaces usually doesn't.

### Premature Optimization

- **DON'T:** Sacrifice clarity for performance before you have actual evidence — a measurement, a profile, a realistic load test — that the code in question is a meaningful bottleneck. An unmeasured guess about what's "probably slow" is frequently wrong, and the clever, hard-to-read version optimized against a guess often isn't even faster in practice.
```
// Bad: sacrifices readability for a "speedup" nobody measured or needed
const r = a.reduce((p,c,i)=>i%2?p:[...p,c*2],[]);

// Good: clear first; optimize later, with evidence, if this is ever a hot path
const doubledEvenIndexedItems = items
    .filter((_, index) => index % 2 === 0)
    .map(item => item * 2);
```

- **DO:** Write the clear, straightforward version of the code first, and treat targeted optimization as a deliberate follow-up step justified by a measurement, applied only to the specific part of the code that the measurement identified as the actual bottleneck — not applied speculatively across the whole codebase.

- **DON'T:** Assume that a general best practice discussed for high-throughput or latency-critical systems automatically applies to a rarely-called, non-critical piece of code. Optimization has a real cost in the form of reduced readability and increased risk of bugs; that cost should only be paid where the performance actually matters.

### Complexity Budget

- **DO:** Treat overall system complexity as a finite, shared budget rather than an unlimited resource each individual decision can draw from for free. A single clever solution might be locally justified, but a codebase where every module independently claims its own justified bit of extra complexity ends up, in aggregate, far harder to work in than any single decision would suggest.

- **DON'T:** Solve a simple problem with a sophisticated general-purpose mechanism (a rules engine, a plugin architecture, a full state machine framework) when a straightforward conditional or a small, direct function would do the job just as well today. Match the sophistication of the solution to the actual complexity of the problem, not to what would be more interesting or impressive to build.

- **DO:** When choosing between two designs that both solve the problem correctly, prefer the one a new team member could understand fastest, all else being roughly equal. Simplicity that speeds up everyone's future understanding is a real, measurable engineering benefit, not just an aesthetic preference.

### Cleverness vs. Clarity

- **DON'T:** Write a dense, clever one-liner that exploits an obscure language feature or a chain of operations purely to show mastery of the language, when a slightly longer, more obvious version would communicate the same logic more clearly. Code is a communication medium first and a performance of skill a distant second; a reader forced to puzzle out a clever trick before understanding what a piece of code does has been actively slowed down by it.

- **DO:** Optimize for the reader who is tired, in a hurry, or unfamiliar with the specific trick being used. If a piece of code requires the reader to already know an obscure idiom to parse it at a glance, that's a cost paid by every future reader who doesn't already know it, for a benefit (fewer characters, a marginally shorter function) that rarely matters as much as the cost.

- **DON'T:** Equate "concise" with "good." A shorter piece of code that takes longer to understand is not actually simpler — the real measure of simplicity is how quickly and correctly a reader can build an accurate mental model of what the code does, not how few characters or lines it took to write.

### Configuration and Option Sprawl

- **DON'T:** Let a component's configuration surface grow indefinitely by adding a new flag or option every time a new use case arrives, instead of asking whether the use case is better served by a different call, a different component, or an explicit rejection. Every configuration option is a hidden branch that multiplies the number of behavioral combinations the component can be in, most of which are never actually tested or even used.

- **DO:** Periodically audit a heavily configurable component for options that are effectively always set to the same value everywhere they're used, or that were added for a use case that no longer exists, and remove them. An option nobody varies isn't flexibility — it's just unexercised complexity sitting in the code and in every reader's way.

- **DON'T:** Treat "make it configurable" as a free way to avoid a design decision. A configuration flag doesn't eliminate the decision of what the right default behavior is; it just defers making that decision explicitly, and pushes the cost of understanding all the possible combinations onto every future reader and maintainer instead.

## State & Side Effects

Most hard-to-reproduce bugs trace back to state that changed somewhere the reader wasn't looking. Minimizing where state lives, who can change it, and how far its effects reach is one of the highest-leverage things a codebase can do for its own long-term debuggability.

### Minimizing Mutable Shared State

- **DO:** Limit how many parts of the codebase can read and write the same mutable state. The more places that can modify a given piece of state, the harder it becomes to answer "why does this value currently look the way it does," because the answer could be any one of many call sites, possibly running concurrently.

- **DON'T:** Share a single mutable object between multiple components or modules as a convenient way to pass information around, when a value could instead be passed explicitly through function arguments and return values. Shared mutable objects create implicit coupling between components that have no obvious relationship in the code itself.
```
// Bad: components communicate through a shared mutable object
const sharedState = { total: 0, discount: 0 };
function applyDiscount() { sharedState.total -= sharedState.discount; }
function addItem(price) { sharedState.total += price; } // order-dependent, fragile

// Good: state flows explicitly through calls, not through a shared object
function calculateTotal(items, discount) {
    return items.reduce((sum, i) => sum + i.price, 0) - discount;
}
```

- **DO:** Scope mutable state as narrowly as possible — local to a function or a single, small module — and only widen its scope when there's a clear, deliberate reason multiple parts of the system need to observe or change it.

### Avoiding Action at a Distance

- **DON'T:** Write code where calling one function unexpectedly changes behavior somewhere else entirely unrelated in the codebase, through a shared reference, a global flag, or an event with far-reaching and non-obvious listeners. A reader who is debugging function A shouldn't have to already know that function B, called from a completely different module, quietly altered the state A depends on.

- **DO:** Make dependencies between components explicit and traceable — through function parameters, constructor injection, or clearly named and scoped events — so that a reader can follow the actual chain of cause and effect by reading the code, rather than by discovering it through a debugger or a production incident.

- **DON'T:** Rely on the order in which independent pieces of code happen to execute (e.g., relying on one module's initialization side effect running before another module reads a value it sets) unless that ordering is enforced explicitly by the code's structure. Implicit ordering dependencies are invisible in the source and break the moment execution order changes for any unrelated reason.

### Immutability Where Practical

- **DO:** Prefer creating new values over mutating existing ones when the language and performance characteristics of the situation reasonably allow it. Immutable values can be freely shared, passed around, and reasoned about without worrying that some other part of the system silently changed them out from under you.
```
// Bad: mutation makes it unclear whether callers share the same object
function addTag(tags, newTag) {
    tags.push(newTag); // mutates the caller's array
    return tags;
}

// Good: return a new value, leave the original untouched
function addTag(tags, newTag) {
    return [...tags, newTag];
}
```

- **DO:** Mark values as immutable/constant wherever the language provides a mechanism to do so and the value is genuinely not meant to change after creation — this turns an accidental mutation into a compile-time or immediate runtime error instead of a subtle bug discovered much later.

- **DON'T:** Treat immutability as an absolute rule to apply everywhere regardless of cost — in hot paths where allocation overhead is measured and matters, or where a mutable local accumulator inside a tightly scoped function is clearly simpler and safer than threading new copies through, a controlled, narrowly scoped mutation is a reasonable engineering tradeoff.

### Global State

- **DON'T:** Reach for a global variable, singleton, or ambient/static state as the default way to make a value accessible from many places in the codebase. Global state makes every function that touches it implicitly coupled to every other function that touches it, makes tests dependent on execution order, and makes the true set of a function's inputs invisible from its signature.

- **DO:** Pass shared configuration, context, or resources explicitly through function parameters, constructors, or a well-defined and narrowly scoped context mechanism, so that what a piece of code depends on is visible at its point of use instead of hidden behind a global lookup.

- **DON'T:** Use global mutable state to work around a design that doesn't otherwise have a clean way to pass a value from one place to another. That's usually a sign the responsibility boundaries between components need rethinking, not a sign that a global is the natural solution.

### Pure Functions Where Possible

- **DO:** Prefer writing logic as pure functions — output depends only on input, with no observable side effects — wherever the underlying operation is naturally a computation rather than an interaction with the outside world. Pure functions are trivially testable (no setup, no mocking, no ordering concerns), safely reusable, and safe to run concurrently.
```
// Bad: mixes pure calculation with I/O and external state in one function
function computeAndLogTotal(items) {
    const total = items.reduce((s, i) => s + i.price, 0);
    console.log(`Total: ${total}`); // side effect entangled with the calculation
    lastComputedTotal = total; // hidden global mutation
    return total;
}

// Good: the calculation is pure; side effects happen at the call site
function computeTotal(items) {
    return items.reduce((s, i) => s + i.price, 0);
}
const total = computeTotal(items);
console.log(`Total: ${total}`);
```

- **DO:** Push side effects to the boundaries of a call graph (the top-level orchestration code) and keep the inner, decision-making logic pure, so that the parts of the system doing the actual business logic can be tested and understood in isolation from the parts that talk to the outside world.

- **DON'T:** Assume purity is an all-or-nothing property that has to apply to an entire module or system before it's worth pursuing. Even converting a handful of the most logic-heavy, most-tested functions in a codebase to be pure meaningfully reduces the surface area a reader or a test has to account for.

### Idempotency and Predictable Re-execution

- **DO:** Design operations that might be retried, re-run, or re-delivered (network calls, background jobs, message handlers, anything crossing a boundary that can fail partway through) to be idempotent — running them twice with the same input should produce the same end state as running them once. Failures at exactly the wrong moment are a certainty in any non-trivial system, and idempotency is what keeps a retry from turning a transient hiccup into duplicated data or a duplicated side effect.
```
// Bad: retrying this after a network blip double-charges the customer
function chargeCustomer(order) {
    paymentGateway.charge(order.customerId, order.amount);
}

// Good: a stable idempotency key lets a retry be recognized and skipped
function chargeCustomer(order) {
    paymentGateway.charge(order.customerId, order.amount, {
        idempotencyKey: order.id,
    });
}
```

- **DON'T:** Assume "this will only ever run once" for any operation that crosses a process, network, or queue boundary. Anything that can be retried by infrastructure, redelivered by a message broker, or re-triggered by a user double-clicking a button eventually will be, regardless of how unlikely it seems during development.

### Concurrency-Safe State

- **DO:** Treat any state that could plausibly be read or written from more than one concurrent execution path (multiple threads, multiple processes, multiple requests, multiple asynchronous callbacks) as requiring explicit thought about synchronization, rather than assuming sequential-looking code will behave sequentially at runtime. A race condition doesn't announce itself in the code — it only shows up as an intermittent, hard-to-reproduce bug much later.

- **DON'T:** Assume that reading and then writing a shared value (check-then-act) is safe just because the two operations are written on adjacent lines. Without an explicit guarantee of exclusivity, another concurrent execution can slip in between the read and the write and invalidate the assumption the code was relying on.

### Isolating Non-Determinism

- **DO:** Isolate genuinely non-deterministic inputs — the current time, random number generation, network responses, environment-dependent values — behind a narrow, explicit seam (an injected clock, an injected random source, a passed-in value) rather than reaching for them directly from deep inside business logic. Logic that reads the system clock or generates a random value directly, buried several calls deep, cannot be tested deterministically and cannot be reasoned about without also accounting for wall-clock time or randomness.
```
// Bad: non-determinism buried directly inside the logic being tested
function isExpired(token) {
    return Date.now() > token.expiresAt; // impossible to test deterministically
}

// Good: non-determinism is injected, so the logic itself is deterministic and testable
function isExpired(token, now) {
    return now > token.expiresAt;
}
```

- **DON'T:** Let non-deterministic values leak into the core decision-making logic of the system in a way that makes tests flaky or behavior unreproducible. A test that occasionally fails because it depended on real elapsed time or genuine randomness usually indicates the underlying code should have taken that value as an explicit input instead.

### Resource Lifecycle and Cleanup

- **DO:** Guarantee that every resource explicitly acquired (a file handle, a network connection, a lock, a database transaction) is released on every possible exit path from the code that acquired it, including error paths — using your language's structured mechanism for guaranteed cleanup (a `finally`-equivalent, a scoped/managed resource construct) rather than relying on cleanup code at the end of the "normal" path only.
```
// Bad: the lock is only released on the success path
acquireLock(resourceId);
doWork(); // if this throws, the lock is held forever
releaseLock(resourceId);

// Good: cleanup happens on every exit path, including exceptions
acquireLock(resourceId);
try {
    doWork();
} finally {
    releaseLock(resourceId);
}
```

- **DON'T:** Assume a resource will be cleaned up "eventually" by a garbage collector or runtime without an explicit release, when that resource is scarce, held externally (a database connection, a lock visible to other processes), or has effects outside the current process's memory. Relying on non-deterministic finalization for anything with real, timely cost outside the process is a common source of resource exhaustion and deadlocks under load.

- **DO:** Make ownership of a shared or scarce resource clear from the code — which part of the system is responsible for eventually releasing it. Ambiguous ownership (two components each assuming the other will clean up) reliably leads to either a leak (neither cleans up) or a double-release bug (both do).

## Dependencies

Every dependency you add is code you didn't write but are now responsible for — its bugs become your bugs, its security vulnerabilities become your vulnerabilities, and its abandonment becomes your maintenance burden. Treat adding one as a decision with ongoing cost, not a one-time convenience.

### Is This Dependency Actually Justified?

- **DO:** Weigh a new dependency against the actual complexity of what it replaces. If the functionality needed is a small amount of well-understood logic — a handful of lines with no tricky edge cases — writing it directly is often cheaper over the dependency's whole lifetime than taking on an external package, its transitive dependencies, and its update cadence.
```
// Pulling in an entire package for something this small is
// rarely worth the added dependency surface:
import { isEven } from "some-tiny-utility-package";

// Six lines of well-understood logic don't need an external dependency:
function isEven(n) { return n % 2 === 0; }
```

- **DO:** Reach for an established, well-maintained dependency when the problem is genuinely hard, security-sensitive, or easy to get subtly wrong (cryptography, date/time handling across time zones, parsing complex formats). These are exactly the cases where "just write it yourself" tends to produce quietly broken code that a specialized, battle-tested library has already handled correctly.

- **DON'T:** Add a dependency to save a small amount of short-term effort without considering the long-term cost of tracking its updates, auditing its security advisories, and understanding its behavior when something goes wrong in production. A dependency you don't understand is a liability you've delegated, not eliminated.

- **DO:** Consider the size and scope of what a dependency actually brings in relative to what you need from it. Pulling in a large, general-purpose library to use one small piece of its functionality adds far more surface area — code size, attack surface, transitive dependencies — than the value it delivers.

### Freshness and Maintenance Status

- **DO:** Check a dependency's maintenance signals before adopting it — recent releases, how actively issues and pull requests are triaged, whether there's more than one active maintainer, and whether it has a healthy usage base. A project with no meaningful activity in years is a risk: a future security issue or compatibility break may simply never get fixed upstream.

- **DON'T:** Assume a dependency that was well-maintained when it was first added stays that way forever. Periodically re-evaluate long-standing dependencies, especially ones central to the system, since maintainers change, projects get abandoned, and yesterday's safe choice can quietly become today's liability.

- **DO:** Prefer dependencies with a track record of prompt security patches and clear versioning practices over ones that make breaking changes unpredictably or leave known vulnerabilities open for long stretches.

### Avoiding Dependency Bloat

- **DON'T:** Let the dependency tree grow unchecked simply because adding one more package is individually easy. Each dependency adds to install size, build time, attack surface, and the number of places a supply-chain compromise could enter the codebase; the cumulative cost of many "small, harmless" additions is rarely small.

- **DO:** Periodically audit the dependency list for packages that are no longer actually used, that duplicate functionality another dependency already provides, or that were added for a feature that has since been removed. Unused or redundant dependencies are pure cost with no offsetting benefit.

- **DON'T:** Add a second dependency that solves largely the same problem as one already in the project (e.g., two different HTTP client libraries, two different date libraries) without a strong, specific reason. Redundant dependencies increase bundle size or footprint and force every future contributor to learn and remember which one to use where.

### Vetting Before Adding

- **DO:** Check a dependency's license compatibility with your project before adding it, especially for anything that will ship to end users or be distributed — a license mismatch discovered after the fact can force an emergency rewrite or removal under time pressure.

- **DO:** Check a dependency's known security history — past vulnerabilities, how quickly they were patched, and whether any are currently open and unpatched — before adopting it, and keep checking through automated vulnerability scanning after adoption, since new vulnerabilities can surface in code that was safe when you first added it.

- **DON'T:** Add a dependency solely because it's popular or trending, without briefly reading through its source, its issue tracker, or its changelog to get a sense of its quality and stability. Popularity correlates with quality but doesn't guarantee it, and it says nothing about whether the package fits your specific use case.

- **DO:** Prefer a dependency with a minimal, well-scoped footprint and few transitive dependencies of its own over one that pulls in a sprawling tree of indirect dependencies to do the same job — every transitive dependency is a piece of code you've implicitly taken on without directly choosing it.

### Version Pinning and Supply-Chain Awareness

- **DO:** Pin dependency versions (or use a lockfile mechanism) so that builds are reproducible and a dependency can't silently change underneath you between one install and the next. An unpinned dependency that gets a new release with a breaking change or a compromised update can affect your system without anyone on your team having made a deliberate decision.

- **DO:** Review what actually changed before upgrading a dependency to a new major or otherwise significant version, rather than upgrading blindly on the assumption that "newer is better." A changelog review, however brief, catches breaking changes and unexpected new behavior before they reach production instead of after.

- **DON'T:** Grant a dependency more trust than its role warrants. A build-time or development-only tool typically doesn't need the same scrutiny as a dependency that runs in production with access to user data — but a small, seemingly unimportant dependency deep in the tree can still be a supply-chain risk if it runs in a sensitive context, so match the scrutiny to the actual exposure, not to how prominent the dependency looks.

### Build vs. Buy (or Adopt)

- **DO:** Frame the dependency decision explicitly as build-vs-adopt, weighing the ongoing maintenance cost of owning custom code against the ongoing cost of tracking and trusting someone else's. Neither option is free; the right call depends on how central, how stable, and how well-understood the problem actually is.

- **DON'T:** Default to always building custom just to avoid a dependency, when doing so means quietly re-implementing (and re-debugging, and re-securing) a well-solved problem from scratch. "No dependencies" is not automatically the safer or cheaper choice — it just moves the cost from "tracking someone else's maintenance" to "being the only maintainer yourself."

### Internal Dependencies Deserve the Same Rigor

- **DO:** Apply the same discipline around versioning, breaking changes, and communication to internal shared libraries and packages (a company's own shared component, an internal monorepo package, a platform team's shared utility) that you'd expect of a well-run external dependency. An internal dependency is still a dependency — teams consuming it are still exposed to whatever it does, and a careless breaking change causes exactly the same kind of damage as an external one, just with less warning since nobody thinks to check an internal package's changelog.

- **DON'T:** Assume that because you (or a team you know) control an internal dependency, it's exempt from deprecation notices, versioning discipline, or migration paths when it changes. Consumers of an internal package are frequently teams with no visibility into its development and no advance warning beyond what's actually communicated to them.

### Isolating Third-Party Dependencies Behind Your Own Interface

- **DO:** Wrap a significant third-party dependency behind a narrow interface of your own design, especially when that dependency is called from many places across the codebase, rather than letting its specific API and types spread directly into unrelated business logic throughout the system. When the dependency eventually needs to be upgraded in a breaking way, replaced, or worked around, the change stays contained to the wrapper instead of rippling through every call site that used the dependency directly.
```
// Bad: the third-party library's specific API is scattered across business logic
function chargeCustomer(order) {
    return thirdPartyGateway.createCharge({ amount_cents: order.total * 100, ... });
}
// dozens of other call sites also call thirdPartyGateway directly

// Good: business logic depends on your own interface, not the library directly
function chargeCustomer(order) {
    return paymentProcessor.charge(order.customerId, order.total);
}
// only paymentProcessor's implementation knows about thirdPartyGateway
```

- **DON'T:** Apply this wrapping reflexively to every dependency regardless of how small, stable, or narrowly used it is — wrapping a tiny, rarely-used utility purely on principle adds a layer of indirection that doesn't pay for itself. Reserve the wrapper pattern for dependencies that are either central to the system, likely to change, or used from many unrelated places.

## Code Review & Collaboration Norms

A code review that only checks formatting and bikesheds variable names has failed at the review's actual job. The point of review is a second mind verifying that the change does what it claims, does it safely, and will be maintainable by someone other than its author.

### Reviewing for Substance, Not Just Style

- **DO:** Review primarily for correctness, readability, and long-term maintainability — does the change actually solve the stated problem, are there edge cases it misses, will another engineer be able to understand and safely modify this code in six months. Style issues matter, but they're the cheapest kind of feedback to give and the least valuable kind to spend a review's attention on.

- **DON'T:** Let a review devolve into nitpicking personal style preferences that a formatter or linter should be enforcing automatically. If the same stylistic comment ("prefer this bracket style," "I'd use a different name here") keeps coming up across reviews, that's a sign it belongs in an automated tool or a documented convention, not in every individual review's back-and-forth.

- **DO:** Actively look for what the change might break — missed edge cases, race conditions, error paths that aren't handled, assumptions that don't hold for all callers — rather than only checking that the code does what the author says it does on the intended happy path.

- **DO:** Evaluate whether the change is testable and whether it's actually tested, not just whether it currently passes the tests it comes with. Ask whether the tests included would actually catch a regression if the logic broke later, not just whether tests exist as a checkbox.

### Giving Actionable, Specific Feedback

- **DO:** Make review feedback specific and actionable — point to the exact line or scenario, explain the concern concretely, and where possible suggest a direction for a fix. "This could fail" is far less useful than "this will throw if the list is empty; consider handling that case explicitly."
```
// Less useful review comment:
"This function is confusing."

// More useful review comment:
"This function mixes validation and persistence — could you split it into
validateOrder() and saveOrder() so each has a single, testable responsibility?"
```

- **DON'T:** Leave vague, unresolvable feedback ("this feels off," "not sure about this") without explaining what specifically raises the concern. Feedback the author can't act on just creates a round trip to ask what you meant, and often the underlying concern doesn't survive being asked to be more specific.

- **DO:** Distinguish, in review comments, between a blocking issue (a real bug, a security problem, a correctness concern) and a non-blocking suggestion (a stylistic preference, a "nice to have" improvement, a note for a possible follow-up). Marking the severity explicitly keeps the author from either over-indexing on a minor nit or missing a genuine blocker buried among nits.

- **DO:** Explain the reasoning behind a requested change, not just the change itself. "Move this to a separate function" is a demand; "move this to a separate function so it can be unit tested independently of the database call" is a teaching moment the author can generalize to their next piece of code.

### Reviewing the Tests, Not Just the Code

- **DO:** Read the tests included with a change as carefully as the production code, and check that they actually assert something meaningful about the behavior rather than merely executing the code path without checking its result. A test that runs a function and asserts nothing about its output, or asserts something trivially true, provides false confidence and is arguably worse than no test at all — it makes the change look safer than it is.

- **DON'T:** Treat the presence of tests, or a high coverage number, as sufficient evidence a change is well-tested without checking what those tests actually verify. Coverage measures which lines executed during a test run, not whether the test would catch a regression — a test can execute every line of a function and still fail to check that the function computed the right answer.

### Small, Reviewable Diffs

- **DO:** Keep individual changes small and focused on one logical unit of work. A reviewer can hold a 100-line, single-purpose diff in their head and reason carefully about every line; a 2,000-line diff spanning five unrelated concerns gets skimmed, not reviewed, no matter how conscientious the reviewer is.

- **DON'T:** Bundle unrelated changes into a single review (a bug fix, an unrelated refactor, and a dependency bump all in one diff). Bundling makes it hard to review each concern on its own merits, hard to revert just one part if something goes wrong, and hard to understand the change's actual history later.

- **DO:** Split a large feature into a sequence of smaller, independently reviewable and, where possible, independently mergeable changes (e.g., landing groundwork and refactors before the feature itself, behind a flag if needed), rather than presenting the entire feature as one massive diff at the end.

### The Reviewer's Responsibility

- **DON'T:** Approve a change without actually understanding what it does and why. A rubber-stamp approval defeats the entire purpose of review — it exists to catch the author's blind spots specifically, which a reviewer who didn't genuinely engage with the change cannot do.

- **DO:** Ask questions in review when something isn't clear, rather than assuming the author got it right because the code looks plausible. A reviewer's confusion about the code is itself valuable information — if the reviewer can't follow it, other future readers likely won't either.

- **DO:** Take the time to actually run or trace through non-trivial logic during review when that's feasible, rather than only reading it. Logic that looks correct on a skim can hide an off-by-one error, an incorrect boundary condition, or a misunderstood API contract that only becomes apparent when traced carefully.

- **DON'T:** Treat review as an adversarial gate to pass or a formality to get through quickly. Both the author and the reviewer share the same goal — a change that's correct and maintainable — and framing review as a collaborative check rather than a hurdle produces better outcomes and better working relationships over time.

### Review Turnaround and Etiquette

- **DO:** Review pending changes promptly. A change that sits unreviewed for days blocks its author, encourages them to context-switch away and lose momentum, and often invites scope creep as more commits pile onto the same branch while it waits.

- **DON'T:** Let review tone read as personal criticism of the author rather than an evaluation of the code. Phrasing feedback around the code itself ("this function assumes X, but callers can pass Y" rather than "you forgot about Y") keeps the conversation focused on the work and easier for the author to receive without becoming defensive.

- **DO:** Acknowledge what's good about a change, not only what needs to change. A review that's entirely a list of problems, with no recognition of a solid approach or a clean piece of code, reads as more discouraging than the feedback actually warrants and can make an author defensive toward even the valid points.

### Automating What Review Shouldn't Have to Catch

- **DO:** Push formatting, import ordering, whitespace, and other purely mechanical style concerns onto an automated formatter or linter run before a human ever looks at the diff. Every one of those issues a human reviewer catches manually is a moment of their limited attention spent on something a machine could have caught for free.

- **DON'T:** Rely on human reviewers to consistently catch the same category of mechanical issue over and over across many reviews. If a class of problem keeps recurring in review comments, that's a signal to add a linter rule, a type check, or a test — not to keep manually re-flagging it indefinitely.

### Self-Review Before Requesting Review

- **DO:** Read through your own diff, as if you were the reviewer, before sending it out. Authors routinely catch their own leftover debug statements, dead code, missing edge cases, and unclear naming on a self-review pass that they would have otherwise handed off for a reviewer to find, wasting a whole review round-trip on something the author could have caught alone.

- **DO:** Leave explanatory notes directly on your own diff, before a reviewer even looks at it, for anything non-obvious — a chunk of code that looks odd but is intentional, an approach that was chosen over an apparently simpler alternative for a specific reason. This answers the reviewer's likely first question before they have to ask it and speeds up the whole review.

- **DON'T:** Submit a change for review that its own author hasn't actually read end to end since assembling it from several editing sessions. A diff nobody has read in full, including its author, is not actually ready to consume someone else's review attention yet.

## Refactoring Discipline

Refactoring is supposed to be the safe kind of change — the behavior stays identical, only the internal structure improves. That safety is only real if the discipline around it is followed; refactoring done carelessly is just a rewrite with extra confidence.

### Small, Verifiable Steps

- **DO:** Refactor in a sequence of small steps, each one independently verifiable (tests pass, behavior is unchanged), rather than as one large, sweeping restructuring done all at once. Small steps mean that if something breaks, the cause is obvious and localized; a giant restructuring that breaks something leaves you searching through hundreds of changed lines for the culprit.

- **DO:** Run the test suite (or otherwise verify behavior) after each meaningful refactoring step, not just once at the very end. Catching a behavior change immediately after the step that introduced it is far cheaper than discovering it after several more steps have been layered on top.

- **DON'T:** Let a refactor sprawl beyond its original, stated scope while you're in the middle of it. It's tempting to fix "just one more thing" you noticed along the way, but each unrelated fix folded into an in-progress refactor increases the size of the eventual diff and the risk that something in the combined change is wrong.

### Never Mixing Refactoring with Behavior Changes

- **DON'T:** Combine a structural refactor (renaming, extracting functions, reorganizing files, changing internal implementation) with an actual behavior change (a bug fix, a new feature, a changed business rule) in the same commit or the same pull request. When both are mixed together, a reviewer can't tell which lines are "safe, mechanical restructuring" and which lines actually need scrutiny for correctness — and if something regresses, it's unclear which of the two changes caused it.
```
// Bad: a single commit that both renames things AND fixes a bug
// "Refactor payment module and fix rounding bug"
// (reviewer now has to untangle which lines are cosmetic and which
//  actually changed behavior, and a revert takes back both at once)

// Good: separate commits/PRs, each independently understandable and revertible
// Commit 1: "Rename PaymentMgr to PaymentProcessor, extract validateCard()"
// Commit 2: "Fix rounding error in calculateTax() for fractional cents"
```

- **DO:** Land a pure refactor as its own commit or pull request, with an explicit statement that behavior is unchanged, separate from any commit that changes what the system actually does. This makes both changes independently reviewable, independently revertible, and independently understandable in the project's history later.

- **DO:** When a refactor reveals a genuine bug along the way, note it and fix it as a separate, subsequent change rather than silently folding the fix into the refactor. The refactor's diff should be explainable purely in terms of "moved things around," with no behavioral claims to verify beyond "nothing changed."

### Tests Before Refactoring

- **DO:** Make sure there's adequate test coverage of the current behavior before starting a non-trivial refactor, and if coverage is missing for the part you're about to touch, write the missing tests first, against the existing behavior, before changing anything. Tests written before a refactor are what let you trust that "the structure changed but the behavior didn't."

- **DON'T:** Refactor a piece of code with unclear or absent behavior guarantees and no tests, purely on the assumption that you understand its full behavior well enough to preserve it by hand. Complex or old code frequently has subtle behavior (an edge case, a quirk some caller depends on) that isn't obvious from reading it, and only a test written against the actual current behavior will catch a break.

- **DO:** Treat characterization tests (tests that capture what a piece of code currently does, even if that behavior looks questionable) as a legitimate and often necessary first step for safely refactoring poorly-tested legacy code, separate from the question of whether that behavior is actually correct.

### The Boy Scout Rule vs. Unauthorized Rewrites

- **DO:** Leave code you touch slightly cleaner than you found it, when the improvement is small, low-risk, and directly adjacent to the change you're already making — a clearer name, a small extracted function, a removed piece of now-dead code. These small, incremental improvements compound over time into a much healthier codebase, at negligible individual cost or risk.

- **DON'T:** Use a small, unrelated task as an excuse to perform a large, unplanned rewrite of a file, module, or system you happened to be passing through. A sprawling rewrite tucked inside an otherwise small change surprises reviewers, balloons the diff and its risk, and often isn't coordinated with anyone who has context on why the code is the way it is.

- **DO:** Propose and discuss large-scale restructuring as its own deliberate, planned piece of work — with its own review, its own testing plan, and, if it affects other teams, their input — rather than executing it unilaterally as a side effect of an unrelated task.

- **DON'T:** Assume that code you find confusing or outdated must be bad and safe to rewrite wholesale. Code that looks wrong at first glance sometimes encodes a hard-won fix for a subtle bug or edge case that isn't obvious from reading it alone; understand why it's the way it is — check its history and the people who wrote it if needed — before assuming it needs a rewrite rather than a smaller, more targeted improvement.

### Large-Scale Migrations

- **DO:** Migrate large systems incrementally, running the old and new approaches side by side where possible, and cutting traffic or usage over gradually rather than all at once. An incremental migration lets you validate the new approach against real behavior at small scale before it's the only thing left standing, and gives you a clear rollback path at every step along the way.

- **DON'T:** Attempt a "big bang" rewrite of a large, business-critical system as a single, long-running effort that replaces everything at once at the end. Long-running rewrites tend to fall behind the continuing evolution of the system they're replacing, accumulate enormous integration risk that's only discovered at the very end, and are far harder to partially roll back if something is wrong.

- **DO:** Keep a large migration reversible for as long as possible — feature-flagged, running behind a toggle, with the old path still intact — so that a problem discovered after cutover can be mitigated by reverting the flag rather than by an emergency rollback of the entire migration.

### Renaming and Refactoring with Tooling

- **DO:** Use automated, tooling-assisted renames and structural refactors (where your language and editor ecosystem support them) over manual find-and-replace for anything beyond a trivial, single-file change. Tooling-assisted renames correctly handle scoping, references, and edge cases that a manual text search routinely misses or over-matches.

- **DON'T:** Perform a wide rename or structural change by hand across many files without a final, careful search for anything the automated tooling (or your manual process) might have missed — a string used in a place the tooling doesn't understand (a config file, a piece of dynamically constructed code, a log message) can silently be left referring to the old name.

### Communicating Refactors That Affect Others

- **DO:** Give advance notice to teams or individuals who actively work in an area of the codebase before starting a significant refactor there, especially one that will touch shared interfaces, widely used utilities, or code with several concurrent contributors. A refactor that collides with someone else's in-flight work causes painful merge conflicts and wasted effort on both sides that a heads-up would have avoided entirely.

- **DON'T:** Silently refactor code that other people depend on or are actively modifying without any communication, even when you're confident the refactor is purely internal and behavior-preserving. "Purely internal" from your point of view can still mean a large, conflict-generating diff from theirs, and the courtesy of a heads-up costs little compared to the friction it prevents.

## Consistency

A codebase with one clear way to do each kind of thing is easier to read than one where every file reflects a different author's personal taste, even if each individual file, taken in isolation, is well written. Consistency is a property of the whole system, and it has to be actively maintained, not just hoped for.

### Matching Existing Conventions

- **DO:** Follow the conventions already established in the codebase you're working in — naming style, file organization, error-handling patterns, testing approach — even when they differ from your personal preference. A codebase that's internally consistent is easier for everyone to navigate than one where each contributor's section reflects their own individual taste.

- **DON'T:** Introduce a new pattern, library, or convention for something the codebase already has an established way of doing, just because you personally prefer a different approach. Every additional way of doing the same kind of thing is one more thing every future contributor has to learn, recognize, and choose correctly between.
```
// Existing codebase convention: errors are returned as a Result type
function parseConfig(text) -> Result<Config, ParseError> { ... }

// Bad: introduces a second, inconsistent error-handling style
// in a new function, just because the author prefers exceptions
function parseSettings(text) { if (!valid) throw new Error("bad"); ... }

// Good: match the existing convention, even if it's not your first choice
function parseSettings(text) -> Result<Settings, ParseError> { ... }
```

- **DO:** Raise a disagreement with an existing convention through a discussion with the team and, if there's agreement, a deliberate, documented, codebase-wide migration — not by unilaterally introducing a competing pattern in the one file you happen to be editing.

### One Way to Do a Thing

- **DO:** Standardize on a single approach for each recurring kind of problem within a codebase — one way to handle errors, one way to structure a module, one way to fetch data, one way to test a component — and document that choice somewhere discoverable, so new contributors don't have to reverse-engineer the convention from example code.

- **DON'T:** Let multiple competing solutions to the same problem accumulate side by side in a codebase (three different date-formatting helpers, two different state-management approaches) without ever consolidating them. Each new contributor who finds inconsistent precedent will reasonably copy whichever example they happened to find first, and the number of competing patterns only grows over time.

- **DO:** When you notice competing patterns already in the codebase, treat consolidating them as valuable, explicitly scoped work worth prioritizing — not just an unavoidable side effect of the codebase's history to permanently live with.

### The Cost of Inconsistency

- **DO:** Recognize that inconsistent style and patterns impose a real, ongoing cognitive tax on every reader — every file a developer opens forces them to first figure out which local convention is in play before they can even start understanding the actual logic. That overhead is invisible in any single file but adds up across an entire codebase and every developer who works in it.

- **DON'T:** Dismiss consistency concerns as "just style" with no real engineering cost. Inconsistency measurably slows down onboarding, increases the rate of subtle bugs (an assumption that holds in most of the codebase but not in the inconsistent corner), and makes automated tooling (linters, formatters, codemods) harder to apply uniformly.

- **DO:** Weigh the value of a locally "better" pattern against the cost of introducing yet another way of doing things codebase-wide. A marginally superior approach applied inconsistently is very often worse for the codebase as a whole than a merely-good approach applied everywhere.

### Consistency Across Teams and Service Boundaries

- **DO:** Extend consistency expectations across team and service boundaries for anything that crosses them — shared API conventions, shared error-response shapes, shared authentication patterns, shared naming for cross-cutting concepts like pagination or timestamps. Two teams solving the same cross-cutting problem in two different ways multiplies the learning cost for anyone who has to work across both.

- **DON'T:** Assume that because one team or one service is independently owned, it's free to diverge from shared conventions without coordination. Independent ownership is about who decides, not license to ignore agreements that make the whole system coherent for everyone who has to integrate with more than one part of it.

### Enforcing Consistency with Tooling

- **DO:** Encode agreed-upon conventions into automated tooling — linters, formatters, static analysis rules, templates, scaffolding generators — wherever possible, so that consistency is maintained by default rather than by everyone remembering and manually applying a style guide on every change.

- **DON'T:** Rely solely on a written style guide or onboarding documentation to maintain consistency over time. Written guidance degrades in practice as a codebase grows and turnover happens — automated enforcement is what actually holds the line once the people who originally agreed on the convention have moved on to other things.

### Consistency vs. Dogmatism

- **DO:** Treat an established convention as the strong default, but allow a deliberate, explained exception when a specific situation genuinely doesn't fit the general rule. A convention applied so rigidly that it produces a clearly worse outcome in an unusual case has stopped serving the goal consistency exists for in the first place — a shared, low-friction way of working.

- **DON'T:** Break an established convention silently. If a specific case genuinely warrants deviating from the norm, say so explicitly — in a comment, in the review discussion, or in documentation — so the deviation reads as a deliberate, considered exception rather than as an inconsistency nobody can explain later.

### Making Conventions Discoverable

- **DO:** Write down a codebase's key conventions somewhere a new contributor will actually find them — project documentation, a contributing guide, comments at the top of a canonical example file — rather than leaving them to be inferred purely by reading enough existing code to notice the pattern. A convention only a few long-tenured people know about isn't really a shared convention; it's tribal knowledge that new contributors will inevitably violate through no fault of their own.

- **DON'T:** Assume a new contributor will correctly infer an unwritten convention from a handful of example files, especially when the codebase itself contains inconsistent precedent from before the convention was settled. Given ambiguous or conflicting examples, a new contributor will reasonably copy whichever one they saw first, perpetuating exactly the inconsistency the convention was meant to prevent.

### A Consistent Representation for "No Value"

- **DO:** Settle on one consistent way to represent "absent," "empty," or "unknown" for a given kind of data across the codebase, and use it uniformly, rather than letting different parts of the system independently choose their own convention for the same concept.
```
// Bad: three different conventions for "no value" scattered across one codebase
function findUser(id) { return null; }        // module A: null means not found
function findAccount(id) { return undefined; } // module B: undefined means not found
function findOrder(id) { return -1; }           // module C: a sentinel means not found
// every caller now has to remember which convention applies to which function

// Good: one consistent convention used everywhere for this concept
function findUser(id) { return null; }
function findAccount(id) { return null; }
function findOrder(id) { return null; }
```

- **DON'T:** Let the choice of how to represent absence vary by author or by module instead of by a deliberate, codebase-wide decision. A caller who has to remember, function by function, whether "not found" means `null`, `undefined`, an empty collection, or a special sentinel value is paying a real, avoidable cognitive tax on every single call.

## Technical Debt

Technical debt, used well, is a legitimate and sometimes correct engineering tradeoff — shipping something imperfect now in exchange for a real, understood cost paid later. It stops being legitimate the moment it's incurred silently, forgotten, or treated as if it cost nothing.

### Intentional vs. Accidental Debt

- **DO:** Distinguish between debt taken on deliberately, with a clear understanding of the tradeoff at the time (shipping a simpler solution now, knowingly deferring a more complete one), and debt that accumulates by accident — through lack of awareness, skipped tests, or simply not knowing a better approach existed. The two require very different responses: intentional debt needs a plan to pay it down, accidental debt needs the underlying skill or process gap fixed.

- **DON'T:** Let debt accumulate purely through neglect — skipped error handling, missing tests, copy-pasted logic — without anyone consciously deciding that tradeoff was worth making. Debt that nobody chose is debt nobody is tracking, and it tends to compound silently until it surfaces as a difficult, expensive problem.

- **DO:** Treat the decision to take on debt as a real engineering decision deserving the same level of scrutiny as any other design choice — what's being traded away, what it will cost to fix later, and whether that tradeoff is actually worth it for the situation at hand (a genuine deadline, an experiment that might get thrown away, a low-risk area of the codebase).

### Documenting Debt When Taken On Deliberately

- **DO:** Record deliberately taken-on debt somewhere visible and durable — a tracked ticket, a clearly marked comment with a reason and a reference, a section in project documentation — rather than letting it live only in the memory of whoever made the decision. Debt that isn't written down effectively doesn't exist to anyone who joins the project later, which means it never gets paid down because nobody besides the original author even knows it's there.
```
// Bad: silent shortcut, no trace of why or that it's temporary
function getExchangeRate(currency) {
    return 1.0; // TODO
}

// Good: the shortcut and its reasoning are on the record
// KNOWN LIMITATION (see TICKET-482): hardcoded to 1.0 until the
// live rates API contract is finalized. Safe short-term because
// this only affects the currently USD-only beta cohort.
function getExchangeRate(currency) {
    return 1.0;
}
```

- **DO:** Explain the reasoning behind a piece of intentional debt when documenting it, not just the fact that it exists — what was skipped, why it was an acceptable tradeoff at the time, and what conditions should trigger addressing it. Future readers need to judge whether the original tradeoff still holds, which they can't do without knowing what it originally weighed.

- **DON'T:** Bury a debt marker somewhere unlikely to ever be seen again (an obscure comment in a rarely opened file, with no ticket or searchable marker). If the debt is worth tracking at all, it's worth tracking somewhere that will actually surface it again — a project's issue tracker, a consistently used marker convention that's periodically searched and reviewed, or equivalent.

### "Good Enough for Now" Is Not Free

- **DON'T:** Treat a "temporary" shortcut as though it has no ongoing cost simply because it isn't causing visible problems yet. Debt accrues interest — a shortcut that was minor when introduced tends to become entangled with more and more of the system the longer it survives, making it progressively more expensive to fix the later it's addressed.

- **DO:** Periodically revisit tracked technical debt and actively decide whether to pay it down, given current priorities, rather than letting it sit indefinitely by default. A debt item that's been open for years with no revisiting has effectively become permanent, whether or not anyone intended that.

- **DON'T:** Let "we'll fix it later" substitute for an actual plan. A debt item with no owner, no rough timeline, and no trigger condition for revisiting it is, in practice, a decision to never fix it — which may sometimes be the right call, but should be made explicitly rather than by default.

- **DO:** Factor the ongoing cost of carrying known debt — the extra caution it demands, the slower velocity it imposes on code near it, the risk it poses — into prioritization discussions alongside new feature work, rather than treating debt paydown as work that only happens when there's nothing more urgent competing for attention. Debt that's actually worth carrying should be able to justify itself against that comparison; debt that can't shouldn't have been taken on in the first place, or should be scheduled for repayment now.

### Distinguishing Debt from Bugs and Missing Features

- **DO:** Keep technical debt conceptually distinct from an outright bug (something that's simply broken and produces wrong behavior now) and from a missing feature (something that was never built). Debt specifically describes a working solution that was deliberately built in a way that will cost more to change or extend later — collapsing all three into one undifferentiated backlog makes it hard to prioritize any of them correctly, since they carry very different urgency and different arguments for why they matter.

- **DON'T:** Relabel an outright bug as "technical debt" to make it sound like a lower-priority, optional cleanup item. A bug producing incorrect behavior for users right now needs to be triaged and prioritized as a bug; calling it debt is a way of quietly deprioritizing something that shouldn't be deprioritized.

### Communicating Debt to Non-Engineering Stakeholders

- **DO:** Translate technical debt into terms a non-engineering stakeholder can actually weigh against other priorities — the concrete risk it poses (an outage, a security exposure, an inability to ship a needed feature quickly), not just an abstract appeal to "code quality." Stakeholders make better tradeoff decisions when the cost of debt is expressed in terms of what it threatens, not in terms internal to the codebase.

- **DON'T:** Assume that leadership or product stakeholders will infer the cost of accumulating debt on their own without it being actively surfaced. Debt that lives only in engineers' heads or in a backlog nobody outside engineering reads effectively doesn't exist for the purposes of prioritization decisions made above the engineering team.

### Cleaning Up Temporary Constructs

- **DO:** Treat temporary scaffolding — feature flags meant to be short-lived, compatibility shims from a migration, a dual-write path kept "just in case" during a cutover — as debt with a specific, known expiration condition, and actually remove it once that condition is met. Temporary constructs that outlive their purpose accumulate exactly like any other form of debt, except they're often invisible because everyone still remembers, incorrectly, that they were meant to be short-term.

- **DON'T:** Let a feature flag, migration shim, or compatibility layer become permanent by default through simple neglect. Every one left in place indefinitely is a live branch that has to be understood, tested, and reasoned about by everyone who touches the surrounding code, long after the reason for its existence has stopped applying.

## Quick Checklist
- Names reveal intent without needing an explanatory comment alongside them.
- Names are searchable, pronounceable, and free of cryptic project-only abbreviations.
- One consistent casing convention per identifier category, applied everywhere.
- No type or implementation detail baked into a name that will go stale when the implementation changes.
- Booleans read as yes/no questions (`is`/`has`/`can`/`should`) and avoid negative phrasing.
- A rename touches every place the old name appeared — declaration, comments, logs, siblings.
- No name promises behavior (e.g., "read-only") that the code doesn't actually deliver.
- Quantities that could plausibly be in more than one unit (time, size, money) encode the unit in the name or use a dedicated type.
- Every function has a single, nameable responsibility and reads as one clear thing.
- Side effects are obvious from a function's name and are never hidden inside something that looks like a pure read.
- Function arguments stay few; related parameters are grouped into a single meaningful structure.
- No boolean flag parameter silently switches a function's internal behavior.
- Commands (state changes) and queries (reads) are kept separate, not mixed in one function.
- Required call ordering is enforced by the API's shape, not left to documentation or convention.
- Comments explain why, never restate what the code already says.
- No commented-out dead code lingers — version control already remembers it.
- Every `TODO`/`FIXME` carries enough context (or a ticket link) to be actionable later.
- A stale or now-incorrect comment is fixed in the same change that invalidates it.
- Confusing code gets renamed or refactored before it gets a comment explaining the confusion.
- Commit messages and PR descriptions explain why a change was made, not just what changed.
- READMEs and API docs are updated in the same change as the behavior they describe.
- Errors fail fast and loudly for programming errors and violated invariants.
- No `catch` block (or equivalent) silently swallows an error with no logging, handling, or re-throw.
- Recoverable, expected failures and unrecoverable, exceptional failures are modeled and handled differently.
- Every error message carries enough context (identifiers, inputs, operation) to debug without reproducing it.
- Wrapped errors preserve the original error instead of discarding it.
- Exceptions (or equivalents) are reserved for exceptional conditions, not used for routine, expected branching.
- Every call that waits on something outside the process (network, lock, queue) has an explicit, reasoned timeout.
- `try`/catch blocks stay narrow and catch only the specific error type actually expected, not a broad catch-all.
- No speculative abstraction is built for a need that doesn't concretely exist yet (YAGNI).
- Shared abstractions are extracted after a third genuine repetition, not the second.
- No interface, factory, or config layer exists for something that has exactly one implementation and no real second one coming.
- An abstraction stretched with special-case flags to fit a new caller is a candidate to split back apart, not to keep bending.
- Deeply nested conditionals are flattened with guard clauses and early returns.
- Complex boolean conditions are extracted into a well-named predicate.
- Mutable state is scoped as narrowly as possible and shared only when there's a clear, deliberate reason.
- No component silently mutates a shared object as its way of communicating with another component.
- Dependencies between components are explicit (parameters, constructors, defined events), not implicit ordering or shared globals.
- Values are treated as immutable by default where the language and performance situation reasonably allow it.
- Global/singleton state is avoided in favor of values passed explicitly through the call chain.
- Business logic is written as pure functions wherever feasible; side effects live at the system's edges.
- Acquired resources (files, connections, locks) are released on every exit path, including error paths.
- A new dependency is justified by real complexity or risk, not by saving a few lines of easy code.
- Dependencies are checked for maintenance activity, license compatibility, and security history before adoption.
- The dependency tree is periodically audited for unused, redundant, or abandoned packages.
- Code review focuses on correctness, readability, and maintainability, not just style nitpicks a linter should catch.
- Review feedback is specific and actionable, and marks clearly whether it's blocking or a suggestion.
- Diffs stay small and scoped to one logical change so a reviewer can actually reason about all of it.
- Unrelated changes (refactor, fix, dependency bump) are never bundled into the same diff.
- A reviewer only approves a change they actually understood — no rubber-stamping.
- Refactors proceed in small, independently verifiable steps, checked against tests after each one.
- A refactor and a behavior change are never mixed into the same commit or pull request.
- Tests covering current behavior exist (or are written first) before a non-trivial refactor begins.
- Small, low-risk cleanups are made in passing (boy-scout rule); large rewrites are proposed and planned separately, not slipped into an unrelated change.
- Existing codebase conventions are followed even when they differ from personal preference; disagreements go through team discussion, not unilateral deviation.
- Each recurring kind of problem has one established solution in the codebase, not several competing ones left to accumulate.
- The codebase uses one consistent representation for "no value" for a given kind of data, not a mix of null/undefined/sentinels.
- Deliberately taken-on technical debt is documented with its reasoning and a reference, not left as tribal knowledge.
- Debt is periodically revisited and consciously paid down or re-justified, never left open indefinitely by default.
- "Good enough for now" shortcuts are recognized as carrying an ongoing cost, not treated as free.
