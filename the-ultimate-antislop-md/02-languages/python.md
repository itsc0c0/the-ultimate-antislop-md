# Python

## Style & Naming

- **DO:** Use `snake_case` for functions, variables, methods, and module names; `PascalCase` for classes and exception types; `UPPER_SNAKE_CASE` for module-level constants. This is PEP 8's naming convention, and mixed conventions in the same codebase force readers to context-switch on every line.
```python
# Bad
class userAccount:
    MaxRetries = 3
    def GetBalance(self): ...

# Good
class UserAccount:
    MAX_RETRIES = 3
    def get_balance(self): ...
```
- **DO:** Choose descriptive names over short cryptic ones, even for local variables — `retry_count` instead of `rc`, `elapsed_seconds` instead of `es`. A name is documentation that never goes stale the way a comment can.
- **DON'T:** Use single-character names outside a tightly-scoped loop counter, comprehension variable, or trivial lambda parameter. `for i in range(10)` is fine; `d = compute(x, y, z)` at function scope is not — nobody six months from now knows what `d` means.
```python
# Bad — d, r, and t give no clue what they hold a few lines later
d = fetch_data()
r = transform(d)
t = r.total

# Good
customer_data = fetch_data()
transformed_records = transform(customer_data)
total_revenue = transformed_records.total
```
- **DO:** Prefix non-public attributes and helper functions with a single leading underscore (`_helper`), and reserve the name-mangled double-leading-underscore (`__attr`) for the narrow case of avoiding attribute collisions in a subclassing hierarchy. A single underscore is a convention, not enforcement — say so in review rather than relying on it as security.
- **DON'T:** Shadow builtins as variable or parameter names (`list`, `dict`, `id`, `type`, `str`, `input`, `sum`, `filter`, `min`, `max`). It silently masks the builtin for the rest of that scope and produces confusing `TypeError`s far from the shadowing line.
```python
# Bad
def process(list, str):
    return list.append(str)  # 'list' no longer refers to the builtin type

# Good
def process(items: list[str], value: str) -> None:
    items.append(value)
```
- **DO:** Name boolean variables and functions with an `is_`, `has_`, `can_`, or `should_` prefix (`is_valid`, `has_permission`, `should_retry`). It reads as a yes/no question at the call site instead of forcing the reader to guess the polarity.
```python
# Bad — reads ambiguously; is "active" a status string, a verb, something else?
active = check_user(user)
if active:
    ...

# Good — unambiguous at both definition and call site
is_active = check_user_is_active(user)
if is_active:
    ...
```
- **DON'T:** Use Hungarian notation or type suffixes in names (`user_list`, `count_int`, `nameStr`). Type hints already convey type; embedding it in the identifier is redundant and rots when the type changes but the name doesn't.
- **DO:** Name a function after what it *does*, using a verb phrase for anything with a side effect (`send_email`, `delete_user`) and a noun phrase or `get_`/`is_`/`has_`-prefixed name for anything that just returns a value with no side effect (`total_price`, `get_total_price`, `is_expired`) — this distinction helps a reader predict whether calling something is safe to do speculatively (just to inspect a value) or will actually change program state.
```python
# Ambiguous — does calling this just read a value, or does it also do something?
def user_status(user):
    user.last_checked = datetime.now()  # a side effect hidden behind a noun-like name
    return "active" if user.is_active else "inactive"

# Clearer — the verb signals a side effect is happening
def refresh_and_get_user_status(user):
    user.last_checked = datetime.now()
    return "active" if user.is_active else "inactive"
```
- **DON'T:** Give a function a name that implies it only computes and returns a value when it actually also mutates state, writes to a file, or makes a network call — a caller reasonably assumes a noun-like, `get_`-prefixed name is safe to call multiple times with no consequence, and a hidden side effect behind that name is a common source of confusing bugs when a refactor calls it in a new context expecting it to be pure.
- **DO:** Keep line length at the project's configured limit — PEP 8's historical default is 79 columns, but most modern Python projects standardize on 88 (Black's default) or 100–120. Pick one project-wide and let the formatter enforce it instead of debating it per pull request.
- **DON'T:** Use `l`, `O`, or `I` as standalone identifiers — in many fonts they're visually indistinguishable from `1` and `0`, and PEP 8 calls this out explicitly.
- **DO:** Name collections with plural nouns (`users`, `pending_orders`) rather than appending `_list`/`_array`/`_collection`. The plural alone signals "more than one" without duplicating type information the annotation already states.
- **DO:** Keep module and package names short, all-lowercase, and free of underscores when reasonably possible (`textutils`, not `Text_Utils.py`) — this matches the convention of the standard library and keeps `import` statements uncluttered.
- **DO:** Write docstrings for every public module, class, function, and method following PEP 257: an imperative one-line summary, a blank line, then further detail (args, returns, raises) if needed.
```python
def fetch_user(user_id: int) -> User:
    """Fetch a user by ID.

    Args:
        user_id: The primary key of the user to retrieve.

    Raises:
        UserNotFoundError: If no user exists with the given ID.
    """
```
- **DON'T:** Write comments that restate the obvious next line (`# increment i` above `i += 1`, `# loop over users` above `for user in users:`). A comment should explain intent or a non-obvious *why*, not narrate syntax the reader can already see.
- **DO:** Use f-strings for string interpolation and formatting in Python 3.6+. They're more readable than `%`-formatting or `.format()` and, unlike string concatenation with `+`, don't require manual type coercion for non-string values.
```python
# Bad
msg = "User " + str(user.id) + " has " + str(user.balance) + " credits"
msg2 = "User %s has %s credits" % (user.id, user.balance)

# Good
msg = f"User {user.id} has {user.balance} credits"
```
- **DON'T:** Mix string quote styles arbitrarily within a project. Pick one default (commonly double quotes, matching Black's default) and switch to the other only when the string itself contains that quote character, to avoid escaping.
- **DO:** Order imports in three groups separated by a blank line — standard library, then third-party, then local/first-party — each group alphabetized. `isort` (or `ruff`'s import-sorting rules) automates this so it never needs manual attention.
```python
# Good
import json
import os
from pathlib import Path

import requests
from pydantic import BaseModel

from myapp.models import User
from myapp.utils import retry
```
- **DON'T:** Use wildcard imports (`from module import *`) in application or library code. It pollutes the namespace, breaks static analysis and "find usages" tooling, and hides exactly which module a name came from.
- **DON'T:** Alias an import to something shorter purely to save keystrokes when the shorter name creates ambiguity or collides with a common variable name elsewhere in the file (`import pandas as pd` and `import numpy as np` are widely recognized, safe conventions; `import my_custom_module as m` is not, since `m` conveys nothing on its own to a reader unfamiliar with that specific choice).
```python
# Confusing — "d" conveys nothing, and shadows a very common single-letter name
from datetime import date as d
today = d.today()

# Clear
from datetime import date
today = date.today()
```
- **DO:** Use trailing commas in multi-line collection literals, function calls, and `def` signatures. It keeps version-control diffs to a single added/removed line when a new item is appended, instead of also touching the previous line's comma.
- **DON'T:** Chain multiple unrelated operations — comprehensions, walrus assignments, ternaries — into a single dense line just because Python's syntax allows it. Optimize for the next reader, not for keystroke count.
```python
# Bad
result = [y for x in data if (y := transform(x)) is not None and y.valid and (z := y.score) > threshold]

# Good
transformed = (transform(x) for x in data)
valid_items = (item for item in transformed if item is not None and item.valid)
result = [item for item in valid_items if item.score > threshold]
```
- **DO:** Reserve the walrus operator (`:=`) for cases where it removes genuine duplication or clarifies control flow (e.g., `while (chunk := file.read(8192)):`), not as a habit applied everywhere an assignment could theoretically be inlined.
- **DO:** Keep each function focused on a single responsibility; if a function needs internal comments to divide it into "sections," that's usually a signal to extract those sections into named helper functions instead.
- **DON'T:** Put multiple statements on one line with semicolons, or collapse a block onto the same line as its `if`/`for` header in new code (`if x: return`). One statement per line is the norm; a single-line guard clause is a defensible, narrow exception when it improves scanability.
- **DO:** Suffix custom exception class names with `Error` (`InvalidTokenError`, `InsufficientFundsError`), matching the convention the standard library uses for its own exception hierarchy.
- **DO:** Use `is` / `is not` for identity comparisons — especially against `None`, `True`, and `False` — and reserve `==` / `!=` for value equality.
- **DON'T:** Rely on `is` for value comparisons between small integers or short strings just because it happens to "work" in casual testing — CPython caches and interns some small integers and string literals as an implementation detail, so `is` can appear to work by coincidence for `x is 5` while failing unpredictably for `x is 500` or a dynamically-built string. `is` is for identity, `==` is for value equality; conflating them based on an implementation detail is a latent bug.
```python
>>> a = 5
>>> b = 5
>>> a is b        # True — small ints are cached by CPython, but this is an implementation detail
True
>>> a = 5000
>>> b = 5000
>>> a is b        # False — outside the small-int cache range
False
```
```python
# Bad
if x == None or x == True:
    ...

# Good
if x is None or x is True:
    ...
```
- **DON'T:** Assign a `lambda` to a name (`f = lambda x: x + 1`). Define it with `def` instead — PEP 8 flags this explicitly because named `def` functions produce a real `__name__` in tracebacks and introspection, while a lambda bound to a variable just reports `<lambda>`.
- **DO:** Check emptiness of a collection with plain truthiness (`if not items:`) rather than comparing its length to zero (`if len(items) == 0:`) — the truthiness form is shorter, avoids computing a length just to compare it, and is the idiom experienced Python readers expect.
```python
# Less idiomatic
if len(items) == 0:
    return

# Idiomatic
if not items:
    return
```
- **DON'T:** Write `if len(items) > 0:` to check for a non-empty collection — `if items:` says the same thing more directly, reserving an explicit `len()` comparison for when the actual numeric count is what matters, not just presence or absence.
- **DO:** Let an automatic formatter (Black or `ruff format`) own whitespace, line-breaking, and quote normalization. Treating formatting as a solved, automated problem removes an entire category of unproductive review comments.
- **DO:** Curate a package's public surface with an explicit `__all__` in `__init__.py` when the package is meant to expose a stable API. It documents intent and controls what `from package import *` (and star-import linting) actually exposes, rather than leaving it to whatever happens not to be underscore-prefixed.
- **DON'T:** Abbreviate words inconsistently across a codebase (`cfg` in one module, `config` in another, `conf` in a third for the same concept). Pick one spelling for a concept and use it everywhere, including in related function names.

### Consistent Return Values

- **DO:** Have every code path through a function return a value of the same conceptual type/shape, rather than sometimes returning `None`, sometimes a value, and sometimes an empty collection to mean overlapping things. Inconsistent return shapes force every caller to defensively check what they got back.
```python
# Bad — three different "nothing here" signals for what should be one concept
def get_active_users(team_id):
    team = find_team(team_id)
    if team is None:
        return None                       # team doesn't exist
    if not team.users:
        return []                         # team exists but is empty
    active = [u for u in team.users if u.is_active]
    if not active:
        return False                      # inconsistent — a bool now?!
    return active

# Good — one predictable shape; "no results" is always an empty list,
# and a genuinely different failure (missing team) is a raised exception
def get_active_users(team_id: int) -> list[User]:
    team = find_team(team_id)              # raises TeamNotFoundError if missing
    return [u for u in team.users if u.is_active]
```
- **DON'T:** Return a bare tuple with positional meaning from a function with more than two or three values, where callers have to remember which position means what. Return a `dataclass`/`NamedTuple` instead so each field is named at the call site.
- **DO:** Keep a function's truthiness behavior predictable when its result might be used in a boolean context — an empty list, `0`, and `""` are all falsy in Python, which is usually fine, but be deliberate about whether "empty result" and "no result at all" need to be distinguishable to callers (see the exceptions-vs-sentinels discussion under Error Handling).

### Guard Clauses Over Nested Conditionals

- **DO:** Return (or `continue`/`raise`) early on an invalid or edge-case condition at the top of a function, instead of wrapping the entire remaining body in a nested `if`. Each guard clause removes one level of nesting for everything after it, which keeps the function's main logic flat and readable instead of buried several indent levels deep.
```python
# Bad — the actual logic is buried three indent levels deep
def process_order(order):
    if order is not None:
        if order.items:
            if order.status == "pending":
                total = sum(item.price for item in order.items)
                return total
            else:
                return None
        else:
            return None
    else:
        return None

# Good — guard clauses handle the edge cases up front; the real logic is flat
def process_order(order):
    if order is None:
        return None
    if not order.items:
        return None
    if order.status != "pending":
        return None
    return sum(item.price for item in order.items)
```
- **DON'T:** Take guard clauses to the opposite extreme of stacking ten early returns for conditions that would be clearer grouped into one combined check (`if not order or not order.items or order.status != "pending":`) — the goal is readability, not a mechanical rule that every condition needs its own line no matter how related they are.
- **DO:** Reserve deep nesting for cases where it genuinely reflects the problem's structure (a recursive tree traversal, an intentionally staged multi-condition state machine) rather than treating "flatten everything" as a universal rule that overrides a case where nesting is the clearest representation of the actual logic.

### Boolean Expressions & Negation

- **DO:** Write boolean conditions in their most directly readable form, avoiding double negatives (`if not user.is_not_active:`) and unnecessary comparisons to `True`/`False` (`if is_valid == True:`) — a boolean value is already a boolean; testing it directly (`if is_valid:` / `if not is_valid:`) says the same thing with less to parse.
```python
# Bad — double negative, and a redundant comparison to True
if not user.is_not_active == True:
    grant_access()

# Good
if user.is_active:
    grant_access()
```
- **DO:** Use `all()`/`any()` with a generator expression to combine several related boolean conditions, instead of a long chain of `and`/`or` that's hard to scan, when the conditions are checking the same kind of thing across a collection.
```python
# Harder to scan
valid = field1_ok and field2_ok and field3_ok and field4_ok and field5_ok

# Clearer once there's a natural collection to check over
valid = all(check(field) for field in fields)
```
- **DON'T:** Rely on De Morgan's law transformations (`not (a and b)` vs `not a or not b`) without double-checking the transformed condition actually matches the original intent — this is a common source of off-by-one-negation logic bugs, especially inside a guard clause where getting the polarity backwards silently inverts the entire function's behavior for every input.

### Working with Optional Values

- **DO:** Use `getattr(obj, "attr", default)` for optionally-present attributes and `dict.get(key, default)` for optionally-present keys, rather than a `try/except AttributeError`/`try/except KeyError` when a simple default is all that's needed — it's shorter and doesn't risk the `try` block accidentally catching an unrelated exception from deeper inside a property getter.
- **DO:** Chain `or` for a simple "use this value, or fall back to that one if falsy" pattern, but be deliberate about it — `value or default` treats `0`, `""`, and `[]` as "missing" too, which is often not what's intended when the value's *presence*, not its truthiness, is what should trigger the fallback.
```python
# Subtly wrong if a limit of exactly 0 is a valid, meaningful value
effective_limit = limit or DEFAULT_LIMIT   # a limit of 0 incorrectly becomes DEFAULT_LIMIT

# Correct — checks for None specifically, not falsiness
effective_limit = limit if limit is not None else DEFAULT_LIMIT
```
- **DON'T:** Chain multiple `.get()` calls or attribute accesses on a deeply nested, possibly-partial structure without a plan for the intermediate `None`s (`data.get("user", {}).get("address", {}).get("zip")` works but silently returns `None` for any missing level, hiding which level was actually missing). For anything beyond one or two levels, validate the whole structure up front (Pydantic, an explicit shape check) rather than defensively chaining `.get()` calls deeper and deeper.

### Docstring Conventions in Practice

- **DO:** Pick one docstring convention (Google style, NumPy style, or reST/Sphinx style) project-wide and enforce it with a linter (`pydocstyle`, or Ruff's `D` rules configured for that convention) rather than letting each contributor's docstrings drift into whatever format they personally prefer.
```python
# Google style
def transfer_funds(from_account: str, to_account: str, amount: float) -> None:
    """Transfer funds between two accounts.

    Args:
        from_account: The source account ID.
        to_account: The destination account ID.
        amount: The amount to transfer, in the account's base currency.

    Raises:
        InsufficientFundsError: If from_account's balance is below amount.
    """

# NumPy style — equally valid, just a different convention; don't mix the two
def transfer_funds(from_account: str, to_account: str, amount: float) -> None:
    """
    Transfer funds between two accounts.

    Parameters
    ----------
    from_account : str
        The source account ID.
    to_account : str
        The destination account ID.
    amount : float
        The amount to transfer, in the account's base currency.
    """
```
- **DON'T:** Let a docstring's documented behavior silently drift out of sync with the function's actual implementation after an edit. A wrong docstring is worse than no docstring — it actively misleads the next reader instead of leaving them to check the code. Update the docstring in the same change that changes the behavior it describes.
- **DO:** Document raised exceptions in a function's docstring when they're part of its intended contract (not every possible exception, but the ones a caller should reasonably anticipate and might want to catch).

### Magic Numbers & Literals

- **DO:** Extract a bare numeric or string literal into a named constant when its meaning isn't self-evident from context, especially if the same value appears more than once. A named constant documents intent and gives future edits a single place to change it.
```python
# Bad — what is 86400? what is 3?
if (datetime.now() - last_login).total_seconds() > 86400 and failed_attempts >= 3:
    lock_account(user)

# Good
SECONDS_PER_DAY = 86400
MAX_FAILED_LOGIN_ATTEMPTS = 3

if (datetime.now() - last_login).total_seconds() > SECONDS_PER_DAY and failed_attempts >= MAX_FAILED_LOGIN_ATTEMPTS:
    lock_account(user)
```
- **DON'T:** Extract every literal into a constant reflexively, including ones whose meaning is already obvious in context (`page + 1` doesn't need `INCREMENT = 1`). Reserve named constants for values that are either non-obvious, reused, or likely to change together.

### Line Continuation & Multi-line Constructs

- **DO:** Rely on Python's implicit line continuation inside parentheses, brackets, and braces for splitting a long expression across multiple lines, rather than the explicit backslash (`\`) continuation character — implicit continuation is visually cleaner and doesn't break if a trailing space sneaks in after the backslash (which silently causes a syntax error).
```python
# Bad — backslash continuation is fragile and visually noisy
total = first_value + \
        second_value + \
        third_value

# Good — implicit continuation inside parentheses
total = (
    first_value
    + second_value
    + third_value
)
```
- **DO:** Break a long function call's arguments one-per-line (with a trailing comma) once it no longer fits comfortably on one line, rather than letting the formatter or the author wrap it awkwardly mid-argument — this is exactly the kind of decision to delegate to an automatic formatter (Black/`ruff format`) rather than deciding by hand each time.
- **DON'T:** Use a backslash continuation to keep an `if`/`while` condition on one logical line when parenthesizing the condition would allow the same implicit continuation more safely and readably.

## Type Hints

- **DO:** Annotate the public signature of every function and method — parameters and return type — even in application code, not just libraries. Type hints double as documentation and let static checkers (mypy, pyright) and editors catch mismatches before runtime.
```python
# Bad
def calculate_discount(price, percent):
    return price * (1 - percent / 100)

# Good
def calculate_discount(price: float, percent: float) -> float:
    return price * (1 - percent / 100)
```
- **DO:** Prefer the modern union syntax `X | None` and `int | str` (Python 3.10+) over `Optional[X]` and `Union[int, str]` in new code targeting 3.10+; for code that must support 3.9 or earlier, either use `Optional`/`Union` from `typing` or add `from __future__ import annotations` to opt into the newer syntax at parse time while keeping runtime compatibility.
```python
# Python 3.10+
def find_user(user_id: int) -> User | None: ...

# Python 3.7–3.9 compatible, still with modern syntax
from __future__ import annotations
def find_user(user_id: int) -> User | None: ...
```
- **DON'T:** Reach for `typing.Any` as a default escape hatch when a signature is hard to type. `Any` disables type checking for that value entirely and silently propagates — prefer a precise type, a `TypeVar`, a `Protocol`, or, as a last resort, a narrower `object` that forces callers to check before use.
- **DO:** Model structured dict-shaped data with `TypedDict` instead of a bare `dict[str, Any]`, so key names and per-key value types are checked statically.
```python
# Bad
def build_response(data: dict) -> dict:
    return {"status": data["status"], "code": data["code"]}

# Good
class ApiResponse(TypedDict):
    status: str
    code: int

def build_response(data: ApiResponse) -> ApiResponse:
    return {"status": data["status"], "code": data["code"]}
```
- **DO:** Use `typing.Protocol` for structural typing when you care about an object's shape (methods it has) rather than its concrete class — this is Python's equivalent of duck-typed interfaces and works well for dependency injection without forcing inheritance.
- **DO:** Use `Literal` to constrain a parameter to a specific, closed set of string/int values instead of typing it as a bare `str`, so a typo like `"puase"` is caught statically instead of failing at runtime.
```python
# Bad
def set_status(status: str) -> None: ...

# Good
def set_status(status: Literal["pending", "active", "paused", "closed"]) -> None: ...
```
- **DON'T:** Write type hints that lie about what a function actually does — e.g., annotating a return as `list[User]` when the function can return `None` on a miss, or annotating a parameter as `str` when `None` is an accepted sentinel. A type checker that trusts an inaccurate hint gives false confidence, which is worse than no hint at all.
- **DO:** Run mypy or pyright in CI, ideally in a reasonably strict mode (`--strict` or `strict = true` for mypy; `typeCheckingMode = "strict"` or `"standard"` for pyright), so type errors are caught before merge rather than discovered as production bugs.
- **DON'T:** Suppress a type error with a bare `# type: ignore` and no explanation. Use a targeted `# type: ignore[error-code]` and, when the reason isn't obvious from context, a short trailing comment — an unexplained suppression is indistinguishable from "the author didn't understand the error" six months later.
```python
# Bad
result = legacy_api(payload)  # type: ignore

# Good
result = legacy_api(payload)  # type: ignore[no-untyped-call]  # legacy_api has no stubs yet
```
- **DO:** Guard imports that exist only for type-checking (to avoid circular imports or runtime dependency on a type-only package) behind `if TYPE_CHECKING:`, and use a string forward reference or `from __future__ import annotations` so the annotation still resolves for the checker without executing at runtime.
```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from myapp.models import Order  # only needed for the type checker

def process(order: "Order") -> None: ...
```
- **DO:** Accept the most general type a function actually needs for parameters (e.g., `Iterable[int]` rather than `list[int]` if you only iterate once) and return the most specific type callers can rely on (e.g., `list[int]` rather than `Iterable[int]` if you materialize a list). This is the "be liberal in what you accept, precise in what you return" principle applied to typing.
- **DO:** Use `cast()` only when you have information the type checker cannot infer (e.g., after a runtime `isinstance` check the checker can't follow), and treat every `cast()` as a spot to double-check rather than a routine tool.
- **DO:** Use `Self` (Python 3.11+, or `typing_extensions.Self` earlier) as the return type for methods that return an instance of the calling class, especially in builder patterns or `classmethod` constructors, so subclasses inherit the correct return type automatically.
```python
class QueryBuilder:
    def where(self, clause: str) -> Self:
        self._clauses.append(clause)
        return self
```
- **DON'T:** Annotate every single local variable inside a function body when the type is already obvious from the right-hand side (`count: int = 0`). Let inference do that work; reserve explicit local annotations for cases where inference genuinely can't determine the type (e.g., an empty collection literal that needs a hint).
- **DO:** Use `@overload` from `typing` to precisely describe a function whose return type depends on an input's literal value or type (e.g., a `get()` that returns `T` when a default is given and `T | None` otherwise), instead of collapsing everything into one imprecise union signature.
- **DO:** Reach for `TypeVar` and `Generic` when writing a container or utility function whose input and output types should be linked (e.g., a `first(items: Iterable[T]) -> T`), so callers get precise types back instead of `Any`.
```python
T = TypeVar("T")

def first(items: Iterable[T]) -> T | None:
    return next(iter(items), None)
```
- **DON'T:** Treat `NewType` and a plain type alias as interchangeable. A type alias (`UserId = int`) is just a name for the same type and doesn't stop you from passing a raw `int` where a `UserId` is expected; `NewType("UserId", int)` creates a distinct type the checker enforces, which is what you want when mixing up two `int`-based IDs (e.g., `UserId` vs `OrderId`) would be a real bug.
- **DO:** Keep `mypy`/`pyright` configuration in `pyproject.toml` (or a dedicated config file) checked into the repo, so type-checking behavior is identical for every contributor and in CI — not dependent on each developer's local IDE defaults.
- **DON'T:** Ignore a growing pile of type errors by excluding whole modules from the checker's scope indefinitely. If a module can't be fully typed yet, track it explicitly (a short-lived `exclude` entry with a linked issue) rather than letting the exclusion list become permanent and unbounded.
- **DO:** Use `Final` to mark a name that should never be reassigned (module-level constants, or an attribute set once in `__init__`), so both the type checker and human readers know it's immutable by contract.
```python
MAX_CONNECTIONS: Final = 100
```
- **DO:** Type dataclass and `NamedTuple` fields explicitly — the type annotation is what makes `@dataclass` generate `__init__` correctly, not just documentation, so leaving it off is a functional bug, not a style nit.
- **DON'T:** Assume a third-party package is fully typed just because `import` succeeds without error. Check whether it ships inline types (`py.typed` marker) or needs a separate `types-<package>` stub package from `typeshed`; otherwise its calls will silently type-check as `Any` and hide real bugs.
- **DO:** Write a local `.pyi` stub file (or inline `Protocol`) for a third-party dependency that has neither inline types nor a `typeshed` stub package available, when that dependency is used extensively enough in typed code for the missing coverage to be a real gap — a minimal stub covering just the functions/classes actually used is enough, and doesn't require stubbing the library's entire surface.
```python
# stubs/untyped_lib.pyi — a minimal stub covering only what this project actually uses
def fetch(url: str, timeout: float = ...) -> bytes: ...

class Client:
    def __init__(self, api_key: str) -> None: ...
    def get(self, path: str) -> dict[str, object]: ...
```
- **DON'T:** Let generated or hand-written stub files silently drift out of sync with the real library's actual signatures after a dependency upgrade — a stub that lies about a function's real signature is worse than no stub, since it actively tells the type checker something false and suppresses the errors that would otherwise have caught a real incompatibility.
- **DO:** Use `assert_type()` or `reveal_type()` (mypy/pyright debugging aids, stripped or ignored at runtime as appropriate) when you're unsure what a checker actually infers for an expression, instead of guessing from documentation alone.
- **DO:** Treat a passing type checker as a floor, not a ceiling — types catch shape mismatches, not logic errors (a function can be perfectly well-typed and still compute the wrong answer). Keep tests for correctness; keep types for structural safety.
- **DO:** Reach for a runtime type-checking/validation library (`typeguard`, `beartype`, or Pydantic's `validate_call`) at a specific, deliberate boundary when static checking alone isn't enough — for example, a plugin system loading user-supplied callables, or a public library function whose callers might not be type-checking their own code — rather than assuming static hints alone protect a boundary that untyped or unchecked callers can reach.
```python
from beartype import beartype

@beartype
def calculate_discount(price: float, percent: float) -> float:
    return price * (1 - percent / 100)

# A caller passing a string still raises immediately, with a clear message,
# instead of producing a confusing TypeError deep inside the function body
calculate_discount("100", 10)  # beartype raises immediately at the boundary
```
- **DON'T:** Apply runtime type-checking decorators to every function throughout an entire codebase by default — they add real per-call overhead, and for internal code already covered by a static type checker and a decent test suite, the additional runtime check is often redundant. Reserve it for genuine trust boundaries.

### Static Types vs. Runtime Validation

- **DO:** Understand that standard type hints are erased at runtime and checked only by external tools (mypy, pyright) — Python itself does not enforce them. A function annotated `def f(x: int)` will happily execute if called with a string; nothing raises automatically.
```python
def add(a: int, b: int) -> int:
    return a + b

add("1", "2")  # runs fine at runtime, returns "12" — no TypeError, hints are not enforced
```
- **DO:** Use a runtime-validating library (Pydantic, `attrs` with validators, or manual checks) at the boundary where external, untrusted, or serialized data enters your program (an HTTP request body, a config file, a CLI argument, a queue message) — this is precisely the boundary where static type hints provide no actual protection, because the data's shape isn't known until runtime.
```python
from pydantic import BaseModel, Field

class CreateOrderRequest(BaseModel):
    user_id: int
    items: list[str] = Field(min_length=1)
    discount_percent: float = Field(ge=0, le=100)

# Raises a clear ValidationError if the incoming JSON doesn't match — instead of
# a confusing downstream KeyError/TypeError several function calls later
request = CreateOrderRequest.model_validate(raw_json)
```
- **DON'T:** Assume static type hints alone are sufficient input validation for a public API endpoint or CLI. A type checker verifies the code *you wrote* is internally consistent; it says nothing about what a real caller actually sends at runtime.
- **DO:** Keep the two concerns distinct in your head: static types are a development-time contract between your own functions, checked before the code ever runs; runtime validation is a production-time guard against data you don't control. Most non-trivial applications need both, not one instead of the other.

### Generics & `collections.abc`

- **DO:** Use the built-in generic collection types (`list[int]`, `dict[str, int]`, `set[str]`) directly as annotations on Python 3.9+, rather than importing `List`, `Dict`, `Set` from `typing` — the `typing` aliases are legacy, kept only for compatibility with pre-3.9 code.
```python
# Legacy — required before Python 3.9, unnecessary now for a 3.9+ target
from typing import List, Dict, Set

def dedupe(items: List[int]) -> Set[int]:
    return set(items)

# Current
def dedupe(items: list[int]) -> set[int]:
    return set(items)
```
```python
# Legacy (pre-3.9 compatibility only)
from typing import List, Dict
def process(items: List[int]) -> Dict[str, int]: ...

# Current
def process(items: list[int]) -> dict[str, int]: ...
```
- **DO:** Annotate function parameters with the most abstract type from `collections.abc` that the function actually needs (`Iterable`, `Sequence`, `Mapping`, `Callable`) rather than a concrete type like `list` or `dict`, when the function doesn't rely on concrete-type-specific behavior. This lets callers pass any compatible object — a generator, a tuple, a custom mapping — without an unnecessary conversion.
```python
# Overly restrictive — forces callers to materialize a list even if they have a generator
def total(values: list[float]) -> float:
    return sum(values)

# More flexible — accepts anything iterable
from collections.abc import Iterable
def total(values: Iterable[float]) -> float:
    return sum(values)
```

### Protocols & Callable Types

- **DO:** Annotate a parameter that accepts "anything callable with this signature" using `Callable[[ArgType, ...], ReturnType]` (or `collections.abc.Callable`), rather than a vague `Callable` with no signature or an untyped `object`, so the checker verifies callers actually pass a compatible function.
```python
from collections.abc import Callable

def apply_twice(func: Callable[[int], int], value: int) -> int:
    return func(func(value))
```
- **DO:** Define a `Protocol` when multiple unrelated classes across the codebase share a method signature you want to depend on structurally, instead of forcing them into a common base class purely to satisfy a type hint. This is especially useful for dependency injection in tests, where a `Protocol`-typed parameter accepts both the real implementation and a hand-written or mocked test double without any inheritance relationship.
```python
from typing import Protocol

class SupportsSave(Protocol):
    def save(self, path: str) -> None: ...

def export_all(items: list[SupportsSave], path: str) -> None:
    for item in items:
        item.save(path)

# Both a real Document class and an unrelated, ad hoc test double satisfy
# SupportsSave structurally — neither needs to inherit from it explicitly.
```
- **DON'T:** Reach for `ABC`/`abstractmethod`-based abstract base classes purely to get a type-checkable interface when structural typing (`Protocol`) would do the job with less coupling — reserve `ABC` for cases where you also want the runtime enforcement of "this class cannot be instantiated until it implements these methods," not just static shape-checking.

### Common Typing Mistakes

- **DON'T:** Assume a mutable generic container is safely "covariant" — a `list[Dog]` is not type-compatible with a parameter typed `list[Animal]`, even though `Dog` is an `Animal` subtype, because a function receiving `list[Animal]` could legally insert a `Cat` into what the caller thinks is still a `list[Dog]`. A type checker correctly rejects this; if the assignment is genuinely intended to be read-only, type the parameter as the covariant `Sequence[Animal]` instead.
```python
class Animal: ...
class Dog(Animal): ...
class Cat(Animal): ...

def add_cat(animals: list[Animal]) -> None:
    animals.append(Cat())

dogs: list[Dog] = [Dog()]
add_cat(dogs)  # a type checker flags this — and rightly so; it would corrupt the list

# If add_cat only needs to *read*, accept the covariant Sequence instead:
def count_animals(animals: Sequence[Animal]) -> int:
    return len(animals)  # safe to pass list[Dog] here
```
- **DON'T:** Widen a function's parameter type to `Any` just to make a type-checker error disappear without understanding why it fired. That silences the specific error but also disables checking for every future call to that function — investigate the actual mismatch (often a genuinely missing case, not a false positive) before reaching for `Any`.
- **DON'T:** Annotate a function's return type as a union that's wider than what it can actually return (e.g., `-> int | str | None` when it only ever returns `int | None`), just to "be safe." An overly wide return type forces every caller to handle cases that can't actually happen, which either produces dead code or unjustified `# type: ignore`s downstream.
- **DO:** Re-run the type checker after any change to a function's signature, not just after changes to its body — a narrowed or widened parameter/return type can silently break callers elsewhere in the codebase that a test suite alone might not catch if those call sites aren't exercised by existing tests.

### Typed Configuration & Settings

- **DO:** Model application configuration as a typed object (a Pydantic `BaseSettings` class, or a typed dataclass populated from environment variables/a config file at startup) instead of passing around a loosely-typed `dict` or reading `os.environ[...]` scattered throughout the codebase. A typed settings object validates once at startup — failing fast with a clear error — rather than failing unpredictably deep in whatever code path first happens to read a missing or malformed setting.
```python
# Bad — each module reads os.environ directly, with no central validation
def connect_to_db():
    host = os.environ["DB_HOST"]  # KeyError deep inside connect_to_db if unset
    port = int(os.environ["DB_PORT"])  # ValueError if not numeric
    ...

# Good — one place validates all configuration, at startup, with clear errors
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    db_host: str
    db_port: int = 5432
    debug: bool = False

settings = Settings()  # raises a clear, complete validation error if anything is missing/malformed
```
- **DON'T:** Read the same environment variable in multiple different places across the codebase, each with its own default value or type-coercion logic, which can drift out of sync (one call site defaulting to `False`, another to `True`, for the same setting). Read each configuration value in exactly one place — the settings object's definition — and pass the resolved, typed value from there.
- **DO:** Fail fast at application startup on invalid or missing required configuration, rather than allowing the application to start and only discovering the misconfiguration when the relevant code path is first hit, possibly in production under real user traffic.

### Type Aliases for Readability

- **DO:** Introduce a type alias (`type UserId = int` on 3.12+, or `UserId: TypeAlias = int` earlier) for a complex or repeated type expression, so a long generic signature reads as a domain concept instead of a wall of brackets repeated at every use site.
```python
# Without an alias — repeated, dense, and easy to get subtly wrong when copy-pasted
def merge_results(a: dict[str, list[tuple[int, float]]], b: dict[str, list[tuple[int, float]]]) -> dict[str, list[tuple[int, float]]]:
    ...

# With an alias — the signature reads as domain concepts
ScoresByCategory: TypeAlias = dict[str, list[tuple[int, float]]]

def merge_results(a: ScoresByCategory, b: ScoresByCategory) -> ScoresByCategory:
    ...
```
- **DON'T:** Introduce a type alias purely to shorten a name that was already short and clear (`UserList = list[User]` adds a layer of indirection over a type that was already perfectly readable as `list[User]`). Reserve aliases for expressions that are genuinely long, generic, or repeated enough to benefit from a name.
- **DO:** Prefer `NewType` over a plain type alias specifically when you want the checker to enforce that two structurally-identical types (like two different `int`-based IDs) aren't accidentally interchanged — covered earlier under Type Hints, but worth remembering that a plain alias and `NewType` solve different problems and aren't interchangeable despite looking similar.

## Idiomatic Python

### JSON & Serialization

- **DON'T:** Hand-write JSON serialization for a custom object by manually building a `dict` at every call site. Implement it once — a `to_dict()`/`from_dict()` pair, a Pydantic model's `.model_dump()`, or a `default=` function passed to `json.dumps()` for types `json` doesn't natively support (like `datetime` or `Decimal`) — so serialization logic exists in exactly one place.
```python
# Bad — ad hoc, repeated at every call site, easy to get inconsistent
data = {"id": order.id, "total": str(order.total), "created": order.created_at.isoformat()}
json.dumps(data)

# Good — serialization logic lives once, on the type itself
class Order(BaseModel):
    id: int
    total: Decimal
    created_at: datetime

json_str = order.model_dump_json()
```
- **DON'T:** Assume `json.dumps()` can serialize arbitrary Python objects by default — it natively handles only `dict`, `list`, `str`, numbers, `bool`, and `None`; anything else (a `datetime`, a `Decimal`, a custom class, a `set`) raises `TypeError` unless given an explicit `default=` encoder function or converted beforehand.
```python
# Bad — raises TypeError: Object of type datetime is not JSON serializable
json.dumps({"created_at": datetime.now(timezone.utc)})

# Good
json.dumps({"created_at": datetime.now(timezone.utc)}, default=str)
```
- **DO:** Validate deserialized JSON against an explicit schema immediately after parsing (covered under Security's Deserialization section) rather than accessing keys directly off the raw parsed `dict` throughout the rest of the code — this both documents the expected shape in one place and fails with one clear error instead of a `KeyError` from whichever line happens to access a missing field first.

### Working with Text Encoding

- **DO:** Specify `encoding="utf-8"` explicitly on every `open()` call that reads or writes text, rather than relying on the platform's default encoding — already covered under Packaging's cross-platform section, but worth restating here as a general text-handling habit, not just a portability concern: an explicit encoding also documents intent for whoever reads the code next.
- **DON'T:** Decode bytes from an external source (a file upload, a network response) without handling the possibility that the declared or assumed encoding is wrong — a `UnicodeDecodeError` raised deep in a data pipeline, far from where the bytes were actually read, is a common and avoidable source of confusing failures. Validate/handle the encoding at the point of ingestion, with an explicit `errors=` strategy (`"strict"`, `"replace"`, `"ignore"`) chosen deliberately rather than left to the default.

### Guiding Principles Behind These Rules

- **DO:** Use the long-standing aphorisms collected in PEP 20 ("the Zen of Python") as a genuine decision-making tool, not decoration — when torn between two equally-working implementations, the one that's more explicit, flatter, and more readable is very often the better one, and that instinct is exactly what most of the specific rules in this section are operationalizing.
- **DO:** Treat "there should be one obvious way to do it" as a reason to follow the codebase's existing convention for a solved problem (how errors are raised, how config is read, how tests are structured) rather than introducing a second, equally-valid-in-isolation way to do the same thing — consistency within a codebase usually matters more than which of two reasonable approaches is marginally better in isolation.
- **DON'T:** Use "practicality beats purity" as a justification for skipping error handling, tests, or types on code that will actually be maintained — that principle is about pragmatic trade-offs in genuinely ambiguous design decisions, not a license to skip fundamentals that were never actually in tension with getting the immediate task done.
- **DO:** Prefer a solution that's simple and slightly less clever over one that's dense and impressive-looking but harder for the next reader to verify — code is read far more often than it's written, and "explicit is better than implicit" is as much a readability principle for the next maintainer as it is a style preference.

### Comprehensions & Generator Expressions

- **DO:** Use a list/set/dict comprehension in place of a manual `for` loop that only appends/collects into an empty collection. It's shorter, and Python can often optimize the comprehension form better than an equivalent append loop.
```python
# Bad
squares = []
for n in range(10):
    squares.append(n * n)

# Good
squares = [n * n for n in range(10)]
```
- **DON'T:** Nest more than two levels of comprehension, or mix multiple `if` clauses and `for` clauses until the line requires re-reading twice to parse. Past that complexity, a regular loop with named intermediate variables is more readable, not less "Pythonic."
```python
# Bad — hard to parse at a glance
pairs = [(x, y) for x in range(10) if x % 2 == 0 for y in range(10) if y % 2 != 0 if x != y]

# Good
pairs = []
for x in range(10):
    if x % 2 != 0:
        continue
    for y in range(10):
        if y % 2 == 0 or x == y:
            continue
        pairs.append((x, y))
```
- **DO:** Use a generator expression (`(x for x in ...)`) instead of a list comprehension when the result is only iterated once and doesn't need to be indexed, reversed, or measured with `len()`. It avoids materializing the whole collection in memory.
```python
# Bad — builds a full list just to sum it
total = sum([process(x) for x in huge_dataset])

# Good — streams one item at a time
total = sum(process(x) for x in huge_dataset)
```
- **DO:** Use a dict or set comprehension directly rather than building a list of tuples and calling `dict()`/`set()` on it.
```python
# Bad
pairs = [(user.id, user.name) for user in users]
by_id = dict(pairs)

# Good
by_id = {user.id: user.name for user in users}
```
- **DON'T:** Use a comprehension purely for its side effects (calling a function per item and discarding the result), e.g. `[print(x) for x in items]`. Comprehensions communicate "I'm building a collection"; a `for` loop communicates "I'm doing something for each item." Using the wrong one for the wrong reason misleads the reader and wastes memory building a throwaway list.
- **DO:** Prefer `any()` and `all()` with a generator expression over manually looping with a flag variable to detect whether some/every element matches a predicate.
```python
# Bad
found = False
for user in users:
    if user.is_admin:
        found = True
        break

# Good
found = any(user.is_admin for user in users)
```

### Context Managers

- **DO:** Use a `with` statement for any resource that must be released deterministically — files, locks, database connections/transactions, network sockets. It guarantees cleanup runs even when an exception is raised, which manual `try`/`finally` code is easy to get wrong.
```python
# Bad
f = open("data.txt")
data = f.read()
f.close()  # never runs if f.read() raises

# Good
with open("data.txt") as f:
    data = f.read()
```
- **DON'T:** Call `.close()` manually "for safety" on top of a `with` block, and don't rely on garbage collection (`__del__`) to release resources — CPython's refcounting makes it work often enough to hide the bug, but it isn't guaranteed, especially on PyPy or with reference cycles.
- **DO:** Write a custom context manager with `contextlib.contextmanager` for setup/teardown pairs that recur in your codebase (e.g., "start a timer and log duration," "begin and commit/rollback a transaction"), instead of copy-pasting the same `try`/`finally` block everywhere.
```python
from contextlib import contextmanager
import time

@contextmanager
def timed(label: str):
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        logger.info("%s took %.3fs", label, elapsed)

with timed("bulk_import"):
    run_bulk_import()
```
- **DO:** Use `contextlib.ExitStack` when the number of context managers to enter isn't known until runtime (e.g., opening a variable number of files), instead of hand-rolling nested `try`/`finally` blocks.
- **DO:** Implement `__enter__`/`__exit__` directly on a class-based context manager when it needs to hold state across calls or be reused as an object, rather than forcing everything through the generator-based `@contextmanager` decorator.
- **DON'T:** Swallow exceptions inside `__exit__` by returning a truthy value unless that's genuinely the contract you intend to offer (e.g., a "suppress and log" context manager, clearly named as such). Silently returning `True` from `__exit__` makes an exception vanish with no trace, which is a difficult bug to track down.
- **DO:** Use `contextlib.suppress(SomeError)` instead of an empty `try/except: pass` block when you deliberately want to ignore a *specific* exception type — it says the same thing in one line and can't accidentally swallow unrelated errors.
```python
# Bad
try:
    os.remove(temp_path)
except OSError:
    pass

# Good
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove(temp_path)
```

### Generators & Iterators

- **DO:** Write a generator function (using `yield`) when producing a sequence of values lazily, especially over large or unbounded data, instead of building and returning a full list. Callers who only need the first few items, or who want to pipe it into another lazy operation, benefit directly.
```python
# Bad — reads the whole file into memory
def read_lines(path):
    with open(path) as f:
        return f.readlines()

# Good — yields one line at a time
def read_lines(path):
    with open(path) as f:
        for line in f:
            yield line.rstrip("\n")
```
- **DO:** Implement `__iter__`/`__next__` (or simply a generator method) on custom container classes that represent a sequence, so they work naturally with `for`, comprehensions, `sum()`, `any()`, and other iterable-consuming builtins instead of exposing an ad hoc `.get_all_items()` method.
- **DON'T:** Iterate a generator twice expecting it to reset — a generator is exhausted after one full pass. If you need to iterate multiple times, materialize it into a list, or write a class implementing `__iter__` that returns a fresh generator each time.
```python
# Bad — second loop sees nothing, the generator is exhausted
results = (compute(x) for x in data)
total = sum(results)
maximum = max(results)  # ValueError: max() arg is an empty sequence

# Good
results = [compute(x) for x in data]
total = sum(results)
maximum = max(results)
```
- **DO:** Use `itertools` (`chain`, `islice`, `groupby`, `takewhile`, `pairwise`, etc.) for common lazy-iteration patterns instead of reimplementing them by hand — they're implemented in C, well-tested, and communicate intent immediately to a reader who knows the module.
- **DO:** Close generators that hold open resources (files, connections) properly — prefer wrapping the resource acquisition inside the generator with a `with` block so `GeneratorExit` triggers cleanup, rather than expecting callers to remember to call `.close()`.

### Dataclasses vs. Plain Classes vs. Named Tuples

- **DO:** Use `@dataclass` for classes whose primary purpose is to hold data, especially when you'd otherwise hand-write `__init__`, `__repr__`, and `__eq__`. It removes boilerplate and keeps the field list as the single source of truth.
```python
# Bad — hand-rolled boilerplate
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __repr__(self):
        return f"Point(x={self.x}, y={self.y})"
    def __eq__(self, other):
        return isinstance(other, Point) and self.x == other.x and self.y == other.y

# Good
@dataclass
class Point:
    x: float
    y: float
```
- **DO:** Set `frozen=True` on a dataclass that represents an immutable value (a coordinate, a money amount, a config snapshot). Immutability prevents an entire class of bugs where shared instances are mutated unexpectedly, and makes the instance hashable so it can go in a `set` or be a `dict` key.
- **DO:** Use `field(default_factory=list)` (or `dict`, `set`, or a lambda) for any dataclass field whose default is a mutable object — the dataclass decorator raises a `ValueError` at class-definition time if you try a bare mutable literal as a default, precisely to stop the classic shared-mutable-default bug before it happens.
```python
@dataclass
class Order:
    items: list[str] = field(default_factory=list)  # not items: list = []
```
- **DON'T:** Reach for `@dataclass` when the class has significant behavior, invariants to enforce on construction, or needs to hide its internal representation. A dataclass's auto-generated `__init__` accepts whatever the field types allow; if you need validation or encapsulation, either add `__post_init__` validation, use a regular class, or (for stricter validation and serialization) reach for `pydantic`.
- **DO:** Use `NamedTuple` (typed, via `typing.NamedTuple`) for small, truly immutable, tuple-like records that also need to support positional unpacking or tuple-equality with plain tuples — e.g., a coordinate returned from a function that callers might destructure as `x, y = get_position()`.
- **DON'T:** Use a plain, untyped `tuple` or a raw `dict` as an ad hoc "struct" when its fields have fixed meaning (e.g., returning `(name, age, email)` from a function). Callers have to remember field order and the type checker can't help — use a dataclass, `NamedTuple`, or `TypedDict` instead so field access is named and checked.
```python
# Bad — what do these positions mean?
def get_user_info(user_id: int) -> tuple:
    return (user.name, user.age, user.email)

name, age, email = get_user_info(42)

# Good
@dataclass
class UserInfo:
    name: str
    age: int
    email: str

def get_user_info(user_id: int) -> UserInfo: ...
```
- **DO:** Use `__slots__` (or `@dataclass(slots=True)` on 3.10+) for high-volume value objects where memory footprint or attribute-access speed matters — it prevents an arbitrary `__dict__` per instance and catches accidental typo'd attribute assignment as an `AttributeError`.
- **DON'T:** Add mutable default state to a class body expecting each instance to get its own copy — a mutable class attribute is shared by every instance unless explicitly reassigned in `__init__`.
```python
# Bad — every Cart instance shares the same list
class Cart:
    items = []
    def add(self, item):
        self.items.append(item)

# Good
class Cart:
    def __init__(self):
        self.items = []
    def add(self, item):
        self.items.append(item)
```

### `pathlib` vs. `os.path`

- **DO:** Use `pathlib.Path` for filesystem path manipulation in new code instead of `os.path` string functions. `Path` objects are composable with `/`, carry methods for common operations (`.exists()`, `.read_text()`, `.glob()`), and eliminate a whole class of manual string-joining bugs.
```python
# Bad
import os
config_path = os.path.join(os.path.dirname(__file__), "config", "settings.json")
with open(config_path) as f:
    data = f.read()

# Good
from pathlib import Path
config_path = Path(__file__).parent / "config" / "settings.json"
data = config_path.read_text()
```
- **DON'T:** Manually concatenate path components with string `+` or f-strings (`base_dir + "/" + filename`). This breaks on Windows (backslash separator), mishandles trailing slashes, and doesn't normalize `..`/`.` segments the way `Path` or `os.path.join` do.
- **DO:** Use `Path.glob()` / `Path.rglob()` for pattern-based file discovery instead of manually walking directories with `os.listdir()` and filtering by extension in a loop.
- **DO:** Use `Path.read_text()` / `Path.write_text()` / `Path.read_bytes()` / `Path.write_bytes()` for simple whole-file I/O instead of the `open(...) as f: ...` dance when you don't need streaming or fine-grained control — it's shorter and still resource-safe (the file handle is opened and closed internally).
- **DO:** Convert a `Path` to `str` explicitly (`str(path)`) only at the boundary where an API genuinely requires a string (some older third-party libraries, some `subprocess` argument lists) — most standard library and modern third-party APIs accept `PathLike` objects directly.
- **DON'T:** Mix `os.path` and `pathlib` idioms within the same function or module. Pick `pathlib` for new code and, when touching legacy `os.path` code substantially, migrate the touched section rather than layering a second style on top.
- **DO:** Use `Path.mkdir(parents=True, exist_ok=True)` to create a directory (and any missing parent directories) idempotently, instead of manually checking `if not path.exists(): path.mkdir()` — the manual check has a race condition (another process could create the directory between the check and the `mkdir` call) that `exist_ok=True` avoids entirely.
```python
# Bad — check-then-act race condition, and doesn't create missing parent dirs
if not output_dir.exists():
    output_dir.mkdir()

# Good — atomic with respect to "already exists", and creates parents as needed
output_dir.mkdir(parents=True, exist_ok=True)
```
- **DO:** Use `Path.with_suffix()`, `Path.with_name()`, and `Path.stem` for deriving a related filename (changing an extension, swapping a base name while keeping the directory) instead of manually slicing or splitting the string representation of a path.
```python
# Bad — manual string manipulation, breaks on paths with multiple dots
output_path = str(input_path).rsplit(".", 1)[0] + ".json"

# Good
output_path = input_path.with_suffix(".json")
```

### Unpacking, Iteration Helpers, and Other Idioms

- **DO:** Use `enumerate()` when you need both the index and the value while iterating, instead of manually tracking a counter variable.
```python
# Bad
i = 0
for user in users:
    print(i, user.name)
    i += 1

# Good
for i, user in enumerate(users):
    print(i, user.name)
```
- **DO:** Use `zip()` to iterate multiple sequences in lockstep instead of indexing each one by a shared counter.
```python
# Bad
for i in range(len(names)):
    print(names[i], scores[i])

# Good
for name, score in zip(names, scores):
    print(name, score)
```
- **DO:** Use tuple/iterable unpacking (`a, b = b, a`; `first, *rest = items`; `x, y, z = point`) instead of manual indexing when destructuring known-shape sequences. It's both more readable and communicates the expected shape.
```python
# Manual indexing — easy to get an off-by-one wrong, and less self-documenting
first = items[0]
rest = items[1:]

# Unpacking — the shape is visible directly in the assignment
first, *rest = items

# Swapping without a temporary variable
a, b = b, a
```
- **DO:** Use `collections.Counter` for frequency counting instead of a hand-rolled `dict` with manual `if key not in counts: counts[key] = 0` bookkeeping.
```python
# Bad — manual bookkeeping, easy to get the initialization check wrong
counts = {}
for word in words:
    if word not in counts:
        counts[word] = 0
    counts[word] += 1

# Good — Counter also gives you .most_common(), arithmetic between counters, etc.
from collections import Counter
counts = Counter(words)
top_three = counts.most_common(3)
```
```python
# Bad
counts = {}
for word in words:
    if word not in counts:
        counts[word] = 0
    counts[word] += 1

# Good
from collections import Counter
counts = Counter(words)
```
- **DO:** Use `collections.defaultdict` when a dict's values are naturally an accumulating collection (list, set, int counter), instead of repeated `dict.setdefault()` calls or manual key-existence checks.
```python
# Bad — manual key-existence check before every append
groups = {}
for item in items:
    if item.category not in groups:
        groups[item.category] = []
    groups[item.category].append(item)

# Good
from collections import defaultdict
groups = defaultdict(list)
for item in items:
    groups[item.category].append(item)
```
- **DO:** Use `dict.get(key, default)` or `dict.setdefault(key, default)` instead of a `try: ... except KeyError:` or an `if key in d:` check, when a default value is all you need.
```python
# More verbose than necessary
try:
    timeout = config["timeout"]
except KeyError:
    timeout = 30

# Concise
timeout = config.get("timeout", 30)
```
- **DON'T:** Use `type(x) == SomeClass` for type checks — use `isinstance(x, SomeClass)`, which also respects subclassing (and accepts a tuple of types for an "is one of" check).
- **DO:** Use `functools.reduce` only when the accumulation genuinely doesn't fit a simpler built-in (`sum`, `any`, `all`, `max`, `min`) or a plain loop — `reduce` is general-purpose but often less readable than the specific tool that already exists for common reductions, and a plain `for` loop with an accumulator is frequently clearer than a `reduce` call for anything beyond a trivial one-liner.
```python
# Harder to read than it needs to be for something this common
from functools import reduce
total = reduce(lambda acc, x: acc + x, values, 0)

# Clearer — sum() already exists for exactly this
total = sum(values)

# reduce earns its place for a genuinely custom accumulation with no direct builtin
merged_config = reduce(lambda acc, layer: {**acc, **layer}, config_layers, {})
```
- **DO:** Use dictionary union operators (`|` and `|=`, Python 3.9+) for merging dicts, instead of `{**a, **b}` unpacking, when a new merged object is what's needed — it reads more directly as "combine these two mappings."
```python
# Still fine, but the | operator is more direct for this specific case
merged = {**defaults, **overrides}

# Equivalent, arguably more readable as "union these two mappings"
merged = defaults | overrides
```
- **DO:** Use `str.join()` to concatenate an iterable of strings, not a loop that repeatedly does `result += item`. String concatenation in a loop is O(n²) in CPython for immutable strings because each `+=` allocates a new string.
```python
# Bad — O(n^2), reallocates the whole string every iteration
result = ""
for word in words:
    result += word + " "

# Good — O(n)
result = " ".join(words)
```
- **DON'T:** Use `str.format()` with positional placeholders (`"{} {}".format(a, b)`) or bare `%s` formatting in new code where an f-string would be clearer — reserve `.format()` for the specific cases where it's still the right tool (a format string assembled dynamically from a template stored separately from the values, or a translatable string where the template and substitution happen at different points in the code).
```python
# An f-string can't be built from a template string loaded at runtime,
# since f-strings are evaluated at the point they're written — this is
# the case where .format() is still the correct choice, not a fallback
template = load_translated_string("greeting")  # e.g. "Hello, {name}!"
message = template.format(name=user.name)
```
- **DO:** Use `*args` and `**kwargs` in a function signature only when the function genuinely needs to accept a variable, unknown-in-advance number of positional or keyword arguments (a decorator wrapping an arbitrary function, a thin proxy forwarding to another API) — not as a default way to avoid writing out a fixed, known parameter list, which throws away the self-documentation and type-checking a real signature provides.
```python
# Bad — the signature says nothing about what this function actually needs
def create_user(*args, **kwargs):
    return User(name=kwargs["name"], email=kwargs["email"])

# Good — the real parameters are visible, documented, and type-checked
def create_user(name: str, email: str) -> User:
    return User(name=name, email=email)

# *args/**kwargs used for its actual purpose — forwarding to an unknown wrapped function
def log_calls(func):
    def wrapper(*args, **kwargs):
        logger.debug("calling %s", func.__name__)
        return func(*args, **kwargs)
    return wrapper
```
- **DO:** Use keyword-only arguments (parameters after a bare `*` in the signature) for boolean flags and any parameter where the call-site meaning wouldn't be obvious from a positional value alone — it forces callers to write `create_report(data, include_totals=True)` instead of the ambiguous `create_report(data, True)`.
```python
# Bad — what does the positional True mean at the call site without checking the signature?
def create_report(data, include_totals):
    ...
create_report(sales_data, True)

# Good — self-documenting at every call site
def create_report(data, *, include_totals: bool):
    ...
create_report(sales_data, include_totals=True)
```
- **DO:** Use multiple assignment / chained context (e.g., `a = b = compute()`) sparingly and only when all targets genuinely should reference the same object — it's a common source of the "why did mutating one variable change the other" bug when the value is mutable.
- **DON'T:** Use mutable objects (lists, dicts, sets) as default argument values in function signatures. Default argument values are evaluated exactly once, at function-definition time, and the same object is reused across every call that doesn't override it — leading to state silently leaking between unrelated calls.
```python
# Bad — the same list is reused and grows across every call
def add_item(item, cart=[]):
    cart.append(item)
    return cart

# Good
def add_item(item, cart=None):
    if cart is None:
        cart = []
    cart.append(item)
    return cart
```
- **DO:** Copy a list/dict/set explicitly (`list(x)`, `dict(x)`, `x.copy()`, or `copy.deepcopy(x)` for nested structures) when you intend to hand out an independent snapshot, rather than assuming assignment (`y = x`) makes a copy — in Python, assignment binds a new name to the same object.
- **DO:** Use structural pattern matching (`match`/`case`, Python 3.10+) for dispatching on the *shape* of a value (a tagged union of dataclasses, a nested dict/list pattern), where it reads more clearly than a chain of `isinstance`/`elif` checks. Don't force it onto a simple two- or three-way `if`/`elif` that reads fine as-is.
- **DON'T:** Use `assert` statements to validate user input, API request bodies, or any condition that must hold in production. Assertions are stripped entirely when Python is run with the `-O` optimization flag, so relying on them for validation means that validation silently disappears in an optimized deployment. Reserve `assert` for internal invariants and use explicit `if ...: raise ValueError(...)` for real validation.
```python
# Bad — vanishes under python -O
def withdraw(account, amount):
    assert amount <= account.balance, "insufficient funds"
    account.balance -= amount

# Good
def withdraw(account, amount):
    if amount > account.balance:
        raise InsufficientFundsError(f"cannot withdraw {amount}, balance is {account.balance}")
    account.balance -= amount
```

### Enums Instead of Magic Values

- **DO:** Use `enum.Enum` (or `enum.IntEnum`/`enum.StrEnum` on 3.11+) to represent a fixed, named set of related constants, instead of a collection of loose module-level string or integer literals. An enum groups the related values under one namespace, is exhaustively checkable by static analysis, and prevents passing an unrelated value where a specific member is expected.
```python
# Bad — loose string constants, no grouping, no protection against typos
STATUS_PENDING = "pending"
STATUS_ACTIVE = "active"
STATUS_CLOSED = "closed"

def set_status(status: str) -> None: ...
set_status("actve")  # typo — no error until it fails downstream

# Good
class OrderStatus(enum.Enum):
    PENDING = "pending"
    ACTIVE = "active"
    CLOSED = "closed"

def set_status(status: OrderStatus) -> None: ...
set_status(OrderStatus.ACTIVE)  # typo-proof; IDE autocompletes valid members
```
- **DON'T:** Use a plain integer or string as a stand-in for a fixed set of states/modes/categories when the codebase would benefit from the type safety, `match`-ability, and self-documentation an `Enum` provides — especially when the same "which values are valid" question keeps being answered by grepping for string literals scattered across the code.
- **DO:** Use `enum.Flag`/`enum.IntFlag` for a set of independent, combinable boolean options (permission bits, feature toggles that can be OR'd together), instead of a collection of separate boolean parameters or a raw bitmask of undocumented integer constants.
```python
class Permission(enum.Flag):
    READ = enum.auto()
    WRITE = enum.auto()
    DELETE = enum.auto()

user_permissions = Permission.READ | Permission.WRITE
if Permission.WRITE in user_permissions:
    allow_edit()
```
- **DO:** Add behavior to an `Enum` via methods when each member has associated logic (e.g., a `Color` enum with a `.to_hex()` method per member), keeping the "what are the valid values" and "what does each value do" concerns co-located instead of a separate `if/elif` chain keyed on the enum elsewhere in the code.
```python
# Scattered — the logic for each status lives far away from its definition
class OrderStatus(enum.Enum):
    PENDING = "pending"
    SHIPPED = "shipped"

def get_status_color(status):
    if status == OrderStatus.PENDING:
        return "yellow"
    elif status == OrderStatus.SHIPPED:
        return "green"

# Co-located — the enum is the single place that knows about its own members
class OrderStatus(enum.Enum):
    PENDING = "pending"
    SHIPPED = "shipped"

    @property
    def color(self) -> str:
        return {"pending": "yellow", "shipped": "green"}[self.value]
```

### Decorators & `functools`

- **DO:** Reach for `functools.wraps` inside any custom decorator that wraps a function, so the wrapped function retains its original `__name__`, `__doc__`, and signature for introspection, debugging, and tooling (including help(), tracebacks, and some static analyzers).
```python
# Bad — the decorated function's identity is lost; help(slow_query) shows "wrapper"
def log_calls(func):
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

# Good — functools.wraps preserves the original function's metadata
from functools import wraps
def log_calls(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper
```
- **DO:** Use `functools.singledispatch` for a function whose behavior should branch on the runtime type of its first argument, instead of a long `isinstance` `if/elif` chain — it's more extensible (new types can register a handler without editing the original function) and communicates the dispatch intent directly.
```python
from functools import singledispatch

@singledispatch
def serialize(value):
    raise TypeError(f"cannot serialize {type(value)}")

@serialize.register
def _(value: int) -> str:
    return str(value)

@serialize.register
def _(value: list) -> str:
    return "[" + ",".join(serialize(v) for v in value) + "]"
```
- **DO:** Use `functools.partial` to pre-bind some arguments of a function for reuse (e.g., as a callback, or to adapt a function's signature to an API that expects fewer arguments) instead of writing a throwaway `lambda` or a small wrapper function that just forwards arguments.
```python
# Bad — a trivial wrapper just to fix one argument
def make_adder(n):
    def add(x):
        return x + n
    return add

# Good
from functools import partial
add_five = partial(operator.add, 5)
```
- **DON'T:** Write a custom decorator for cross-cutting behavior (retrying, caching, timing, access control) that a well-maintained standard-library or third-party utility already provides (`functools.lru_cache`, `functools.cache`, `tenacity.retry`). Reserve a hand-written decorator for genuinely project-specific behavior.
```python
# A reasonable, genuinely project-specific decorator — not something
# a standard library utility already provides
def require_permission(permission: Permission):
    def decorator(func):
        @wraps(func)
        def wrapper(request, *args, **kwargs):
            if permission not in request.user.permissions:
                raise PermissionDeniedError(permission)
            return func(request, *args, **kwargs)
        return wrapper
    return decorator

@require_permission(Permission.DELETE)
def delete_order(request, order_id):
    ...
```
- **DO:** Keep decorators focused on one cross-cutting concern each, and stack multiple decorators rather than writing one that tries to do logging, caching, and retries all at once — each decorator should be independently understandable, testable, and reusable.

### Properties & Encapsulation

- **DO:** Use `@property` when an attribute needs computed, validated, or lazily-evaluated access, while still letting callers use plain attribute syntax (`obj.value`, not `obj.get_value()`). This gets you both Python's normal attribute-access idiom and the ability to add logic behind it later without changing the calling code.
```python
class Circle:
    def __init__(self, radius: float):
        self.radius = radius

    @property
    def area(self) -> float:
        return math.pi * self.radius ** 2

    @property
    def radius(self) -> float:
        return self._radius

    @radius.setter
    def radius(self, value: float) -> None:
        if value < 0:
            raise ValueError("radius cannot be negative")
        self._radius = value
```
- **DON'T:** Add a `@property` for every single attribute "just in case validation is needed later." Start with a plain attribute; convert it to a property only when actual computed or validated behavior is needed — Python's attribute access is the same syntax either way, so there's no upfront cost to deferring this decision, unlike languages that require getters/setters from the start.
```python
# Unnecessary ceremony — plain_attr has no validation or computation at all
class Point:
    def __init__(self, x):
        self._x = x
    @property
    def x(self):
        return self._x
    @x.setter
    def x(self, value):
        self._x = value  # no validation happening — this property adds nothing

# Start simple; add @property later, only if/when real logic is needed
class Point:
    def __init__(self, x):
        self.x = x
```
- **DO:** Use `functools.cached_property` for a computed attribute that's expensive to calculate but doesn't change after the first access, instead of manually memoizing it into a private attribute inside a regular `@property` getter.
```python
class Report:
    def __init__(self, rows):
        self.rows = rows

    @functools.cached_property
    def summary_stats(self) -> dict:
        # expensive aggregation, computed once and cached on the instance
        return compute_stats(self.rows)
```

### Structural Pattern Matching (`match`/`case`)

- **DO:** Use `match`/`case` (3.10+) when dispatching on the *shape* of a value — a tagged union of dataclasses, a nested list/dict structure, or a value that needs both a type check and a destructure in one step — where it reads far more directly than the equivalent chain of `isinstance` checks and manual attribute access.
```python
# Without match — verbose, and the structure of each case is hard to see at a glance
def handle_event(event):
    if isinstance(event, UserCreated):
        return f"welcome {event.name}"
    elif isinstance(event, UserDeleted):
        return f"goodbye user {event.user_id}"
    elif isinstance(event, OrderPlaced) and event.total > 1000:
        return f"large order: {event.order_id}"
    return "unhandled event"

# With match — the shape of each case, including guards, is visible directly
def handle_event(event):
    match event:
        case UserCreated(name=name):
            return f"welcome {name}"
        case UserDeleted(user_id=user_id):
            return f"goodbye user {user_id}"
        case OrderPlaced(order_id=order_id, total=total) if total > 1000:
            return f"large order: {order_id}"
        case _:
            return "unhandled event"
```
- **DON'T:** Force a simple two- or three-branch `if`/`elif` on a plain scalar value into `match`/`case` just because the syntax is newer — a plain `if x == "a":` reads at least as clearly for a small number of simple value comparisons, and `match` earns its keep specifically on structural/destructuring cases.
- **DO:** Use a wildcard `case _:` to handle the unmatched/default case explicitly, the same way a final `else` closes off an `if`/`elif` chain — an unhandled `match` with no default silently falls through without executing any branch, which can hide a missed case.

### Multiple Inheritance & Method Resolution Order

- **DO:** Prefer composition (an object holding a reference to another object and delegating to it) over multiple inheritance for combining independent pieces of behavior, unless there's a specific, well-understood reason multiple inheritance is the better fit (such as intentionally using cooperative mixins with `super()`). Multiple inheritance in Python resolves method calls through the C3 linearization (MRO), which is correct but easy to reason about incorrectly once more than two base classes with overlapping method names are involved.
```python
# Fragile — behavior of a shared method name depends on MRO, easy to get wrong
class Loggable:
    def save(self):
        print("logging save")

class Cacheable:
    def save(self):
        print("caching save")

class Document(Loggable, Cacheable):
    pass
# Document().save() -> only "logging save" prints; Cacheable.save() is shadowed
# unless every save() in the chain deliberately calls super().save()

# Clearer — composition makes the relationship and call order explicit
class Document:
    def __init__(self, logger, cache):
        self._logger = logger
        self._cache = cache
    def save(self):
        self._logger.log("saving")
        self._cache.store(self)
```
- **DO:** Call `super().__init__(**kwargs)` (and design mixins to cooperatively forward unused keyword arguments) when multiple inheritance with mixins genuinely is the right tool, so the whole MRO chain initializes correctly regardless of which concrete class ends up using a given mixin.
- **DON'T:** Assume `super()` always refers to "the immediate parent class as written in the class definition" — it actually refers to the next class in the MRO, which depends on the full inheritance graph of the *instance's* concrete class, not just the class where `super()` is called. This matters specifically in multiple-inheritance/mixin hierarchies.

### Copying: Shallow vs. Deep

- **DO:** Understand the difference between a shallow copy (`copy.copy()`, `list(x)`, slicing `x[:]`) and a deep copy (`copy.deepcopy()`) before choosing one — a shallow copy duplicates the outer container but still shares references to any mutable objects nested inside it, so mutating a nested object through the copy also mutates the original.
- **DO:** Implement `__copy__`/`__deepcopy__` on a custom class only when the default behavior (copying each attribute) is actually wrong for that class — most classes never need this; it's specifically for cases like a class wrapping a resource handle or maintaining an internal cache that shouldn't be blindly duplicated along with the rest of the object's state.
```python
class ConnectionPool:
    def __init__(self, size):
        self.size = size
        self._connections = [make_connection() for _ in range(size)]

    def __deepcopy__(self, memo):
        # a copy should get its own fresh connections, not share the originals
        new_pool = ConnectionPool(self.size)
        return new_pool
```
```python
import copy

original = {"tags": ["a", "b"]}
shallow = copy.copy(original)
shallow["tags"].append("c")
print(original["tags"])  # ['a', 'b', 'c'] — the nested list was shared, not copied!

deep = copy.deepcopy(original)
deep["tags"].append("d")
print(original["tags"])  # ['a', 'b', 'c'] — unaffected; deepcopy copied the nested list too
```
- **DON'T:** Reach for `copy.deepcopy()` by default "to be safe" on every copy, without considering whether a shallow copy (cheaper, and correct when nested objects are themselves immutable or intentionally shared) would do. `deepcopy` has real overhead, especially on large or deeply nested structures, and can fail or behave unexpectedly on objects holding non-copyable resources (open file handles, locks, network connections).

### Dates, Times, and Timezones

- **DON'T:** Create or compare "naive" `datetime` objects (no attached timezone) in any code that deals with more than one timezone, or that will ever run in a different timezone than it was written in. A naive `datetime` is ambiguous — `datetime(2026, 3, 1, 12, 0)` means a different real instant depending on which timezone it's implicitly assumed to be in, and comparing a naive datetime to an aware one raises `TypeError` at the worst possible moment.
```python
# Bad — naive datetime; ambiguous which timezone "now" means, and not comparable
# to any timezone-aware value without an explicit, easy-to-forget conversion
scheduled_at = datetime(2026, 3, 1, 12, 0)

# Good — explicit, unambiguous, and safely comparable to other aware datetimes
from datetime import datetime, timezone
scheduled_at = datetime(2026, 3, 1, 12, 0, tzinfo=timezone.utc)
```
- **DO:** Store and pass timestamps as timezone-aware UTC internally throughout a system (in the database, between services), and convert to a user's local timezone only at the final display/presentation layer — this avoids an entire class of off-by-some-hours bugs caused by timezone conversions happening inconsistently at different points in the system.
- **DON'T:** Use `datetime.utcnow()` in new code — it returns a *naive* datetime that represents UTC without saying so, which is exactly the ambiguous situation the previous point warns about, and it's deprecated as of Python 3.12 in favor of the explicit form. Use `datetime.now(timezone.utc)` instead, which is both aware and explicit.
```python
# Deprecated and ambiguous — naive, even though it's UTC
now = datetime.utcnow()

# Correct — aware and explicit about being UTC
now = datetime.now(timezone.utc)
```
- **DO:** Use the `zoneinfo` module (standard library, 3.9+) for named timezone handling (`ZoneInfo("America/New_York")`) rather than fixed UTC offsets, when daylight saving time or a specific region's rules matter — a fixed offset doesn't shift with DST the way a named zone correctly does.
- **DON'T:** Do date arithmetic by manually adding/subtracting a fixed number of seconds/days when calendar-aware arithmetic is actually needed (e.g., "one month later" isn't a fixed number of days, and adding 24 hours isn't always "the same time tomorrow" across a DST transition). Use `dateutil.relativedelta` or equivalent calendar-aware utilities for calendar-unit arithmetic, reserving `timedelta` for genuinely fixed-duration arithmetic.

### The `warnings` Module for Deprecations

- **DO:** Use `warnings.warn(..., DeprecationWarning)` to signal that a function/parameter is deprecated and will be removed in a future version, instead of only mentioning it in a docstring or changelog that callers may never read — a runtime warning surfaces at the point of actual use, where it's much more likely to be noticed by whoever needs to migrate.
```python
import warnings

def old_function(x):
    warnings.warn(
        "old_function() is deprecated and will be removed in v2.0; use new_function() instead",
        DeprecationWarning,
        stacklevel=2,
    )
    return new_function(x)
```
- **DO:** Pass `stacklevel=2` (or higher, depending on call depth) so the warning is attributed to the *caller's* line, not the line inside the deprecated function itself — without it, the warning points at code the caller can't do anything about, defeating the point of surfacing it at the call site.
- **DON'T:** Silently change or remove a public function's behavior without a preceding deprecation period (a version where the old behavior still works but warns) for anything with external consumers — an abrupt breaking change with no warning period is a common source of unnecessary downstream breakage.

### String Formatting & Numeric Precision

- **DO:** Use format specifiers within f-strings (`f"{value:.2f}"`, `f"{count:,}"`, `f"{ratio:.1%}"`) for controlled numeric presentation, instead of manually rounding and concatenating strings.
```python
# Bad — manual rounding and string-building
formatted = str(round(price, 2)) + " USD"

# Good
formatted = f"{price:.2f} USD"
formatted_count = f"{count:,}"       # 1234567 -> "1,234,567"
formatted_pct = f"{ratio:.1%}"       # 0.4567  -> "45.7%"
```
- **DON'T:** Use floating-point arithmetic for money or other values that require exact decimal precision — binary floating point cannot represent most decimal fractions exactly, and repeated arithmetic compounds the rounding error. Use `decimal.Decimal` (constructed from strings, not floats) for currency and other exact-decimal domains.
```python
# Bad — classic float imprecision: this prints 0.30000000000000004, not 0.3
total = 0.1 + 0.2

# Good
from decimal import Decimal
total = Decimal("0.1") + Decimal("0.2")  # exactly Decimal("0.3")
```
- **DO:** Compare floating-point values with a tolerance (`math.isclose(a, b)`) rather than direct equality (`a == b`), since floating-point arithmetic rarely produces bit-exact results across different computation paths that are mathematically equal.
- **DO:** Use `int`/`Decimal` for counting money in a smallest indivisible unit (cents, not dollars) in systems where financial correctness matters, avoiding both floating-point rounding error and the ambiguity of "how many decimal places does this currency use."

### Managing Multiple Context Managers

- **DO:** Use a single `with` statement with multiple comma-separated context managers (Python 3.10+ also allows parenthesized multi-line form) when acquiring several independent resources together, instead of nesting a separate `with` block for each one — it's flatter, and each resource's cleanup still runs correctly in reverse order if a later one fails to acquire.
```python
# Bad — unnecessary nesting for two independent resources
with open("input.csv") as infile:
    with open("output.csv", "w") as outfile:
        process(infile, outfile)

# Good — Python 3.10+ parenthesized form
with (
    open("input.csv") as infile,
    open("output.csv", "w") as outfile,
):
    process(infile, outfile)
```
- **DO:** Use `contextlib.ExitStack` specifically when the set of context managers to hold open isn't known until runtime (a variable-length list of files, one per item in a dynamic collection) — it lets you `enter_context()` each one in a loop while still guaranteeing correct, reverse-order cleanup of everything successfully entered so far, even if a later one fails.
```python
from contextlib import ExitStack

def merge_files(paths: list[Path], output_path: Path) -> None:
    with ExitStack() as stack:
        files = [stack.enter_context(open(p)) for p in paths]
        with open(output_path, "w") as out:
            for f in files:
                out.write(f.read())
```

### Sorting & Key Functions

- **DO:** Use the `key=` parameter with `sorted()`/`list.sort()`/`min()`/`max()` to specify what to sort/compare by, instead of implementing a custom comparison function or pre-transforming the collection into tuples just to control ordering.
```python
# Bad — verbose and easy to get wrong with cmp-style comparisons
def compare(a, b):
    return (a.last_name, a.first_name) < (b.last_name, b.first_name)

# Good
users.sort(key=lambda u: (u.last_name, u.first_name))
```
- **DO:** Use `operator.attrgetter`/`operator.itemgetter` for a `key=` function that just extracts one or more attributes/items — they're implemented in C and communicate the intent more directly than an equivalent `lambda`.
```python
from operator import attrgetter
users.sort(key=attrgetter("last_name", "first_name"))
```
- **DO:** Pass `reverse=True` to `sorted()`/`.sort()` for descending order rather than negating a numeric key or reversing the result afterward, which is both less efficient and less clear about intent for non-numeric keys.

### Debug-Friendly `__repr__`

- **DO:** Implement `__repr__` on custom classes (when not already generated by `@dataclass`) to return an unambiguous, ideally eval-able representation showing the object's key state — this is what shows up in a debugger, a REPL, a pytest assertion failure diff, and a log's `%r`/`!r` formatting, and a good one saves real debugging time.
```python
# Bad — the default repr gives no useful information
class Order:
    def __init__(self, order_id, total):
        self.order_id = order_id
        self.total = total
# repr(order) -> "<__main__.Order object at 0x7f8b2c0a1d90>"

# Good
class Order:
    def __init__(self, order_id, total):
        self.order_id = order_id
        self.total = total
    def __repr__(self) -> str:
        return f"Order(order_id={self.order_id!r}, total={self.total!r})"
# repr(order) -> "Order(order_id='A123', total=49.99)"
```
- **DON'T:** Implement `__str__` to return internal/debugging detail and leave `__repr__` as the useless default, backwards from their intended roles — `__repr__` is for developers/debugging (unambiguous), `__str__` is for end users (readable); if only one is defined, `__repr__` is the one to prioritize since `__str__` falls back to it automatically.
- **DO:** Use `!r` (calls `repr()`) rather than plain `{value}` (calls `str()`) when formatting a value for a log message or an error message meant for a developer — `repr()` distinguishes `None` from `"None"` and shows quotes around strings, which matters when diagnosing exactly what value a variable held.

## Error Handling

- **DON'T:** Use a bare `except:` clause. It catches every exception, including `KeyboardInterrupt` and `SystemExit`, which means Ctrl-C stops working as expected and the process can swallow signals meant to shut it down cleanly. Catch specific exception types, or at minimum `except Exception:` if you genuinely need a catch-all.
```python
# Bad
try:
    process(data)
except:
    log.error("something went wrong")

# Good
try:
    process(data)
except ValidationError as exc:
    log.error("invalid data: %s", exc)
```
- **DON'T:** Catch `Exception` broadly and then discard it (`except Exception: pass`) or log a generic message without the original error. This hides real bugs — a `TypeError` from a programming mistake looks identical to an expected, recoverable condition, and both disappear silently.
```python
# Bad — a NullPointer-style bug looks identical to a handled case
try:
    result = risky_operation()
except Exception:
    result = None

# Good — only the specific, expected failure is handled; everything else propagates
try:
    result = risky_operation()
except TimeoutError:
    result = None
    log.warning("risky_operation timed out, falling back to None")
```
- **DO:** Catch the most specific exception type that the failing call can actually raise, and look it up rather than guessing — check the library's documentation or source for its exception hierarchy instead of reflexively catching `Exception`.
- **DO:** Define a custom exception hierarchy rooted in a single base exception per package/library (e.g., `class MyAppError(Exception): ...`, with `ValidationError(MyAppError)`, `NotFoundError(MyAppError)` beneath it). This lets callers catch broadly (`except MyAppError`) when they don't care about the distinction, or narrowly when they do, without ever needing to catch bare `Exception`.
```python
class PaymentError(Exception):
    """Base class for all payment-processing errors."""

class CardDeclinedError(PaymentError):
    def __init__(self, reason: str):
        super().__init__(f"card declined: {reason}")
        self.reason = reason

class InsufficientFundsError(PaymentError):
    pass
```
- **DO:** Use `raise NewError(...) from original_exc` (or let Python's implicit exception chaining happen naturally inside an `except` block) when converting one exception type to another. This preserves the original traceback as `__cause__`, so debugging shows the full causal chain instead of losing the root cause.
```python
# Bad — original traceback and message are lost
try:
    parsed = json.loads(raw)
except json.JSONDecodeError:
    raise ConfigError("invalid config file")

# Good — original exception is chained and visible in the traceback
try:
    parsed = json.loads(raw)
except json.JSONDecodeError as exc:
    raise ConfigError("invalid config file") from exc
```
- **DON'T:** Use `raise NewError(...) from None` to intentionally suppress the original traceback unless you have a deliberate reason (e.g., the original exception is genuinely irrelevant noise, like a low-level retry loop's intermediate failures). Suppressing it by default removes debugging information for no benefit.
- **DO:** Put the smallest possible amount of code inside a `try` block — just the statement(s) that can actually raise the exception you're handling — so the `except` clause can't accidentally catch an unrelated failure from a different line.
```python
# Bad — a bug in parse_response() would be mis-attributed to the network call
try:
    response = requests.get(url, timeout=5)
    data = parse_response(response)
except requests.RequestException:
    data = None

# Good
try:
    response = requests.get(url, timeout=5)
except requests.RequestException:
    return None
data = parse_response(response)
```
- **DO:** Prefer EAFP ("easier to ask forgiveness than permission" — try the operation and catch the exception) over LBYL ("look before you leap" — check preconditions first) for operations where the check-then-act sequence has a race condition, such as file access or dict lookups in concurrent code. It's also generally considered more idiomatic Python for common cases like dict/attribute access.
```python
# LBYL — has a TOCTOU race in concurrent contexts, and is slower on the happy path
if key in config:
    value = config[key]
else:
    value = default

# EAFP — atomic, and idiomatic
try:
    value = config[key]
except KeyError:
    value = default
```
- **DON'T:** Use exceptions for routine, expected control flow where a return value communicates the same thing more cheaply and clearly (e.g., raising `StopSearchException` to break out of a loop instead of just using `break`, or raising to signal "not found" when returning `None` or a sentinel is the established convention for that function). Reserve exceptions for genuinely exceptional, error-like conditions.
- **DO:** Use the `else` clause on a `try` block for code that should run only when no exception was raised, keeping it clearly separate from both the risky code and the cleanup code.
```python
try:
    conn = connect_to_db()
except ConnectionError:
    log.error("failed to connect")
    return
else:
    # only runs if connect_to_db() succeeded
    run_migrations(conn)
finally:
    conn.close() if 'conn' in locals() else None
```
- **DO:** Use `finally` (or, better, a context manager) for cleanup that must run whether or not an exception occurred — releasing locks, closing connections, deleting temp files.
- **DON'T:** Return `None`, `-1`, `False`, or another ambiguous "error sentinel" from a function to signal failure when the caller has no reliable way to distinguish that from a legitimate result. If `None` is itself a valid successful return value, an error sentinel is inherently ambiguous — raise an exception instead, or return an explicit `Result`/`Ok`/`Err`-style wrapper if the codebase has adopted that pattern.
```python
# Bad — indistinguishable from "found a user named None" or a real -1 balance
def find_user(user_id):
    ...
    return None  # not found, or an actual error?

# Good
def find_user(user_id: int) -> User:
    ...
    raise UserNotFoundError(user_id)
```
- **DO:** Include enough context in an exception's message to debug it without a debugger attached — the relevant identifiers, the value that failed validation, the operation being attempted — not just a generic "an error occurred."
```python
# Bad
raise ValueError("invalid input")

# Good
raise ValueError(f"invalid discount percent {percent!r}: must be between 0 and 100")
```
- **DO:** Use `assert` only for internal invariants that should be impossible to violate if the code is correct (a sanity check on your own logic), never for validating external input, arguments from a public API, or user-supplied data — see the earlier note on `-O` stripping assertions.
- **DO:** Use a context manager (`contextlib.contextmanager` wrapping a `try/finally`, covered under Idiomatic Python) to guarantee compensating/rollback logic runs on failure partway through a multi-step operation, instead of manually repeating cleanup code in every `except` branch of a complex operation.
```python
@contextmanager
def staged_deployment(deployment_id: str):
    mark_deployment_in_progress(deployment_id)
    try:
        yield
        mark_deployment_succeeded(deployment_id)
    except Exception:
        mark_deployment_failed(deployment_id)
        rollback_deployment(deployment_id)
        raise

with staged_deployment(deployment_id):
    apply_migration()
    restart_services()
    run_smoke_tests()
```
- **DON'T:** Leave a multi-step operation in a partially-completed, inconsistent state after a failure partway through, with no compensating action — either make each step idempotent and safely retryable, or wrap the whole sequence so a failure at any point triggers a defined rollback/cleanup path, rather than leaving whatever succeeded so far in place with no record of the partial state.
- **DON'T:** Re-raise a caught exception as `raise e` inside its own `except` block — this resets the traceback to start at the re-raise point, hiding where the exception actually originated. Use a bare `raise` (no argument) to re-raise the current exception with its original traceback intact.
```python
# Bad — traceback now starts here, not at the real failure site
except ValueError as e:
    log.error("bad value")
    raise e

# Good — original traceback preserved
except ValueError:
    log.error("bad value")
    raise
```
- **DO:** Group related, independent failures with `ExceptionGroup` and `except*` (Python 3.11+) when a single operation can produce multiple concurrent errors (e.g., `asyncio.TaskGroup` collecting failures from several tasks), rather than only ever surfacing the first exception and discarding the rest.
- **DO:** Log exceptions with `logger.exception(...)` (inside an `except` block) rather than `logger.error(str(exc))`, so the full traceback is captured in the log output instead of just the exception's message.
```python
# Bad — traceback is lost, only the message survives
except Exception as exc:
    logger.error(f"failed: {exc}")

# Good — full traceback is logged automatically
except Exception:
    logger.exception("processing failed")
```
- **DON'T:** Let a library-level function decide to `sys.exit()`, print an error, or otherwise handle an exception in a way that assumes it's running as the top-level program. Libraries should raise; only the application's entry point should decide how to present or recover from an error.
- **DO:** Use Python 3.11+'s `Exception.add_note()` to attach additional diagnostic context to an exception as it propagates through multiple layers, when that context would otherwise require either a wrapped exception at every layer or losing the detail entirely. Notes appear in the traceback alongside the original exception, giving a fuller picture of what was happening at each level without needing a new exception type per layer.
```python
try:
    process_batch(batch_id)
except Exception as exc:
    exc.add_note(f"while processing batch {batch_id}, item {current_item_index}")
    raise
```
- **DON'T:** Rely on parsing an exception's string message to determine what kind of error occurred (`if "not found" in str(exc):`). Exception messages are meant for humans and can change wording between library versions without warning; catch by exception *type* (or a documented, stable error code/attribute the library provides), never by matching message text.
- **DO:** Catch multiple exception types in one `except` clause with a tuple when they should genuinely be handled identically, instead of duplicating the same handler body across several separate `except` blocks — but only when the handling really is identical; if two exception types need different recovery logic, keep them in separate clauses even though it's more code.
```python
# Repetitive — identical handling duplicated across two clauses
try:
    response = call_external_api()
except TimeoutError:
    return cached_fallback()
except ConnectionError:
    return cached_fallback()

# Concise — one clause, since the handling is genuinely the same
try:
    response = call_external_api()
except (TimeoutError, ConnectionError):
    return cached_fallback()
```
- **DON'T:** List a broad exception type before a more specific one in a sequence of `except` clauses for the same `try` block — Python matches clauses in order, so a broader type listed first (e.g., `except Exception:` before `except ValueError:`) shadows the more specific clause entirely, which never runs.
- **DO:** Use a custom exception's `__init__` to enforce that it's always constructed with the context it needs (e.g., requiring a `field` argument rather than making it optional), so it's structurally impossible to raise that exception type without the diagnostic information every catcher will need — this is a small design choice that pays off directly the first time someone debugs a production incident using that exception's structured data instead of just its message.
- **DON'T:** Define an exception class with a mutable default argument in its `__init__` (the same mutable-default-argument pitfall covered under Idiomatic Python, but worth calling out specifically for exceptions since they're easy to overlook as "just another class") — an exception carrying a mutable list/dict default shares that same footgun of state leaking across instances that were never supposed to be related.
```python
# Bad — the same list object is shared across every ValidationError instance
class ValidationError(Exception):
    def __init__(self, message, errors=[]):
        super().__init__(message)
        self.errors = errors

# Good
class ValidationError(Exception):
    def __init__(self, message, errors=None):
        super().__init__(message)
        self.errors = errors if errors is not None else []
```

### Retries & Transient Failures

- **DO:** Use a battle-tested retry library (`tenacity`, `backoff`) for retrying operations that fail transiently (a flaky network call, a rate-limited API, a momentarily unavailable database), instead of hand-rolling a retry loop — they handle exponential backoff, jitter, max-attempt limits, and which exceptions to retry on correctly, edge cases that are easy to get subtly wrong by hand.
```python
# Bad — no backoff (hammers the failing service), no jitter, retries forever
def fetch_with_retry(url):
    while True:
        try:
            return requests.get(url, timeout=5)
        except requests.RequestException:
            continue

# Good
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(5), wait=wait_exponential(multiplier=1, min=1, max=10))
def fetch_with_retry(url):
    return requests.get(url, timeout=5)
```
- **DON'T:** Retry an operation that isn't idempotent (e.g., a payment charge, a non-idempotent POST that creates a new resource each time) without an idempotency key or equivalent safeguard — a retried non-idempotent call after a timeout can silently duplicate the side effect (double-charging, creating two records) even though the first attempt actually succeeded on the server side.
- **DON'T:** Retry on every exception type indiscriminately. Retry only errors that are plausibly transient (network timeouts, 5xx responses, specific "rate limited" errors); retrying a `ValidationError` or `4xx` client error just repeats the same failure and wastes time and quota.
- **DO:** Cap total retry time/attempts and surface the final failure clearly (with the last underlying error attached via exception chaining) rather than retrying silently forever or swallowing the eventual failure after retries are exhausted.

### Validation Layers & Error Surfaces

- **DO:** Validate input as early as possible — at the boundary of a function, an API endpoint, or a CLI command — and fail with a clear, specific exception immediately, rather than letting invalid data propagate deep into business logic where the eventual failure is far removed from its actual cause.
```python
# Bad — invalid input isn't caught until it fails deep inside processing,
# with an error that gives no clue it started from a bad argument
def process_order(quantity, price):
    return _apply_tax(_apply_discount(quantity * price))

# Good — fails fast, right where the bad input actually enters
def process_order(quantity: int, price: float):
    if quantity <= 0:
        raise ValueError(f"quantity must be positive, got {quantity}")
    if price < 0:
        raise ValueError(f"price cannot be negative, got {price}")
    return _apply_tax(_apply_discount(quantity * price))
```
- **DO:** Design a consistent error-response shape for anything exposed over an API (a stable JSON error envelope with a machine-readable code and a human-readable message), and map internal exception types to that shape at a single, centralized boundary (an exception handler/middleware), rather than formatting error responses ad hoc at every endpoint.
- **DON'T:** Leak internal implementation details (a raw stack trace, an internal file path, a database error message) to an external API caller. Log the full detail server-side, and return a sanitized, appropriately generic error to the client — especially important for anything that could reveal exploitable information to an attacker.

### Custom Exception Design Patterns

- **DO:** Attach structured data to a custom exception (as instance attributes set in `__init__`) rather than only embedding values in the formatted message string — this lets calling code programmatically inspect *why* it failed (e.g., `exc.field_name`, `exc.limit`) instead of parsing the human-readable message text.
```python
# Bad — the only way to get the field name back out is regex-parsing the message
raise ValidationError(f"field 'email' failed validation: invalid format")

# Good — structured data the caller can act on programmatically
class ValidationError(Exception):
    def __init__(self, field: str, reason: str):
        self.field = field
        self.reason = reason
        super().__init__(f"field {field!r} failed validation: {reason}")

try:
    validate(payload)
except ValidationError as exc:
    return {"error": "validation_failed", "field": exc.field, "reason": exc.reason}
```
- **DO:** Give each exception subclass in a hierarchy a distinct, specific meaning rather than creating one broad exception type that's raised with a different message for every case — the whole point of a hierarchy is letting callers catch precisely the failure modes they can meaningfully handle differently.
- **DON'T:** Create a deep, over-specified exception hierarchy for distinctions no calling code will ever actually need to handle differently (e.g., `EmailValidationError`, `PhoneValidationError`, `AddressValidationError` all subclassing `ValidationError` when every caller only ever catches the base `ValidationError` anyway). Match hierarchy granularity to how callers actually need to discriminate between failures.

### Handling Errors Across Boundaries

- **DO:** Translate a lower-level/third-party library's exception into your own domain exception at the boundary where you call that library, rather than letting the library's exception type leak into and propagate through your own business logic and public API. Callers of your code should generally depend on your exception hierarchy, not on which specific HTTP client, database driver, or ORM you happen to use underneath.
```python
# Bad — every caller of get_user() now has to know requests-specific exceptions
def get_user(user_id: int) -> User:
    response = requests.get(f"{API_BASE}/users/{user_id}", timeout=5)
    response.raise_for_status()
    return User(**response.json())

# Good — the underlying HTTP library's exception is translated at the boundary
def get_user(user_id: int) -> User:
    try:
        response = requests.get(f"{API_BASE}/users/{user_id}", timeout=5)
        response.raise_for_status()
    except requests.RequestException as exc:
        raise UserServiceUnavailableError(f"could not fetch user {user_id}") from exc
    return User(**response.json())
```
- **DO:** Decide, deliberately and consistently, what a given layer of a system does when it catches an exception it can't fully resolve: re-raise as a different, more meaningful type; retry; degrade gracefully with a fallback; or propagate unchanged. Make that decision explicit in code (and ideally in a comment when the choice isn't obvious), rather than letting it be an accident of whatever `try/except` shape was easiest to write.
- **DON'T:** Catch an exception at a low layer only to immediately re-raise the exact same exception with no transformation, added context, logging, or cleanup — an empty pass-through `except SomeError: raise` adds a stack frame and visual noise without doing anything a bare absence of the `try/except` wouldn't already achieve.

### Errors in Batch & Background Processing

- **DO:** Decide explicitly whether a batch operation should fail entirely on the first error (fail-fast) or continue processing remaining items and report all failures at the end (best-effort), and implement whichever is actually appropriate for the use case — don't let this be an accident of whether the loop happens to have a `try/except` inside it or not.
```python
# Best-effort: collect and report every failure, without one bad row aborting the whole batch
def import_rows(rows: list[dict]) -> ImportResult:
    succeeded, failed = [], []
    for i, row in enumerate(rows):
        try:
            record = validate_and_create(row)
            succeeded.append(record)
        except ValidationError as exc:
            failed.append(RowError(index=i, row=row, reason=str(exc)))
    return ImportResult(succeeded=succeeded, failed=failed)
```
- **DON'T:** Silently drop failed items from a batch operation with no record of what failed or why — a bulk import that just skips bad rows with no reporting mechanism gives the caller no way to know the import was incomplete, let alone which records need attention.
- **DO:** Make failures in background/async jobs (a queue worker, a scheduled task) visible somewhere a human will actually see them — structured logging with alerting, a dead-letter queue, a monitoring dashboard — rather than only writing to a log file nobody is watching. An exception that only exists in a log line no one reads is functionally the same as a silently swallowed exception.

### Graceful Degradation

- **DO:** Design a fallback behavior explicitly for a non-critical dependency that might be unavailable (a recommendations service, an analytics call, a nice-to-have enrichment lookup) so its failure degrades the user experience rather than failing the entire request — but only when partial functionality is genuinely acceptable for that specific dependency; a payment step or an auth check should generally fail loudly, not "degrade gracefully" into skipping the check.
```python
def get_product_page(product_id: int) -> ProductPage:
    product = fetch_product(product_id)  # core data — let this raise if it fails
    try:
        recommendations = fetch_recommendations(product_id)
    except RecommendationServiceError:
        logger.warning("recommendations unavailable for product %s", product_id)
        recommendations = []  # acceptable degraded experience — not a broken page
    return ProductPage(product=product, recommendations=recommendations)
```
- **DON'T:** Apply "graceful degradation" to a failure that should actually stop the operation — silently proceeding with a missing authorization check, an unconfirmed payment, or corrupted core data "so the request doesn't fail" turns a visible, fixable error into a much worse silent correctness or security problem.
- **DO:** Consider a circuit breaker pattern (via a library like `pybreaker`, or a hand-rolled failure-counting wrapper for simple cases) around calls to a dependency that's prone to extended outages, so repeated calls stop hammering an already-failing service and instead fail fast for a cooldown period — this protects both the failing downstream service and the calling service's own resources (threads, connections) from being consumed waiting on calls likely to fail anyway.

## Packaging & Dependency Management

- **DO:** Define project metadata, dependencies, and build configuration in a single `pyproject.toml` (PEP 517/518/621), rather than splitting them across `setup.py`, `setup.cfg`, and `requirements.txt`. It's the current standard, understood by pip, build, and every modern packaging tool.
```toml
# pyproject.toml
[project]
name = "myapp"
version = "1.2.0"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.31,<3",
    "pydantic>=2.5,<3",
]

[project.optional-dependencies]
dev = ["pytest>=8", "ruff>=0.5", "mypy>=1.10"]
```
- **DON'T:** Write a new `setup.py` with imperative build logic for a typical pure-Python package. `setup.py` executes arbitrary code at install time (a security and reproducibility concern) and has been superseded by declarative `pyproject.toml` for the vast majority of packaging needs; keep it only for the rare case of genuinely dynamic build steps a declarative config can't express.
- **DO:** Create and activate a virtual environment (`venv`, `virtualenv`, or a tool-managed one via Poetry/uv/Pipenv) for every project, and never install project dependencies into the system or user-global Python. Global installs cause version conflicts between unrelated projects and make builds non-reproducible.
```bash
# Good
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```
- **DON'T:** Commit a virtual environment directory (`.venv/`, `venv/`, `env/`) to version control. It's large, platform-specific, and trivially reproducible from the lockfile/dependency spec — add it to `.gitignore` instead.
- **DO:** Pin dependencies with a lockfile (`poetry.lock`, `uv.lock`, `Pipfile.lock`, or a pip-compile-generated `requirements.txt`) that records exact resolved versions, including transitive dependencies, and commit that lockfile. Reproducible installs across machines and CI depend on it — an unpinned `requirements.txt` with only loose version ranges can resolve differently on different days.
- **DO:** Express *direct* dependencies in `pyproject.toml` with sensible version ranges (not exact pins), and let the lockfile pin the exact resolved graph. Over-pinning direct dependencies to a single exact version in `pyproject.toml` itself makes the package hard to use alongside other packages with slightly different needs; that precision belongs in the lockfile, not the abstract dependency declaration.
- **DO:** Understand what each version-constraint operator actually promises before choosing one: `>=2.1,<3` allows any 2.x release (protects against an unreviewed major/breaking bump while allowing minor features and patches); `~=2.1.0` (compatible release) allows patch-level updates only (`2.1.x`, not `2.2`); an exact pin (`==2.1.0`) allows nothing to change until a human edits it. Choose deliberately based on how much you trust the dependency's own versioning discipline, not out of habit.
```toml
[project]
dependencies = [
    "requests>=2.31,<3",     # allow any 2.x release — trusts requests' semver discipline
    "some-internal-lib~=1.4.0",  # patch releases only — a library still stabilizing its API
    "a-notoriously-unstable-pkg==0.3.2",  # exact pin — this one has broken minor releases before
]
```
- **DON'T:** Leave a dependency unconstrained (no version specifier at all) for anything that isn't trivially replaceable, purely to "always get the latest" — an application, unlike a widely-reused library, generally benefits more from controlled, reviewed upgrades (via the lockfile and a deliberate `pip-compile --upgrade`/`poetry update`) than from silently picking up whatever the newest release happens to be on the next install.
- **DO:** Choose one dependency/environment manager for a project — `uv`, `Poetry`, `pip-tools` + `venv`, or plain `pip` + `venv` are all legitimate — and use it consistently. Mixing `poetry add` and manual `pip install` in the same project produces a `poetry.lock` that silently drifts from what's actually installed.
- **DO:** Use `pip-compile` (from `pip-tools`) or an equivalent lock-generation step when using plain `pip`, so `requirements.txt` is generated from a minimal `requirements.in` rather than hand-maintained — hand-edited requirements files drift and rarely record transitive pins accurately.
- **DON'T:** Install packages with `sudo pip install` or into the system Python. This can break OS-level tooling that depends on the system interpreter and its packages, and it's a symptom of not using virtual environments at all.
- **DO:** Adopt the `src/` layout (`src/myapp/...` with tests outside it at the project root) for anything beyond a small script, so the package is only importable when properly installed — this catches "it works on my machine because I'm running from the project root" bugs that a flat layout (`myapp/` next to `setup.py`/`pyproject.toml`) can hide.
- **DO:** Install a package in editable mode (`pip install -e .`) during local development instead of manipulating `sys.path` or running scripts from inside the package directory. It makes the package importable everywhere in the environment while still reflecting live source edits.
- **DON'T:** Manually append to `sys.path` at the top of scripts (`sys.path.append("../..")`) to make imports work. It's fragile (depends on the current working directory at run time), doesn't play well with tooling, and is almost always a sign the project should be installed as a package instead.
```python
# Bad
import sys
sys.path.append("../../")
from myapp.utils import helper

# Good — myapp is installed (editable or not) into the active environment
from myapp.utils import helper
```
- **DO:** Follow semantic versioning (`MAJOR.MINOR.PATCH`) for a published package's version number, and bump `MAJOR` for any breaking change to its public API — consumers rely on version ranges (`>=1,<2`) to mean "safe to upgrade automatically."
- **DO:** Define console-script entry points in `pyproject.toml` (`[project.scripts]`) for anything meant to be run as a CLI command, instead of telling users to run `python path/to/script.py` directly.
```toml
[project.scripts]
myapp = "myapp.cli:main"
```
- **DON'T:** Vendor a copy of a third-party library's source directly into your own repository "to avoid a dependency," except as a deliberate, documented last resort (e.g., no packaged distribution exists, or a security-sensitive single-purpose need). Vendored code stops receiving upstream security fixes and version updates unless someone remembers to manually sync it.
- **DO:** Use `platformdirs` (the maintained successor to `appdirs`) to resolve the correct, OS-appropriate location for application config, cache, and data directories, rather than hardcoding a path like `~/.myapp` that ignores platform conventions (`%APPDATA%` on Windows, `~/Library/Application Support` on macOS, XDG base directories on Linux).
```python
from platformdirs import user_config_dir, user_cache_dir

config_path = Path(user_config_dir("myapp")) / "settings.toml"
cache_path = Path(user_cache_dir("myapp")) / "response_cache.db"
```
- **DON'T:** Write application state, logs, or cache files directly into the installed package's own directory (`site-packages/myapp/...`) — that location may not be writable by the running user (a system-wide install), isn't the conventional place for mutable runtime data, and can be wiped out entirely on the next upgrade/reinstall of the package.
- **DO:** Read configuration secrets (API keys, database URLs, credentials) from environment variables or a secrets manager, not from values committed to the repository — including in a `.env` file, which should be `.gitignore`d, with only an `.env.example` template committed.
- **DO:** Separate runtime dependencies from development-only dependencies (test runners, linters, type checkers) using `[project.optional-dependencies]` or a dependency-group mechanism, so production installs don't pull in tooling they'll never use.
- **DON'T:** Depend on an unpinned "latest" version of a critical dependency in production deployment configuration (e.g., a Dockerfile doing `pip install django` with no version constraint). A transitive or direct upstream release can silently change behavior or break the build on a date you don't control.
- **DO:** Document the minimum supported Python version explicitly via `requires-python` in `pyproject.toml`, and verify it in CI across that range (or at least the oldest and newest supported versions), rather than only ever testing against whatever version happens to be on the primary developer's machine.
- **DO:** Use a `Makefile`, `tox.ini`, `nox` config, or documented `just`/npm-script-style task runner to standardize common commands (`test`, `lint`, `format`, `build`) so contributors don't need to reverse-engineer the right invocation from CI config.

### Containerizing Python Applications

- **DO:** Pin the base image to a specific Python version tag (`python:3.12-slim`), not `python:latest` or an unversioned tag, so a container build today produces the same interpreter version as a build next month — an unpinned base image can silently jump a Python minor/major version on rebuild.
```dockerfile
# Bad — resolves to whatever "latest" points to on build day, drifts over time
FROM python:latest

# Good — explicit, reproducible
FROM python:3.12-slim
```
- **DO:** Copy dependency manifests (`pyproject.toml`, the lockfile) and install dependencies in their own `Dockerfile` layer *before* copying the rest of the application source. Docker's layer caching then skips the (often slow) dependency-install step on rebuilds that only change application code.
```dockerfile
# Good — dependency layer is cached separately from source code
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .
```
- **DON'T:** Run the application process as root inside the container. Create and switch to a non-root user in the Dockerfile — running as root inside a container is unnecessary privilege that widens the blast radius of any container-escape or dependency-compromise scenario.
- **DO:** Use a multi-stage build to keep the final image free of build-only tooling (compilers, dev dependencies) — build in one stage, then copy only the installed environment/application into a slim final stage. Smaller images mean a smaller attack surface and faster deploys.
- **DON'T:** Bake secrets (API keys, credentials) into a Docker image via `ENV` instructions or files copied into the image. Anyone with access to the image (including its layer history) can extract them — inject secrets at runtime instead (environment variables passed at container start, a mounted secrets volume, or a secrets manager integration).
- **DO:** Set `PYTHONDONTWRITEBYTECODE=1` and `PYTHONUNBUFFERED=1` as container environment variables for typical containerized deployments — the former avoids writing `.pyc` files into an ephemeral, frequently-rebuilt image, and the latter ensures `print`/logging output isn't buffered and delayed inside the container's stdout, which would otherwise delay it reaching the container log collector.

### Versioning & Release Tooling

- **DO:** Automate version bumping and changelog generation (via `bump-my-version`, `python-semantic-release`, or an equivalent tool driven by conventional commit messages) rather than manually editing a version string in multiple places, which is easy to forget or get inconsistent across `pyproject.toml`, `__init__.py`, and a changelog file.
- **DO:** Keep a single source of truth for the package version — ideally `pyproject.toml`'s `[project] version`, with `importlib.metadata.version("mypackage")` used at runtime to read it back if the version needs to be accessible in code — instead of hardcoding the version string separately in a `__version__` variable that can drift out of sync.
- **DON'T:** Publish a release to a package index (PyPI or a private index) without a corresponding git tag, changelog entry, and passing CI run. An untagged, undocumented release is difficult to trace back to the exact source that produced it when a bug report comes in later.

### Build Backends & Editable Installs

- **DO:** Declare an explicit `[build-system]` table in `pyproject.toml` naming the build backend (`setuptools`, `hatchling`, `flit-core`, `pdm-backend`, etc.) and its required version — this is what lets `pip install` (or any PEP 517-compliant tool) know how to actually build the package, independent of which dependency manager a contributor happens to use day-to-day.
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```
- **DON'T:** Assume every build backend handles editable installs (`pip install -e .`) identically — modern backends implement PEP 660 editable installs (often via a lightweight `.pth`-based or import-hook mechanism), which behave slightly differently from `setuptools`' traditional `.egg-link` approach. If editable installs behave unexpectedly (e.g., a new submodule isn't picked up until reinstall), check the specific backend's documented editable-install behavior rather than assuming it's a bug.
- **DO:** Pick a build backend based on what the project actually needs — `hatchling` or `flit-core` for a straightforward pure-Python package (simpler configuration, less legacy baggage), `setuptools` when compiled extensions or complex build steps require its ecosystem — rather than defaulting to whichever one happened to be used in the last unrelated project.

### Reproducible Builds & Hash Verification

- **DO:** Generate lockfiles with per-package hashes (`pip-compile --generate-hashes`, or the hash pinning most modern lockfile formats include by default) so `pip install` refuses to install a package whose downloaded content doesn't match the hash recorded at lock time — this defends against a compromised or tampered package being silently substituted for the trusted one, even if it shares the same name and version number.
```bash
# requirements.txt generated with --generate-hashes
requests==2.31.0 \
    --hash=sha256:58cd2187c01e70e6e26505bca751777aa9f2ee0b7f4300988b709518771b32) \
    --hash=sha256:942c5a758f98d790eaed1a29cb6eefc7ffb0d1cf7af05c3d2791656dbd6ad1e2
```
- **DO:** Run installs with `pip install --require-hashes` (or the equivalent enforcement in whichever tool is used) in CI and production deployment pipelines specifically, where reproducibility and supply-chain integrity matter most, even if local development installs are more lenient for convenience.
- **DON'T:** Treat "the lockfile pins exact versions" as equivalent to "the lockfile guarantees exact content" — a version pin alone doesn't stop a compromised package index (or a yanked-and-replaced release under unusual circumstances) from serving different bytes for the same version string; hash verification is the guarantee version pinning alone doesn't provide.

### Monorepos & Private Indexes

- **DO:** Use a private package index (a self-hosted `pypiserver`/`devpi`, or a managed offering) for internal packages shared across multiple projects, rather than distributing them via git-URL dependencies or copy-pasted source. A real index gives you proper versioning, dependency resolution, and the same install experience as a public package.
- **DO:** For a monorepo housing multiple installable Python packages, keep each package's own `pyproject.toml` with explicit, real dependencies on its monorepo siblings (via path dependencies in development, resolved to real version constraints for publishing) rather than relying on everything sharing one flat `sys.path` and implicit availability.
- **DON'T:** Let internal packages depend on each other through ad hoc `sys.path` manipulation or symlinks as a substitute for a real, declared dependency. It works until someone tries to build, containerize, or install just one of the packages in isolation, at which point the hidden dependency breaks silently.

### Comparing Common Dependency Workflows

- **DO:** Understand what each common workflow actually gives you before picking one for a new project — `uv` (fast, single binary, handles venvs/locking/Python-version installs), `Poetry` (mature, integrated dependency management plus packaging/publishing), and `pip-tools` (minimal — just compiles a lockfile from a `requirements.in`, leaving venv creation and installation to plain `pip`/`venv`) solve overlapping but not identical problems, and the "best" one depends on the team's existing tooling and how much the project needs beyond dependency locking.
```bash
# uv — fast, single tool for venv + install + lock
uv venv && uv pip sync requirements.lock
uv add requests           # adds to pyproject.toml and updates the lock

# Poetry — integrated dependency + build + publish workflow
poetry add requests
poetry install
poetry build && poetry publish

# pip-tools — minimal, layered on top of plain pip/venv
python -m venv .venv && source .venv/bin/activate
pip-compile requirements.in -o requirements.txt
pip-sync requirements.txt
```
- **DON'T:** Switch a project's dependency tooling mid-stream without migrating the lockfile and updating CI/documentation together in one change — a half-migrated project (some contributors on Poetry, others on plain pip, a stale `requirements.txt` next to a `poetry.lock` nobody updates) is worse than either tool used consistently.
- **DO:** Document, in the project's README or contributing guide, exactly which commands to run to set up a working development environment from a clean checkout — the specific tool matters less than every contributor being able to reproduce the same, correct environment from those documented steps alone.

### Cross-Platform Pitfalls

- **DON'T:** Hardcode a path separator (`"data" + "/" + "file.txt"`, or a backslash on Windows-only code) instead of using `pathlib.Path`'s `/` operator, which resolves to the correct separator for whatever OS the code actually runs on.
- **DON'T:** Assume file paths are case-insensitive (true on default Windows and macOS filesystems, false on Linux) — a reference to `Config.json` when the real file is `config.json` may work in local development on macOS/Windows and fail immediately in a Linux-based CI or production container.
- **DON'T:** Assume line endings are always `\n` when reading text cross-platform — Python's text-mode file I/O already does universal newline translation by default, but code that shells out to external tools, reads raw bytes, or hardcodes `\n` in a string comparison against file content can behave differently on a file that was saved with Windows-style `\r\n` endings.
- **DO:** Set `PYTHONUTF8=1` (or otherwise ensure UTF-8 I/O explicitly, e.g. `open(path, encoding="utf-8")`) rather than relying on a platform's default locale/encoding for text file I/O — a script that works locally on a machine with a UTF-8 default locale can read garbled text or raise `UnicodeDecodeError` on a system (notably some Windows configurations) whose default encoding isn't UTF-8.
```python
# Bad — relies on the platform's default encoding, which varies
with open("data.txt") as f:
    text = f.read()

# Good — explicit, and identical behavior on every platform
with open("data.txt", encoding="utf-8") as f:
    text = f.read()
```
- **DON'T:** Shell out to a platform-specific command (`ls`, `dir`, `del`) via `subprocess` when a portable standard-library equivalent exists (`os.listdir`/`pathlib.Path.iterdir`, `os.remove`/`Path.unlink`) — the portable version works identically across operating systems and avoids depending on a shell command's exact behavior and availability.

## Linting & Formatting

- **DO:** Use `ruff` as a fast, unified linter (and optionally formatter, via `ruff format`) covering what used to require separate `flake8`, `isort`, `pyupgrade`, and several plugin tools. Consolidating reduces config drift between overlapping tools and speeds up CI.
- **DO:** Use `black` (or `ruff format`, which is largely Black-compatible) as the single source of truth for code formatting, and configure editors/pre-commit to run it automatically on save/commit. Once a formatter owns whitespace and line-wrapping, formatting stops being a topic for code review.
- **DON'T:** Argue about formatting choices (spaces around operators, where to break a long line, single vs. double quotes) in code review once a formatter is adopted. If the formatter's default output looks wrong for a specific case, that's a discussion about the formatter's config, not about hand-editing around it.
- **DO:** Run `isort` (or Ruff's `I` rule set) to keep import ordering and grouping automatic and consistent, rather than relying on each contributor to order imports by hand.
- **DO:** Enable pre-commit hooks (via the `pre-commit` framework) that run the formatter, linter, and at least a fast subset of type-checking before every commit, so style and trivial correctness issues never reach code review.
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```
- **DO:** Run the linter, formatter check (`--check` mode, not auto-fix), and type checker in CI as required checks, not just locally — a local pre-commit hook can be skipped (`--no-verify`) or simply not installed by a contributor.
- **DON'T:** Disable a broad category of lint rules project-wide just to silence a handful of false positives. Suppress the specific line with a targeted `# noqa: <code>` (or ruff's equivalent) and a short reason, so the rest of the codebase still benefits from that check.
```python
# Bad — turns off unused-import checking everywhere
# ruff: noqa: F401

# Good — one line, with a reason
import legacy_module  # noqa: F401  # imported for its side effects (registers plugin)
```
- **DO:** Keep linter and formatter configuration in `pyproject.toml` under `[tool.ruff]` / `[tool.black]` / `[tool.mypy]`, version-controlled alongside the code, so every contributor and CI run uses identical settings.
- **DO:** Turn on Ruff rule categories beyond the default set deliberately (e.g., `B` for bugbear-style likely-bug patterns, `SIM` for simplification, `UP` for modernizing syntax to the target Python version, `S` for basic security checks) rather than leaving the linter at its most permissive defaults indefinitely.
- **DON'T:** Treat a clean lint pass as equivalent to a code review. Linters catch style and a subset of correctness smells; they don't catch wrong business logic, missing test coverage, or poor API design.
- **DO:** Use `ruff --fix` (or the equivalent auto-fix mode) to batch-apply safe, mechanical fixes, but review the diff before committing — most fixes are safe, but a small number can change behavior (e.g., some `SIM` rules touching short-circuit evaluation) and deserve a human glance.

### Beyond Style: Security, Docstring, and Complexity Linters

- **DO:** Run `bandit` (or Ruff's `S` rule category, which covers much of the same ground) as a dedicated security linter in CI — it flags patterns like `eval()`/`exec()` use, `assert` used for validation, hardcoded passwords, `subprocess` calls with `shell=True`, and weak cryptographic primitives, catching a category of issue that style-focused linting doesn't target at all.
- **DO:** Enforce docstring presence and format on public APIs with `pydocstyle` (or Ruff's `D` rule category) when a project has committed to documented public interfaces, so missing or malformed docstrings are caught the same way a missing type hint would be.
- **DO:** Track cyclomatic complexity (via `radon`, `xenon`, or Ruff's `C901` rule) to flag functions that have grown too many branching paths to reason about or test thoroughly, using it as a prompt to consider refactoring rather than a hard gate that blocks legitimate complexity.
- **DON'T:** Enable every available linter category at maximum strictness on day one of an existing, previously-unlinted codebase. Introduce stricter rule sets incrementally (often via a baseline that ratchets — new code must pass, existing violations are tracked but grandfathered) so the tooling becomes a forward-looking gate rather than an unmanageable wall of pre-existing violations.

### Per-File Rule Exceptions

- **DO:** Scope a legitimate, permanent rule exception to the specific file or directory it applies to, using the linter's per-file-ignore configuration, rather than a project-wide disable — a test directory reasonably allowing `assert` statements (which `bandit`/Ruff's `S` rules otherwise flag) shouldn't require disabling that check for application code too, where it's a genuinely useful signal.
```toml
[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101"]          # assert is fine and expected in tests
"src/myapp/legacy/**/*.py" = ["D"]  # legacy module not yet docstring-compliant
```
- **DON'T:** Accumulate inline `# noqa` suppressions without any reason comment as the default response to a linter complaint. A reviewer (or the same author, months later) has no way to tell a deliberate, justified suppression from one added just to make a red CI check go green.
```python
# Bad — no indication of why this is safe to ignore
result = eval(expression)  # noqa

# Good — the reason is right there for the next reader
result = eval(expression)  # noqa: S307 — expression is a hardcoded constant, not user input
```
- **DO:** Periodically audit accumulated suppressions (`grep` for `noqa`/`type: ignore` across the codebase, or a linter's own reporting of ignored violations) as part of routine maintenance — a suppression added for a since-fixed reason, or one that's silently started masking a real new issue after a refactor, is easy to forget about once it's in place.

### Configuring Ruff, Black, and mypy Together

- **DO:** Keep line-length, target-Python-version, and quote-style settings consistent across the formatter and linter configs — a formatter set to an 88-column line length while the linter is configured for 79 produces constant, unfixable friction between the two tools.
```toml
# pyproject.toml — one coherent configuration
[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "S"]

[tool.black]
line-length = 100
target-version = ["py311"]

[tool.mypy]
python_version = "3.11"
strict = true
```
- **DO:** Wire the formatter, linter, and type checker into the same CI job (or clearly labeled parallel jobs) that runs on every pull request, and make it a required check before merge — a tool that's configured but not enforced in CI degrades into something only a few conscientious contributors actually run locally.
```yaml
# .github/workflows/ci.yml (excerpt)
- run: ruff check .
- run: ruff format --check .
- run: mypy src/
- run: pytest
```
- **DON'T:** Let CI linting and local pre-commit hooks drift out of sync (different rule sets, different versions of the same tool). Pin tool versions identically in both places, ideally by having pre-commit and CI both read from the same `pyproject.toml` configuration rather than duplicating flags.

### Editor Integration & Migrating from Legacy Tools

- **DO:** Configure the team's editors/IDEs to run the formatter on save and surface linter/type-checker diagnostics inline, using the same configuration files committed to the repository — so what a developer sees while typing matches exactly what CI will enforce, instead of a developer discovering a violation only after pushing.
- **DO:** Migrate incrementally from a legacy toolchain (separate `flake8` + `isort` + `pyupgrade` + `pylint`) to a consolidated one (`ruff`) by first running the new tool in parallel/report-only mode, comparing its findings against the old setup, and only removing the legacy tools once the team is confident the new configuration covers the checks that actually mattered — a silent, all-at-once swap can quietly drop a category of check nobody notices is now missing.
- **DON'T:** Keep both an old and a new linter permanently configured "just in case" once the migration is validated — running two overlapping tools indefinitely doubles CI time for checks that mostly duplicate each other, and their differing opinions on edge cases becomes a recurring source of contributor confusion about which tool's verdict to trust.
- **DO:** Periodically review which lint rules are actually enabled versus which are available — a linter's rule set grows over time, and a configuration written once and never revisited misses newer checks (including newer security and bug-pattern rules) that would otherwise be caught for free.

## Testing

### pytest Fundamentals

- **DO:** Use `pytest` as the default test framework for new Python projects. Its plain `assert`-based syntax, fixture system, and parametrization are more concise and expressive than the `unittest.TestCase` class-based, JUnit-derived style, while still running `unittest`-style tests when needed for legacy code.
```python
# unittest style — verbose
class TestAdd(unittest.TestCase):
    def test_add_positive(self):
        self.assertEqual(add(2, 3), 5)

# pytest style — a plain function and a plain assert
def test_add_positive():
    assert add(2, 3) == 5
```
- **DO:** Name test files `test_*.py` or `*_test.py` and test functions `test_*`, matching pytest's default discovery convention, so tests are found automatically without custom configuration.
- **DO:** Structure each test around Arrange-Act-Assert (set up inputs, perform the action under test, assert the outcome), with each phase visually distinct, even without literal comment labels — a test that interleaves setup and assertions is harder to scan.
```python
def test_apply_discount_reduces_price():
    # Arrange
    cart = Cart(items=[Item(price=100)])
    # Act
    cart.apply_discount(percent=10)
    # Assert
    assert cart.total == 90
```
- **DON'T:** Write a test whose name doesn't describe the behavior being verified (`test_1`, `test_case_a`, `test_function`). A failing test's name should tell you what broke before you even open the file.
```python
# Bad
def test_1():
    assert calculate_shipping(5, "US") == 12.5

# Good
def test_calculate_shipping_charges_flat_rate_for_us_orders():
    assert calculate_shipping(weight_kg=5, country="US") == 12.5
```
- **DO:** Assert on specific, meaningful values and, where relevant, on exception types/messages — not just `assert result` (truthiness) when a more precise assertion is available and would catch a wider range of regressions.
- **DON'T:** Assert on incidental implementation details (private attribute names, the exact order of dict keys before Python 3.7 guarantees, internal call counts that aren't part of the public contract) when the test's actual concern is externally observable behavior. Tests coupled to implementation details break on harmless refactors and stop being trustworthy.
- **DO:** Write one focused assertion (or a small, closely-related group of assertions checking one behavior) per test function, rather than combining several unrelated checks into one giant test — when a multi-assertion test fails, `pytest` reports only the first failing assertion by default, hiding what else might also be broken, and the test's name can no longer describe one clear behavior.
```python
# Bad — one test doing three unrelated things; a failure doesn't say which broke
def test_user_creation():
    user = create_user("ana@example.com", "Ana")
    assert user.email == "ana@example.com"
    order = create_order(user, items=[...])
    assert order.total == 100
    assert send_welcome_email(user) is True

# Good — each test verifies one behavior, and each has a name saying which
def test_create_user_sets_email():
    user = create_user("ana@example.com", "Ana")
    assert user.email == "ana@example.com"

def test_create_order_calculates_total():
    order = create_order(sample_user, items=[...])
    assert order.total == 100
```

### Fixtures

- **DO:** Use pytest fixtures (`@pytest.fixture`) for setup/teardown and shared test data, instead of `setUp`/`tearDown` methods or module-level globals mutated by tests. Fixtures are explicit (declared as test parameters), composable, and support proper scoping.
```python
@pytest.fixture
def db_session():
    session = create_test_session()
    yield session
    session.rollback()
    session.close()

def test_user_creation(db_session):
    user = User(name="Ana")
    db_session.add(user)
    db_session.commit()
    assert db_session.query(User).count() == 1
```
- **DO:** Choose fixture scope (`function` (default), `class`, `module`, `session`) deliberately based on cost and mutability — an expensive, read-only resource (a test database schema, a loaded ML model) can be `session`-scoped, but anything a test mutates should stay `function`-scoped to avoid state leaking between tests.
- **DON'T:** Give a broadly-scoped fixture mutable state that tests modify — a `session`-scoped fixture whose object is mutated by one test can make an unrelated, later test fail (or worse, silently pass for the wrong reason) depending on test execution order.
- **DO:** Put fixtures shared across multiple test files in a `conftest.py` at the appropriate directory level, so pytest auto-discovers them without explicit imports — that's the mechanism the framework provides specifically for shared fixtures.
- **DO:** Use pytest's built-in fixtures (`tmp_path` for a fresh temporary directory, `monkeypatch` for safely patching attributes/env vars/dict entries with automatic teardown, `capsys`/`capfd` for capturing stdout/stderr) instead of reimplementing the same tempfile or manual patch/restore logic per test.
```python
def test_writes_output_file(tmp_path):
    output_file = tmp_path / "result.txt"
    write_report(output_file, data={"total": 42})
    assert output_file.read_text() == "total: 42\n"

def test_reads_api_key_from_env(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-key-123")
    assert load_api_key() == "test-key-123"
```
- **DON'T:** Write to the real filesystem, a real network endpoint, or a shared/production database from a unit test. Use `tmp_path`, mocked HTTP clients, and an isolated test database (or in-memory equivalent) so tests are hermetic and safe to run in parallel and in any order.

### Parametrization

- **DO:** Use `@pytest.mark.parametrize` to run the same test logic across multiple input/expected-output pairs, instead of copy-pasting near-identical test functions or looping over cases manually inside one test (which reports only the first failure and hides how many cases actually failed).
```python
# Bad — a failure only shows the first mismatch, and adding a case means copy-pasting a function
def test_is_valid_email_1():
    assert is_valid_email("a@b.com") is True
def test_is_valid_email_2():
    assert is_valid_email("not-an-email") is False

# Good — pytest reports each case as its own result
@pytest.mark.parametrize("email, expected", [
    ("a@b.com", True),
    ("not-an-email", False),
    ("", False),
    ("a@b", True),
])
def test_is_valid_email(email, expected):
    assert is_valid_email(email) is expected
```
- **DO:** Give parametrize cases readable `ids` (either explicit `ids=[...]` or via a descriptive first element in the tuple) when the raw parameter values would render as unreadable test IDs in the output.
- **DON'T:** Parametrize a test with cases that actually exercise meaningfully different code paths requiring different assertions — that's a sign you need separate, clearly-named tests, not one parametrized test straining to cover unrelated behavior.

### Mocking & Test Doubles

- **DO:** Use `unittest.mock` (`Mock`, `MagicMock`, `patch`) — or the `pytest-mock` plugin's `mocker` fixture for automatic teardown — to isolate the unit under test from slow, flaky, or external dependencies (network calls, the current time, randomness, the filesystem).
```python
from unittest.mock import patch

def test_get_weather_handles_api_timeout():
    with patch("myapp.weather.requests.get", side_effect=TimeoutError):
        result = get_weather("Berlin")
    assert result is None
```
- **DO:** Patch a dependency at the location where it's *used* (looked up), not where it's *defined* — `patch("myapp.weather.requests")`, not `patch("requests")`, if `myapp.weather` did `import requests`. This is one of the most common sources of a mock that silently doesn't take effect.
- **DON'T:** Over-mock a test until it no longer exercises any real logic — if every collaborator is mocked and the assertions only check that mocks were called with certain arguments, the test verifies the mock setup, not the actual behavior of the code. Mock at the boundary (external I/O, network, time) and let the real logic under test run.
- **DO:** Use `spec=` (or `spec_set=`, or `create_autospec()`) when creating a `Mock`/`MagicMock` for an existing class or object, so the mock raises an `AttributeError` if test code calls a method that doesn't actually exist on the real object — an unconstrained `Mock()` happily accepts calls to any attribute name, silently masking a typo'd method name or a method that was renamed/removed from the real class.
```python
# Bad — a typo'd method name on the mock is silently accepted
mock_client = Mock()
mock_client.fecth_user.return_value = sample_user  # typo: "fecth" — no error here

# Good — spec constrains the mock to the real class's actual interface
mock_client = create_autospec(APIClient, instance=True)
mock_client.fecth_user.return_value = sample_user  # AttributeError: no such method
```
- **DON'T:** Assert that a mock "was called" as a substitute for verifying the actual outcome of the code under test, when checking the real outcome is possible and more meaningful — verifying `mock_send_email.called is True` confirms an email-sending function was invoked, but verifying it was called with the *correct* recipient and content is what actually protects against a real regression.
- **DO:** Use `assert_called_once_with(...)` and similar precise Mock assertions rather than the looser `assert mock.called`, when the specific arguments a collaborator was invoked with matter to correctness.
- **DON'T:** Leave a `patch()` active beyond the scope of the test that needs it (e.g., patching at module level without cleanup). Use `patch()` as a context manager, as a decorator, or via `mocker`/`monkeypatch` fixtures, all of which automatically undo the patch after the test — manual patch/restore code is easy to get wrong when a test fails partway through.
- **DO:** Prefer dependency injection (passing a collaborator as a constructor/function argument) over monkeypatching a module attribute, when designing new code — it makes substituting a test double trivial without needing `mock.patch` at all, and it makes the dependency visible in the function's signature.

### Coverage, Flakiness, and Test Design

- **DO:** Track test coverage (`pytest-cov` / `coverage.py`) as a signal for *finding untested code*, not as a target to be gamed. 100% line coverage doesn't mean the tests check the right things — a test that calls a function without asserting on its result still counts as "covered."
- **DON'T:** Use `time.sleep()` in a test to wait for an asynchronous operation, background thread, or eventual-consistency condition to finish. It's slow when the condition resolves quickly and flaky when it doesn't resolve within the fixed sleep — poll with a timeout, use the library's own synchronization primitives, or use a proper async test utility instead.
```python
# Bad — flaky under load, wastes time otherwise
worker.start()
time.sleep(2)
assert worker.is_done

# Good
worker.start()
wait_until(lambda: worker.is_done, timeout=5)
```
- **DO:** Keep unit tests independent and order-independent — running a single test file, a single test function, or the whole suite in a randomized order (pytest-randomly is useful for surfacing hidden ordering dependencies) should all pass identically.
- **DON'T:** Share mutable state between tests via module-level globals, class attributes, or files left over from a previous test run. Each test should set up everything it depends on and clean up after itself (or rely on fixture teardown to do so).
- **DO:** Write tests for edge cases and failure paths explicitly — empty input, `None`, boundary values, the exception a function is documented to raise — not only the "happy path" with typical input.
- **DO:** Consider property-based testing (`hypothesis`) for functions with a large or algebraic input space (parsers, serialization round-trips, math utilities) where example-based tests can only ever cover a hand-picked subset of cases and are prone to missing the exact edge case that breaks in production.
```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sorted_output_is_actually_sorted(items):
    result = my_sort(items)
    assert result == sorted(items)
```
- **DON'T:** Test a private, internal implementation function directly (`_internal_helper`) from outside its module as if it were part of the public contract, unless the module's own test suite is deliberately testing internals for a good reason (e.g., a genuinely complex private algorithm). Prefer testing through the public API so the test survives internal refactors.
- **DO:** Keep test execution fast — a slow test suite gets skipped, run less often, or run only in CI, which delays feedback. Separate genuinely slow integration/end-to-end tests (marked with `@pytest.mark.slow` or placed in a separate directory) from the fast unit test suite developers run on every save.
- **DO:** Run the test suite in parallel (`pytest-xdist`'s `-n auto`) once it's grown large enough that wall-clock time meaningfully affects developer feedback loops or CI cost — but first ensure tests are actually isolated (no shared mutable fixtures, no fixed temp-file names, no dependence on execution order) since parallel execution surfaces exactly those hidden coupling bugs that a serial run can mask.
- **DON'T:** Treat a flaky test (one that fails intermittently with no code change) as acceptable background noise to be re-run until it passes. A flaky test that's tolerated erodes trust in the whole suite — once "just re-run it" becomes normal, a genuine new failure is far more likely to be dismissed the same way. Investigate and fix (or, as a last resort, explicitly quarantine with a tracked ticket) flaky tests promptly rather than normalizing retries.

### Test Data Builders & Factories

- **DO:** Use a factory library (`factory_boy`) or a small hand-written builder function to construct test objects with sensible defaults, overriding only the fields a specific test cares about — this keeps each test focused on what it's actually verifying instead of repeating a full, irrelevant object construction in every test.
```python
# Bad — every test repeats the full constructor, burying what actually matters
def test_discount_applies_to_active_user():
    user = User(id=1, name="Ana", email="ana@example.com", status="active", created_at=datetime.now())
    assert calculate_discount(user) == 0.1

# Good — a factory supplies sane defaults; the test only overrides what it cares about
def test_discount_applies_to_active_user():
    user = UserFactory(status="active")
    assert calculate_discount(user) == 0.1
```
- **DON'T:** Copy-paste a large, realistic-looking fixture object (e.g., a giant hand-typed dict mimicking an API response) into multiple test files. Centralize it as a fixture or factory so it's updated in one place when the underlying shape changes, instead of drifting into several slightly-different copies.
- **DO:** Use snapshot testing (e.g., `syrupy`, `pytest-snapshot`) deliberately and sparingly — for output that's genuinely large/complex and stable (rendered templates, serialized structures) — not as a default substitute for writing real assertions. An unreviewed snapshot update (`--snapshot-update`) can silently "bless" a regression as the new expected output if nobody actually reads the diff before approving it.

### Testing Async Code & CI Matrices

- **DO:** Use a dedicated HTTP-mocking library (`responses` for the `requests` library, `respx` for `httpx`) to intercept and mock outbound HTTP calls at the transport level in tests, rather than mocking the client object's methods directly — a transport-level mock exercises the same request-building and response-parsing code paths as production, catching bugs that mocking `client.get` entirely would miss.
```python
import responses

@responses.activate
def test_fetch_user_parses_response():
    responses.add(
        responses.GET,
        "https://api.example.com/users/42",
        json={"id": 42, "name": "Ana"},
        status=200,
    )
    user = fetch_user(42)
    assert user.name == "Ana"

@responses.activate
def test_fetch_user_raises_on_500():
    responses.add(responses.GET, "https://api.example.com/users/42", status=500)
    with pytest.raises(UserServiceUnavailableError):
        fetch_user(42)
```
- **DON'T:** Let tests make real HTTP calls to a live third-party API. Beyond being slow and flaky (subject to the real service's uptime, rate limits, and network conditions), it also means tests can fail for reasons that have nothing to do with the code under test, and a test suite that depends on network access can't run in an isolated/offline CI environment.
- **DO:** Use `pytest-asyncio` (with `@pytest.mark.asyncio` or its `asyncio_mode = "auto"` config) to write `async def` test functions that directly `await` the async code under test, instead of wrapping every async call in `asyncio.run()` inside a synchronous test.
```python
# Bad — manually driving the event loop inside a sync test
def test_fetch_user():
    result = asyncio.run(fetch_user(42))
    assert result.id == 42

# Good — with pytest-asyncio configured
@pytest.mark.asyncio
async def test_fetch_user():
    result = await fetch_user(42)
    assert result.id == 42
```
- **DO:** Use `unittest.mock.AsyncMock` (or `pytest-mock`'s async-aware patching) when mocking a coroutine function — a plain `Mock`/`MagicMock` used in place of an `async def` returns a `Mock` object instead of something awaitable, which raises a `TypeError` the moment the code under test tries to `await` it.
```python
# Bad — patch() defaults to MagicMock, which isn't awaitable
with patch("myapp.client.fetch", return_value={"id": 1}):
    await fetch_and_process()  # TypeError: object dict can't be used in 'await'

# Good
with patch("myapp.client.fetch", new_callable=AsyncMock, return_value={"id": 1}):
    await fetch_and_process()
```
- **DO:** Run the test suite against every Python version the project claims to support (via a CI matrix, or locally with `tox`/`nox`) rather than only the version installed on the primary developer's machine — a feature or syntax available in one supported version but not another is otherwise only caught by an end user on the unsupported version.
- **DO:** Run tests in CI against the exact dependency versions pinned in the lockfile, and separately consider a periodic (not every-commit) job that tests against the latest allowed dependency versions, to catch upstream breaking changes before they surprise a routine dependency bump.

### Contract & Compatibility Testing

- **DO:** Add a contract test (verifying the actual shape of a request/response against a shared schema, such as an OpenAPI spec or a Pact contract) at the boundary between two independently-deployed services, when both sides are maintained by different teams or deployed on independent schedules — this catches an incompatible API change before it reaches production, rather than relying solely on each side's own unit tests (which can't see how the other side actually behaves).
- **DON'T:** Assume that both sides of an internal API integration will always be tested and deployed together just because they're in the same organization — independently deployable services can (and eventually will) drift out of sync, and a contract test is what catches that drift before it becomes a production incident.
- **DO:** Version an API deliberately (in the URL, a header, or the payload) when a breaking change is unavoidable, and maintain the previous version for a defined deprecation window — this gives consumers (including ones you don't directly control) time to migrate, rather than forcing an instantaneous, uncoordinated cutover.

### Testing CLI Applications

- **DO:** Use a CLI framework's built-in test runner (`click.testing.CliRunner`, `typer.testing.CliRunner`) to invoke commands in-process and assert on exit codes and output, instead of shelling out to a real subprocess for every test — in-process invocation is dramatically faster and gives direct access to raised exceptions for debugging test failures.
```python
from click.testing import CliRunner
from myapp.cli import cli

def test_greet_command_outputs_expected_message():
    runner = CliRunner()
    result = runner.invoke(cli, ["greet", "--name", "Ana"])
    assert result.exit_code == 0
    assert "Hello, Ana!" in result.output

def test_greet_command_fails_without_required_option():
    runner = CliRunner()
    result = runner.invoke(cli, ["greet"])
    assert result.exit_code != 0
```
- **DO:** Use pytest's `capsys` fixture to capture and assert on stdout/stderr for a plain `argparse`-based CLI with no dedicated test runner, instead of manually redirecting `sys.stdout` and restoring it by hand.
```python
def test_main_prints_result(capsys):
    main(["--input", "5"])
    captured = capsys.readouterr()
    assert captured.out.strip() == "Result: 25"
```
- **DON'T:** Test a CLI's behavior only by manually running it once during development and eyeballing the output. Automate CLI tests the same as any other code path — argument parsing, exit codes, and error messages are exactly the kind of thing that silently breaks during a refactor if nothing actually exercises them.

### Test File Organization

- **DO:** Mirror the source package's structure in the test directory (`src/myapp/services/orders.py` tested by `tests/services/test_orders.py`) so any contributor can find the tests for a given module by pattern alone, without needing a mental index of where things ended up.
```
src/myapp/
    services/
        orders.py
        payments.py
tests/
    services/
        test_orders.py
        test_payments.py
    conftest.py
```
- **DON'T:** Dump every test into one flat `tests.py` or a single giant `test_all.py` file once a project has grown past a trivial size — a monolithic test file becomes slow to navigate, produces noisy diffs when multiple people edit unrelated tests, and often ends up with tests unintentionally sharing state through module-level fixtures.
- **DO:** Keep one test module's tests scoped to one source module's behavior, with a `conftest.py` at each directory level holding only the fixtures genuinely shared across that scope — a fixture used by exactly one test file belongs in that file, not promoted to a shared `conftest.py` "for consistency."

### Unit, Integration, and End-to-End Tests

- **DO:** Keep the test suite's shape roughly pyramid-like — many fast, isolated unit tests; fewer integration tests exercising real collaboration between components (a real test database, a real HTTP client against a local server); fewer still full end-to-end tests driving the whole system. Each layer catches different bugs, and the fast layer should be the one run most frequently.
- **DON'T:** Rely exclusively on end-to-end tests to catch bugs that a unit test would catch far faster and more precisely. A slow, broad E2E suite that's the *only* safety net makes every change expensive to verify and pushes feedback minutes or hours away from the moment the bug was introduced.
- **DO:** Mark and separate tests by speed/scope (`@pytest.mark.unit`, `@pytest.mark.integration`, `@pytest.mark.e2e`, or separate directories) so a developer can run just the fast unit tests during active development and reserve the full suite for CI or pre-merge.
- **DON'T:** Call a test an "integration test" when it's actually a unit test with unnecessary real I/O attached (e.g., hitting a real database for a test that doesn't need to verify database-specific behavior at all). Extra I/O without a corresponding reason to test that boundary just makes the test slower and flakier for no added coverage.

### Testing Exceptions & Error Paths

- **DO:** Use `pytest.raises(SomeException)` as a context manager to assert that a specific exception type is raised, and additionally match on the message with `match=` (a regex against `str(exc)`) when the specific error content matters, not just the exception's type.
```python
import pytest

def test_withdraw_raises_when_amount_exceeds_balance():
    account = Account(balance=50)
    with pytest.raises(InsufficientFundsError, match="insufficient funds"):
        account.withdraw(100)

def test_withdraw_error_carries_structured_data():
    account = Account(balance=50)
    with pytest.raises(InsufficientFundsError) as exc_info:
        account.withdraw(100)
    assert exc_info.value.requested == 100
    assert exc_info.value.available == 50
```
- **DON'T:** Wrap an entire multi-statement test body in one `pytest.raises()` block when only one specific statement is expected to raise — later statements inside the block become dead code that will never actually execute once the exception fires, silently hiding the fact that they're untested.
```python
# Bad — the assert on line 2 never actually runs; the exception on line 1 exits the block first
with pytest.raises(ValueError):
    parsed = parse_config(bad_input)
    assert parsed.mode == "default"  # unreachable, gives false confidence

# Good — isolate exactly the statement expected to raise
with pytest.raises(ValueError):
    parse_config(bad_input)
```
- **DO:** Test that a function does *not* raise, for input that's a valid edge case a previous bug incorrectly treated as an error — an explicit "this should succeed" regression test is as valuable as an explicit "this should fail" test when a bug specifically involved an incorrect exception being raised.

### Fixture Composition & Indirect Parametrization

- **DO:** Compose fixtures by having one fixture depend on another (a fixture function taking another fixture as a parameter) to build up test setup in layers, instead of one large fixture that does everything — this lets individual tests depend on only the layer they need.
```python
@pytest.fixture
def db_engine():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()

@pytest.fixture
def db_session(db_engine):
    Session = sessionmaker(bind=db_engine)
    session = Session()
    yield session
    session.rollback()
    session.close()

@pytest.fixture
def sample_user(db_session):
    user = User(name="Ana", email="ana@example.com")
    db_session.add(user)
    db_session.commit()
    return user
```
- **DO:** Use `pytest.fixture(params=[...])` (indirect parametrization) when the same test logic needs to run once per fixture variant (e.g., once against SQLite and once against Postgres, or once per supported serialization format), rather than duplicating the test function for each variant.
```python
@pytest.fixture(params=["json", "yaml", "toml"])
def config_format(request):
    return request.param

def test_round_trip_serialization(config_format):
    data = {"key": "value"}
    serialized = serialize(data, format=config_format)
    assert deserialize(serialized, format=config_format) == data
```

### Common pytest Mistakes

- **DON'T:** Mark a fixture `autouse=True` unless every single test in its scope genuinely needs it to run. An overused `autouse` fixture becomes invisible magic — a test can depend on setup that isn't mentioned anywhere in its own signature, which makes the test harder to understand in isolation and harder to debug when that hidden setup is the actual cause of a failure.
```python
# Bad — every test silently gets a mocked clock, whether it needs one or not,
# and nothing in a failing test's signature reveals that this is happening
@pytest.fixture(autouse=True)
def freeze_time():
    with freeze_time_lib.freeze_time("2026-01-01"):
        yield

# Good — explicit; only tests that need frozen time opt in
@pytest.fixture
def frozen_time():
    with freeze_time_lib.freeze_time("2026-01-01"):
        yield

def test_expiry_calculation(frozen_time):
    ...
```
- **DON'T:** Define a fixture and a test in a way that creates a dependency cycle or an unclear dependency chain across multiple `conftest.py` files at different directory levels — keep fixture dependencies shallow and, when a fixture is only relevant to one test module, define it in that module rather than promoting it to a shared `conftest.py` "just in case."
- **DON'T:** Forget that a fixture using `yield` for teardown only runs its teardown code if the setup portion (before `yield`) completed without raising — an exception during setup skips teardown entirely, which matters if setup partially acquires a resource before failing.
- **DO:** Use `pytest.ini`/`pyproject.toml`'s `[tool.pytest.ini_options]` to register custom markers explicitly (`markers = ["slow: marks tests as slow"]`) — an unregistered marker still works but emits a warning on every run, and explicit registration also gives contributors a discoverable, documented list of the markers the project actually uses.

### Mocking Time, Randomness, and External Clocks

- **DO:** Inject the current time as a parameter (or through an injectable clock object) rather than calling `datetime.now()`/`time.time()` directly inside business logic, so tests can supply a fixed, known time instead of needing to patch a built-in module — this also makes the function's dependency on "the current time" visible in its signature.
```python
# Harder to test — the function's behavior secretly depends on wall-clock time
def is_subscription_expired(subscription) -> bool:
    return datetime.now() > subscription.expires_at

# Easier to test — the time dependency is explicit and injectable
def is_subscription_expired(subscription, now: datetime) -> bool:
    return now > subscription.expires_at

def test_is_subscription_expired_after_expiry_date():
    subscription = SubscriptionFactory(expires_at=datetime(2026, 1, 1))
    assert is_subscription_expired(subscription, now=datetime(2026, 6, 1)) is True
```
- **DO:** When calling time directly is unavoidable (deep in a large existing codebase, or a third-party library that calls it internally), use `freezegun`/`time-machine` to freeze or control time for the duration of a test, rather than making a test's pass/fail status depend on when it happens to run.
- **DON'T:** Write a test whose outcome depends on real, unmocked randomness or real, unmocked wall-clock time in a way that makes the test intermittently fail (a test asserting behavior "close to now" without an explicit tolerance, or a probabilistic assertion with no fixed seed). Seed random number generators explicitly (`random.seed(...)`, or better, inject a `Random` instance) in any test exercising code that uses randomness.

## Async Python

- **DO:** Use `async`/`await` (and `asyncio`) when a program's bottleneck is I/O-bound concurrency — many simultaneous network requests, database queries, or socket connections — where tasks spend most of their time waiting, not computing. For CPU-bound work, `asyncio` provides no speedup on its own; reach for multiprocessing or a native extension instead.
- **DON'T:** Call a blocking, synchronous function (`time.sleep`, a synchronous `requests.get`, blocking file I/O, a CPU-heavy pure-Python loop) directly inside an `async def` coroutine. It blocks the single-threaded event loop, stalling every other coroutine scheduled on it — not just the one making the call.
```python
# Bad — blocks the entire event loop for 2 seconds
async def fetch_data():
    time.sleep(2)
    return requests.get(url).json()

# Good — yields control back to the event loop while waiting
async def fetch_data():
    await asyncio.sleep(2)
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()
```
- **DO:** Use `asyncio.to_thread()` (3.9+) or `loop.run_in_executor()` to run a genuinely blocking or CPU-bound call from within async code, so it executes on a separate thread instead of stalling the event loop.
```python
async def process_image(path: Path) -> bytes:
    # cpu_heavy_resize is a synchronous, CPU-bound function
    return await asyncio.to_thread(cpu_heavy_resize, path)
```
- **DON'T:** Create a "fire and forget" task with `asyncio.create_task()` and immediately let the returned `Task` object go out of scope without keeping a reference to it. CPython's event loop only holds a *weak* reference internally, so the task can be garbage-collected mid-execution, silently cancelling it.
```python
# Bad — the Task object has no strong reference and may be GC'd before it finishes
async def handle_request():
    asyncio.create_task(log_analytics_event())

# Good — keep a reference (e.g., in a set owned by the enclosing scope)
background_tasks: set[asyncio.Task] = set()

async def handle_request():
    task = asyncio.create_task(log_analytics_event())
    background_tasks.add(task)
    task.add_done_callback(background_tasks.discard)
```
- **DO:** Use `asyncio.gather()` (or `asyncio.TaskGroup` on 3.11+) to run independent coroutines concurrently instead of `await`-ing them one after another, when there's no dependency between them — sequential awaiting of independent I/O calls throws away the entire benefit of async.
```python
# Bad — sequential, each await blocks until the previous completes
user = await fetch_user(user_id)
orders = await fetch_orders(user_id)
reviews = await fetch_reviews(user_id)

# Good — all three requests run concurrently
user, orders, reviews = await asyncio.gather(
    fetch_user(user_id), fetch_orders(user_id), fetch_reviews(user_id)
)
```
- **DO:** Prefer `asyncio.TaskGroup` (3.11+) over `asyncio.gather()` for new code when you want structured concurrency — it ensures all child tasks are awaited or cancelled together and surfaces multiple failures as an `ExceptionGroup`, instead of `gather()`'s default behavior of only propagating the first exception while other tasks keep running unsupervised.
- **DON'T:** Assume `asyncio.gather()` cancels the other coroutines the moment one of them raises — by default it doesn't; the remaining awaitables keep running in the background unless you explicitly handle cancellation, which can surprise code that expected an early exit on first failure.
- **DO:** Use an async-native library for any I/O the async code path depends on (`httpx.AsyncClient` or `aiohttp` for HTTP, `asyncpg`/`aiomysql` or an async ORM driver for databases, `aiofiles` for file I/O when it matters) rather than calling a synchronous library's blocking API inside a coroutine.
- **DO:** Use `async with` and `async for` for async context managers and async iterators respectively (e.g., an async database connection pool, an async generator streaming paginated API results) — using the plain `with`/`for` forms on an async-only object either fails immediately or silently does the wrong thing depending on the library.
- **DON'T:** Mix a blocking synchronous call stack with `asyncio.run()` calls nested inside it, or call `asyncio.run()` more than once for what's conceptually one continuous async workflow. `asyncio.run()` creates a fresh event loop and tears it down when it returns; calling it repeatedly (or from within an already-running loop) is inefficient at best and raises `RuntimeError: asyncio.run() cannot be called from a running event loop` at worst.
- **DO:** Set explicit timeouts on awaited I/O (`asyncio.timeout()` on 3.11+, `asyncio.wait_for()` earlier, or the HTTP client's own timeout parameter) rather than letting a coroutine await indefinitely on a hung connection.
```python
async def fetch_with_timeout(url: str) -> dict:
    async with asyncio.timeout(5):
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            return response.json()
```
- **DON'T:** Use a plain `threading.Lock` inside async code — it blocks the thread (and therefore the whole event loop) while waiting. Use `asyncio.Lock`, `asyncio.Semaphore`, or `asyncio.Queue`, which are designed to cooperatively yield control instead of blocking.
- **DO:** Handle `asyncio.CancelledError` deliberately when a coroutine needs to run cleanup on cancellation (closing a connection, releasing a resource) — catch it, do the cleanup, and re-raise it, since swallowing `CancelledError` breaks the cancellation contract that callers (and `TaskGroup`/`gather`) depend on.
```python
async def worker():
    try:
        await do_long_running_work()
    except asyncio.CancelledError:
        await cleanup()
        raise
```
- **DO:** Limit concurrency with `asyncio.Semaphore` (or a bounded task group) when firing off many concurrent requests to a rate-limited or resource-constrained external service, instead of launching an unbounded number of simultaneous tasks with `gather()` over a large input list.
```python
sem = asyncio.Semaphore(10)

async def fetch_one(url):
    async with sem:
        async with httpx.AsyncClient() as client:
            return await client.get(url)

results = await asyncio.gather(*(fetch_one(u) for u in urls))
```
- **DON'T:** Define a function as `async def` when it never actually awaits anything. An unnecessary `async def` forces every caller to await it (or wrap it) for no benefit, and signals a concurrency behavior that doesn't exist.
- **DO:** Understand that `asyncio` concurrency in standard CPython is single-threaded cooperative multitasking — it interleaves I/O-bound work on one OS thread, but doesn't parallelize CPU-bound Python code the way multiple processes (or a GIL-free build) do. Reach for `multiprocessing`, `concurrent.futures.ProcessPoolExecutor`, or moving hot code into a compiled extension for CPU-bound parallelism.

### Structured Concurrency in Depth

- **DO:** Prefer `asyncio.TaskGroup` (3.11+) as the default way to launch and supervise a set of related child tasks, since it guarantees that if one task fails, the others are cancelled and awaited before the surrounding `async with` block exits — you can't accidentally leave an orphaned task running after an error, which is a real risk with manually tracked `create_task()` calls.
```python
# Good — structured: all three tasks are guaranteed to be resolved (completed,
# failed, or cancelled) before this block exits; a failure in one cancels the rest
async def sync_all_accounts(account_ids: list[int]) -> None:
    async with asyncio.TaskGroup() as tg:
        for account_id in account_ids:
            tg.create_task(sync_account(account_id))
```
- **DO:** Catch `ExceptionGroup` (or use `except*`, Python 3.11+) around a `TaskGroup` block when individual task failures need distinct handling, since a `TaskGroup` that has multiple failing children raises one `ExceptionGroup` bundling all of them, not just the first.
```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(sync_account(1))
        tg.create_task(sync_account(2))
except* ConnectionError as eg:
    for exc in eg.exceptions:
        logger.warning("account sync failed: %s", exc)
```
- **DON'T:** Assume cancelling a parent task automatically stops all the blocking work its children are doing — cancellation in `asyncio` only takes effect at the next `await` point. A coroutine executing a long synchronous computation (or an `await` on something that doesn't check for cancellation, like certain C-extension calls) won't actually stop until it reaches a cancellable await, so genuinely long CPU-bound spans still need to be chunked or offloaded to a thread/process to remain responsive to cancellation.
- **DO:** Use `asyncio.shield()` deliberately, and sparingly, when a specific sub-operation must run to completion even if its parent task is cancelled (e.g., flushing a write that's already in progress) — but be aware it can leave a "shielded" operation running after the code that launched it has moved on, so pair it with an explicit plan for what happens to its result.

### Streaming & Backpressure

- **DO:** Use `asyncio.Queue` to connect a producer coroutine to one or more consumer coroutines when the production rate might outpace consumption, instead of an unbounded in-memory list — a bounded `Queue` (via `maxsize`) applies natural backpressure, making the producer `await` (and pause) when the queue is full rather than growing memory usage without limit.
- **DO:** Use `asyncio.as_completed()` when processing a batch of concurrent awaitables and you want to handle each result as soon as it's ready, rather than waiting for all of them via `gather()` before processing any — useful when downstream processing of an early-finishing task shouldn't wait on a slower sibling task.
```python
async def fetch_all_and_process(urls: list[str]) -> None:
    tasks = [asyncio.create_task(fetch(url)) for url in urls]
    for coro in asyncio.as_completed(tasks):
        result = await coro  # handles whichever task finishes first, in completion order
        await process(result)
```
- **DON'T:** Assume `asyncio.as_completed()` preserves the original input order in the results it yields — it yields in completion order, not input order, which is an easy source of subtly mismatched results if calling code assumes otherwise (`gather()`, by contrast, does preserve input order in its final returned list).
```python
async def producer(queue: asyncio.Queue):
    for item in generate_items():
        await queue.put(item)  # blocks here if the queue is full — backpressure
    await queue.put(None)  # sentinel signaling completion

async def consumer(queue: asyncio.Queue):
    while (item := await queue.get()) is not None:
        await process(item)
```
- **DON'T:** Read an entire large streamed HTTP response or file into memory (`response.content`, `.read()`) when only incremental processing is actually needed. Use the client library's streaming interface (`httpx`'s `.aiter_bytes()`/`.aiter_lines()`, or a file's async line iteration) to process data incrementally and keep memory bounded regardless of the source's total size.

### Bridging Synchronous and Asynchronous Code

- **DO:** Use `asyncio.run()` exactly once, at the top-level entry point of an async program, to start the event loop and run the top-level coroutine to completion — everything else in the program should be reached through `await` from within that running loop, not by calling `asyncio.run()` again from a nested function.
```python
# Good — a single, top-level entry point
async def main() -> None:
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(*(fetch(client, url) for url in urls))
    print(results)

if __name__ == "__main__":
    asyncio.run(main())
```
- **DO:** Use `asyncio.to_thread()` (or `loop.run_in_executor()`) when synchronous code needs to call into a library that only offers a blocking API, rather than blocking the event loop directly — already covered above, but the reverse direction matters too: when synchronous code needs to call an `async def` function, run it with `asyncio.run()` only if there's no event loop already active in that thread; if one might be, that's a sign the calling code itself should become async, or should hand the coroutine to a dedicated background event-loop thread instead.
- **DON'T:** Try to call an `async def` function directly as if it were synchronous (`result = fetch_data()` without `await`). This doesn't raise an error immediately — it silently returns a coroutine *object* instead of running the coroutine's body, which is a distinctively confusing failure mode because no exception fires at the call site; the bug only surfaces later when code tries to use the un-awaited coroutine object as if it were the real result.
```python
# Bad — returns a coroutine object, not the fetched data; no error here
def get_data():
    return fetch_data()  # missing await; fetch_data() body never actually runs

result = get_data()
print(result.status)  # AttributeError: 'coroutine' object has no attribute 'status'

# Good
async def get_data():
    return await fetch_data()
```
- **DO:** Treat "RuntimeWarning: coroutine '...' was never awaited" as a bug to fix immediately, not a warning to ignore — it means a coroutine was created but its body never ran, which is very often silently dropping real work (an unsent request, a skipped database write).

### Async Generators & Iteration

- **DO:** Write an `async def` generator function (using `yield` inside it) to expose a lazily-produced, asynchronously-fetched sequence — such as paginated API results — and consume it with `async for`, instead of eagerly awaiting and collecting every page into a list before the caller can process any of it.
```python
async def iter_all_users(client: httpx.AsyncClient) -> AsyncIterator[User]:
    page = 1
    while True:
        response = await client.get("/users", params={"page": page})
        data = response.json()
        if not data["results"]:
            return
        for raw_user in data["results"]:
            yield User(**raw_user)
        page += 1

async def main():
    async for user in iter_all_users(client):
        await process(user)  # starts processing the first page immediately
```
- **DON'T:** Mix a plain generator (`def` with `yield`) and an async generator (`async def` with `yield`) interchangeably, or attempt to iterate an async generator with a plain `for` loop — an async generator requires `async for`, and a regular `for` loop raises a `TypeError` immediately because the object doesn't implement the synchronous iterator protocol.

### Custom Async Context Managers

- **DO:** Implement `__aenter__`/`__aexit__` (or use `contextlib.asynccontextmanager` for the generator-based shorthand) for a resource whose setup/teardown itself needs to `await` something — an async database connection, an async lock, an async-native network client — so it can be used naturally with `async with` and get correct cleanup on both normal exit and exception.
```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def acquire_connection(pool):
    conn = await pool.acquire()
    try:
        yield conn
    finally:
        await pool.release(conn)

async def run_query(pool, sql):
    async with acquire_connection(pool) as conn:
        return await conn.fetch(sql)
```
- **DON'T:** Implement a synchronous `__enter__`/`__exit__` on a class whose setup/teardown actually needs to await something, just to make it usable with a plain `with` — this either forces a blocking call inside what should be async code, or silently skips awaiting work that needed to happen. If teardown needs to `await`, the type needs `__aenter__`/`__aexit__` and `async with`, not a synchronous shim.

### Long-Lived Connections: WebSockets & Polling

- **DO:** Structure a WebSocket handler as a loop that awaits incoming messages and reacts to them, running for the lifetime of the connection, rather than treating it like a single request/response call — and wrap the receive loop so that a client disconnect (typically surfaced as a specific exception from the WebSocket library) breaks the loop and runs cleanup instead of propagating as an unhandled error on every routine disconnect.
```python
async def websocket_handler(websocket):
    await register(websocket)
    try:
        async for message in websocket:
            await handle_message(websocket, message)
    except ConnectionClosed:
        pass  # a client disconnecting is a normal, expected event, not an error
    finally:
        await unregister(websocket)
```
- **DON'T:** Poll a resource in a tight loop with no delay (`while True: check_status()`) waiting for a state change, whether from sync or async code — this wastes CPU and can hammer whatever's being polled. Use an appropriate wait/backoff interval, or better, an actual notification mechanism (a webhook, a pub/sub subscription, a WebSocket push) when one is available instead of polling at all.
- **DO:** Set a heartbeat/ping-pong mechanism on long-lived WebSocket connections (many WebSocket libraries provide this built in) to detect and clean up dead connections that disconnected without a clean close handshake (a client that lost network connectivity abruptly) — without it, a server can accumulate open-but-dead connection objects indefinitely.

### Async Database Access Patterns

- **DO:** Use a connection pool (`asyncpg.create_pool()`, SQLAlchemy's async engine with its built-in pooling) rather than opening a new database connection per request — connection setup (including any TLS handshake) has real latency, and an unbounded number of concurrent connections can overwhelm the database under load.
```python
import asyncpg

async def create_pool() -> asyncpg.Pool:
    return await asyncpg.create_pool(dsn=DATABASE_URL, min_size=5, max_size=20)

async def get_user(pool: asyncpg.Pool, user_id: int) -> dict | None:
    async with pool.acquire() as conn:
        return await conn.fetchrow("SELECT * FROM users WHERE id = $1", user_id)
```
- **DON'T:** Share a single database connection object across concurrently-running coroutines/tasks without a lock or a pool — most database drivers' connection objects aren't safe for concurrent use from multiple coroutines at once, and interleaved queries on one shared connection can produce corrupted results or protocol errors.
- **DO:** Use parameterized queries (`$1`, `$2`, or the driver's placeholder style) with async database drivers exactly as with synchronous ones — the SQL-injection risk from string-interpolated queries (covered under Security) applies identically regardless of whether the call is awaited.
- **DON'T:** Wrap a synchronous ORM's blocking database calls in `async def` functions without actually using an async-capable driver/ORM underneath — an `async def` function that calls a synchronous SQLAlchemy session's `.query()` still blocks the event loop for the duration of that call, providing none of async's concurrency benefit despite the `async`/`await` syntax being present.

## Performance

- **DON'T:** Use a mutable object as a default argument value (covered in Idiomatic Python above) — worth repeating here specifically as a performance/correctness footgun, since the resulting "shared state" bug often first shows up as a mysterious performance or memory issue (an ever-growing list) rather than an obvious crash.
- **DON'T:** Rely on module-level global mutable state (a global dict cache, a global counter, a global list accumulating results) as the default way to share data across functions. It makes code harder to test in isolation, breaks under concurrent/parallel execution, and creates hidden coupling between unrelated call sites. Pass state explicitly (function arguments, a class instance, a context object) instead.
```python
# Bad — hidden shared mutable state, not thread-safe, hard to test in isolation
_cache = {}
def get_user(user_id):
    if user_id not in _cache:
        _cache[user_id] = db.query(user_id)
    return _cache[user_id]

# Good — cache is an explicit, owned object
class UserRepository:
    def __init__(self, db):
        self._db = db
        self._cache: dict[int, User] = {}

    def get_user(self, user_id: int) -> User:
        if user_id not in self._cache:
            self._cache[user_id] = self._db.query(user_id)
        return self._cache[user_id]
```
- **DO:** Use `functools.lru_cache` (or `functools.cache` on 3.9+) to memoize a pure function with a bounded, hashable input space instead of hand-rolling a global dict cache — it's a single decorator, thread-safe, and comes with a `.cache_clear()`/`.cache_info()` for introspection and testing.
```python
from functools import lru_cache

@lru_cache(maxsize=1024)
def fibonacci(n: int) -> int:
    return n if n < 2 else fibonacci(n - 1) + fibonacci(n - 2)
```
- **DON'T:** Check membership with `in` against a `list` inside a hot loop when the list is large and checked repeatedly — list membership is O(n) per check. Use a `set` or `dict` (O(1) average-case lookup) when membership testing is the operation you actually need.
```python
# Bad — O(n) per lookup, O(n*m) overall
allowed = ["gold", "platinum", "diamond"]
for user in users:
    if user.tier in allowed:  # linear scan every time
        notify(user)

# Good — O(1) per lookup
allowed = {"gold", "platinum", "diamond"}
for user in users:
    if user.tier in allowed:
        notify(user)
```
- **DO:** Vectorize numeric operations over large arrays with `numpy`/`pandas` (or another array library) rather than iterating element-by-element in a pure-Python `for` loop. A vectorized operation runs the loop in compiled C rather than the CPython interpreter, often 10-100x faster for large data.
```python
# Bad — pure-Python loop over a NumPy array defeats the point of using NumPy
total = 0
for x in large_array:
    total += x * x

# Good
total = (large_array ** 2).sum()
```
- **DON'T:** Reach for `pandas`/`numpy` and vectorization as a reflex on small, one-off collections where a plain Python loop or comprehension is both fast enough and clearer. Vectorization is a real-data-volume optimization, not a universal default — profile before optimizing.
- **DON'T:** Use `DataFrame.apply()` with a row-wise Python function as the default way to transform a pandas column — it's implemented as a Python-level loop under the hood and gives up most of pandas' performance advantage. Prefer a vectorized operation on the underlying Series/columns directly, and reserve `.apply()` for logic that genuinely can't be expressed as a vectorized operation.
```python
# Bad — row-wise apply falls back to a slow Python-level loop
df["total"] = df.apply(lambda row: row["price"] * row["quantity"], axis=1)

# Good — vectorized column operation, runs in compiled code
df["total"] = df["price"] * df["quantity"]
```
- **DON'T:** Grow a pandas DataFrame by repeatedly calling `pd.concat()` (or the removed `.append()`) inside a loop, one row/chunk at a time — like list concatenation, this reallocates the underlying array on every call. Collect rows into a plain Python list (or a list of DataFrames) and concatenate once at the end.
- **DO:** Avoid repeated attribute or dict-key lookups inside a hot loop by binding them to a local variable once beforehand, since local variable access is faster than repeated attribute resolution in CPython's bytecode.
```python
# Slightly slower — re-resolves self.config and its nested keys every iteration
for row in rows:
    process(row, self.config["threshold"], self.config["mode"])

# Faster — resolved once
threshold = self.config["threshold"]
mode = self.config["mode"]
for row in rows:
    process(row, threshold, mode)
```
- **DO:** Profile before optimizing — use `cProfile`/`py-spy`/`line_profiler` (or `timeit` for microbenchmarks) to find the actual bottleneck rather than guessing. Intuition about where Python code spends time is frequently wrong, and "optimizing" a function that accounts for 0.1% of runtime wastes effort and adds unnecessary complexity for no measurable gain.
- **DON'T:** Build a large string by repeated `+=` concatenation in a loop (covered under idiomatic Python's `str.join` bullet) — worth flagging again here specifically as a Big-O performance trap, since it silently degrades from linear to quadratic time as input size grows and often isn't noticed until production data volume exposes it.
- **DO:** Use generators and lazy iteration (covered under Idiomatic Python) for large datasets specifically *because* it avoids allocating memory for the whole collection up front — this is as much a performance consideration as a style one, and matters most exactly when data size is large enough for performance to matter.
- **DON'T:** Call a function that does redundant, repeated work inside a loop when the result doesn't change between iterations (an unchanging regex compile, a repeated file read, a repeated expensive computation with the same arguments). Hoist the invariant computation out of the loop, or memoize it.
- **DO:** Use `bisect.insort`/`bisect.bisect` to maintain a sorted list with O(log n) lookups and O(n) insertion (still faster than re-sorting from scratch each time), instead of appending and re-sorting the whole collection after every insertion when a running sorted order needs to be maintained.
```python
# Bad — re-sorts the entire list after every single insertion
scores = []
for score in incoming_scores:
    scores.append(score)
    scores.sort()

# Good — binary-search insertion keeps the list sorted incrementally
import bisect
scores = []
for score in incoming_scores:
    bisect.insort(scores, score)
```
- **DO:** Use `bisect.bisect_left`/`bisect.bisect_right` for O(log n) lookups into an already-sorted sequence (finding an insertion point, a range boundary) instead of a linear scan (`for i, x in enumerate(sorted_list): if x >= target: break`), which is O(n) and unnecessary once the data is already sorted.
```python
# Bad — recompiles the same regex on every call
def validate(text):
    pattern = re.compile(r"^[A-Za-z0-9_]+$")
    return bool(pattern.match(text))

# Good — compiled once at module load
_VALID_PATTERN = re.compile(r"^[A-Za-z0-9_]+$")
def validate(text):
    return bool(_VALID_PATTERN.match(text))
```
- **DO:** Use `__slots__` (already covered under dataclasses) when instantiating a very large number of small objects, since it removes the per-instance `__dict__` and measurably reduces memory overhead at scale.
- **DON'T:** Use deep recursion for a problem that's naturally iterative when the recursion depth scales with input size — CPython has a default recursion limit (commonly 1000) and no tail-call optimization, so a deeply recursive function on large input raises `RecursionError` where an equivalent loop wouldn't.
- **DO:** Batch I/O operations (bulk database inserts, batched API requests) instead of issuing one round-trip per item in a loop — this is very often the dominant cost in real applications, dwarfing pure-CPU micro-optimizations.
```python
# Bad — one round trip per row
for record in records:
    db.execute("INSERT INTO events (payload) VALUES (?)", (record,))

# Good — one round trip for the whole batch
db.executemany("INSERT INTO events (payload) VALUES (?)", records)
```
- **DO:** Cache expensive, repeatable computations at the appropriate layer — in-process (`lru_cache`), or an external cache (Redis, memcached) for values shared across processes/machines — but invalidate deliberately (TTL, explicit eviction on write) rather than letting a cache grow unbounded or serve stale data indefinitely.
- **DON'T:** Assume the N+1 query problem only applies to ORMs in other languages — it's just as easy to write in SQLAlchemy/Django ORM/any Python ORM by looping over a collection and lazily accessing a related object per iteration, each triggering its own query. Use eager loading (`select_related`/`prefetch_related`/`joinedload`, or a manual batched query) when you know you'll need the related data for every item.
```python
# Bad — one query for the orders, then one more query per order for its user (N+1)
orders = Order.objects.all()
for order in orders:
    print(order.user.email)

# Good — a single query with the join done up front
orders = Order.objects.select_related("user")
for order in orders:
    print(order.user.email)
```
- **DO:** Choose the right data structure for the access pattern before micro-optimizing code around a poor one — e.g., a `collections.deque` for frequent appends/pops from both ends (O(1)) instead of a `list` (O(n) for left-end operations), or a `heapq` for repeatedly retrieving the smallest item instead of re-sorting a list each time.
- **DO:** Use `array.array` or NumPy arrays instead of a plain `list` when storing a large, homogeneous collection of numbers — a Python `list` stores pointers to individually-boxed objects, while a typed array stores raw values contiguously, which is both far more memory-efficient and faster for numeric operations at scale.
- **DON'T:** Build a large collection with repeated single-item insertions at the front of a `list` (`items.insert(0, x)`) inside a loop — each call is O(n) because every existing element has to shift, making the whole loop O(n²). Append to the end (O(1) amortized) and reverse once at the end, or use a `collections.deque` (O(1) appends at both ends), instead.
- **DO:** Process very large datasets in bounded chunks rather than loading everything into memory at once — read a huge file in fixed-size blocks, page through a large database result set instead of fetching it all in one query, or batch-write output incrementally — so memory usage stays roughly constant regardless of the total input size.
```python
# Bad — loads the entire result set into memory before processing any of it
def export_all_orders(db):
    orders = db.execute("SELECT * FROM orders").fetchall()  # could be millions of rows
    for order in orders:
        write_to_export_file(order)

# Good — streams results in bounded chunks, memory usage stays flat
def export_all_orders(db, chunk_size=1000):
    cursor = db.execute("SELECT * FROM orders")
    while chunk := cursor.fetchmany(chunk_size):
        for order in chunk:
            write_to_export_file(order)
```
- **DON'T:** Assume a "process in chunks" loop is automatically safe to parallelize across chunks without checking for shared state or ordering dependencies between them — some chunked workloads are embarrassingly parallel (safe to run concurrently with a process pool) while others depend on sequential order (a running total, a stateful transformation) and would produce wrong results if chunks ran out of order or concurrently without coordination.

### Profiling in Practice

- **DO:** Reach for the right profiling tool for the question being asked: `cProfile` (with `snakeviz` or `pstats` for visualization) for "which function is consuming the most wall-clock time," `line_profiler` for "which specific line inside this one hot function is slow," `py-spy` for sampling a *running* production process without restarting it or adding instrumentation, and `timeit` for comparing two small snippets' relative speed.
```bash
# Whole-program profiling
python -m cProfile -o profile.stats myapp/main.py
python -m pstats profile.stats  # then: sort cumulative; stats 15

# Sample a live process without any code changes or restart
py-spy top --pid 12345
```
- **DON'T:** Optimize based on a hunch about where time is going. Profile first — the actual bottleneck in real Python programs is very often in an unexpected place (serialization, logging overhead, a single N+1 query) rather than the tight numeric loop an engineer's intuition jumps to.
- **DO:** Re-profile after each optimization to confirm it actually helped and to find the *next* bottleneck — after fixing the biggest cost, a different function usually becomes the new largest cost, and guessing at further optimization without re-measuring wastes effort on parts that no longer matter.
- **DON'T:** Micro-optimize a code path that profiling shows accounts for a negligible fraction of total runtime, at the cost of readability. A 2x speedup on code that's 0.5% of total execution time is not worth the complexity if it makes the code harder to understand — spend optimization effort where profiling shows it will actually move the needle.

### Memory Awareness & Caching Layers

- **DO:** Use `tracemalloc` (standard library) or `memory_profiler` when memory usage — not CPU time — is the actual concern (a service that grows unbounded, an out-of-memory crash under load), rather than assuming a CPU profiler's output tells you anything useful about memory.
- **DO:** Reach for an external cache (Redis, memcached) instead of an in-process cache (`lru_cache`, a module-level dict) once a value needs to be shared across multiple processes or machines — an in-process cache gives each worker its own inconsistent copy, and a cold-started new worker gets no benefit from what other workers have already cached.
- **DON'T:** Cache a value indefinitely with no eviction policy or TTL when the underlying data can change — a permanent in-process cache of "current user permissions" or "current price," for example, will silently serve stale data forever once the source changes, with no mechanism to notice.
- **DO:** Size-bound any in-process cache explicitly (`lru_cache(maxsize=...)`, a bounded LRU structure) rather than using an unbounded dict as a cache — an unbounded cache is a slow memory leak that eventually degrades or crashes a long-running process under varied enough input.

### Choosing a Concurrency Model

- **DO:** Match the concurrency tool to the actual bottleneck: `asyncio` (or plain sequential code with a thread pool) for I/O-bound work with many concurrent waits; `threading` for I/O-bound work using libraries that don't support `asyncio` (threads still release the GIL during blocking I/O and C-extension calls); `multiprocessing`/`ProcessPoolExecutor` for CPU-bound work that needs true parallelism across cores, since the GIL prevents CPU-bound Python bytecode from running in parallel across threads in the standard CPython build.
```python
# CPU-bound: use processes to get real parallelism across cores
from concurrent.futures import ProcessPoolExecutor

def cpu_heavy(n: int) -> int:
    return sum(i * i for i in range(n))

with ProcessPoolExecutor() as pool:
    results = list(pool.map(cpu_heavy, [10_000_000] * 4))

# I/O-bound: use a thread pool (or asyncio) — threads sit idle waiting on I/O,
# and multiprocessing's process-startup and IPC overhead would dominate here
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=20) as pool:
    responses = list(pool.map(lambda url: requests.get(url, timeout=5), urls))
```
- **DON'T:** Reach for `multiprocessing` to parallelize I/O-bound work — the overhead of spawning processes and serializing data between them (pickling arguments and return values across the process boundary) typically outweighs any benefit for tasks that are mostly waiting on network/disk, where threads or `asyncio` handle the same workload with far less overhead.
- **DON'T:** Expect `threading` to speed up a CPU-bound pure-Python computation. The GIL means only one thread executes Python bytecode at a time regardless of how many threads you create — CPU-bound work needs `multiprocessing`, a C extension that releases the GIL (as NumPy operations do), or a free-threaded (no-GIL) Python build where available.
- **DO:** Be aware that data passed to a `ProcessPoolExecutor`/`multiprocessing` worker must be picklable, and that each worker process has its own separate memory — mutating a shared object inside a worker does not affect the parent process or other workers. Use a `multiprocessing.Manager`, shared memory, or a message-passing queue when workers genuinely need to share or coordinate state.

### Multiprocessing: Fork vs. Spawn Pitfalls

- **DO:** Be explicit about the multiprocessing start method (`multiprocessing.set_start_method("spawn")`, or rely on the platform default) rather than assuming `fork`'s behavior everywhere — `fork` (the default on Linux) copies the parent process's entire memory, including any already-open resources, while `spawn` (the default on Windows and macOS since 3.8) starts a fresh interpreter and re-imports everything, which can behave very differently for code with import-time side effects.
```python
import multiprocessing as mp

if __name__ == "__main__":
    mp.set_start_method("spawn")  # explicit and consistent across platforms
    with mp.Pool(4) as pool:
        results = pool.map(worker_fn, items)
```
- **DON'T:** Assume a resource opened before forking (a database connection, a file handle, a random number generator's state) is safe to use independently in the child process after a `fork` — a forked child inherits the parent's open file descriptors and connection state, and using the same connection object from both parent and child concurrently causes corruption. Open per-process resources *inside* the worker function, after the fork/spawn has happened, not before.
- **DON'T:** Put non-picklable objects (open file handles, database connections, thread locks, lambdas, local closures) into the arguments or return value of a function submitted to `ProcessPoolExecutor`/`multiprocessing.Pool` — pickling is how data crosses the process boundary, and any of those will raise a `PicklingError` (or silently fail in more confusing ways) rather than working as expected.
- **DO:** Guard multiprocessing entry-point code with `if __name__ == "__main__":` on every platform, but treat it as strictly required on Windows/macOS with the `spawn` start method — without it, `spawn` re-imports the main module in each child process, which re-executes top-level code and can trigger infinite recursive process spawning.

### Startup Time & Import Costs

- **DO:** Keep module-level import statements limited to what the module actually needs at load time, and defer an expensive or rarely-needed import (a large ML library, an optional integration) into the function that actually uses it, when import time measurably affects startup latency — such as a CLI tool where every invocation pays the cost of every top-level import even for a code path that never uses it.
```python
# Bad — every invocation of the CLI pays the cost of importing a heavy library,
# even for subcommands that never touch it
import pandas as pd
import numpy as np
import torch

def cli_version_command():
    print(__version__)

# Good — heavy, rarely-needed imports are deferred to where they're actually used
def cli_version_command():
    print(__version__)

def cli_train_command():
    import torch  # only paid for by the one subcommand that needs it
    ...
```
- **DON'T:** Apply deferred/local imports as a blanket default for ordinary dependencies — this trades a small, usually irrelevant startup-time saving for real readability cost (it's no longer obvious what a module depends on just by reading its top). Reserve the technique for genuinely expensive imports and cases with a specific, measured startup-time concern (CLIs, serverless functions with cold-start constraints).
- **DO:** Measure import time (`python -X importtime script.py`) before optimizing it, for the same reason profiling precedes any other performance work — the actual biggest contributor to slow startup is often one specific heavy dependency, not a broad, diffuse problem that needs restructuring the whole import graph.
- **DO:** Pay particular attention to import-time cost in contexts where it's paid repeatedly and matters directly to user-facing latency — serverless/FaaS cold starts, CLI tools invoked frequently in scripts, and any process that restarts often — versus a long-running server process where import cost is paid once and amortized over the process's whole lifetime.

### Algorithmic Complexity Awareness

- **DO:** Recognize when a piece of code's complexity class (not just its constant-factor speed) will become the bottleneck as input size grows — an O(n²) nested-loop membership check is fine for a 50-item list and a real problem for a 500,000-item one. Reasoning about Big-O before writing the loop is often cheaper than discovering the problem after it's in production.
```python
# O(n * m) — for every item in `orders`, scans the entire `blocked_ids` list
def filter_blocked(orders, blocked_ids):
    return [o for o in orders if o.customer_id not in blocked_ids]

# O(n + m) — converting blocked_ids to a set makes each membership check O(1)
def filter_blocked(orders, blocked_ids):
    blocked = set(blocked_ids)
    return [o for o in orders if o.customer_id not in blocked]
```
- **DON'T:** Nest loops over the same or related collections without noticing the multiplicative growth — a "for each user, for each of that user's orders, for each item in that order" triple loop over real-world data sizes can quietly become the slowest part of a system, especially once it's also doing a database or network call at the innermost level (compounding the algorithmic cost with I/O cost).
- **DO:** Estimate realistic data scale before deciding an algorithm's complexity doesn't matter — code that will only ever run against a few dozen items doesn't need Big-O concern, but the same code copy-pasted into a context processing millions of rows absolutely does; the complexity analysis is only actionable once matched to actual expected scale.

## Security

### Injection Risks

- **DON'T:** Build SQL queries by interpolating user input directly into a query string (f-strings, `%`, `.format()`, or plain concatenation). This is classic SQL injection — a value like `"'; DROP TABLE users; --"` can alter the query's meaning entirely. Always use parameterized queries / bound placeholders, which the database driver escapes correctly.
```python
# Bad — SQL injection
query = f"SELECT * FROM users WHERE username = '{username}'"
cursor.execute(query)

# Good — parameters are bound safely by the driver, not string-interpolated
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))
```
- **DO:** Use an ORM's query builder (Django ORM, SQLAlchemy's expression language) or the DB-API's parameter substitution for every query that includes any externally-influenced value, including values that "look like" they'd be safe (numeric IDs, enum-like strings) — attacker-controlled input is unsafe regardless of what it's expected to look like.
- **DON'T:** Pass `shell=True` to `subprocess.run()`/`subprocess.Popen()` when any part of the command includes user-controlled or externally-sourced input. `shell=True` invokes a shell that interprets metacharacters (`;`, `|`, `&&`, backticks), turning untrusted input into arbitrary command execution.
```python
# Bad — shell injection if filename contains e.g. "; rm -rf ~"
subprocess.run(f"convert {filename} out.png", shell=True)

# Good — arguments passed as a list, no shell interpretation of metacharacters
subprocess.run(["convert", filename, "out.png"])
```
- **DON'T:** Use `os.system()` for the same reason — it always goes through the shell. Use `subprocess.run()` with an argument list instead, which also gives you proper return-code handling, timeouts, and captured output.
- **DO:** Validate and allowlist (rather than denylist) any user input that will be used to construct a filesystem path, shell argument, or external command — check that it matches an expected pattern (e.g., a UUID, a known set of filenames) instead of trying to strip out "dangerous" characters after the fact.
- **DON'T:** Use `eval()` or `exec()` on any input that isn't fully trusted, hardcoded, and controlled by the developer. Both execute arbitrary Python code with the full privileges of the running process — there is essentially no way to "sandbox" them safely against untrusted input in pure Python.
```python
# Bad — arbitrary code execution if expr is user-supplied
result = eval(expr)

# Good — use a real parser/interpreter scoped to the operations you actually need
import ast, operator
_OPS = {ast.Add: operator.add, ast.Sub: operator.sub, ast.Mult: operator.mul}
def safe_eval_arithmetic(expr: str) -> float:
    node = ast.parse(expr, mode="eval").body
    return _eval_node(node)  # walks the AST, only allowing _OPS and literals
```
- **DON'T:** Use string formatting to build a file path from user input without normalizing and validating it against a base directory. A value like `"../../etc/passwd"` in a "filename" parameter can escape an intended directory (path traversal) unless the resolved path is checked against the allowed root.
```python
# Bad — path traversal via "../"
def read_upload(filename):
    return (UPLOAD_DIR / filename).read_text()

# Good — resolve and verify the path stays within UPLOAD_DIR
def read_upload(filename):
    target = (UPLOAD_DIR / filename).resolve()
    if not target.is_relative_to(UPLOAD_DIR.resolve()):
        raise ValueError("invalid filename")
    return target.read_text()
```
- **DO:** Use Jinja2's (or your templating engine's) autoescaping for any HTML output that includes user-controlled data, and never mark user input `| safe` (or otherwise bypass autoescaping) without a specific, reviewed reason — disabling autoescaping on user data is how cross-site scripting (XSS) vulnerabilities get introduced.
- **DON'T:** Parse untrusted XML with a parser that resolves external entities by default (classic `xml.etree.ElementTree`/`xml.dom.minidom` on older Python configurations, or `lxml` with default settings) — this enables XML External Entity (XXE) attacks that can read local files or trigger server-side request forgery. Use `defusedxml`, or explicitly disable entity resolution, when parsing XML from an untrusted source.
- **DON'T:** Build a command for `subprocess` by concatenating a base command string with user input even when `shell=True` is avoided, if the result is then parsed with `shlex.split()` on a string that itself contains untrusted substitutions — validate/allowlist the untrusted piece first, since a naive `shlex.split(f"convert {filename} out.png")` still lets a crafted `filename` containing shell-like tokens produce unexpected argument splitting.
```python
# Bad — user-controlled filename could contain spaces/quotes that change argument parsing
cmd = shlex.split(f"convert {filename} -resize 100x100 out.png")
subprocess.run(cmd)

# Good — build the argument list directly; each element is passed through verbatim,
# with no string parsing/splitting step for user input to interfere with
subprocess.run(["convert", filename, "-resize", "100x100", "out.png"])
```
- **DO:** Set an explicit, reasonable `timeout` on every `subprocess.run()` call, the same way you would for an HTTP request — a spawned child process that hangs (or is a deliberately crafted denial-of-service input causing the invoked tool itself to hang) can otherwise block the calling process indefinitely.
- **DON'T:** Pass `capture_output=True` (or manually piped stdout/stderr) on a subprocess call that might produce a very large amount of output when the output isn't actually needed or is only checked for a return code — buffering unbounded output in memory is both a performance and, for untrusted subprocesses, a potential memory-exhaustion risk. Redirect to `subprocess.DEVNULL` when output genuinely isn't needed.

### Deserialization

- **DON'T:** Unpickle data from an untrusted or externally-reachable source (a network socket, a user upload, a message queue fed by external systems). `pickle.load()` can execute arbitrary code as a side effect of deserialization — this isn't a hardening gap to fix later, it's a fundamental property of the pickle protocol.
```python
# Bad — arbitrary code execution if data comes from an untrusted source
obj = pickle.loads(request.body)

# Good — use a data-only format for anything crossing a trust boundary
obj = json.loads(request.body)
```
- **DO:** Use `json`, or a schema-validated format (protobuf, msgpack with a defined schema, or a Pydantic model parsing JSON), for any serialization that crosses a trust boundary — process to process, service to service, or client to server. Reserve `pickle` for trusted, same-process/same-deployment use cases like caching your own objects internally.
- **DON'T:** Use `yaml.load()` without specifying a safe loader. The default/full loader can construct arbitrary Python objects from tags embedded in the YAML document, which is a known code-execution vector for untrusted YAML. Use `yaml.safe_load()` always, unless you have a specific, narrow need for a custom-tagged loader you've fully audited.
```python
# Bad — full loader can instantiate arbitrary Python objects
config = yaml.load(untrusted_input)

# Good
config = yaml.safe_load(untrusted_input)
```
- **DO:** Validate deserialized data against an explicit schema (Pydantic, `attrs` with validators, or manual checks) before trusting its shape and using it — a successfully-parsed JSON document can still contain unexpected types, missing fields, or malicious values (e.g., extremely large numbers, deeply nested structures for a decompression-bomb-style DoS).

### Secrets, Credentials & Cryptography

- **DON'T:** Hardcode API keys, passwords, tokens, or connection strings directly in source code, even "temporarily" or in a private repository. Once committed, a secret is in the repository's history essentially forever (found by history search, forks, or exposed if the repo later becomes public) — load secrets from environment variables, a secrets manager, or an encrypted config the deployment environment injects.
```python
# Bad
STRIPE_API_KEY = "sk_live_51H8x..."

# Good
STRIPE_API_KEY = os.environ["STRIPE_API_KEY"]
```
- **DO:** Use the `secrets` module (not `random`) for anything security-sensitive — session tokens, password-reset tokens, API keys, CSRF tokens. `random`'s default generator (Mersenne Twister) is deterministic and predictable given enough output, and is explicitly documented as unsuitable for cryptographic use.
```python
# Bad — predictable, not cryptographically secure
token = str(random.randint(100000, 999999))

# Good
import secrets
token = secrets.token_urlsafe(32)
```
- **DON'T:** Store passwords in plaintext or hash them with a fast general-purpose hash (MD5, SHA-1, unsalted SHA-256) — these are designed to be fast, which makes brute-forcing leaked hashes cheap. Use a password-hashing algorithm designed to be slow and salted (bcrypt, scrypt, or Argon2, via a maintained library) for anything storing user credentials.
- **DO:** Use `hmac.compare_digest()` (constant-time comparison) instead of `==` when comparing secrets, tokens, or MAC/signature values — a naive `==` comparison on strings short-circuits on the first mismatched character, and the resulting timing difference can theoretically be exploited to guess a secret byte-by-byte.
```python
# Bad — timing side-channel on secret comparison
if provided_token == expected_token:
    ...

# Good
import hmac
if hmac.compare_digest(provided_token, expected_token):
    ...
```
- **DON'T:** Log secrets, tokens, full credit card numbers, or other sensitive values, even at debug level — logs are often retained, shipped to third-party aggregators, and accessible to a broader set of people than the original request. Redact or omit sensitive fields before logging a request/response payload.
- **DO:** Set an explicit `timeout` on every outbound HTTP request (`requests.get(url, timeout=5)`, `httpx` client timeouts). A request with no timeout can hang indefinitely on a slow or unresponsive server, tying up a thread/connection and turning a remote issue into a local outage.
```python
# Bad — can hang forever if the server never responds
response = requests.get(url)

# Good
response = requests.get(url, timeout=10)
```
- **DON'T:** Disable TLS certificate verification (`requests.get(url, verify=False)`, or an `ssl` context with `CERT_NONE`) outside of a narrowly-scoped, well-understood local development or testing context. Disabling verification defeats the protection against man-in-the-middle attacks, and it's easy for a `verify=False` added "temporarily" to survive into production.
- **DO:** Store a hashed password with a per-password random salt generated by the hashing library itself (as `bcrypt`/`argon2`/`passlib` all do automatically), never a single fixed salt shared across all users — a shared salt means a single precomputed rainbow-table-style attack works against every user's password simultaneously instead of needing to be redone per user.
```python
import bcrypt

# Hashing: bcrypt generates and embeds a fresh random salt per call
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt())

# Verifying: the salt is read back out of the stored hash automatically
bcrypt.checkpw(candidate_password.encode(), hashed)
```
- **DON'T:** Roll a custom password-hashing scheme (a hand-written combination of a fast hash function plus a manually-managed salt) instead of using an established, audited library — password hashing has enough subtle failure modes (timing attacks, insufficient work factor, salt reuse, incorrect comparison) that a battle-tested library is worth the dependency in every realistic case.

### Web & API-Specific Pitfalls

- **DO:** Validate and constrain all external input at the boundary (request bodies, query parameters, headers, file uploads) — size limits, type checks, allowed value ranges — using something like Pydantic/marshmallow/framework-native validation, rather than trusting client-supplied data to already be well-formed.
- **DON'T:** Build outbound requests to a URL supplied (even indirectly) by a client without restricting the destination — an attacker can use your server as a proxy to reach internal-only services (Server-Side Request Forgery). Validate/allowlist destination hosts for any "fetch this URL" feature, and block requests to internal/private IP ranges unless explicitly intended.
- **DO:** Apply the principle of least privilege to database credentials, cloud IAM roles, and service accounts used by application code — a background job that only reads data shouldn't run with a database user that has `DROP TABLE` privileges.
- **DO:** Verify a webhook's signature (using the sending service's documented HMAC scheme, comparing with `hmac.compare_digest`) before trusting or acting on its payload — a webhook endpoint is, by definition, a publicly reachable URL that accepts POST requests, and without signature verification anyone who discovers the URL can send fabricated events.
```python
import hmac, hashlib

def verify_webhook_signature(payload: bytes, signature_header: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), payload, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature_header)

@app.post("/webhooks/payments")
async def handle_webhook(request: Request):
    body = await request.body()
    signature = request.headers.get("X-Signature", "")
    if not verify_webhook_signature(body, signature, WEBHOOK_SECRET):
        raise HTTPException(status_code=401, detail="invalid signature")
    event = json.loads(body)
    process_payment_event(event)
```
- **DON'T:** Trust a webhook's claimed source based only on the request's origin IP or a client-supplied header claiming which service sent it — both can be spoofed. Signature verification (or, at minimum, mutual TLS) is the actual trust boundary, not the payload's self-reported identity.
- **DON'T:** Trust a file's declared `Content-Type` or its extension as proof of its actual content when handling uploads — validate the real file type (e.g., checking magic bytes/signature, or using a library that inspects actual content) before processing it as an image, archive, or other structured format, especially before passing it to a parser with a history of vulnerabilities.

### Dependency & Supply-Chain Security

- **DO:** Run a dependency vulnerability scanner (`pip-audit`, GitHub's Dependabot alerts, `safety`) in CI, and treat a flagged known-CVE dependency as something to triage promptly — most real-world breaches exploit a known, already-patched vulnerability in a dependency that simply wasn't updated.
- **DON'T:** Install packages from an unverified or typo-prone source name without checking it's the package you intend (a real risk from typosquatting on PyPI — e.g., `python-dateutil` vs. a similarly-named malicious package). Double-check package names before adding a new dependency, especially ones introduced by a quick copy-pasted `pip install` command from an unfamiliar source.
- **DO:** Pin dependencies via a lockfile (covered under Packaging) partly *as a security measure*, not just for reproducibility — an unpinned dependency can be silently upgraded to a compromised version (a supply-chain attack on a popular package) between one install and the next.
- **DON'T:** Run untrusted third-party code (a downloaded script, a plugin, a `pip install` from an unknown index) with elevated privileges or broad filesystem/network access "just to try it." Sandbox or containerize evaluation of unfamiliar code when practical.
- **DO:** Validate that a file upload's actual size stays within an explicit, enforced limit before or while reading it, rather than trusting a client-declared `Content-Length` header alone — a header can be absent, wrong, or deliberately misleading, and reading an unbounded stream into memory based on trusting it is a straightforward denial-of-service vector.
```python
# Bad — trusts a client-controlled header and reads without a hard cap
async def handle_upload(request):
    body = await request.body()  # could be gigabytes if the client sends them
    save_file(body)

# Good — enforce a real limit while reading, independent of any declared header
MAX_UPLOAD_BYTES = 10 * 1024 * 1024

async def handle_upload(request):
    body = bytearray()
    async for chunk in request.stream():
        body.extend(chunk)
        if len(body) > MAX_UPLOAD_BYTES:
            raise HTTPException(status_code=413, detail="file too large")
    save_file(bytes(body))
```
- **DON'T:** Extract an uploaded archive (zip, tar) without checking each member's resulting path stays within the intended extraction directory — a maliciously crafted archive entry with a path like `../../etc/cron.d/evil` can write outside the intended directory during extraction (a "zip slip" vulnerability), and Python's own archive modules do not guard against this automatically on all versions.

### Rate Limiting & Abuse Prevention

- **DO:** Apply rate limiting to any endpoint that's expensive to compute, sends external communications (email, SMS), or is a plausible target for credential-stuffing/brute-force attempts (login, password reset, OTP verification) — using a framework's built-in rate-limiting middleware or a dedicated library, keyed by IP and/or account identifier as appropriate.
- **DON'T:** Rely solely on client-side throttling (disabling a submit button, a JavaScript debounce) to prevent abuse. Client-side controls are trivially bypassed by anyone calling the API directly; the actual limit must be enforced server-side.
- **DO:** Return a generic, identical error message for "user not found" and "wrong password" on a login endpoint, rather than distinguishing them — a distinguishable response lets an attacker enumerate valid usernames/emails by observing which one produces which error.
```python
# Bad — leaks which emails are registered
if user is None:
    raise HTTPException(404, "no account with that email")
if not verify_password(password, user.password_hash):
    raise HTTPException(401, "incorrect password")

# Good — identical response regardless of which check failed
if user is None or not verify_password(password, user.password_hash):
    raise HTTPException(401, "invalid email or password")
```

### Framework-Provided Defenses

- **DO:** Rely on your web framework's built-in CSRF protection, secure cookie flags (`HttpOnly`, `Secure`, `SameSite`), and templating autoescaping rather than disabling or reimplementing them — these defaults exist specifically because the failure modes they prevent are common and easy to get wrong when hand-rolled.
- **DON'T:** Configure CORS with a wildcard `Access-Control-Allow-Origin: *` alongside credentialed requests (cookies, `Authorization` headers) — most frameworks won't even allow this combination by default because it effectively lets any website make authenticated requests to your API on a logged-in user's behalf. Allowlist specific trusted origins instead.
- **DO:** Keep the web framework, its extensions, and the language runtime itself on a supported, patched version — a meaningful share of real-world web application compromises exploit a known vulnerability in outdated framework or runtime code, not a novel flaw in application logic.

### Mass Assignment & Over-Posting

- **DON'T:** Build a database model or update an existing record directly from a raw request payload (`Model(**request.json)`, or `for key, value in request.json.items(): setattr(user, key, value)`) without an explicit allowlist of updatable fields. This lets a client set fields it was never meant to control — a `role` or `is_admin` field a form never exposed, but that the underlying model happens to have — a vulnerability class known as mass assignment / over-posting.
```python
# Bad — a client can include "is_admin": true in the request body and it
# silently gets applied, even though the update form never showed that field
def update_profile(user, request_data: dict):
    for key, value in request_data.items():
        setattr(user, key, value)

# Good — only the fields this endpoint is meant to let clients change are updated
ALLOWED_PROFILE_FIELDS = {"display_name", "bio", "avatar_url"}

def update_profile(user, request_data: dict):
    for key, value in request_data.items():
        if key in ALLOWED_PROFILE_FIELDS:
            setattr(user, key, value)
```
- **DO:** Use a dedicated input schema (a Pydantic model, a serializer/DTO class) that explicitly lists only the fields a given endpoint accepts, separate from the full internal data model — this makes the allowlist structural and enforced by the type system, rather than an easy-to-forget manual check repeated at every mutation point.
- **DON'T:** Reuse the same model/schema for both reading (serializing output) and writing (accepting input) when the two should expose a different field set — an output serializer that includes internal fields (`password_hash`, `internal_notes`) is safe for reading but dangerous if the same shape is also accepted as writable input.

### Regular Expressions & Denial of Service

- **DON'T:** Build a regular expression with nested, ambiguous quantifiers (e.g., `(a+)+b`) from or against untrusted input without considering catastrophic backtracking — certain crafted inputs can make such a pattern's matching time blow up exponentially, letting a single request tie up a worker for an extremely long time (ReDoS, a real and repeatedly-exploited denial-of-service vector).
```python
# Bad — vulnerable to catastrophic backtracking on a crafted input like
# "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!"
PATTERN = re.compile(r"^(a+)+$")

# Good — write the equivalent pattern without ambiguous nested repetition
PATTERN = re.compile(r"^a+$")
```
- **DO:** Set a reasonable maximum input length before applying a regular expression to untrusted, externally-supplied text, as a defense-in-depth measure alongside writing non-pathological patterns in the first place.
- **DO:** Consider the `regex` third-party module's or the standard library's built-in timeout-aware alternatives, or a non-backtracking engine, for regular expressions that must run against genuinely untrusted input at scale, rather than relying entirely on manual pattern review to rule out worst-case behavior.

### Log Injection & Untrusted Data in Logs

- **DON'T:** Write raw, unsanitized user input directly into log messages without considering that the value itself might contain characters designed to forge additional log entries (embedded newlines faking a second log line, or terminal escape sequences that alter how the log looks when viewed in a terminal). Structured logging (passing values as separate fields via `extra=`, rather than interpolating them into the message string) sidesteps most of this by keeping user data in its own field rather than the free-text message.
```python
# Vulnerable to log forging — a username containing an embedded newline plus a
# fake log line can make the log stream appear to contain an entry it doesn't
logger.info(f"login attempt for user: {username}")

# Better — the value is a distinct structured field, not interpolated into free text
logger.info("login attempt", extra={"username": username})
```
- **DON'T:** Assume data is safe to log just because it originated inside your own system — a value that started as user input (a display name, a comment, a search query) and was merely stored and retrieved is still attacker-influenced data by the time it reaches a log call.
- **DO:** Sanitize or truncate unbounded user-controlled strings before logging them (very long strings, unexpected binary content, embedded control characters), both to keep log output readable and to avoid a user-controlled value causing excessive log volume or storage cost as a denial-of-service vector.

### Secrets in CI/CD & Configuration Files

- **DO:** Store CI/CD secrets (deploy keys, cloud credentials, API tokens) in the CI platform's dedicated secrets store (GitHub Actions secrets, GitLab CI/CD variables marked protected/masked, etc.), never as plaintext in the pipeline YAML file or a committed `.env` checked into the repository powering that pipeline.
- **DON'T:** Echo or log a secret's value during a CI run "to debug why it's not working" — most CI platforms mask known secret values in logs automatically, but only for values registered as secrets; printing a derived or partially-transformed form of the secret, or a secret pulled from an unregistered source, bypasses that masking and can leak it into stored, often widely-readable, build logs.
- **DO:** Scope CI/CD credentials as narrowly as possible (a deploy key limited to one repository, a cloud role limited to the specific resources a pipeline needs to touch) rather than reusing one broad, powerful credential across every pipeline — a compromised pipeline (via a malicious dependency or a supply-chain attack in a third-party CI action) then has bounded, not organization-wide, blast radius.
- **DON'T:** Grant a third-party CI action/plugin broad repository or secret access without checking its provenance and pinning it to a specific commit SHA (not a mutable tag like `@v1`) — an unpinned third-party action can be silently updated by its maintainer (or an attacker who compromises the maintainer's account) to exfiltrate secrets on the next run.

### Temporary Files & File Permissions

- **DO:** Use the `tempfile` module (`tempfile.NamedTemporaryFile`, `tempfile.TemporaryDirectory`, `tempfile.mkstemp`) to create temporary files and directories, rather than constructing a "random-ish" path by hand under `/tmp`. `tempfile` creates the file securely (avoiding predictable-name race conditions where an attacker pre-creates a file at a guessed path) and handles cleanup.
```python
# Bad — a predictable path is a race condition/symlink-attack risk on a shared system,
# and nothing guarantees cleanup if the process crashes
path = f"/tmp/report_{os.getpid()}.csv"
with open(path, "w") as f:
    f.write(data)

# Good — securely created, unique, and cleaned up automatically
import tempfile
with tempfile.NamedTemporaryFile(mode="w", suffix=".csv", delete=True) as f:
    f.write(data)
    f.flush()
    process_file(f.name)
```
- **DON'T:** Write a file containing secrets or other sensitive data with default, world-readable permissions on a multi-user system. Set restrictive permissions explicitly (`os.chmod(path, 0o600)`, or `os.open()` with an explicit mode at creation time to avoid a brief window where the file is world-readable before permissions are tightened).
- **DO:** Clean up temporary files and directories deterministically (via `tempfile`'s context-manager support, or an explicit `try`/`finally`) rather than leaving cleanup to eventual OS-level temp-directory garbage collection, especially for anything containing sensitive data that shouldn't linger on disk longer than necessary.

## Logging vs. Print

- **DO:** Use the `logging` module (or a structured-logging library built on it, like `structlog`) for all runtime diagnostic output in application and library code, instead of `print()`. Logging gives you levels, per-module control, timestamps, structured output, and the ability to route output to files/log aggregators without touching call sites.
```python
# Bad
print(f"Processing order {order.id}")
print(f"ERROR: order {order.id} failed: {exc}")

# Good
logger = logging.getLogger(__name__)
logger.info("Processing order %s", order.id)
logger.error("Order %s failed: %s", order.id, exc)
```
- **DON'T:** Leave debugging `print()` statements in committed code. They're easy to forget, clutter real output (including a library's caller's stdout, which the library has no business writing to), and provide none of logging's level-based filtering — a stray `print()` in a hot loop can also measurably slow things down via flushed I/O.
- **DO:** Use `logger = logging.getLogger(__name__)` at module scope so log records carry the originating module's name automatically, instead of a single ad hoc root-logger call or a hardcoded string identifying the source.
- **DO:** Choose the log level deliberately: `DEBUG` for detailed diagnostic info useful only during development, `INFO` for normal operational milestones, `WARNING` for unexpected-but-recoverable situations, `ERROR` for failures that need attention, `CRITICAL` for failures threatening the whole application. Logging everything at `INFO` (or worse, everything at the same level) defeats the purpose of having levels at all.
- **DON'T:** Configure logging (handlers, formatters, log level) inside library code. A library should just call `logging.getLogger(__name__)` and emit records; only the application's entry point should call `logging.basicConfig()` or otherwise decide where logs go and how they're formatted — a library that configures the root logger can silently override or conflict with the application's own logging setup.
- **DO:** Use the logging module's lazy `%s`-style argument substitution (`logger.info("value: %s", value)`) rather than pre-formatting the string with an f-string (`logger.info(f"value: {value}")`), at least for any log call on a hot path. The f-string version always pays the formatting cost even if the log level would filter the message out; the `%s` form only formats if the record is actually going to be emitted.
```python
# Less efficient — the f-string is always evaluated, even if DEBUG is disabled
logger.debug(f"cache state: {expensive_repr(cache)}")

# Better — argument formatting is deferred until (and unless) the record is emitted
logger.debug("cache state: %s", expensive_repr(cache))
```
- **DO:** Use `logger.exception(...)` inside an `except` block (covered under Error Handling) so the traceback is captured automatically, instead of manually formatting the exception into the message.
- **DON'T:** Log sensitive data — passwords, tokens, full payment details, personally identifiable information beyond what's operationally necessary — even at `DEBUG` level. Treat log output as something that will eventually be read by more people, and stored for longer, than you expect.
- **DO:** Include correlation/request IDs in log output for services handling concurrent requests, so log lines from a single logical request can be filtered and reconstructed even when interleaved with other requests' log output in a shared stream.
- **DO:** Use structured logging (key-value fields, or a JSON formatter) in production services rather than free-form message strings, when logs are consumed by a log aggregator/search system — structured fields (`user_id=42`, `duration_ms=340`) are filterable and aggregatable in a way that free text embedded in a message isn't.
```python
# Less useful for querying later
logger.info(f"Request completed for user {user_id} in {duration_ms}ms")

# More useful — structured fields a log platform can index and query
logger.info("request_completed", extra={"user_id": user_id, "duration_ms": duration_ms})
```
- **DON'T:** Use `print()` for genuine CLI *output* (the actual result a command-line tool is meant to produce, meant to be piped or read by the user) and confuse that with logging — CLI tools legitimately use `print()`/`sys.stdout.write()` for their primary output while still using `logging` for diagnostic/debug/error information, typically routed to `stderr`.
- **DO:** Set the root/application log level from configuration (an environment variable, a config file) rather than hardcoding it, so verbosity can be raised in production for debugging a specific incident without a code change and redeploy.
- **DON'T:** Use bare `logging.warning(...)`/`logging.info(...)` module-level function calls scattered through application code as a substitute for a properly named logger — they implicitly use the root logger, which makes it impossible to control verbosity or routing per-module later.

### Log Levels Per Environment

- **DO:** Run with a more verbose log level (`DEBUG` or `INFO`) in development and a less verbose one (`WARNING` or `INFO`) in production by default, controlled through configuration rather than a code change — development benefits from seeing more detail, while a production service logging every `DEBUG` line at scale generates cost (storage, ingestion, noise) without proportional benefit.
- **DO:** Keep the ability to temporarily raise verbosity in a specific production environment (a per-service or per-request log-level override) for active incident debugging, then revert it — rather than either permanently running production at `DEBUG` (expensive, noisy) or having no way to get more detail when actually needed.
- **DON'T:** Log at `ERROR` or `CRITICAL` for conditions that are actually expected and routinely recoverable (a cache miss, a normal validation rejection, a client retry succeeding on its second attempt) — reserving high-severity levels for genuinely actionable problems keeps alerting on those levels meaningful; if everything routine is logged as an error, real errors get lost in the noise and alert fatigue sets in.

### Contextual Logging in Request-Scoped Code

- **DO:** Attach request-scoped context (a request ID, an authenticated user ID, a trace ID) to every log line emitted during that request automatically, using the logging framework's contextual mechanisms (`contextvars`-based filters/adapters, or a web framework's built-in request-logging middleware), rather than manually passing and re-threading that context through every function's parameters just so it can be logged.
```python
import contextvars, logging

request_id_var = contextvars.ContextVar("request_id", default="-")

class RequestIdFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id_var.get()
        return True

logger = logging.getLogger(__name__)
logger.addFilter(RequestIdFilter())
# formatter: "%(asctime)s [%(request_id)s] %(levelname)s %(message)s"

# middleware sets request_id_var.set(...) once per incoming request;
# every log call within that request automatically includes it
```
- **DON'T:** Rely on manually formatting the request ID into every individual log message string (`logger.info(f"[{request_id}] processing order")`) throughout a codebase — it's repetitive, easy to forget at some call sites, and doesn't help lines logged by library code that has no way to know about your application's request ID convention.

### Structured Logging Libraries

- **DO:** Consider `structlog` (or an equivalent structured-logging library) for a service where logs are consumed by a log platform that indexes structured fields, instead of building structured output manually on top of the standard `logging` module — it provides bound loggers that carry context automatically, consistent key-value output, and integrates with `logging`'s handlers rather than replacing them outright.
```python
import structlog

log = structlog.get_logger()

def process_order(order_id: int, user_id: int) -> None:
    order_log = log.bind(order_id=order_id, user_id=user_id)
    order_log.info("order_processing_started")
    try:
        result = charge_payment(order_id)
    except PaymentError as exc:
        order_log.error("order_processing_failed", error=str(exc))
        raise
    order_log.info("order_processing_completed", total=result.total)
```
- **DON'T:** Introduce a structured-logging library for a small script or a codebase with no log-aggregation platform actually consuming structured fields — the standard `logging` module with a sensible formatter is simpler and entirely sufficient when nothing downstream benefits from the extra structure.

### Configuring Handlers, Formatters, and Rotation

- **DO:** Configure logging once, centrally, at the application entry point — via `logging.config.dictConfig()` (preferred for anything beyond the trivial case) or a small number of explicit `basicConfig()`/handler-attachment calls — rather than scattering ad hoc handler setup across multiple modules, which tends to produce duplicated or conflicting output.
```python
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "default": {"format": "%(asctime)s %(levelname)s %(name)s: %(message)s"},
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler", "formatter": "default"},
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "app.log",
            "maxBytes": 10_000_000,
            "backupCount": 5,
            "formatter": "default",
        },
    },
    "root": {"level": "INFO", "handlers": ["console", "file"]},
}
logging.config.dictConfig(LOGGING_CONFIG)
```
- **DO:** Use a rotating handler (`RotatingFileHandler` by size, or `TimedRotatingFileHandler` by time interval) for any application writing logs directly to a local file, so log files don't grow without bound and eventually fill the disk — unless logs are instead shipped straight to a managed log aggregation service, which typically handles retention itself.
- **DON'T:** Attach the same handler to a logger more than once across repeated module imports or repeated calls to a setup function — in long-running processes (notably ones that re-run setup code, like certain test suites or reloaders) this duplicates every log line once per accidental re-registration. Guard setup code to run exactly once, or check `logger.handlers` before adding a new one.

## Common AI-Assistant Mistakes in Python

This section targets patterns specific to AI-generated Python — the failure modes that show up disproportionately often when a model writes code quickly from a prompt rather than from lived experience maintaining a codebase.

### Hallucinated APIs & Fabricated Behavior

- **DON'T:** Invent a function, method, class, or parameter name that sounds plausible but doesn't exist in the library being used (e.g., a `requests.Session.get_json()` that isn't a real method, or a keyword argument a real function never defined). This is the single most damaging AI-specific failure mode in generated Python — the code looks correct, often passes a superficial read, and fails only at runtime, sometimes deep in a code path that isn't exercised until production.
```python
# Bad — pandas.DataFrame has no such method; this is a plausible-looking fabrication
df.drop_duplicates_by_column("email")

# Good — the real API
df.drop_duplicates(subset=["email"])
```
- **DO:** Verify an unfamiliar library call actually exists — by checking installed version documentation, the library's source/`__init__.py` exports, or `help()`/`dir()` on the real object — before relying on it, rather than pattern-matching from a similar-sounding API in a different library or an older/newer version of the same one.
- **DON'T:** Mix up APIs between similar libraries (writing `pandas`-style method chains against a `polars` DataFrame, or `requests`-style calls against `httpx`, or `unittest.TestCase` assertion methods inside a plain pytest function). Each library has its own real surface area; superficial similarity between libraries in the same problem space is not equivalence.
```python
# Bad — .iloc / .loc are pandas idioms; polars uses a different API entirely
row = polars_df.iloc[0]

# Good
row = polars_df.row(0)
```
- **DON'T:** Cite a specific function's parameter list, default value, or return type from memory with confidence when the library version in use isn't confirmed. Library APIs change between major (and sometimes minor) versions — a parameter that existed in one version may be renamed, removed, or have a different default in the version actually pinned in the project.
- **DO:** Cross-check generated code against the project's actual installed dependency versions (`pip show <package>`, the lockfile) rather than assuming the latest documentation online matches what's running, especially for fast-moving libraries.
- **DON'T:** Fabricate a plausible-sounding standard library function that doesn't exist (e.g., `os.path.get_extension()`, `str.remove_prefix_if_exists()`) instead of the real one (`os.path.splitext()`, `str.removeprefix()` on 3.9+). When unsure whether a convenience function exists, it's safer to write the two or three lines of definitely-correct code than to guess at a shortcut.
- **DON'T:** Claim a package supports a feature (async support, a particular file format, a specific integration) without verification, then write code as if that claim were established fact. State uncertainty explicitly, or verify with a quick check, rather than presenting a guess as settled.

### Python 2 Habits Leaking into Python 3

- **DON'T:** Use `print` as a statement (`print "hello"`) instead of a function call — this is a Python 2 syntax error in Python 3, and while an AI assistant rarely writes this verbatim, small remnants of it (missing parens around multiple arguments, assuming `print` returns something) still surface in generated code trained on mixed-era examples.
- **DON'T:** Use `xrange()` instead of `range()`. Python 3's `range()` already returns a lazy, memory-efficient sequence object — `xrange` doesn't exist in Python 3 at all and its use is an immediate `NameError`.
```python
# Bad — Python 2 only, raises NameError on Python 3
for i in xrange(1000000):
    ...

# Good
for i in range(1000000):
    ...
```
- **DON'T:** Use `dict.has_key(k)` — it was removed in Python 3. Use `k in dict` instead.
```python
# Bad — AttributeError on Python 3
if config.has_key("timeout"):
    ...

# Good
if "timeout" in config:
    ...
```
- **DON'T:** Rely on Python 2's implicit integer division (`/` truncating for two ints) — in Python 3, `/` always performs true (float) division, and `//` is the explicit floor-division operator. Code that assumes `5 / 2 == 2` is a Python-2-ism that silently produces wrong numeric results on Python 3 rather than an error.
```python
# Ambiguous / wrong assumption carried over from Python 2 thinking
average_batch_size = total_items / num_batches  # this is now a float, e.g. 3.5

# Explicit about intent
whole_batches = total_items // num_batches       # floor division, an int
average_batch_size = total_items / num_batches   # true division, a float
```
- **DON'T:** Use `unicode()` or the `u"..."` string prefix as if `str` and `unicode` were still separate types. In Python 3, all `str` is already Unicode text; `unicode()` doesn't exist, and the `u"..."` prefix is a harmless no-op kept only for Python 2/3 compatibility shims — new code needs neither.
- **DON'T:** Catch exceptions with the old comma syntax (`except ValueError, e:`) — this is a `SyntaxError` in Python 3. Use `except ValueError as e:`.
- **DON'T:** Use `raw_input()` — it was renamed to `input()` in Python 3 (Python 3's `input()` already returns a string, unlike Python 2's `input()`, which evaluated the entry as an expression).
- **DON'T:** Iterate over `dict.iteritems()`/`.iterkeys()`/`.itervalues()` — these were removed in Python 3. Plain `.items()`, `.keys()`, and `.values()` already return lightweight view objects in Python 3, not full list copies, so there's no separate lazy-iterator API to reach for.
- **DON'T:** Use old-style string formatting exclusively (`"%s" % value`) as if f-strings and `.format()` didn't exist, particularly in freshly generated code with no compatibility constraint. It's not wrong on Python 3, but defaulting to it signals training on older-era code rather than current idiom.
- **DON'T:** Write a class without inheriting implicitly from `object` concerns in mind — this specific issue (old-style vs. new-style classes) is moot on Python 3, where every class is already a "new-style" class whether or not `(object)` is written explicitly; including `(object)` explicitly is unnecessary boilerplate in Python 3-only code, not a requirement.

### Environment & Dependency Mismanagement

- **DON'T:** Tell a user to `pip install <package>` without any mention of a virtual environment, or generate a script that assumes packages are already available in whatever Python happens to run it. Always frame dependency installation in terms of a project's virtual environment (`python -m venv .venv && source .venv/bin/activate && pip install ...`) or its declared dependencies (`pyproject.toml`, `requirements.txt`).
- **DON'T:** Generate a `requirements.txt` with no version constraints at all (`requests`, `pandas`, `flask`) when reproducibility matters, or conversely pin every dependency to an exact version with no stated reason when the project would benefit from compatible version *ranges*. Match the pinning strategy to context: exact pins in a lockfile, ranges in a direct-dependency declaration.
- **DON'T:** Add a new dependency for functionality the standard library already provides — reaching for a whole external package to do `datetime` math, basic HTTP requests without any advanced needs, or simple path manipulation when `pathlib`/`datetime`/`urllib` already cover it. Each added dependency is a maintenance and supply-chain-risk cost; justify it against what's already available.
- **DON'T:** Assume a package is already installed in the target environment just because it's common or well-known. Either state the dependency explicitly (add it to `pyproject.toml`/`requirements.txt`) or check for it before importing, rather than writing an `import` for a package the project has never declared.
- **DON'T:** Ignore an existing project's chosen dependency manager and introduce a different one in generated instructions (suggesting `pip install` commands for a project that uses Poetry, or `poetry add` for a project that uses plain `pip` + `requirements.txt`). Detect and follow the convention already in use.
- **DO:** Check for a `pyproject.toml`, `Pipfile`, `requirements.txt`, or lockfile already present in the project before assuming how dependencies should be declared or installed, and match whatever mechanism is already established rather than introducing a second, conflicting one.

### Flat Scripts Instead of Structured Programs

- **DON'T:** Write every request as one long, flat, top-level script with no functions, no `if __name__ == "__main__":` guard, and all logic executing at import time. This makes the code untestable (importing it runs it), unreusable as a module, and impossible to compose into a larger program.
```python
# Bad — runs immediately on import, can't be tested or reused
data = load_data("input.csv")
cleaned = [row for row in data if row["value"] is not None]
print(sum(row["value"] for row in cleaned))

# Good — importable, testable, and only runs when executed directly
def load_and_clean(path: str) -> list[dict]:
    data = load_data(path)
    return [row for row in data if row["value"] is not None]

def main() -> None:
    cleaned = load_and_clean("input.csv")
    print(sum(row["value"] for row in cleaned))

if __name__ == "__main__":
    main()
```
- **DON'T:** Produce a single giant function that does input parsing, business logic, and output formatting all in one body with no decomposition, just because the prompt described one task. Break work into small, named, independently testable functions even for a "quick script" — it costs little and pays off the moment the script needs a fix or a test.
- **DON'T:** Default to a flat single-module script for anything that's clearly going to grow into a real project (a request that mentions "the app," multiple features, or ongoing development). Set up a proper package structure (`src/<package>/`, `__init__.py`, separated modules by responsibility) from the start rather than requiring an awkward later migration out of one 800-line `main.py`.
- **DON'T:** Put unrelated responsibilities in one module (database access, HTTP handling, and business logic all defined together in a single `app.py`) once the codebase has grown past a trivial size. Split by responsibility (`models.py`, `api.py`, `services.py`, or a more domain-oriented split) so each module has one clear reason to change.
- **DO:** Match structural complexity to the actual scope of the request — a genuinely one-off 15-line script doesn't need a package, a `pyproject.toml`, and a test suite, and adding that ceremony for a throwaway script is its own form of over-engineering. Judge scope from context rather than reflexively applying either extreme.

### Exception-Handling Anti-Patterns

- **DON'T:** Wrap large blocks of unrelated code in a single broad `try: ... except Exception: ...` "just in case," especially when the `except` block only logs a generic message or silently passes. This is one of the most common AI-generated anti-patterns — it looks defensive and safety-conscious, but it actually hides real bugs (typos, wrong argument types, logic errors) behind a facade of "handled" errors, and makes production incidents far harder to diagnose.
```python
# Bad — hides everything from a network timeout to a typo in an attribute name
try:
    user = fetch_user(user_id)
    order = create_order(user, items)
    send_confirmation_email(order)
except Exception as e:
    print(f"Error: {e}")
    return None

# Good — handle only what you can actually recover from, at the point it happens
try:
    user = fetch_user(user_id)
except UserNotFoundError:
    raise OrderCreationError(f"cannot create order: user {user_id} not found")

order = create_order(user, items)

try:
    send_confirmation_email(order)
except EmailDeliveryError:
    logger.warning("order %s created but confirmation email failed", order.id)
```
- **DON'T:** Add a defensive `try/except` around code where the "exception" being guarded against is actually impossible given the code's own logic (e.g., catching `KeyError` right after explicitly checking `if key in dict:` two lines above). Defensive code should correspond to a real, reachable failure mode, not an imagined one — superfluous handlers add noise and can mask an actual bug if the "impossible" case ever does occur due to a later edit.
- **DON'T:** Generate inconsistent error-handling styles across a single response — some functions raising custom exceptions, others returning `None` on failure, others returning a tuple of `(success, error)` — without an explicit, stated reason. Pick one convention (exceptions are the Python-idiomatic default) and apply it uniformly within a codebase.
- **DON'T:** Add a fallback/default value inside an `except` block that silently changes program behavior in a way the caller can't detect (e.g., returning `0`, an empty list, or a cached stale value on any error, with no logging and no way for the caller to know something went wrong). If a caller might need to react differently to "no data because it's genuinely empty" versus "no data because something failed," don't erase that distinction by treating both as `[]`.

### Global State & Mutable Defaults

- **DON'T:** Reach for a module-level global variable as the default way to share state between functions in generated code, when a class, an explicitly passed parameter, or a small context object would keep the dependency visible and testable. Global mutable state generated "to make it work quickly" tends to survive into the final codebase unchallenged.
- **DON'T:** Initialize an expensive shared resource (a database connection, an HTTP client, a loaded ML model) as a bare module-level global that gets created at import time with no lifecycle management, error handling, or way to substitute a test double — this couples resource creation to the accident of when a module happens to be imported, and makes it very difficult to test code that uses it without triggering the real resource's setup.
```python
# Bad — connects to a real database the instant this module is imported,
# with no way to swap in a test double and no handling if the connection fails
db = create_database_connection()

def get_user(user_id):
    return db.query(User).get(user_id)

# Good — the dependency is explicit and injectable
class UserRepository:
    def __init__(self, db_connection):
        self._db = db_connection
    def get_user(self, user_id):
        return self._db.query(User).get(user_id)
```
```python
# Bad — hidden global state, hard to test, breaks under concurrent use
_request_count = 0
def handle_request(req):
    global _request_count
    _request_count += 1
    ...

# Good — state is owned by an object, explicit, and trivially testable in isolation
class RequestHandler:
    def __init__(self):
        self.request_count = 0
    def handle_request(self, req):
        self.request_count += 1
        ...
```
- **DON'T:** Use the `global` keyword to mutate a module-level variable from inside a function as a routine pattern. Needing `global` to make code work is usually a sign the function should instead take the value as a parameter and/or return an updated value, or that the state belongs on an object.
```python
# Bad — a hidden dependency on module state, invisible in the function's signature
_total = 0
def add_to_total(amount):
    global _total
    _total += amount

# Good — the state and the function that mutates it are visibly connected
class RunningTotal:
    def __init__(self):
        self.total = 0
    def add(self, amount):
        self.total += amount
```
- **DON'T:** Use a class purely as a namespace for global-like state and functions with no actual instances ever created (every method effectively `@staticmethod`, no real object identity) — if there's genuinely no per-instance state, a module with plain functions already provides that namespacing in Python, without the ceremony of a class that's never instantiated.
- **DON'T:** Default a function argument to a mutable literal (`def f(items=[])`, `def f(config={})`) — already covered under Idiomatic Python, but worth flagging specifically here because it's a mistake AI-generated code produces with notable frequency, since the syntax reads as intuitively correct to anyone unfamiliar with when Python evaluates default arguments.
- **DON'T:** Introduce a singleton pattern (a module-level instance, a class-level cache dict) as a first resort for "shared configuration" or "shared client" needs, without considering dependency injection. A hidden singleton makes unit testing harder (tests can leak state into each other via the shared instance) and obscures a function's real dependencies from its signature.

### Comments, Docstrings & Over-Explanation

- **DON'T:** Add a comment above every single line restating what that line already says in plain sight (`# create an empty list` above `items = []`, `# call the function` above a function call). This is a frequent AI-generated-code smell — it reads as padding rather than genuine documentation and actually makes the code harder to scan, since real signal (the comments that explain *why*) gets buried among comments with no information content.
```python
# Bad — every line narrated, none of it adds information
# Initialize the total to zero
total = 0
# Loop through each item in the list
for item in items:
    # Add the item's price to the total
    total += item.price
# Return the total
return total

# Good — no comment needed; the code already says what it does
total = sum(item.price for item in items)
```
- **DON'T:** Write a multi-paragraph docstring, with a full `Args`/`Returns`/`Raises` breakdown, for a trivial one-line function whose name and signature already communicate everything (`def is_even(n: int) -> bool: return n % 2 == 0`). Match documentation depth to actual complexity — reserve detailed docstrings for functions with non-obvious behavior, side effects, or subtle argument semantics.
- **DO:** Use comments to explain *why* a piece of code exists in its current, possibly non-obvious form — a workaround for a specific bug in a dependency, a business rule that isn't derivable from the code itself, a deliberate trade-off — rather than *what* the code does, which should be readable from the code itself.
- **DON'T:** Leave placeholder or TODO-style comments in generated code presented as finished work (`# TODO: handle edge cases`, `# implement error handling here`) without either actually implementing that part or explicitly flagging to the user that the code is incomplete. Silently shipping a stub dressed as a complete implementation is worse than clearly stating what's missing.
- **DON'T:** Generate excessive inline type commentary in comments that duplicates an actual type hint already present on the same line (`x: int = 5  # x is an integer`). If the information is already expressed in code (via a type hint, a clear name, a docstring), repeating it in a comment adds no value.
- **DON'T:** Add a "section banner" comment (a row of `#`s or `====` framing a one-word label like `# --- Helper Functions ---`) to visually divide a file into zones, as a substitute for actually splitting that file into separate modules once it's grown large enough to need dividing. A banner comment treats a symptom (a file that's hard to navigate) without fixing the actual cause (too much unrelated content living in one file).
- **DO:** Let a function's name, its parameter names, and its type hints carry as much of the "what does this do" burden as they can before reaching for a comment — a well-named function with well-named parameters and precise types often needs no comment at all to be immediately understandable, which is a better outcome than a poorly-named one propped up with an explanatory comment.

### Testing Anti-Patterns in Generated Code

- **DON'T:** Generate a test that asserts on a mocked return value rather than real behavior, in a way that makes the test tautological — mocking the exact function under test's dependency to return a fixed value, then asserting the function returns that same fixed value, verifies nothing about actual logic.
```python
# Bad — this test can never fail even if calculate_total is completely broken,
# because it mocks the very computation being tested
def test_calculate_total():
    with patch("myapp.orders.calculate_total", return_value=100):
        assert calculate_total(items) == 100

# Good — exercises the real calculation logic against known inputs
def test_calculate_total():
    items = [Item(price=30), Item(price=70)]
    assert calculate_total(items) == 100
```
- **DON'T:** Generate tests only for the success/happy path and skip error cases, edge cases (empty input, `None`, boundary values), and the specific exceptions a function is documented to raise. A test suite that only exercises the easy case gives false confidence.
- **DON'T:** Write a test with an assertion so loose it would pass for almost any output (`assert result is not None`, `assert len(result) > 0`) when a precise assertion on the actual expected value is available and meaningful.
- **DON'T:** Generate a full test file without also verifying it actually runs and passes against the real code (when execution is possible) — a test that has a subtle bug of its own (asserting the wrong thing, referencing an undefined fixture, mocking the wrong target) provides false confidence indistinguishable from a real, passing test until someone actually runs it.

### Ignoring Project Conventions & Existing Code

- **DON'T:** Introduce a different code style, naming convention, or error-handling pattern than what's already established in the surrounding codebase, just because it's not the pattern most commonly seen in training data. Read the existing code first and match its conventions — the goal is a codebase that reads as if one team wrote it, not a patchwork of each contribution's default style.
- **DON'T:** Re-implement a utility function that already exists elsewhere in the project (a validation helper, a formatting function, a retry decorator) instead of finding and reusing it. Duplicate logic drifts out of sync over time and increases the surface area for bugs.
- **DON'T:** Add a new abstraction layer, interface, or configuration option that nothing in the codebase actually needs yet, on the theory that it might be useful someday ("speculative generality"). Unused flexibility has a real maintenance cost and no realized benefit — build the abstraction when a second real use case actually appears, not preemptively.
- **DON'T:** Ignore an already-typed codebase's conventions by adding new, unannotated functions once the rest of the project has type hints throughout, or vice versa — introducing type hints inconsistently, on only some new functions, in a codebase that has deliberately stayed untyped. Match the surrounding code's typing discipline.
- **DON'T:** Silently change or remove existing behavior (an argument's default value, a function's return type, an error type raised) while ostensibly making an unrelated change, without calling out that the behavior changed. An incidental behavior change buried inside a larger diff is exactly the kind of regression code review is least likely to catch.

### Over-Engineering vs. Under-Engineering

- **DON'T:** Wrap a straightforward data-fetch-and-transform task in an unwarranted stack of design-pattern ceremony — an abstract factory, a strategy interface with a single concrete implementation, a plugin registry — for a script that has exactly one, well-known way it will ever be used. Match architectural weight to actual, present requirements, not to hypothetical future flexibility.
```python
# Bad — a factory and strategy interface for a script with one real implementation
class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, price: float) -> float: ...

class StandardDiscountStrategy(DiscountStrategy):
    def calculate(self, price: float) -> float:
        return price * 0.9

class DiscountStrategyFactory:
    @staticmethod
    def create(kind: str) -> DiscountStrategy:
        if kind == "standard":
            return StandardDiscountStrategy()
        raise ValueError(kind)

# Good — this is the entire actual requirement
def apply_standard_discount(price: float) -> float:
    return price * 0.9
```
- **DON'T:** Swing the opposite direction and cram unrelated concerns into one function/module because separating them felt like "too much structure" for a request that's actually going to be maintained and extended long-term (an ongoing application, not a one-off script). Both extremes — needless abstraction and no structure at all — are slop; the fix is reading the actual scope of the request, not defaulting to either pole.
- **DO:** Ask (or reasonably infer from context) whether code is a one-off script, a prototype, or the start of a maintained application before deciding how much structure, testing, and abstraction it warrants — the appropriate amount of engineering ceremony genuinely differs between those cases, and treating a throwaway script like a production service (or vice versa) both count as failures to match effort to purpose.

### Fabricated Confidence & Unverified Claims

- **DON'T:** State that generated code "handles all edge cases," "is fully optimized," or "follows best practices" as a blanket claim without having actually verified specific edge cases, measured performance, or checked the claim against the specific best practices that apply. Vague, unearned confidence is itself a form of slop — it reads as thorough while conveying no actual verified information.
- **DON'T:** Describe a workaround or a partial solution using language that implies a complete, robust fix when it isn't one ("this resolves the race condition" for a fix that narrows the race window but doesn't eliminate it). Be precise about what a change actually does and doesn't guarantee.
- **DO:** Distinguish clearly, in both code comments and any accompanying explanation, between "this is verified to work" (it ran, and its output was checked against an expectation) and "this is expected to work based on the API/documentation but hasn't been executed" — collapsing that distinction is how unverified assumptions end up shipped as if they were tested facts.

### Ignoring Errors and Warnings from Tooling

- **DON'T:** Generate code that a linter or type checker would immediately flag (an unused import, an unreachable branch, a type mismatch) and present it as finished without running or at least mentally checking it against the project's actual configured tooling. If the tools are available, run them; if they aren't available in the current context, say so rather than implying the code has been checked.
- **DON'T:** Suppress a linter/type-checker warning reflexively (a blanket `# noqa`, a file-wide `# type: ignore` comment, or a rule disabled in configuration) as a first response to an inconvenient warning, rather than reading and addressing what the warning is actually telling you. A warning suppressed without understanding is a bug deferred, not a bug fixed.
- **DO:** Treat a persistent warning that keeps needing to be suppressed across multiple pieces of generated code as a signal to change approach (a different API, a different pattern) rather than a signal to keep adding more suppressions.

### Guessing Instead of Checking Project Context

- **DON'T:** Guess at a project's Python version, framework, or dependency versions from generic assumptions rather than checking what's actually declared (`pyproject.toml`'s `requires-python`, an installed package's actual version, a lockfile) when that information is available in the current context. Code written against the wrong assumed version/framework can look plausible while being subtly or entirely wrong for the real target.
- **DON'T:** Assume a database schema, an API's response shape, or a function's actual current signature based on what's typical or common, when the real definition is available to check (a model file, an OpenAPI spec, the actual function source). A plausible guess that happens to be wrong is worse than pausing to look, because it's presented with the same confidence as a verified fact.
- **DO:** When genuinely unable to check something material to correctness (no access to the real schema, the actual library version, etc.), say so explicitly and flag the assumption being made, rather than silently proceeding as if the guess were confirmed fact.

### Dropping Existing Tests, Comments, or Functionality

- **DON'T:** Remove existing tests, comments, logging statements, or error handling while making an unrelated change, without calling it out — a rewritten function that quietly drops the validation or logging the original had is a regression, even if the new code otherwise "looks cleaner."
- **DON'T:** Simplify a function by removing handling for an edge case that was clearly deliberate (a specific `except` clause for a documented failure mode, a guard against a known-bad input) just because that edge case doesn't appear in the immediate task description. If a change's scope doesn't obviously require touching that handling, leave it alone.
- **DO:** Treat existing tests as a specification of current behavior — if a change makes an existing test fail, either the change has broken something that test was correctly protecting, or the test's expectation has genuinely changed and the test itself needs a deliberate, explained update. Silently modifying or deleting a failing test to make it pass, without addressing whether the underlying behavior change was intended, defeats the entire purpose of having the test.

### Miscellaneous AI-Specific Pitfalls

- **DON'T:** Use `os.path` and `pathlib` inconsistently within the same generated file — reaching for whichever one comes to mind for each individual line rather than picking one and using it consistently, which is a distinctive tell of code assembled from disparate training examples rather than written with a single coherent style in mind.
```python
# Bad — mixes both styles in the same function for no reason
def load_config():
    base = Path(__file__).parent
    config_path = os.path.join(str(base), "config.json")  # converts back to os.path here
    with open(config_path) as f:
        return json.load(f)

# Good — one consistent style throughout
def load_config():
    config_path = Path(__file__).parent / "config.json"
    return json.loads(config_path.read_text())
```
- **DON'T:** Generate code that imports a module but never uses it, or uses a name that was never imported (assuming a name is available because it's commonly imported "by convention" in similar code, without an actual `import` statement present in this file).
- **DON'T:** Silently downgrade to a less capable but "safer-looking" approach that doesn't actually do what was asked — e.g., asked to parse a moderately complex format with a real parser, instead assembling ad hoc string-splitting/regex code that only handles the example case shown in the prompt and breaks on realistic variations.
- **DON'T:** Generate a solution to a narrower problem than what was actually asked, quietly, without flagging the gap (implementing only the "create" operation when the request described full CRUD, or handling only the success path when the request implied error handling was part of the job). If full scope wasn't completed, say what's missing rather than presenting a partial implementation as the complete answer.
- **DON'T:** Repeat a mistake within the same response after already correcting it once — e.g., using a hallucinated function name in one code block, then reusing that same wrong name in a second code block later in the same response even after having gotten the correct name right in between. Internal consistency within one response is a baseline, not an aspiration.
- **DON'T:** Present generated code as fully tested, benchmarked, or "production ready" when none of that verification actually happened. State plainly what was and wasn't verified (does it run, was it tested against real data, was it profiled) rather than implying a level of confidence the actual work doesn't support.
- **DON'T:** Default to synchronous code and then bolt on `async`/`await` keywords superficially without actually restructuring for concurrency (e.g., making a function `async def` but filling it with blocking calls and no `await` on anything that benefits from it, discussed under Async Python) — a common tell of "this looks like it should be async" pattern-matching rather than a reasoned concurrency design.
- **DON'T:** Generate a class with manually written getter/setter methods (`get_name()`/`set_name()`) as a reflex from other languages' conventions. Python idiom is direct attribute access, with `@property` reserved for the specific case of needing computed or validated access — not a blanket Java/C#-style accessor pattern applied to every attribute.
- **DON'T:** Write an explicit `__init__` that only assigns constructor arguments to identically-named attributes, with no other logic, for a class that's really just a data container — that's exactly what `@dataclass` generates automatically, and hand-writing it is pure boilerplate that also has to be kept in sync by hand if a field is ever added or removed.
```python
# Bad — unidiomatic Java-style accessors for a plain attribute
class User:
    def __init__(self, name):
        self._name = name
    def get_name(self):
        return self._name
    def set_name(self, name):
        self._name = name

# Good — direct attribute access; add @property only if validation/computation is needed
class User:
    def __init__(self, name):
        self.name = name
```
- **DON'T:** Assume the newest language feature is always the right choice regardless of the project's actual minimum supported Python version — using `match`/`case`, `X | Y` union syntax, or `tomllib` in code targeting a codebase whose `requires-python` explicitly supports older versions that don't have them, without flagging the version mismatch.
- **DO:** State assumptions explicitly when they materially affect the generated code — the Python version targeted, which library version's API was assumed, whether a function was actually tested — rather than leaving them implicit and letting the user discover a mismatch later.
- **DON'T:** Generate a solution using a heavyweight dependency (a full ORM for a script that touches one table once, a large ML framework for a task solvable with basic statistics) when a much simpler, dependency-free approach solves the actual stated problem just as well. Reaching for the most sophisticated available tool isn't the same as reaching for the *right* one.
- **DON'T:** Produce code that technically satisfies the literal wording of a request while missing its obvious practical intent (writing a function that only handles the single example input shown in the prompt, rather than the general case the example was clearly illustrating). Read past the literal example to the actual underlying requirement.
- **DO:** Ask a clarifying question (or state a reasonable, explicit default assumption) when a request is genuinely ambiguous in a way that would lead to materially different code, rather than silently picking one interpretation and presenting it as the obvious, only reading.

### Reinventing the Standard Library

- **DON'T:** Hand-write a utility that a standard-library module already provides correctly and efficiently — a custom deep-copy function instead of `copy.deepcopy`, manual JSON-escaping instead of the `json` module, a hand-rolled CSV parser instead of the `csv` module, manual date-arithmetic instead of `datetime`/`dateutil`. Reimplementations skip edge cases the standard version has already handled (encoding quirks, quoting rules, leap years, timezone math) and add a maintenance burden for no benefit.
```python
# Bad — a hand-rolled CSV "parser" that breaks on quoted fields containing commas
def parse_csv_line(line):
    return line.strip().split(",")

# Good — the standard library already handles quoting, escaping, and edge cases
import csv
def parse_csv_line(line):
    return next(csv.reader([line]))
```
- **DON'T:** Manually implement a retry loop, an LRU cache, a thread pool, or an argument parser from scratch when `tenacity`/`functools.lru_cache`/`concurrent.futures.ThreadPoolExecutor`/`argparse` already do the job, tested and edge-case-hardened, in the standard library or a well-known dependency already used elsewhere in the ecosystem.
- **DO:** Search for an existing standard-library or well-established third-party solution before writing custom utility code for a common problem (parsing, validation, retrying, caching, CLI argument handling, date/time math) — treat "this feels like a solved problem" as a prompt to check, not an assumption to skip.
```python
# Bad — a hand-rolled "deep merge" that mishandles several real cases
# (lists, non-dict values at the same key, etc.)
def merge(a, b):
    result = dict(a)
    for k, v in b.items():
        if k in result and isinstance(result[k], dict):
            result[k] = merge(result[k], v)
        else:
            result[k] = v
    return result

# Good — a maintained library (e.g. deepmerge, or a well-tested vendored
# utility already used elsewhere in the codebase) already handles the
# edge cases this hand-rolled version misses
from deepmerge import always_merger
merged = always_merger.merge(a, b)
```

### Silent Truncation of Large Files or Refactors

- **DON'T:** Present a partial rewrite of a large file as if it were complete — dropping unrelated functions, trailing code, or entire classes that existed in the original file without calling out the omission. A silently truncated file handed back to the user as "done" causes real data loss the moment it's saved over the original.
- **DON'T:** Claim a multi-file refactor is finished when only some of the affected call sites were actually updated (e.g., renaming a function in its definition but missing several of its call sites elsewhere in the codebase). Search for every usage of a renamed/changed symbol before declaring the refactor complete, and say explicitly which files were and weren't touched.
- **DO:** When a file is too large to comfortably reproduce in full, say so explicitly and either work section-by-section with clear boundaries, or make a targeted, minimal diff-style edit instead of regenerating the whole file from memory and risking silently dropping content that wasn't the focus of the change.
```python
# Bad — asked to add one new field to a dataclass, but the whole file was
# regenerated from memory and quietly lost an unrelated method (calculate_tax)
# that existed in the original file
@dataclass
class Invoice:
    id: int
    total: float
    currency: str  # the newly requested field
    # calculate_tax() existed in the original file and is now silently gone

# Good — a minimal, targeted diff that only touches what was actually asked for
@dataclass
class Invoice:
    id: int
    total: float
    currency: str  # added field

    def calculate_tax(self) -> float:  # preserved, untouched
        return self.total * TAX_RATE
```

### Copy-Pasted Boilerplate Instead of Abstraction

- **DON'T:** Repeat the same multi-line block (a database-connection setup, an API-request-and-retry pattern, a validation sequence) across several generated functions instead of factoring it into one shared helper. Generated code is especially prone to this because each function is often produced somewhat independently, without cross-referencing what was written a few functions earlier in the same response.
```python
# Bad — the same "retry up to 3 times with backoff" logic re-implemented
# slightly differently in each function that happens to need it
def fetch_user_profile(user_id):
    for attempt in range(3):
        try:
            return api.get(f"/users/{user_id}")
        except TimeoutError:
            time.sleep(2 ** attempt)
    raise ServiceUnavailableError()

def fetch_order_history(user_id):
    for i in range(3):
        try:
            return api.get(f"/orders/{user_id}")
        except TimeoutError:
            time.sleep(2 ** i)
    raise ServiceUnavailableError()

# Good — one shared, correctly-implemented retry helper
@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1))
def fetch_user_profile(user_id):
    return api.get(f"/users/{user_id}")

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1))
def fetch_order_history(user_id):
    return api.get(f"/orders/{user_id}")
```
```python
# Bad — the same connect/commit/close dance repeated in every function
def create_user(name, email):
    conn = get_connection()
    try:
        conn.execute("INSERT INTO users (name, email) VALUES (?, ?)", (name, email))
        conn.commit()
    finally:
        conn.close()

def delete_user(user_id):
    conn = get_connection()
    try:
        conn.execute("DELETE FROM users WHERE id = ?", (user_id,))
        conn.commit()
    finally:
        conn.close()

# Good — the repeated pattern is factored into one place
@contextmanager
def db_transaction():
    conn = get_connection()
    try:
        yield conn
        conn.commit()
    finally:
        conn.close()

def create_user(name, email):
    with db_transaction() as conn:
        conn.execute("INSERT INTO users (name, email) VALUES (?, ?)", (name, email))

def delete_user(user_id):
    with db_transaction() as conn:
        conn.execute("DELETE FROM users WHERE id = ?", (user_id,))
```
- **DON'T:** Generate several near-identical functions that differ only by one or two hardcoded values, instead of one parameterized function. Near-duplicate functions are a maintenance trap — a bug fix applied to one copy is easy to forget applying to the others.

### Inconsistent Style Within a Single Response

- **DON'T:** Switch conventions midway through one generated file or one response — starting with f-strings and switching to `.format()` partway through, mixing double and single quotes inconsistently, or using type hints on the first few functions and dropping them for the rest. A single response should read as internally consistent even before it's ever reviewed against the rest of the codebase.
- **DON'T:** Vary error-handling style within one file generated in a single response (some functions raising, others returning `None`, others printing and continuing) without a stated reason — this is a smaller-scale version of the project-convention-drift problem, but occurring within a single piece of generated output where there's no excuse of "matching pre-existing code."
- **DO:** Re-read a generated file for internal consistency (naming, quote style, error handling, typing discipline) before presenting it as finished, the same way a careful human author would proofread before committing.

### Misusing `__init__.py` and Package Structure

- **DON'T:** Put substantial business logic directly inside `__init__.py`. It's a common AI-generated-code habit to treat `__init__.py` as "the main file" of a package, but its conventional role is re-exporting the package's public API (or being empty) — real implementation belongs in named submodules that `__init__.py` imports from.
```python
# Bad — mypackage/__init__.py containing the actual implementation
class UserService:
    def get_user(self, user_id): ...
    def create_user(self, name): ...

# Good — mypackage/services.py holds the implementation;
# mypackage/__init__.py just curates what's public
# mypackage/services.py
class UserService:
    def get_user(self, user_id): ...
    def create_user(self, name): ...

# mypackage/__init__.py
from mypackage.services import UserService
__all__ = ["UserService"]
```
- **DON'T:** Create circular imports by having two modules import from each other at module scope, then "fix" it with a local import buried inside a function as a permanent workaround rather than restructuring the actual dependency (often by extracting the shared piece both modules need into a third module). A local import can be a legitimate, deliberate choice in specific cases (breaking a genuine cycle, deferring an expensive/optional import), but reaching for it reflexively to silence an `ImportError` usually just papers over a design problem.

### Non-Idempotent Generated Scripts

- **DON'T:** Generate a setup/migration/data-seeding script that fails or corrupts data if run a second time, without calling that out. A script that does `CREATE TABLE users (...)` with no `IF NOT EXISTS` guard, or that unconditionally `INSERT`s seed data with no uniqueness check, breaks the moment someone re-runs it after a partial failure — which is a common, expected occurrence for exactly this class of script.
```python
# Bad — fails on every run after the first
cursor.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, email TEXT)")
cursor.execute("INSERT INTO users (email) VALUES ('admin@example.com')")

# Good — safe to run repeatedly
cursor.execute("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, email TEXT UNIQUE)")
cursor.execute(
    "INSERT INTO users (email) VALUES (?) ON CONFLICT(email) DO NOTHING",
    ("admin@example.com",),
)
```
- **DO:** Design setup, migration, and seed scripts to be idempotent by default (safe to run multiple times with the same end result) unless there's a specific, stated reason not to — this is a normal expectation for this category of script in real operational use, not an edge case.

### Fabricated File Paths & Project Structure

- **DON'T:** Reference a file path, module location, or configuration filename that sounds conventional but wasn't actually confirmed to exist in the project (assuming `src/config.py` exists when the project's actual layout is `app/settings/base.py`, or assuming a `.env.example` exists when it doesn't). A plausible-sounding path is still a guess, and code that imports from or writes to a guessed path fails immediately or, worse, silently creates an unintended duplicate file.
- **DON'T:** Assume a standard project layout (a `src/` directory, a specific test directory name, a particular config file location) applies universally without checking the actual repository structure first — conventions vary meaningfully between ecosystems, frameworks, and individual projects, and importing against the wrong assumed structure produces an immediate, confusing `ModuleNotFoundError` at best.
- **DO:** List or inspect the actual directory structure before generating code that references specific file paths, module names, or import paths, when that information is available — this is a small check that prevents a whole class of otherwise-plausible-looking broken code.

### Grouping & Counting Idioms

- **DO:** Use `itertools.groupby` (on pre-sorted input, since it only groups *consecutive* matching elements) or a `collections.defaultdict(list)` accumulation (for input that isn't already sorted/grouped) to group items by a key, instead of a manual nested-loop grouping implementation.
```python
from itertools import groupby
from operator import attrgetter

# groupby requires the input already sorted by the grouping key
orders_by_status = {
    status: list(group)
    for status, group in groupby(sorted(orders, key=attrgetter("status")), key=attrgetter("status"))
}

# defaultdict works regardless of input order, and is often simpler to reach for
from collections import defaultdict
orders_by_status = defaultdict(list)
for order in orders:
    orders_by_status[order.status].append(order)
```
- **DON'T:** Call `itertools.groupby` on input that isn't sorted by the grouping key and expect it to group all matching elements together — it only groups *consecutive* runs, so unsorted input silently produces multiple separate groups for the same key instead of one, which is a common and easy-to-miss correctness bug.

## Quick Checklist
- **DO** follow PEP 8 naming: `snake_case` functions/variables, `PascalCase` classes, `UPPER_SNAKE_CASE` constants.
- **DON'T** shadow builtins (`list`, `dict`, `id`, `type`) or use ambiguous single-letter names outside tiny loop scopes.
- **DO** write imperative, PEP 257-style docstrings for public modules, classes, and functions — summary line first.
- **DON'T** pad code with comments that restate what the next line already says; comment the *why*, not the *what*.
- **DO** annotate public function signatures with type hints and run mypy/pyright in CI, ideally in strict mode.
- **DON'T** default to `typing.Any` as an escape hatch, or leave `# type: ignore` unexplained.
- **DO** validate external/untrusted data at runtime (Pydantic or equivalent) — static types alone don't enforce anything at runtime.
- **DO** prefer comprehensions and generator expressions over manual append-loops; use generators for large/lazy data.
- **DO** use `with` for every resource needing deterministic cleanup — files, locks, connections, transactions.
- **DON'T** use a mutable object (`list`, `dict`, `[]`, `{}`) as a default argument value — use `None` and initialize inside the function.
- **DO** use `@dataclass` for data-holding classes, with `field(default_factory=...)` for mutable defaults and `frozen=True` for immutable ones.
- **DO** use `pathlib.Path` instead of `os.path` string manipulation in new code, consistently within a file.
- **DON'T** use bare `except:`, and don't catch `Exception` broadly only to log-and-swallow it — catch specific types.
- **DO** build a custom exception hierarchy rooted in one base exception per package; use `raise ... from exc` to chain causes.
- **DON'T** use `assert` to validate user input or API arguments — it's stripped under `python -O`; raise a real exception instead.
- **DO** return `None`/sentinels only when genuinely ambiguity-free; prefer raising a specific exception for real failures.
- **DO** manage dependencies via `pyproject.toml` plus a lockfile (uv/Poetry/pip-tools); always work inside a virtual environment.
- **DON'T** install packages into the system/global Python, and don't commit `.venv`/`venv` directories to version control.
- **DO** run `ruff`/`black` (or `ruff format`) and a type checker in CI as required, non-skippable checks, configured in `pyproject.toml`.
- **DON'T** suppress a lint/type warning with a blanket, unexplained `# noqa` — scope it to the specific rule with a reason.
- **DO** write pytest-style tests with fixtures for setup/teardown, `@pytest.mark.parametrize` for repeated cases, and `tmp_path`/`monkeypatch` over ad hoc equivalents.
- **DON'T** over-mock a test until it no longer exercises real logic, or assert only on mock-call-happened rather than actual outcomes.
- **DO** keep tests isolated, order-independent, and fast; separate slow integration/E2E tests from the unit suite.
- **DON'T** use `time.sleep()` to wait on async/concurrent conditions in tests — poll with a timeout or use the library's sync primitives.
- **DO** use `async`/`await` for I/O-bound concurrency; never call a blocking synchronous function directly inside a coroutine.
- **DON'T** create a "fire and forget" `asyncio.create_task()` without keeping a strong reference to it — it can be garbage-collected mid-flight.
- **DO** use `asyncio.TaskGroup`/`gather()` to run independent coroutines concurrently instead of awaiting them sequentially.
- **DO** set explicit timeouts on all awaited I/O and all outbound HTTP requests — never await indefinitely on a hung connection.
- **DON'T** check membership against a large `list` repeatedly in a loop — use a `set`/`dict` for O(1) lookups.
- **DO** vectorize numeric work over large arrays (NumPy/pandas) instead of iterating element-by-element in pure Python.
- **DON'T** build large strings with repeated `+=` in a loop — use `str.join()`.
- **DO** profile (`cProfile`, `py-spy`, `line_profiler`) before optimizing; re-profile after each change to confirm impact.
- **DON'T** ever build SQL by interpolating user input into a query string — use parameterized queries/an ORM, always.
- **DON'T** pass `shell=True` to `subprocess` (or use `os.system`) with any externally-influenced input — pass an argument list instead.
- **DON'T** call `eval()`/`exec()` on anything not fully trusted and hardcoded; don't `pickle.loads()` untrusted data; use `yaml.safe_load()`, never `yaml.load()`.
- **DO** use the `secrets` module (not `random`) for tokens/keys, `hmac.compare_digest()` for secret comparisons, and a real password-hashing library (bcrypt/argon2) for credentials.
- **DON'T** hardcode secrets/credentials in source; read them from environment variables or a secrets manager, never commit `.env`.
- **DO** verify webhook signatures before trusting payloads, and allowlist fields explicitly on any endpoint that writes from client input (avoid mass assignment).
- **DO** use the `logging` module, not `print()`, for all runtime diagnostics; choose log levels deliberately and configure logging once at the entry point.
- **DON'T** log secrets, tokens, or full PII, even at `DEBUG` level; use lazy `%s` formatting in log calls on hot paths, not f-strings.
- **DON'T** hallucinate library functions, parameters, or method signatures — verify against the actual installed version before relying on them.
- **DON'T** mix Python 2 idioms into Python 3 code: no `print` statements, `xrange`, `.has_key()`, `raw_input()`, or old-style `except Err, e:`.
- **DON'T** write every task as a flat, top-level script with no functions and no `if __name__ == "__main__":` guard — structure and test real applications.
- **DON'T** reach for a global mutable variable as the default way to share state — pass parameters explicitly or use a class/context object.
- **DON'T** over-comment obvious code, or under-document genuinely non-obvious behavior, side effects, and workarounds.
- **DON'T** silently drop existing functionality, tests, comments, or error handling while making an unrelated change — call out every behavior change.
- **DO** match structural/architectural weight to actual scope — neither over-engineer a one-off script nor leave a real application unstructured.
- **DON'T** present unverified code as tested, optimized, or "handles all edge cases" — state plainly what was and wasn't actually checked.
- **DO** read and match a codebase's existing conventions (naming, error handling, typing discipline) instead of introducing a competing style.
- **DO** guard clauses over deep nesting; keep boolean conditions free of double negatives and redundant `== True`/`== False` comparisons.
- **DO** use `enum.Enum` for closed sets of related constants instead of loose module-level string/int literals.
- **DO** use timezone-aware `datetime` objects (`datetime.now(timezone.utc)`) everywhere; never compare or store naive datetimes across timezone boundaries.
- **DON'T** use floating-point arithmetic for money — use `decimal.Decimal` constructed from strings, or integer minor units.
- **DO** pin dependencies via a lockfile with hashes for reproducible, tamper-resistant installs; scan for known vulnerabilities (`pip-audit`) in CI.
- **DON'T** grant CI/CD credentials broad, unscoped access, and never pin a third-party CI action to a mutable tag instead of a commit SHA.
- **DO** use guard clauses and early returns to keep validation/edge-case handling flat instead of deeply nested `if` blocks.
- **DON'T** retry a non-idempotent operation (a payment, a resource-creating POST) without an idempotency key — a retried timeout can duplicate the side effect.
- **DO** design setup, migration, and seed scripts to be idempotent by default — safe to re-run after a partial failure.
- **DON'T** trust a client-declared `Content-Length`, MIME type, or file extension — enforce real size limits and validate actual content.
- **DO** treat a passing type checker and a green lint run as a floor, not a substitute for tests, correctness review, or actually running the code.
