# Security & Application Security (AppSec)

## Injection Attacks

Injection flaws happen when untrusted data is interpreted as code, commands, markup, or query syntax instead of being treated purely as data. They remain one of the most common and most damaging classes of vulnerability precisely because the fix — keep data and code channels separate — is simple to state and easy to skip under deadline pressure. Every rule in this subsection reduces to one idea: never build an executable string by concatenating trusted syntax with untrusted values.

### SQL Injection

- **DO:** Use parameterized queries (prepared statements) for every database call that includes any value not hardcoded by the developer. The database driver sends the query structure and the data as separate channels, so user input can never be reinterpreted as SQL syntax no matter what characters it contains.
- **DON'T:** Build SQL strings by concatenating or interpolating user input directly into the query text. A single unescaped quote character in a name field can change a `WHERE` clause's meaning, dump unrelated tables, or bypass a login check entirely.
```python
# BAD: user input is spliced directly into SQL syntax
query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'"
cursor.execute(query)

# GOOD: the driver binds parameters out-of-band from the query text
cursor.execute(
    "SELECT * FROM users WHERE username = %s AND password_hash = %s",
    (username, password_hash),
)
```
- **DO:** Prefer your ORM's or query builder's parameter-binding API (`.filter()`, `.where()`, bound placeholders) over its "raw query" escape hatch whenever the query can be expressed that way. ORMs exist specifically to keep query construction declarative and safe; reaching for `.raw()`/`.execute(rawSql)` reintroduces every SQL-injection risk the ORM was meant to remove.
- **DON'T:** Assume an ORM makes a codebase immune to SQL injection. Most ORMs expose a raw-SQL or raw-fragment method for cases the query builder can't express, and string-concatenating user input into that raw method is exactly as dangerous as raw SQL in any other language.
```javascript
// BAD: ORM used, but the raw-query escape hatch defeats the protection
const users = await db.$queryRawUnsafe(
  `SELECT * FROM users WHERE email = '${email}'`
);

// GOOD: same ORM, parameterized raw query
const users = await db.$queryRaw`SELECT * FROM users WHERE email = ${email}`;
```
- **DO:** Use allowlists (a fixed set of known-valid values) when a query needs a dynamic identifier such as a table name, column name, or sort direction, since those positions cannot be parameterized by the database driver. Map the user-supplied value to one of a small set of literal, developer-controlled strings before it ever reaches the query builder.
```python
# BAD: column name taken directly from user input, cannot be parameterized
order = request.args.get("sort")
cursor.execute(f"SELECT * FROM products ORDER BY {order}")

# GOOD: user input only selects among developer-controlled literals
ALLOWED_SORT_COLUMNS = {"price": "price", "name": "name", "newest": "created_at"}
column = ALLOWED_SORT_COLUMNS.get(request.args.get("sort"), "created_at")
cursor.execute(f"SELECT * FROM products ORDER BY {column}")
```
- **DON'T:** Rely on manual escaping functions (hand-rolled quote-doubling, blacklisting `'` or `;`) as a substitute for parameterization. Escaping routines are notoriously easy to get wrong across encodings, nested quoting contexts, and driver-specific quirks, and a single missed edge case reopens the vulnerability.
- **DO:** Apply the principle of least privilege to the database account the application uses: grant only the specific `SELECT`/`INSERT`/`UPDATE`/`DELETE` permissions on the specific tables the application needs, never a superuser or owner-level account. If injection does occur despite other defenses, a restricted account limits the blast radius to what that account can actually do.
- **DON'T:** Let one application-wide database credential have DDL rights (`DROP`, `ALTER`, `CREATE`), access to unrelated schemas, or the ability to read system tables the application never touches. A successful injection against an over-privileged account can drop tables or exfiltrate credentials it never needed access to in the first place.
- **DO:** Treat second-order SQL injection as a real risk: data that was safely parameterized on the way into the database can still become dangerous if it is later read out of the database and concatenated into a *new* raw query. Parameterize on every query that uses that value, not just the one that first stored it.
- **DO:** Watch for injection in less obvious query surfaces — `ORDER BY`, `LIMIT`/`OFFSET`, dynamic `IN (...)` lists, full-text search expressions, and stored procedure calls built by string concatenation — since these are frequently overlooked while the main `WHERE` clause is properly parameterized.
- **DON'T:** Suppress or ignore database errors that surface only under injection-probing input (mismatched quote counts, syntax errors from a single `'` character) as "flaky test noise." That kind of error is often the first observable sign an endpoint is vulnerable, and it should be investigated rather than silenced.
- **DO:** Use static analysis / SAST tooling (e.g., linters or IDE plugins with taint-tracking rules) in CI to flag string-built queries automatically, since manual code review alone reliably misses a handful of injection sites in any codebase past a few hundred database calls.
- **DON'T:** Trust "it's just an internal admin tool" or "only employees can reach this endpoint" as a reason to skip parameterization. Internal tools are frequently the entry point attackers pivot through after compromising a single low-privileged account, and insider threat is a real risk category on its own.

### Command Injection

- **DO:** Call operating-system commands through APIs that take an argument array, never through a shell string, whenever the language offers that choice. Passing arguments as a list means the shell never re-parses them, so metacharacters like `;`, `|`, `&&`, or backticks in user input cannot chain in an extra command.
```python
# BAD: shell=True hands the whole string to a shell for interpretation
subprocess.run(f"convert {filename} output.png", shell=True)

# GOOD: arguments are passed as a list, never parsed by a shell
subprocess.run(["convert", filename, "output.png"], shell=False)
```
- **DON'T:** Enable a shell interpreter (`shell=True`, `exec()`, backticks, `os.system`) for a command whose arguments include any value derived from user input, a filename supplied by a request, or data pulled from an upload. The shell layer is exactly the thing that turns a data value into executable syntax.
- **DO:** Avoid invoking a general-purpose shell at all when a language-native library achieves the same result. Prefer a library that manipulates images, archives, or PDFs in-process over shelling out to `convert`, `zip`, or `pdftk`, which removes the injection surface entirely along with the process-spawning overhead.
- **DON'T:** Rely on a denylist of "dangerous characters" (semicolons, pipes, backticks) to sanitize input destined for a shell command. Shell metacharacter sets differ across shells and locales, and denylists are reliably incomplete; allowlisting the expected input format (e.g., "digits only," "matches this filename pattern") is the safer approach when shelling out cannot be avoided.
```javascript
// BAD: denylist approach, easy to bypass with an overlooked metacharacter
function isSafe(input) {
  return !/[;&|`$]/.test(input);
}

// GOOD: allowlist approach, only known-good shapes pass
function isSafeFilename(input) {
  return /^[a-zA-Z0-9_-]{1,64}\.pdf$/.test(input);
}
```
- **DO:** Validate and canonicalize filenames and paths before passing them anywhere near a command or a filesystem call, since command injection and path traversal frequently travel together in file-processing features (image conversion, log retrieval, report generation).
- **DON'T:** Build shell commands from configuration values, environment variables, or third-party API responses without the same scrutiny given to direct user input. Any data that originates outside the process boundary — including a value round-tripped through a database that a different, less-trusted service wrote — should be treated as untrusted at the point it reaches a command invocation.
- **DO:** Run subprocesses with the minimum OS-level privilege they need (a dedicated low-privilege user, a restricted container, dropped capabilities) so that if a command injection does occur, it cannot escalate to full host compromise.
- **DO:** Set explicit timeouts and resource limits on any spawned subprocess, since command injection is often paired with resource-exhaustion attempts (fork bombs, infinite loops) once an attacker has any code-execution foothold.
- **DON'T:** Use `eval()`, `exec()`, or a language's dynamic-code-execution primitive to interpret user-supplied "formulas," "expressions," or "scripts" as a shortcut. If a feature genuinely needs a mini expression language, use a purpose-built sandboxed expression evaluator with a restricted grammar, not the host language's own interpreter.

### Cross-Site Scripting (XSS)

- **DO:** Encode output for the specific context it is rendered into — HTML-entity encoding for HTML body content, attribute encoding for HTML attributes, JavaScript-string encoding inside `<script>` blocks, and URL encoding inside `href`/`src` values. The correct escaping function depends entirely on where the data lands, not just that it "gets escaped somehow."
```html
<!-- BAD: raw user value dropped into an attribute with no context-aware encoding -->
<div title="<%= comment.author %>">

<!-- GOOD: attribute-context encoding applied by the templating engine -->
<div title="<%= escapeHtmlAttribute(comment.author) %>">
```
- **DON'T:** Assume one generic "sanitize" or "clean" function is safe for every output context. A function that safely escapes HTML body text can still allow a `javascript:` URL through an `href` attribute or leave a string open to breaking out of a `<script>` block.
- **DO:** Rely on your templating engine's automatic contextual escaping (Jinja2 autoescape, React's JSX text interpolation, Angular's default binding, Rails' ERB `<%= %>`) rather than manually building HTML strings, and keep autoescaping enabled — never globally disable it "to make formatting easier."
- **DON'T:** Use `innerHTML`, `document.write`, `dangerouslySetInnerHTML`, Vue's `v-html`, or Angular's `bypassSecurityTrustHtml` to inject a value that includes any user-controlled or third-party-sourced content, without first passing it through a dedicated HTML sanitizer. These APIs render markup as live HTML/DOM, so any `<script>`, `onerror=`, or `javascript:` payload inside the value executes.
```javascript
// BAD: raw user content rendered as live HTML
element.innerHTML = comment.body;

// GOOD: either avoid innerHTML for user content, or sanitize with a vetted library first
element.textContent = comment.body; // no markup rendering at all
// -- or, if HTML formatting is genuinely required --
element.innerHTML = DOMPurify.sanitize(comment.body);
```
- **DO:** Use a well-maintained sanitization library (e.g., a widely used HTML sanitizer) with an explicit allowlist of permitted tags and attributes when a feature genuinely requires rendering user-authored rich text (comments with formatting, markdown previews). Configure it to strip `<script>`, event handler attributes (`onclick`, `onerror`), and `javascript:`/`data:` URLs by default rather than trying to denylist them by hand.
- **DON'T:** Write a custom regex-based HTML sanitizer. HTML parsing has enough edge cases (malformed tags, encoding tricks, mutation-XSS where a browser's own parser normalizes supposedly-safe markup into something dangerous) that hand-rolled sanitizers are routinely bypassed; use a library maintained by people who track browser parsing quirks full-time.
- **DO:** Deploy a Content Security Policy (CSP) as a defense-in-depth layer against XSS, restricting script sources to trusted origins and disallowing inline script execution (`script-src 'self'` rather than allowing `'unsafe-inline'`). A CSP does not replace output encoding, but it meaningfully limits what a successfully injected script can do or where it can load from.
```
# BAD: overly permissive CSP that allows inline scripts and any script source
Content-Security-Policy: script-src 'unsafe-inline' *;

# GOOD: restrictive CSP with a nonce for the few inline scripts that are unavoidable
Content-Security-Policy: script-src 'self' 'nonce-{random-per-request-value}'; object-src 'none'; base-uri 'self';
```
- **DON'T:** Add `'unsafe-inline'` or `'unsafe-eval'` to a CSP as a quick fix to stop console errors when a legacy inline `<script>` block or `eval`-based library breaks under a new policy. Both directives remove most of the XSS protection the CSP was meant to provide; migrate the inline script to a nonce/hash-based approach or an external file instead.
- **DO:** Distinguish the three XSS categories when reasoning about a fix — reflected (payload comes from the current request, e.g., a search query echoed into results), stored (payload is persisted and served to other users later, e.g., a comment), and DOM-based (the payload never touches the server; client-side JavaScript itself writes untrusted data into the DOM via `location`, `document.referrer`, or a URL fragment). All three need output encoding, but DOM-based XSS often lives entirely in client code that server-side scanners never see.
- **DON'T:** Treat DOM-based XSS as covered by server-side output encoding alone. Client-side JavaScript that reads from `location.hash`, `window.name`, `postMessage` data, or `document.referrer` and writes it into the DOM needs the same context-aware encoding applied on the client, since the server never processes that data at all.
- **DO:** Set cookies that carry session identifiers or authentication tokens as `HttpOnly`, which prevents JavaScript (including any XSS payload that does execute) from reading them via `document.cookie`. This does not stop XSS from happening, but it removes session-hijacking as a payoff for a successful XSS injection.
- **DO:** Use Trusted Types (where the target browsers support it) or an equivalent framework-level guardrail to enforce that only sanitized, framework-approved values can ever reach a DOM XSS sink like `innerHTML`, closing off accidental unsanitized assignments at the platform level rather than relying purely on code review.
- **DON'T:** Reflect user input into HTTP response headers (a custom header echoing a query parameter, a redirect `Location` built from user input) without encoding, since header injection can be chained into response-splitting or reflected script execution depending on how the value is later consumed.

### Template Injection (SSTI)

- **DO:** Keep a hard boundary between "data passed into a template" and "template source text itself." A template engine should only ever render a fixed, developer-authored template file with user data bound into its variables — never compile a template whose *source* is built by concatenating user input.
```python
# BAD: user input becomes part of the template source, not just template data
template = Template("Hello " + user_supplied_name + "!")
template.render()

# GOOD: user input is only ever bound as data into a fixed template
template = Template("Hello {{ name }}!")
template.render(name=user_supplied_name)
```
- **DON'T:** Pass user-controlled strings into a template engine's `render_template_string`/`Template(...)`-style API that compiles arbitrary template source, ever. Most template languages (Jinja2, Twig, Freemarker, Velocity, Handlebars) expose enough expressiveness in their syntax that a successfully injected template fragment can reach arbitrary code execution, not just markup injection.
- **DO:** Use the "render a named, static template file with a data context" pattern exclusively, and if a feature genuinely needs user-authored templates (an email-template editor, a custom-report builder), run that rendering in a sandboxed subprocess or a template engine specifically designed for untrusted authors with a restricted expression grammar and no filesystem/network access from within the template.
- **DON'T:** Assume a template engine's default "sandboxed mode" (where one exists) is airtight without checking its documented limitations. Several mainstream template engines have had sandbox-escape vulnerabilities discovered after being marketed as safe for untrusted templates; treat "sandboxed" template execution as reducing risk, not eliminating it, and prefer avoiding untrusted template source entirely.
- **DO:** Recognize that SSTI and XSS look similar on the surface (both may reflect a payload in the rendered output) but require different fixes — output encoding stops XSS but does nothing for SSTI, because the injection happens during template *compilation*, before any encoding step runs. If probing a value with a template expression (something the engine would evaluate, like a simple arithmetic expression in the engine's own syntax) produces an evaluated result rather than literal text, that signals template injection rather than plain XSS.

### NoSQL, LDAP, and Other Query-Language Injection

- **DO:** Use the driver's structured query-builder API for NoSQL databases (MongoDB's query objects, not string-concatenated JSON; Elasticsearch's query DSL objects) rather than building a query document by string-interpolating user input into JSON text, since the same "data becomes syntax" problem applies to JSON-based query languages.
```javascript
// BAD: user input directly forms part of the query object's structure
const query = JSON.parse(`{"username": "${username}", "password": "${password}"}`);
db.collection('users').findOne(query);

// GOOD: user input is only ever a scalar value bound to a known field
db.collection('users').findOne({ username: String(username), passwordHash: hash });
```
- **DON'T:** Pass request body fields directly into a NoSQL query without validating their *type*, not just their content. A JSON API that accepts `{"username": {"$ne": null}}` instead of a string and forwards it unchecked into a MongoDB query can let an operator-injection payload bypass an authentication check entirely; explicitly cast and validate that fields expected to be strings are actually strings before they reach the query.
- **DO:** Parameterize or properly escape values used to build LDAP filter strings when authenticating against or querying a directory service, using the LDAP-specific escaping rules (different special characters than SQL or HTML) or a library that builds filters programmatically rather than by string concatenation.
- **DO:** Apply the same "never concatenate untrusted input into query syntax" discipline to GraphQL resolvers that build downstream database queries, to XPath queries against XML documents, and to any other structured query or expression language the application touches — the underlying principle is identical across all of them even though the syntax differs.
- **DON'T:** Forget that XML parsers processing user-supplied XML can be abused via XML External Entity (XXE) injection if external entity resolution and DTD processing are left enabled. Disable external entity resolution and DTD processing on any XML parser that handles untrusted input, using the parser's documented secure-configuration flags.
```java
// BAD: default XML parser configuration leaves external entities enabled
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();

// GOOD: external entity and DTD processing explicitly disabled
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);
```

### General Principle: Never Build Executable Text From Untrusted Data

- **DO:** Ask, for every place untrusted data flows into a string that is later interpreted as SQL, shell commands, HTML/JS, a template, a regular expression, or any other syntax — "is this value data, or does it become part of the syntax?" If it becomes part of the syntax, use the language/library's structured, parameterized API for that context instead of string building.
- **DON'T:** Treat injection prevention as something bolted on at the end via a generic "sanitize everything" filter applied globally to all input. Sanitization rules are context-dependent (what's safe for a URL path segment is not safe for a shell argument), so a single blanket filter either breaks legitimate input or misses an entire class of injection.
- **DO:** Add injection-focused test cases to the test suite for any endpoint that accepts free-text input and touches a database, a shell command, a template, or another interpreter — asserting that special characters (quotes, semicolons, template delimiters) are stored and rendered as literal data, not reinterpreted as syntax.
- **DON'T:** Rely solely on a web application firewall (WAF) as the fix for an injection vulnerability found in code. A WAF is a useful additional layer that can catch known attack signatures, but it is not a substitute for parameterized queries and proper encoding at the source, and WAF rules are routinely bypassed by encoding variations the rule didn't anticipate.
- **DO:** Treat GraphQL resolvers with the same suspicion as REST handlers when they build downstream queries: a resolver argument is just as untrusted as a query-string parameter, and nesting it three layers deep inside a GraphQL query does not make it safe to concatenate into SQL, a shell command, or a template.
- **DON'T:** Assume that because an ORM's migration or seed scripts are "developer-only" code, string-built SQL there is safe. Seed and migration scripts are often copy-pasted into application code later, or run against production data with values sourced from a CSV or admin form, silently reintroducing the same risk in a place nobody thought to review for injection.
- **DO:** Treat regular-expression construction from user input as its own injection-adjacent risk (ReDoS): an attacker-supplied pattern, or attacker-supplied input matched against a catastrophic-backtracking pattern, can hang a worker thread. Avoid compiling regular expressions directly from user input, and test developer-authored patterns against pathological inputs, or use a regex engine with linear-time guarantees for anything processing untrusted text.
```javascript
// BAD: a nested-quantifier pattern that can backtrack catastrophically
const isValidEmail = /^([a-zA-Z0-9]+)+@example\.com$/;

// GOOD: a simpler, linear pattern (or a length cap plus a vetted email-validation library)
const isValidEmail = /^[a-zA-Z0-9._%+-]{1,64}@example\.com$/;
```
- **DON'T:** Skip threat-modeling injection risk in message-queue consumers and webhook handlers just because they aren't "the web app." A worker that dequeues a message and builds a shell command, database query, or template from the message body is exactly as exploitable as an HTTP endpoint if that queue can be fed by an external or lower-trust system.

### Prototype Pollution (JavaScript/Node.js)

- **DO:** Recognize prototype pollution as its own injection-adjacent class specific to JavaScript's prototype-based object model: an attacker-controlled key like `__proto__`, `constructor`, or `prototype` merged unsafely into an object can modify `Object.prototype` itself, changing the behavior of every object in the running process, not just the one being merged into.
- **DON'T:** Recursively merge or deep-clone a user-controlled object into an existing object (a "merge config," "extend defaults," or JSON-body-to-options pattern) using a naive hand-rolled merge function or an outdated library version known to be vulnerable to prototype pollution, without filtering dangerous keys.
```javascript
// BAD: a naive deep-merge copies attacker-controlled keys, including
// "__proto__", directly onto the target object's prototype chain
function merge(target, source) {
  for (const key in source) {
    if (typeof source[key] === 'object') {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
merge({}, JSON.parse(untrustedBody)); // {"__proto__": {"isAdmin": true}} pollutes globally

// GOOD: dangerous keys are explicitly rejected before merging, and/or a
// merge utility with documented prototype-pollution protection is used
const FORBIDDEN_KEYS = new Set(['__proto__', 'constructor', 'prototype']);
function safeMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (FORBIDDEN_KEYS.has(key)) continue;
    // ...safe recursive merge for remaining keys
  }
  return target;
}
```
- **DO:** Use `Object.create(null)` or a `Map` instead of a plain object literal for any lookup table keyed by user-controlled strings, which sidesteps prototype-chain-based attacks entirely since the resulting object has no inherited prototype to pollute or be confused with.
- **DON'T:** Assume a JSON schema validation pass alone protects against prototype pollution — schema validators historically have had their own bypasses for exactly this pattern, and the safest mitigation is preventing dangerous keys from ever reaching a merge/assignment operation, not relying solely on upstream validation to catch it.

### GraphQL-Specific Injection and Abuse Considerations

- **DO:** Treat every GraphQL resolver argument as untrusted input requiring the same validation and parameterization discipline as a REST request body, regardless of how deeply nested it is inside the query structure — GraphQL's flexible query shape does not change the trust level of the data flowing through it.
- **DON'T:** Let a GraphQL query's field-selection flexibility become a way to bypass validation applied elsewhere in the codebase to the "same" logical operation exposed via REST. It's common for a REST endpoint to have carefully tuned validation that a GraphQL resolver added later for the same underlying operation doesn't fully replicate; audit resolvers against the same validation checklist as their REST equivalents.
- **DO:** Limit query depth, complexity/cost, and result-set size in a GraphQL API, since GraphQL's ability to nest queries arbitrarily deep (`user { posts { comments { author { posts { comments { ... } } } } } }`) creates a resource-exhaustion / denial-of-service vector that a REST API's fixed endpoint shapes don't have by default.

### HTTP Header and Response-Splitting Injection

- **DO:** Encode or validate any user-controlled value before it's written into an HTTP response header (a custom header echoing input, a `Location` header built from a redirect target, a `Set-Cookie` value derived from user input), since a value containing raw CRLF sequences can, on a server that doesn't reject them, inject additional headers or split the response into two, enabling response-splitting and downstream cache-poisoning or reflected-content attacks.
- **DON'T:** Assume a modern web framework's response-header API automatically makes this safe without checking. Most current frameworks do reject or strip newline characters from header values by default, but a raw, low-level socket write, an older framework version, or a custom proxy layer may not — verify the behavior rather than assuming it, particularly in any code that manually constructs raw HTTP responses.

### Path Traversal

- **DO:** Resolve any user-influenced filename or path to its canonical absolute form and verify it still falls within the intended base directory before opening, reading, writing, or deleting a file. Path traversal happens when a value like `../../etc/passwd` or an absolute path is accepted as part of a filename and walks the resulting file operation outside the directory the developer intended to restrict it to.
```python
# BAD: a user-supplied filename is joined directly onto a base directory with
# no check that the result stays within that directory
def read_report(filename):
    path = os.path.join(REPORTS_DIR, filename)
    with open(path) as f:
        return f.read()
# filename = "../../../../etc/passwd" escapes REPORTS_DIR entirely

# GOOD: the resolved absolute path is verified to still be inside the
# intended base directory before the file is touched
def read_report(filename):
    base = os.path.realpath(REPORTS_DIR)
    path = os.path.realpath(os.path.join(base, filename))
    if not path.startswith(base + os.sep):
        raise ValueError("invalid filename")
    with open(path) as f:
        return f.read()
```
- **DON'T:** Rely on stripping `../` sequences from a filename as the fix for path traversal. Naive stripping is bypassable with encoded variants (`%2e%2e%2f`), doubled sequences that reassemble after one stripping pass (`....//`), absolute paths that don't contain `../` at all, or platform-specific separators (`..\\` on Windows) — canonicalizing the full path and checking it against the base directory, as above, is the reliable fix, not pattern-stripping.
- **DO:** Prefer an indirection layer — a generated internal file identifier mapped to a real path in a database, rather than ever accepting a client-supplied path fragment for file access — for any feature where the set of accessible files is meant to be fully controlled by the application rather than named freely by the user.
- **DON'T:** Trust a filename or path extracted from an uploaded archive, a ZIP entry, or a multipart form field without the same canonical-path validation applied to any other user-controlled path, as covered under File Upload Validation above — archive extraction is one of the most common real-world path-traversal vectors specifically because it's easy to overlook that each entry inside the archive is itself untrusted input.

### Email Header Injection

- **DO:** Use your mail-sending library's structured API for setting recipients, subject, and headers (passing each as a distinct parameter) rather than building a raw email header block by string concatenation, since a user-controlled value containing a raw newline can inject additional headers (adding BCC recipients, altering the subject, or turning a contact form into a spam relay) if concatenated directly into header text.
- **DON'T:** Interpolate an unvalidated, user-supplied "from name," subject, or reply-to address directly into raw email header text. Validate and, where the underlying library requires manual header construction, strip or reject embedded newline characters from any field that flows into an email header.

### Parameterized Queries Across Languages

The specific syntax for binding parameters differs by language and database driver, but the underlying principle — the query structure and the data are sent to the database as separate channels — is identical everywhere. The following examples all express the same safe pattern in different ecosystems, for reference when reviewing or writing database code outside the primary examples used earlier in this section.

```java
// Java: PreparedStatement binds parameters out-of-band from the SQL text
String sql = "SELECT * FROM users WHERE username = ? AND status = ?";
try (PreparedStatement stmt = connection.prepareStatement(sql)) {
    stmt.setString(1, username);
    stmt.setString(2, status);
    ResultSet rs = stmt.executeQuery();
}
```
```php
// PHP: PDO with bound placeholders, never direct interpolation into the query string
$stmt = $pdo->prepare('SELECT * FROM users WHERE username = :username AND status = :status');
$stmt->execute(['username' => $username, 'status' => $status]);
$rows = $stmt->fetchAll();
```
```ruby
# Ruby: ActiveRecord's query interface binds values safely; a raw SQL fragment
# still needs an explicit placeholder rather than string interpolation
User.where(username: username, status: status)
# or, for a raw fragment that can't be expressed with the query builder:
User.where('username = ? AND status = ?', username, status)
```
```go
// Go: database/sql binds parameters positionally via driver-specific placeholders
row := db.QueryRow("SELECT * FROM users WHERE username = $1 AND status = $2", username, status)
```
```csharp
// C#: parameters are added to the command explicitly, never concatenated into the SQL string
using var cmd = new SqlCommand("SELECT * FROM Users WHERE Username = @username AND Status = @status", conn);
cmd.Parameters.AddWithValue("@username", username);
cmd.Parameters.AddWithValue("@status", status);
```
- **DO:** Learn and consistently use the parameter-binding syntax specific to whatever language, framework, and database driver a given codebase uses — the placeholder syntax varies (`?`, `:name`, `$1`, `@name`), but the requirement that user input never be spliced into the SQL text string is constant across every one of them.
- **DON'T:** Assume a language or framework is "safe by default" for database access just because it's not one of the languages most commonly associated with SQL injection examples. Every one of the languages above is equally capable of introducing a SQL injection vulnerability the moment a developer reaches for string concatenation instead of the parameter-binding API shown here.

### Insecure Deserialization in Practice

- **DO:** Recognize that "insecure deserialization" is a language-independent risk with language-specific danger zones: Java's native object serialization, PHP's `unserialize()` on untrusted data, Python's `pickle`, and Ruby's `Marshal.load`/unsafe YAML loading have all had well-documented, exploitable gadget-chain vulnerabilities where deserializing crafted attacker input led to arbitrary code execution — not merely malformed data.
```java
// BAD: native Java deserialization of a byte stream that could originate
// from an untrusted source (a cookie, an uploaded file, a network message)
ObjectInputStream in = new ObjectInputStream(untrustedInputStream);
Object obj = in.readObject(); // can trigger gadget-chain code execution

// GOOD: untrusted data is deserialized only via a data-only format (JSON)
// with a schema-bound mapper, never native object deserialization
MyDto dto = objectMapper.readValue(untrustedInputStream, MyDto.class);
```
```php
// BAD: unserialize() on user-controlled input can trigger PHP object
// injection if any autoloaded class implements a "magic" wakeup/destruct method
$data = unserialize($_COOKIE['session_data']);

// GOOD: a data-only format with no object-instantiation capability
$data = json_decode($_COOKIE['session_data'], true);
```
- **DON'T:** Deserialize any object graph from a source outside the process's own trust boundary using a format capable of instantiating arbitrary classes, even if "no dangerous classes are on the classpath/autoload path today" — a gadget chain typically depends on classes present transitively through dependencies, which changes over time as dependencies are added, making "we checked and there's no gadget chain" a claim that can silently become false with an unrelated dependency upgrade.
- **DO:** Where a self-describing, code-executing serialization format is unavoidable for a specific integration (interoperating with a legacy system, for instance), use the format library's documented allowlist/type-restriction feature to restrict deserialization to a fixed, explicit set of expected classes, rather than accepting arbitrary types.

### Markdown and Rich-Text Rendering Pipelines

- **DO:** Treat a markdown-to-HTML (or BBCode-to-HTML, or any similar lightweight-markup-to-HTML) rendering pipeline as producing untrusted HTML that still needs sanitization before being inserted into the page, since markdown renderers commonly support raw embedded HTML passthrough by default, and a markdown comment containing an embedded `<script>` tag or an `onerror` attribute renders exactly as dangerously as if that HTML had been submitted directly.
```javascript
// BAD: the markdown renderer's raw-HTML passthrough is left enabled, and its
// output is inserted directly into the page with no further sanitization —
// a comment containing embedded HTML renders that HTML as live markup
const html = markdownRenderer.render(userComment); // raw HTML passthrough: on by default
element.innerHTML = html;

// GOOD: raw HTML passthrough is disabled in the renderer's configuration,
// and the rendered output is still passed through a dedicated sanitizer
// as a second layer of defense
const renderer = new MarkdownRenderer({ html: false }); // no raw HTML allowed through
const html = DOMPurify.sanitize(renderer.render(userComment));
element.innerHTML = html;
```
- **DON'T:** Assume a markdown library is inherently safe simply because markdown "looks like" plain text formatting. Most markdown implementations are a thin syntax layer over HTML generation, and many default to permitting embedded raw HTML through the exact syntax users would naturally type if they wanted rich formatting — disable that passthrough explicitly, and sanitize the rendered output regardless.
- **DO:** Apply the same link-safety rules to markdown-generated links as to any other user-controlled `href` — reject or neutralize `javascript:` and other script-executing URL schemes in markdown link syntax (`[text](javascript:...)`), since markdown's link syntax is just as capable of carrying a dangerous URL scheme as a raw HTML anchor tag would be.
- **DON'T:** Render user-authored markdown/rich-text server-side and cache the resulting HTML without re-sanitizing it if the sanitization rules or the rendering library are ever updated. A cached, previously-rendered HTML blob generated under an older, more permissive configuration doesn't automatically pick up a later tightened sanitization policy; re-render (or at minimum re-sanitize) cached rich-text output after a security-relevant change to the rendering pipeline.

### ORM Raw-Query Methods by Framework

- **DO:** Know your specific ORM's raw/unsafe query method by name, since the temptation to reach for it under time pressure is the same across ecosystems, but the specific method name (and its parameterized-safe counterpart) differs — Django's `.raw()` and `extra()` versus its query-builder API, Rails' `where("... #{value}")` string interpolation versus `where(column: value)` or `sanitize_sql_array`, Sequelize's `sequelize.query()` versus its model methods, Hibernate's native SQL queries versus HQL/Criteria with bound parameters.
- **DON'T:** Reach for an ORM's raw-query escape hatch and interpolate a variable directly into the string, in any of these frameworks, when the same operation can be expressed through the framework's parameterized query-builder API or, if raw SQL is genuinely required, through that same raw-query method's own parameter-binding argument (nearly all of them accept bound parameters even in "raw" mode).

## Authentication

### WebAuthn and Passkey Relying Party Considerations

- **DO:** Validate the WebAuthn relying party ID and origin strictly against the expected values on both registration and authentication, since WebAuthn's core security property — that a credential is cryptographically bound to the specific origin it was registered for — only holds if the server actually checks the origin the assertion claims to be for, rather than accepting whatever the client reports.
- **DON'T:** Configure the relying party ID more broadly than necessary (registering credentials against a parent domain when only one specific subdomain needs them). A broader relying party ID means a credential works across more origins than strictly needed, widening the set of contexts in which a compromised subdomain could potentially abuse it.
- **DO:** Store the credential's signature counter (or use a WebAuthn authenticator's other supported clone-detection mechanism) and check it increases on each authentication, since a counter that fails to increase (or decreases) can indicate the credential has been cloned — a signal worth alerting on rather than silently ignoring.
- **DON'T:** Treat "the user has a registered passkey" as sufficient for the highest-sensitivity actions without considering whether step-up confirmation (a fresh user-presence or user-verification gesture at the moment of the sensitive action, not merely a still-valid earlier session) is warranted, consistent with the step-up authentication guidance covered above.

Authentication answers "who is making this request." Weaknesses here tend to be catastrophic rather than incremental, because a broken authentication control often hands over every other control built on top of it. The rules below cover how credentials are stored, how sessions are established and maintained, and why hand-built cryptographic or session schemes consistently underperform well-reviewed, widely deployed standards.

### Password Storage

- **DO:** Hash passwords with a purpose-built, slow, memory-hard algorithm — Argon2id is the current general recommendation, with bcrypt and scrypt as well-established alternatives — before ever storing them. These algorithms are deliberately expensive to compute, which makes large-scale offline guessing of a stolen hash database far slower than it would be with a fast general-purpose hash.
- **DON'T:** Store passwords in plaintext, reversibly encrypted, or hashed with a fast general-purpose digest (MD5, SHA-1, or bare unsalted SHA-256/SHA-512) under any circumstance, including "just for a demo" or "just for an internal tool." Fast hashes can be brute-forced at billions of guesses per second on commodity GPU hardware, and plaintext or reversible storage turns any database leak directly into a full credential leak.
```python
# BAD: a fast, general-purpose hash with no per-password salt or work factor
import hashlib
password_hash = hashlib.sha256(password.encode()).hexdigest()

# GOOD: a slow, memory-hard, salted KDF built for password storage
from argon2 import PasswordHasher
ph = PasswordHasher()
password_hash = ph.hash(password)  # salt and parameters are embedded in the output
```
- **DO:** Let the chosen password-hashing library generate and manage the per-password salt automatically rather than implementing salting by hand. Modern password-hashing libraries embed the salt and algorithm parameters in the stored hash string itself, so verification "just works" without a parallel salt column and without a developer re-deriving the scheme incorrectly.
- **DON'T:** Reuse a single global salt (or no salt at all) across all users' password hashes. A shared or missing salt lets an attacker precompute a single rainbow table that cracks every user's password at once instead of needing a separate effort per account.
- **DO:** Tune the work factor (bcrypt's cost parameter, Argon2's time/memory/parallelism parameters) to the slowest value your infrastructure can absorb at expected login volume — typically hundreds of milliseconds per hash on production hardware — and re-tune it upward as hardware gets faster over the life of the system.
- **DON'T:** Truncate passwords before hashing without being aware of an algorithm's input-length quirks (for example, older bcrypt implementations silently ignore bytes beyond 72), and don't impose an unnecessarily low maximum password length "for compatibility" — a 20-character limit needlessly rejects strong passphrases for no real security benefit.
- **DO:** Rehash and upgrade a user's stored password hash transparently at their next successful login if the stored hash was produced with an older algorithm or weaker parameters than the current standard, so the credential store improves over time without forcing a mass password reset.
- **DON'T:** Log the plaintext password anywhere — not in application logs, not in error messages, not in analytics events — even temporarily during debugging. A password that appears in a log line is now stored in every system that ingests logs (log aggregators, error trackers, backups), each with its own retention and access-control posture.
- **DO:** Enforce a minimum password length (current guidance favors length, generally 12+ characters, over complex composition rules) and check new passwords against a breached-password list (such as a "have I been pwned"-style k-anonymity check) rather than mandating arbitrary complexity rules like "must contain a symbol," which push users toward predictable patterns (`Password1!`).
- **DON'T:** Force periodic password rotation (e.g., "change your password every 90 days") for its own sake absent evidence of compromise. Mandatory rotation without cause has been shown to push users toward weaker, more predictable password variations and is no longer recommended by most current guidance; rotate credentials in response to an actual suspected compromise instead.

### Multi-Factor Authentication

- **DO:** Offer multi-factor authentication (MFA) for any account that can access sensitive data or perform sensitive actions, using a phishing-resistant or at least app-based factor — TOTP authenticator apps, push-based approval, or WebAuthn/FIDO2 security keys and passkeys — as the preferred second factor over SMS.
- **DON'T:** Treat SMS-based one-time codes as a strong MFA factor for anything security-sensitive. SMS is vulnerable to SIM-swapping and SS7-level interception; it is better than no second factor, but a phishable/interceptable channel should not be the only MFA option offered for high-value accounts.
- **DO:** Implement WebAuthn/FIDO2 (hardware security keys or platform passkeys) where feasible, since it is cryptographically bound to the origin the credential was registered for and is resistant to phishing in a way that OTP codes — which a user can be tricked into typing into a fake site — are not.
- **DON'T:** Allow an MFA enrollment or account-recovery flow to become a bypass for MFA itself. A "lost your second factor? verify with email" recovery path that requires nothing more than access to an email inbox re-collapses two-factor security back down to single-factor; require a stronger identity check (support-verified identity proof, backup codes generated at enrollment) for recovery.
- **DO:** Generate and let the user download one-time backup/recovery codes at MFA enrollment time, hashed the same way passwords are before storage, so a lost authenticator device does not permanently lock the user out and does not become a support-desk social-engineering vector.
- **DON'T:** Skip rate-limiting or attempt-throttling on the MFA code-entry step. A 6-digit TOTP code has only one million possible values; without throttling, that space is brute-forceable in a realistic amount of time against an endpoint with no attempt limit.

### Session Management

- **DO:** Generate session identifiers using a cryptographically secure random number generator with sufficient entropy (128 bits or more), never a predictable or sequential value (an incrementing integer, a timestamp, a hash of the username). A guessable session ID lets an attacker enumerate or predict another user's active session.
- **DON'T:** Put sensitive data (user ID, role, permissions) directly inside a session identifier or an unsigned/unencrypted cookie value and trust the client not to tamper with it. Any value the client can read or modify must be treated as attacker-controlled input the next time it's read back, unless it is cryptographically signed or the actual data lives server-side keyed by an opaque session ID.
- **DO:** Regenerate the session identifier immediately after any privilege change — most importantly right after a successful login, and again after privilege elevation (e.g., stepping up to an admin context) — to prevent session fixation, where an attacker seeds a victim with a known session ID before authentication and then reuses it after the victim logs in.
```python
# BAD: the pre-login session ID is kept as-is after authentication succeeds
def login(request, user):
    request.session['user_id'] = user.id
    # session identifier from before login is still in use — fixation risk

# GOOD: a fresh session identifier is issued at the moment of authentication
def login(request, user):
    request.session.cycle_key()          # rotate the session ID
    request.session['user_id'] = user.id
```
- **DON'T:** Let sessions live forever with no expiration. Set both an idle timeout (session expires after a period of inactivity) and an absolute timeout (session expires after a fixed maximum duration regardless of activity), with shorter windows for higher-sensitivity applications.
- **DO:** Invalidate the session server-side on logout — not just clear the cookie client-side — so a captured or replayed session token stops working immediately rather than remaining valid until natural expiration.
- **DON'T:** Store sessions purely client-side (in a JWT or signed cookie treated as the sole source of truth) if the application needs the ability to forcibly revoke a session (on logout, password change, or detected compromise) before its natural expiry. A pure client-side token with no server-side revocation list stays valid until it expires no matter what the server later learns.
- **DO:** Maintain a server-side session store (or a short-lived-token-plus-refresh-token pattern with a revocable refresh token) when immediate revocation matters, and provide users a way to view and terminate their own active sessions/devices.
- **DON'T:** Bind session validity only to the token's existence with no additional integrity checks. Consider binding sessions to characteristics like a consistent user-agent or reasonable IP-range continuity as an additional signal (not as a hard requirement, since legitimate IP changes happen), and flag or challenge sessions with sudden, suspicious changes.

### Secure Cookie Configuration

- **DO:** Set `HttpOnly` on any cookie carrying a session identifier or authentication token so client-side JavaScript cannot read it via `document.cookie`, closing off session theft as a payoff for a successful XSS injection.
- **DO:** Set `Secure` on every authentication-related cookie so the browser only ever transmits it over an HTTPS connection, never in plaintext over HTTP.
- **DO:** Set `SameSite=Lax` or `SameSite=Strict` on session cookies (choosing `Strict` where the login flow allows it) to prevent the cookie from being automatically attached to cross-site requests, which meaningfully reduces CSRF exposure as a defense-in-depth layer alongside a dedicated CSRF token.
```
# BAD: session cookie with no protective flags set
Set-Cookie: session=abc123; Path=/

# GOOD: session cookie hardened against theft and cross-site misuse
Set-Cookie: session=abc123; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=3600
```
- **DON'T:** Set `SameSite=None` on a session cookie unless the application genuinely requires cross-site delivery (e.g., an embedded widget), and if it is required, pair it with `Secure` (mandatory alongside `SameSite=None` in modern browsers) and additional CSRF defenses, since `SameSite=None` opts the cookie back into being sent on cross-site requests.
- **DO:** Scope cookies with the narrowest sensible `Domain` and `Path` attributes rather than defaulting to the broadest possible scope, so a cookie meant for one subdomain or application path isn't unnecessarily exposed to every other subdomain sharing the parent domain.
- **DON'T:** Set an unnecessarily long `Max-Age`/`Expires` on an authentication cookie "for convenience." A session or refresh-token cookie that never expires turns a single stolen cookie into indefinite account access; balance session lifetime against user convenience deliberately rather than defaulting to the longest option.

### Tokens: JWTs and API Keys

- **DO:** Verify a JWT's signature using a fixed, server-configured algorithm and key — never trust an `alg` value taken from the token itself, and explicitly reject the `none` algorithm. Historic JWT libraries that read the algorithm from the untrusted token header enabled attackers to switch a server from asymmetric verification to `none` or to reuse a public key as an HMAC secret.
```javascript
// BAD: verification algorithm is derived from the token's own (attacker-controlled) header
jwt.verify(token, secretOrPublicKey); // algorithm not pinned

// GOOD: the expected algorithm is pinned server-side and 'none' is rejected implicitly
jwt.verify(token, publicKey, { algorithms: ['RS256'] });
```
- **DON'T:** Put sensitive data (passwords, full PII, secrets) inside a JWT payload on the assumption that signing makes it confidential. A JWT's payload is base64-encoded, not encrypted — anyone who can read the token can read its claims; use short-lived, minimal claims and keep sensitive lookups server-side keyed by a subject ID.
- **DO:** Keep JWT (or any bearer-token) lifetimes short, and use a separate, revocable refresh token to obtain new short-lived access tokens, so a leaked access token has a small window of usefulness and a compromised refresh token can be revoked server-side.
- **DON'T:** Issue long-lived, non-revocable JWTs as the sole authentication mechanism for a system that needs to be able to kill a session on logout, password change, or detected compromise. A stateless long-lived token with no revocation list remains valid for its full lifetime no matter what the server later learns.
- **DO:** Treat API keys the same as passwords for storage purposes — hash them before persisting server-side (or store only a lookup prefix plus a hash of the secret portion), show the full key to the user exactly once at creation time, and support per-key scoping, expiration, and independent revocation.
- **DON'T:** Issue a single, unscoped, non-expiring API key per account/organization that grants full access to everything. Scope keys to the minimum set of operations they need, and let users see, name, rotate, and revoke individual keys independently so a leaked key can be killed without breaking every other integration.

### Avoiding Custom Cryptography and Auth Schemes

- **DO:** Use a maintained, widely reviewed authentication library or framework module (the ecosystem's standard auth library, a vetted OAuth/OIDC client, a well-known password-hashing library) instead of hand-rolling login, session, token, or password-reset logic from primitives.
- **DON'T:** Design a custom challenge-response scheme, a custom token format, or a custom "encryption" routine for authentication "because the built-in option seems like overkill." Authentication protocols have decades of accumulated, hard-won lessons about subtle failure modes (timing attacks, replay attacks, downgrade attacks) that a from-scratch design is very unlikely to have accounted for, and this class of mistake is one of the most common sources of critical vulnerabilities in otherwise well-built applications.
- **DO:** Use constant-time comparison functions for any secret comparison (session tokens, API keys, HMAC signatures, password-reset tokens) rather than the language's default equality operator, which typically short-circuits on the first differing byte and can leak timing information about how much of a guess was correct.
```python
# BAD: standard string comparison short-circuits, leaking timing information
if provided_token == stored_token:
    ...

# GOOD: constant-time comparison, independent of where the strings first differ
import hmac
if hmac.compare_digest(provided_token, stored_token):
    ...
```
- **DON'T:** Implement a password-reset flow with a predictable or short-lived-but-guessable token (a sequential ID, a short numeric code with no rate limit, a token derived deterministically from the username and timestamp). Use a long, cryptographically random, single-use token with a short expiration window, invalidate it immediately after use, and invalidate any other outstanding reset tokens for that account once one is used.
- **DO:** Rate-limit and add friction (CAPTCHA, exponential backoff, temporary account lockout with a sane unlock path) to login endpoints, password-reset requests, and MFA code entry, to blunt credential-stuffing and brute-force attempts without permanently locking out legitimate users on a single mistyped password.
- **DON'T:** Reveal through response differences whether a given username/email exists in the system during login or password-reset flows (e.g., "invalid password" vs. "no such user," or a visibly different response time). Use a uniform response and a uniform response time for both cases, which prevents user enumeration as a reconnaissance step for credential-stuffing attacks.
- **DO:** Consider passkeys/WebAuthn passwordless authentication as the direction to move toward for new systems where feasible, since it removes phishable shared secrets from the authentication flow entirely rather than merely adding a second factor on top of one.
- **DON'T:** Conflate "authenticated" with "trustworthy." A successfully authenticated request still needs full input validation and authorization checks — authentication only establishes identity, and treating a logged-in user's input as inherently safe is how authenticated-user-only injection and IDOR vulnerabilities happen.
- **DO:** Bind single-sign-on and OAuth/OIDC "state" and "nonce" parameters correctly and validate them on callback, since skipping state validation opens the login flow to CSRF (an attacker completing their own OAuth flow and tricking a victim's browser into finishing it under the victim's session) and skipping nonce validation opens ID tokens to replay.
- **DON'T:** Implement a "remember me" feature by extending the normal session cookie's lifetime indefinitely. Use a separate, distinct, revocable long-lived token (ideally a rotating token scheme where each use issues a fresh token and invalidates the old one, so a stolen unused token is detected on next legitimate use) rather than simply widening the blast radius of the primary session cookie.
- **DO:** Log authentication events (successful logins, failed attempts, password changes, MFA enrollment/removal, session revocations) to a security-relevant audit trail distinct from general application logs, since these events are exactly what an incident responder needs first when investigating a suspected account compromise.

### OAuth 2.0 and OpenID Connect Integration

- **DO:** Use the Authorization Code flow with PKCE (Proof Key for Code Exchange) for any client that cannot fully protect a client secret — single-page apps, mobile apps, and increasingly recommended even for confidential server-side clients — since PKCE prevents an intercepted authorization code from being redeemed by anyone other than the client that initiated the request.
- **DON'T:** Use the deprecated Implicit flow (tokens returned directly in a URL fragment) for new integrations, and don't skip PKCE on the reasoning that "the client secret is confidential" for a public client type where no secret can actually be kept confidential (a JavaScript app, a mobile app whose binary can be decompiled).
- **DO:** Validate the `redirect_uri` on the authorization server side against an exact, pre-registered allowlist — not a prefix match, not a wildcard pattern — since a loosely validated redirect URI is one of the most common real-world OAuth misconfigurations and can let an attacker redirect the authorization code or token to an attacker-controlled endpoint.
- **DON'T:** Trust an ID token's claims without verifying its signature against the issuer's published keys, its `aud` (audience) claim matches your client ID, its `iss` (issuer) matches the expected identity provider, and it hasn't expired — accepting an ID token's claims (like the user's email or subject identifier) without full verification is equivalent to accepting an unverified assertion of identity from anyone who can craft a similarly shaped token.
- **DO:** Keep access tokens short-lived and use refresh tokens (stored securely, ideally with rotation so each use invalidates the previous refresh token) to obtain new access tokens, consistent with the general token-lifetime guidance above.

### Credential Stuffing and Compromised-Password Defenses

- **DO:** Check new and existing passwords against a known-breached-password database (using a privacy-preserving method such as k-anonymity range queries, which avoids ever transmitting the actual password or its full hash to a third party) and prompt affected users to change their password.
- **DON'T:** Assume strong password-complexity rules alone prevent credential stuffing. Credential stuffing uses real username/password pairs leaked from breaches of *other* services — the password can be perfectly "strong" by complexity rules and still be compromised because the user reused it elsewhere; complexity rules and breach-checking address different risks and both matter.
- **DO:** Detect and respond to credential-stuffing patterns specifically — a high volume of login attempts across many distinct usernames from a small set of sources, or the reverse (many sources attempting the same small set of passwords) — with escalating friction (CAPTCHA, temporary IP-level throttling) distinct from a simple per-account lockout, since credential stuffing is designed to stay under a per-account attempt threshold by spreading guesses across many accounts.
- **DON'T:** Notify a user only after a successful account takeover. Where feasible, proactively notify users when their credentials are found in a known breach dataset (even before any attack against your system is observed) and prompt a password reset, closing the window before it's exploited.

### Device Trust and Step-Up Authentication

- **DO:** Recognize and require additional verification (step-up authentication — a fresh MFA challenge, an email confirmation) for high-risk actions within an already-authenticated session, such as changing the account's email/password, adding a new payment method, or performing a large financial transaction, rather than treating "already logged in" as sufficient authorization for every action regardless of sensitivity.
- **DON'T:** Let a long-lived, low-friction session (kept alive for weeks via a "remember me" token) carry the same implicit trust for a highly sensitive action as it does for routine browsing. Session age and the sensitivity of the action being performed should both factor into whether re-authentication is required.
- **DO:** Consider device recognition/fingerprinting as a *signal* that can reduce friction for a returning, previously-verified device — not as a replacement for authentication itself, and not as a security boundary on its own, since device signals can be spoofed and should only ever supplement, not substitute for, a real credential check.

### Account Recovery and Change Flows

- **DO:** Notify a user through an out-of-band channel (email, and ideally a second channel like SMS/push for high-value accounts) whenever a security-relevant change happens on their account — password change, email change, MFA reset, a new device/session added — so the legitimate owner has a chance to notice and respond to an unauthorized change quickly.
- **DON'T:** Let an email-change flow immediately and silently redirect all future password-reset and account-recovery communication to the new address without first confirming the new address belongs to the account owner. An attacker who has gained temporary access to an account (through a leaked session, for instance) can otherwise permanently lock the real owner out by changing the recovery email before the legitimate owner notices.
- **DO:** Require re-authentication (current password, or a fresh MFA challenge) before allowing a change to the account's own authentication factors — password, MFA methods, recovery email — even within an already-authenticated session, since a hijacked but not-yet-detected session shouldn't be enough on its own to permanently take over the account's recovery path.
- **DON'T:** Allow unlimited concurrent active sessions with no visibility or control for the user. Provide a way for users to see their active sessions/devices and revoke any of them, and consider automatically invalidating other sessions when a password is changed, since a changed password is a common signal that the account owner wants existing access (including any an attacker may have) cut off.

### Federated Identity and Single Logout

- **DO:** Propagate logout across all applications sharing a federated identity/SSO session where the identity provider supports single logout, so that ending a session at the identity provider actually ends access at every connected application rather than leaving still-valid local sessions active elsewhere.
- **DON'T:** Assume that revoking access at the identity provider (disabling a user's SSO account) immediately invalidates every already-issued token/session at every connected downstream application. Depending on the federation protocol and each application's session design, an already-issued token can remain valid until its own natural expiration unless the application also checks for revocation; account offboarding processes need to consider this gap.

### Password Hashing Across Languages

As with parameterized queries, the API shape for password hashing differs by language, but the requirement — a slow, salted, memory-hard algorithm, with the library managing salt generation — is the same everywhere. These examples show the same safe pattern across common ecosystems.

```java
// Java: Spring Security's BCryptPasswordEncoder (or a dedicated Argon2 encoder)
PasswordEncoder encoder = new BCryptPasswordEncoder(12); // work factor 12
String hash = encoder.encode(rawPassword);
boolean matches = encoder.matches(rawPassword, hash);
```
```php
// PHP: the built-in password_hash()/password_verify() pair uses bcrypt (or
// Argon2id when explicitly requested) and manages the salt automatically
$hash = password_hash($rawPassword, PASSWORD_ARGON2ID);
$isValid = password_verify($rawPassword, $hash);
```
```ruby
# Ruby: has_secure_password (bcrypt-backed) on an ActiveRecord model
class User < ApplicationRecord
  has_secure_password  # provides password= and authenticate() using bcrypt
end
user.authenticate(raw_password) # returns the user or false
```
```go
// Go: golang.org/x/crypto/bcrypt
hash, err := bcrypt.GenerateFromPassword([]byte(rawPassword), bcrypt.DefaultCost)
err = bcrypt.CompareHashAndPassword(hash, []byte(rawPassword)) // nil on match
```
```csharp
// C#: ASP.NET Core Identity's PasswordHasher (PBKDF2 by default, configurable)
var hasher = new PasswordHasher<ApplicationUser>();
string hash = hasher.HashPassword(user, rawPassword);
PasswordVerificationResult result = hasher.VerifyHashedPassword(user, hash, rawPassword);
```
- **DO:** Use whichever password-hashing facility is the current recommended default for the specific framework in use (many mainstream web frameworks now ship one built in) rather than reaching for a lower-level, general-purpose hashing library and assembling the salting/iteration logic by hand.
- **DON'T:** Assume a framework's built-in "hash" or "encode" helper is automatically the *password*-appropriate one — some frameworks expose multiple hashing utilities for different purposes (fast general-purpose hashing for cache keys or ETags, versus slow password-specific hashing), and picking the wrong one because the name sounds similar reintroduces the fast-hash password-storage mistake this section is meant to prevent.

### Secure Credential Storage on Mobile and Desktop Clients

- **DO:** Store authentication tokens and any other sensitive credential on mobile and desktop clients using the platform's dedicated secure storage facility (the platform keychain/keystore, an OS-level credential manager) rather than in plain preference files, plain local storage, or an app-level SQLite database with no additional encryption.
- **DON'T:** Store a session token, refresh token, or password in a mobile app's shared preferences / plain local storage without encryption. On a rooted/jailbroken device, or through a backup-extraction attack, plaintext local storage is directly readable, unlike the platform's dedicated secure-storage facility, which is backed by hardware-level protection on most modern devices.
- **DO:** Set the platform's data-protection/backup-exclusion flags on any locally cached sensitive data so it isn't swept into an unencrypted device backup that could later be restored to, or inspected on, a different device.
- **DON'T:** Log sensitive tokens or credentials to a mobile OS's system-wide log (which can be readable by other apps or connected debugging tools on some device/OS configurations) — apply the same "never log secrets" discipline covered under Secrets Management to client-side/mobile logging as well as server-side logging.

### Implementing Rate Limiting

- **DO:** Choose a rate-limiting algorithm appropriate to the traffic pattern being protected against — a fixed or sliding window counter for simple per-account/per-IP throttling, a token-bucket algorithm when short bursts of legitimate traffic should be tolerated but sustained abuse should not — and back the counter with a shared, fast store (an in-memory cache service) reachable by every application instance, not a per-process in-memory counter that a load-balanced deployment would let an attacker trivially bypass by hitting different instances.
```python
# BAD: an in-process counter only limits requests within a single server
# instance — behind a load balancer with multiple instances, an attacker's
# requests are simply spread across instances to bypass the limit entirely
request_counts = {}  # lives in this process's memory only

def is_rate_limited(user_id):
    request_counts[user_id] = request_counts.get(user_id, 0) + 1
    return request_counts[user_id] > 10

# GOOD: a shared, atomic counter in a store every instance reads from
def is_rate_limited(user_id):
    key = f"ratelimit:{user_id}"
    count = redis_client.incr(key)
    if count == 1:
        redis_client.expire(key, 60)  # window resets after 60 seconds
    return count > 10
```
- **DON'T:** Implement rate limiting with a check-then-increment pattern against a shared store that isn't atomic. As with the check-then-act race condition covered under Common Web Vulnerabilities, a non-atomic "read the count, compare, then increment" sequence allows concurrent requests to race past the intended limit; use the store's atomic increment operation (as shown above) rather than separate read and write calls.
- **DO:** Return a clear rate-limit response (HTTP 429, with a `Retry-After` header where applicable) so legitimate clients can back off appropriately, distinguishing rate-limiting from a generic error in both the response code and any client-facing messaging.

### Username Enumeration Through Timing and Side Channels

- **DO:** Make the login, registration, and password-reset code paths take a comparable amount of time whether or not the submitted identifier corresponds to a real account — for example, by always performing a password-hash comparison against some hash (a dummy one when no account exists) rather than short-circuiting immediately when the lookup finds no matching user, so the response time doesn't itself reveal which case occurred.
```python
# BAD: the response returns immediately when no account exists, making the
# "no such user" case measurably faster than the "wrong password" case —
# an attacker can distinguish the two purely by timing, without ever
# seeing a different error message
def login(username, password):
    user = find_user(username)
    if user is None:
        return "invalid credentials"          # fast path — no hash computed
    if not verify_password(password, user.password_hash):
        return "invalid credentials"          # slow path — hash computed
    return authenticated_session(user)

# GOOD: a hash comparison is always performed, keeping the timing profile
# consistent regardless of whether the account exists
def login(username, password):
    user = find_user(username)
    hash_to_check = user.password_hash if user else DUMMY_HASH
    valid = verify_password(password, hash_to_check)
    if user is None or not valid:
        return "invalid credentials"
    return authenticated_session(user)
```
- **DON'T:** Assume that using an identical error *message* for "no such user" and "wrong password" (as covered earlier under Authentication) is sufficient on its own. If the underlying code path still takes a measurably different amount of time for each case, the timing itself is a side channel that reveals the same information the identical message was meant to hide.

### Password Managers and Autofill Compatibility

- **DO:** Use standard, semantic `<input>` types and `autocomplete` attribute values (`autocomplete="current-password"`, `autocomplete="new-password"`, `autocomplete="username"`, `autocomplete="one-time-code"`) on authentication forms, which lets password managers correctly detect, fill, and — importantly — *generate and save* strong, unique passwords for users, directly supporting the "avoid password reuse" outcome this entire section is trying to encourage.
- **DON'T:** Set `autocomplete="off"` on password fields to "improve security." This blocks password managers from offering to save or fill a strong, unique generated password, which measurably pushes users toward the exact behavior — reusing a weaker, memorable password across sites — that makes credential-stuffing attacks effective; it provides no compensating security benefit, since it doesn't meaningfully stop a determined attacker or malware from reading the field.
- **DO:** Allow pasting into password fields. Blocking paste in a password field (sometimes done with the mistaken belief that it improves security) actively interferes with password managers and makes it harder for users to use long, randomly generated passwords, again pushing toward weaker, typeable, reused passwords instead.
- **DON'T:** Design a custom, non-standard login form control (a segmented input requiring one keystroke per box, a virtual on-screen keyboard for password entry) that breaks password-manager autofill "for security," unless there's a specific, well-understood threat (such as keylogging malware) that the specific design genuinely mitigates and that mitigation has been weighed against the cost of degrading password-manager compatibility for the vast majority of users.

### Passwordless Magic-Link Authentication

- **DO:** Generate magic-link tokens with the same rigor as password-reset tokens covered above — long, cryptographically random, single-use, and short-lived (minutes, not hours or days) — since a magic link is functionally a bearer credential for full account access, sent over a channel (email) that itself isn't always fully secure end-to-end.
- **DON'T:** Let a magic link remain valid after it's been used once, or allow the same link to authenticate multiple sessions. Invalidate the token the moment it's successfully used, exactly as with a password-reset token, so a link that leaks after use (forwarded, cached, or found in an inbox someone else later gains access to) can't grant a second login.
- **DO:** Bind a magic link to the context it was requested in where practical — for example, requiring the link to be opened from the same browser/device that requested it (via a short-lived local marker), or at minimum warning the user if the link is being used from a very different context than the request — since email accounts are themselves a common secondary compromise target, and an attacker with inbox access can otherwise complete the login flow just as easily as the legitimate user.
```python
# GOOD: a magic-link token is single-use, short-lived, and invalidated
# immediately on successful login
def request_magic_link(email):
    token = secrets.token_urlsafe(32)
    store_token(token, email, expires_at=now() + timedelta(minutes=15), used=False)
    send_email(email, magic_link_url(token))

def login_with_magic_link(token):
    record = get_token(token)
    if not record or record.used or record.expires_at < now():
        raise InvalidTokenError()
    mark_token_used(record)          # single-use: invalidated immediately
    return authenticated_session(record.email)
```
- **DON'T:** Treat a successful magic-link login as equivalent to a fully verified, MFA-protected login for a highly sensitive account, without considering whether email-account compromise is a realistic threat in the application's specific risk profile. For high-value accounts, pair a magic-link (or any single-factor passwordless) flow with a genuine second factor rather than relying on "possession of the email inbox" as the sole authentication signal.

## Authorization

Authorization answers "what is this authenticated identity allowed to do," and it must be re-checked on every single request — it is not something a login screen establishes once and the rest of the system inherits. The rules below focus on keeping authorization logic centralized, consistent, and enforced at every layer the request passes through, not only in the UI.

### Principle of Least Privilege

- **DO:** Grant each user, service account, and API credential only the specific permissions required for its actual job, and default new roles/permissions to "deny" rather than "allow." A least-privilege model limits how much damage a single compromised account or a single logic bug in one feature can cause.
- **DON'T:** Default a new user, role, or service account to broad or administrative access "to avoid permission errors during development" and plan to lock it down later. Overly broad defaults reliably ship to production because tightening permissions after the fact requires someone to notice and prioritize it, while a permission error during development is immediately visible and easy to fix forward.
- **DO:** Apply least privilege to machine-to-machine credentials just as strictly as to human accounts — a microservice that only reads from one table should hold a database credential scoped to read-only access on that table, not the same broad credential shared across every service in the system.
- **DON'T:** Let a "temporary" elevated-privilege grant (an admin flag flipped for debugging, a broadened IAM policy for a one-off migration) become permanent because nobody scheduled its removal. Time-box privilege escalations explicitly and track them so they expire or get reviewed rather than silently persisting.

### Insecure Direct Object References (IDOR)

- **DO:** Verify, on the server, that the currently authenticated user is actually authorized to access the specific resource identified by an ID in the URL, body, or query string — every single time that resource is accessed, not only when the ID was first handed to the client. IDOR vulnerabilities happen when an application checks that a user is logged in but forgets to check that the requested object actually belongs to (or is shared with) them.
```javascript
// BAD: any authenticated user can fetch any invoice by guessing/incrementing the ID
app.get('/api/invoices/:id', requireAuth, async (req, res) => {
  const invoice = await db.invoices.findById(req.params.id);
  res.json(invoice);
});

// GOOD: ownership is verified as part of the same query, not as an afterthought
app.get('/api/invoices/:id', requireAuth, async (req, res) => {
  const invoice = await db.invoices.findOne({
    id: req.params.id,
    ownerId: req.user.id,
  });
  if (!invoice) return res.status(404).end(); // don't distinguish "not found" from "not yours"
  res.json(invoice);
});
```
- **DON'T:** Treat an unguessable-looking identifier (a UUID instead of a sequential integer) as a substitute for an authorization check. Obscurity in the ID format raises the bar for casual guessing but does nothing if the ID leaks through a referrer header, a shared link, a log file, or another user's response — the authorization check is what actually matters, and the UUID is not one.
- **DO:** Return the same response (typically a generic 404) for "resource does not exist" and "resource exists but you're not authorized to see it," so the authorization check itself doesn't leak information about which IDs are valid to a user probing for other people's data.
- **DON'T:** Perform the ownership/authorization check only on the "read" endpoint for a resource while forgetting it on the "update" or "delete" endpoint for the same resource. IDOR audits should walk every verb (GET/POST/PUT/PATCH/DELETE) for every resource, since it's extremely common for a team to secure the endpoint they thought about first and miss a sibling endpoint added later.
- **DO:** Extend IDOR checks to nested and related resources, not just the top-level object in the URL. A `GET /projects/:projectId/tasks/:taskId` endpoint that checks the caller owns `projectId` but not that `taskId` actually belongs to that same project can let a user read another project's task by ID even while "correctly" scoped at the project level.
- **DON'T:** Assume authorization checks written for the REST API automatically apply to a GraphQL, gRPC, WebSocket, or batch/bulk-export endpoint added later that exposes the same underlying data through a different interface. Each entry point into the data needs its own explicit authorization enforcement; a bulk CSV-export feature is a classic place where per-record authorization gets silently dropped in favor of "just fetch everything and filter... eventually."

### Consistent Access-Control Models (RBAC / ABAC)

- **DO:** Centralize authorization logic in one policy layer (a guard/middleware, a policy object, an authorization library) that every route and resolver calls through, rather than letting each handler independently reimplement its own permission check. A single source of truth for "can this user do this thing to this resource" is auditable in one place and consistent by construction.
- **DON'T:** Scatter ad hoc `if (user.role === 'admin')`-style checks across dozens of handlers, especially when combined with copy-paste. Inconsistent, hand-duplicated authorization logic is exactly how one code path ends up with a subtly different (or entirely missing) check than its siblings.
- **DO:** Choose role-based access control (RBAC) for systems where permissions map cleanly onto a small number of job-function roles, and attribute-based or policy-based access control (ABAC) when authorization depends on finer-grained context (resource ownership, organization membership, time of day, resource sensitivity level) that a fixed role list can't express cleanly — and be consistent about which model governs which part of the system rather than mixing informal ad hoc rules with a formal role system.
- **DON'T:** Let role names imply guarantees the code doesn't actually enforce (a `viewer` role that can, through an overlooked endpoint, still trigger a write). Periodically audit that every permission a role is documented to have (and only those) is what the enforcement layer actually grants, ideally with automated tests that assert access for each role/resource combination.
- **DO:** Model permissions around actions and resources explicitly (`can(user, 'edit', document)`) rather than encoding authorization as scattered boolean flags on the user object (`isEditor`, `isPremium`, `canDeleteStuff`), which become unmaintainable and inconsistent as the number of flags grows.
```python
# BAD: ad hoc boolean flags checked inconsistently across the codebase
if user.is_admin or user.is_owner or user.is_editor:
    document.delete()

# GOOD: a single policy function is the one place this decision is made
if policy.can(user, "delete", document):
    document.delete()
```
- **DON'T:** Implement multi-tenancy authorization by trusting a `tenant_id`/`org_id` value the client sends in the request body or query string. Derive the tenant context from the authenticated session/token server-side, and use it to scope every query — never let a client-supplied tenant identifier determine which tenant's data a query returns.
- **DO:** Enforce tenant isolation at the data-access layer itself (a repository method that always injects the current tenant's scope into every query, or database-level row-level security) so that a missing `WHERE tenant_id = ?` in one forgotten query can't leak cross-tenant data. A defense implemented once at the data layer protects every caller; a convention that "every query must remember to filter by tenant" eventually gets forgotten somewhere.
- **DON'T:** Trust client-supplied role or permission claims embedded in a request body, hidden form field, or an unsigned cookie. Any authorization-relevant claim the client can modify must be re-derived or re-validated server-side from a trusted source (the session, a signed token, a database lookup) rather than accepted at face value.

### Enforcing Authorization at Every Layer

- **DO:** Enforce every authorization rule on the server/API layer as the ultimate source of truth, treating any client-side check (hiding a button, disabling a menu item, a UI route guard) purely as a user-experience convenience. A determined user can call the API directly, bypassing the UI entirely, so a security control that exists only in client-side code is not a security control.
```javascript
// BAD: the delete button is hidden for non-admins, but the API has no server check
// client: {!user.isAdmin ? null : <DeleteButton />}
app.delete('/api/posts/:id', requireAuth, async (req, res) => {
  await db.posts.delete(req.params.id); // no role check here at all
});

// GOOD: the API independently enforces the rule regardless of what the UI shows
app.delete('/api/posts/:id', requireAuth, requireRole('admin'), async (req, res) => {
  await db.posts.delete(req.params.id);
});
```
- **DON'T:** Ship a feature flag, an admin-only endpoint, or a "hidden" internal API path that relies on the client simply not knowing the URL exists ("security through obscurity") instead of an actual authorization check. Undocumented endpoints are routinely discovered through bundled JavaScript, API response inspection, or automated crawling.
- **DO:** Re-check authorization at each service boundary in a multi-service architecture rather than trusting that "the gateway already checked this." If service B can be reached directly (by another internal service, a misconfigured network path, or a future integration), it needs its own authorization enforcement rather than assuming every caller has already passed through the gateway's check.
- **DON'T:** Assume an internal network boundary (VPN-only, same-VPC, "it's behind the firewall") is sufficient authorization on its own for a service that handles sensitive data. Internal network position answers "can this traffic reach the service," not "should this specific caller be allowed to perform this specific action" — apply real authorization checks for internal services too, especially as architectures move toward zero-trust assumptions where network location no longer implies trust.
- **DO:** Verify authorization for background jobs, scheduled tasks, and asynchronous message handlers the same way as for synchronous request handlers — a queued job that processes a user-supplied payload still needs to confirm the actor is allowed to perform that action, since the synchronous request that enqueued it may have passed a check that is no longer valid by the time the job actually runs (a role revoked in between, for instance).
- **DON'T:** Forget authorization checks on file/object storage access paths that bypass the application server — a signed URL, a public S3-style bucket, or a CDN-fronted asset. If a file is meant to be private to a specific user or account, either keep the bucket/object non-public and issue short-lived, scoped signed URLs on demand, or apply access-control rules at the storage layer itself rather than relying on "the URL is hard to guess."
- **DO:** Write authorization-focused tests (not just feature tests with a happy-path admin user) that assert a *non*-privileged user is correctly denied — for every sensitive endpoint, confirm both that the right people can act and that the wrong people cannot, since it's easy for a test suite to only ever exercise the allowed path.
- **DON'T:** Compare roles, permission levels, or other authorization-relevant strings with loose equality, implicit type coercion, or case-insensitive matching that wasn't deliberately designed in. A comparison like `role == "admin"` in a language with loose typing/coercion, or a case-insensitive match against a user-controlled string, can be tricked into an unintended `true` result by a crafted value; use strict, type-safe equality and validate the value against a known enum before comparing it at all.
- **DO:** Design the "deny by default, allow by exception" shape into every authorization check — a function that fails to determine a clear answer should return "denied," not "allowed." A policy check that throws an unexpected error should propagate as a denial, not silently fall through to granting access.
```python
# BAD: an unexpected exception during the permission check falls through to allow
def can_access(user, resource):
    try:
        return resource.owner_id == user.id or user.role == "admin"
    except AttributeError:
        return True  # fails open — dangerous default

# GOOD: any failure to determine access explicitly denies it
def can_access(user, resource):
    try:
        return resource.owner_id == user.id or user.role == "admin"
    except AttributeError:
        return False  # fails closed
```
- **DON'T:** Grant broad access "temporarily" through a support/admin impersonation feature without an audit trail. If customer-support staff need to view an account as that user for troubleshooting, log every impersonation session (who, whose account, when, what was viewed/changed) and time-box the impersonation session automatically.

### Time-of-Check to Time-of-Use (TOCTOU) in Authorization

- **DO:** Perform the authorization check and the action it guards as close together as possible — ideally within the same database transaction or atomic operation — so that a resource's ownership or state can't change in the gap between "checked" and "acted upon."
- **DON'T:** Check authorization once at the start of a long-running or multi-step operation and assume it still holds by the time a later step executes. A permission that was valid when a background job was enqueued may have been revoked by the time the job actually runs; re-check authorization at the point of action, not only at the point of request.
- **DO:** Treat any workflow where a resource can change owner, state, or sensitivity between an initial check and a later dependent action (a document that gets reassigned, a role that gets revoked mid-session) as needing a re-check immediately before the sensitive action executes, not only relying on a check performed earlier in the request lifecycle.

### Authorization Across Services and Gateways

- **DO:** Propagate the original caller's identity and permissions context through a call chain of internal services (via a signed/verified token, not merely a trusted-network assumption) so that a downstream service can make its own correct authorization decision rather than blindly trusting whatever the upstream service forwarded.
- **DON'T:** Let a downstream service treat "the request came from our API gateway" as proof the caller is authorized for the specific action being requested. The gateway authenticating a caller's identity and a downstream service authorizing that identity's specific action are two different checks; collapsing them into one (assuming the gateway already did all necessary authorization) is a common source of the "confused deputy" problem, where a service with broad legitimate access is tricked into performing an action on behalf of a caller who shouldn't be allowed to trigger it.
- **DO:** Watch specifically for the confused-deputy pattern in any service that holds broader privileges than its individual callers (a file-processing service with access to all users' files, acting on behalf of whichever user's request triggered it) — that service must independently verify the calling user is authorized for the specific file/resource in the request, not just that the calling service itself is a trusted internal caller.
- **DON'T:** Use a single shared, static internal API key/secret for all service-to-service authorization decisions. Without per-caller identity carried through the chain, a downstream service has no way to distinguish "user A's legitimate request routed through service B" from "user C's request for user A's data routed through the same trusted service B."

### Default Permissions on New Resources and Features

- **DO:** Default newly created resources (a new document, a new project, a new API integration) to the most restrictive reasonable visibility — private to the creator, or scoped to an explicit set of collaborators — rather than defaulting to organization-wide or public visibility "to reduce friction."
- **DON'T:** Ship a new feature with a permissions model that hasn't been explicitly designed, on the assumption that "we'll figure out fine-grained permissions later." A feature that ships without an access-control model tends to default to either "everyone can see everything" (a data-exposure risk) or "only admins can see anything" (breaking the feature) — neither of which is a deliberate decision, and retrofitting a permissions model after a feature has real user data in it is markedly harder than designing it in from the start.
- **DO:** Re-audit default permissions whenever a resource type's sensitivity changes — a "notes" feature that started as scratch space but has grown to store sensitive customer information needs its default sharing/visibility settings revisited, not left as originally configured for a lower-sensitivity use case.

### Privilege Escalation via Object Relationships

- **DO:** Check authorization along the *entire* chain of relationships a request traverses, not just the final object referenced — a request to "add a comment to task T in project P" needs to confirm the user has access to P, not only that task T exists, since checking only the leaf object can let a user interact with a resource that's nested under a parent they were never granted access to.
- **DON'T:** Allow a user to change a resource's parent/owner reference to one they don't have access to as a way of "moving" it, without checking authorization on *both* the source and destination parent. A "move this document to a different folder" action needs to verify the user can write to the source folder and the destination folder — checking only one side lets a user smuggle a resource into or out of a scope they shouldn't reach.
- **DO:** Watch specifically for escalation through self-service role/permission assignment features — a "manage team members" feature that lets any member (not just admins) invite a new member and assign that new member's role can become a privilege-escalation path if it doesn't independently enforce that the *inviter's own* role permits granting the requested role, and in particular that a non-admin can't grant admin.

### Service Accounts and Machine-to-Machine Authorization

- **DO:** Scope each service account or machine credential's permissions to the exact set of operations that specific integration needs, following the same least-privilege principle applied to human accounts, and give each distinct integration its own credential rather than sharing one service account across multiple unrelated integrations.
- **DON'T:** Let a service account inherit or retain broader access than its current integrations actually use "in case it's needed later." An unused-but-granted permission on a service account is pure downside — it does nothing for the integrations actually running, and it widens what a leaked service-account credential could do.
- **DO:** Apply the same audit-logging and periodic access-review discipline to service accounts as to human accounts — service account credentials tend to be longer-lived, less actively monitored by a human, and less frequently rotated than user credentials, which makes them a disproportionately attractive target once an environment is compromised.

### Administrative Interface Security

- **DO:** Apply defense-in-depth to administrative interfaces specifically — beyond the normal role-based authorization check, consider network-level restriction (VPN-only or IP-allowlisted access), mandatory MFA, and more aggressive session timeouts, since an admin interface compromise typically has outsized impact compared to a compromise of an ordinary user account.
- **DON'T:** Expose an administrative panel on the same public hostname and path structure as the main application with authorization as the only distinguishing control. While authorization should indeed be the authoritative control (never rely on obscurity alone, as covered above), pairing it with additional layers specifically for admin surfaces is a reasonable, proportionate response to how much more damaging an admin-account compromise is.
- **DO:** Log every administrative action with full detail (who, what changed, before/after values where applicable) to the audit trail described under Security Logging, since administrative actions are exactly the category of event an incident investigation most needs to reconstruct.

### Cloud IAM and Infrastructure Permissions

- **DO:** Write cloud IAM policies (AWS IAM, GCP IAM, Azure RBAC) that name specific resources and specific actions rather than using wildcard resources (`*`) or wildcard actions, mirroring the same least-privilege principle applied to application-level roles and permissions — a policy scoped to "this service can read/write to this specific storage bucket" is fundamentally different in risk from "this service has full access to storage."
```json
// BAD: a wildcard action on a wildcard resource grants far more than the
// service actually needs, and is a common result of resolving an "access
// denied" error by broadening the policy instead of identifying the
// specific permission actually required
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}

// GOOD: scoped to the specific actions and the specific resource the
// service actually needs
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::app-user-uploads/*"
}
```
- **DON'T:** Grant a service or developer role standing administrative access to cloud infrastructure "to avoid repeated access requests." Use time-bound, just-in-time elevated access (a temporary role assumption with an expiration, an approval-gated elevation workflow) for the relatively rare cases that genuinely need broader access, rather than a permanently broad standing grant.
- **DO:** Audit cloud storage buckets, database instances, and managed service endpoints for accidental public exposure on a recurring, automated basis (many cloud providers offer a built-in public-access-detection feature) — a storage bucket accidentally left publicly readable is one of the most common, most damaging, and most easily automated-away cloud misconfigurations.
- **DON'T:** Trust a resource's default access configuration when creating new cloud infrastructure, since defaults vary by provider, by resource type, and change over time. Explicitly set and verify the intended access configuration for every new storage bucket, database, or managed service rather than assuming "the default is private" without checking.
- **DO:** Scope cross-account and cross-service trust relationships (an IAM role another AWS account can assume, a service account another GCP project can impersonate) as narrowly as the specific integration requires, and periodically review which external accounts/services hold a trust relationship into your infrastructure, since an overly broad or forgotten trust relationship is effectively a standing backdoor if the trusted party's own security is ever compromised.
- **DON'T:** Reuse the same cloud credentials or IAM role across multiple unrelated services or environments. Separate credentials per service and per environment (as with application secrets) limits how far a single compromised credential can reach, and makes it possible to revoke access for one compromised component without disrupting every other service sharing that credential.

### Automated Authorization Testing

- **DO:** Build a permission-matrix test suite that systematically exercises every (role, resource, action) combination the application's access-control model defines, asserting both that permitted combinations succeed and that every other combination is denied, rather than hand-writing a handful of ad hoc authorization tests that happen to occur to whoever wrote them.
```python
# GOOD: a parameterized test sweeps the full permission matrix rather than
# testing only the specific cases a developer happened to think of
@pytest.mark.parametrize("role,action,expected_allowed", [
    ("owner", "delete", True),
    ("editor", "delete", False),
    ("viewer", "delete", False),
    ("viewer", "read", True),
    ("editor", "read", True),
])
def test_permission_matrix(role, action, expected_allowed):
    user = make_user(role=role)
    assert policy.can(user, action, resource) == expected_allowed
```
- **DON'T:** Let authorization tests only cover the roles and actions that existed when the test suite was first written. Add a new row to the permission-matrix test whenever a new role or a new sensitive action is introduced, treating the matrix as a living document of the system's actual access-control intent, not a one-time exercise.
- **DO:** Include negative authorization tests (an unauthenticated request, a request from a different tenant/organization, a request with a tampered resource ID) in the standard test suite for every new sensitive endpoint, as a required part of the definition of "done" for that endpoint, not an optional follow-up.

### Multi-Tenancy Data Isolation Strategies

- **DO:** Choose a multi-tenant data-isolation strategy deliberately — separate databases per tenant (strongest isolation, highest operational overhead), separate schemas per tenant, or a shared schema with a `tenant_id` column enforced on every table — and understand that the shared-schema approach, while the cheapest to operate, places the entire isolation guarantee on every single query remembering to filter by tenant correctly, which is exactly the kind of "must remember every time" pattern that eventually gets missed somewhere.
- **DON'T:** Rely purely on application-code discipline ("every repository method filters by `tenant_id`") as the *only* enforcement layer for a shared-schema multi-tenant design, if the database offers a stronger guarantee. A single missed `WHERE tenant_id = ?` clause in one query, anywhere in a large codebase, is a cross-tenant data leak — and it only takes one.
- **DO:** Use database-level row-level security (RLS) where the database supports it, so tenant isolation is enforced by the database itself for every query automatically, independent of whether the application code remembered to add a tenant filter — this converts a "must remember every time" risk into a structural guarantee.
```sql
-- GOOD: PostgreSQL row-level security enforces tenant isolation at the
-- database layer — even a query that forgets a WHERE tenant_id clause
-- only ever sees rows belonging to the current session's tenant
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON invoices
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- the application sets the tenant context once per connection/transaction:
SET app.current_tenant_id = '11111111-2222-3333-4444-555555555555';
```
- **DON'T:** Set the tenant context for row-level security (or an equivalent session-level isolation setting) from a value the client can influence directly. Derive it the same way any other authorization-relevant context is derived — from the authenticated session/token server-side — never from a request parameter, exactly as covered under Authorization above for tenant-scoped queries in general.
- **DO:** Test tenant isolation explicitly and continuously (an automated test that creates data in tenant A and asserts a tenant B session cannot read or write it, across every table that carries tenant-scoped data) rather than trusting a one-time manual verification, since this is precisely the kind of guarantee that silently regresses when a new table or query is added later without the same discipline applied.

### Insider Threat and Separation of Duties

- **DO:** Apply separation of duties to the most consequential actions a system permits — the person who approves a production deployment shouldn't necessarily be the same person who wrote and merged it unreviewed; the engineer with database access for debugging shouldn't necessarily also hold the ability to disable audit logging — so that a single compromised or malicious individual account cannot unilaterally cause and simultaneously cover up significant harm.
- **DON'T:** Design access-control policy purely around external threats while leaving every internal, authenticated actor implicitly trusted with unlimited scope. Insider threat — whether a genuinely malicious insider or, far more commonly, a compromised legitimate account being used by an external attacker who has already gained a foothold — is a real category the least-privilege and audit-logging principles throughout this document are equally meant to address, not only external, unauthenticated attackers.
- **DO:** Require a second approval (a "four-eyes" review) for the highest-risk administrative actions — deleting a production database, disabling a security control, granting another account broad access — so that no single set of credentials, however well-intentioned, can unilaterally take an irreversible or highly sensitive action alone.

### Systematic Threat Modeling for Access Control

- **DO:** Walk through a structured threat-modeling exercise (considering, for each significant component: how could someone impersonate a legitimate actor, tamper with data, repudiate an action, read data they shouldn't, cause unavailability, or gain more privilege than intended) when designing a new feature's access-control model, rather than relying purely on ad hoc intuition about what "obviously" needs a permission check.
- **DON'T:** Treat threat modeling as a one-time exercise performed only at initial design time. Revisit the threat model when a feature's data sensitivity changes, when new user roles are introduced, or when the feature is significantly extended — the access-control assumptions that were reasonable for the original, narrower feature don't automatically still hold once its scope has grown.
- **DO:** Involve someone other than the feature's own implementer in a design-time threat-modeling discussion for anything handling sensitive data or performing sensitive actions, since a fresh perspective is more likely to spot an access-control gap the implementer's own mental model of "how this is supposed to be used" has trained them not to see.

## Secrets Management

Secrets — API keys, database credentials, signing keys, third-party tokens — are the crown jewels of most systems, because a single leaked secret often grants an attacker the same access a legitimate service has. The goal is to make sure secrets never live anywhere they can be casually stumbled upon: not in source code, not in logs, not in error messages, not in a chat message pasted "just to debug this quickly."

### Never Hardcoding Secrets in Source

- **DO:** Load all secrets — API keys, database passwords, signing keys, third-party service tokens, webhook signing secrets — from environment variables, a secret manager, or an encrypted configuration store at runtime, never as literal values written into source code.
```python
# BAD: the API key is a literal string baked into the source file
STRIPE_API_KEY = "sk_live_51H8x9K2example..."

# GOOD: the value is read from the environment at startup, never committed
import os
STRIPE_API_KEY = os.environ["STRIPE_API_KEY"]
```
- **DON'T:** Hardcode a secret "just for now" or "just to get this working locally" with the intention of moving it to proper configuration later. This is one of the most common ways real secrets end up permanently in version-control history — the temporary hardcoded value gets committed before anyone remembers to move it, and even a later commit that removes it leaves the original value recoverable from git history indefinitely.
- **DO:** Treat any secret that was ever committed to version control as compromised the moment it's discovered, and rotate it immediately — removing the line in a follow-up commit does not remove it from git history, which remains fetchable by anyone with repository access (including anyone who cloned it before the fix, and potentially the public if the repository is or ever becomes public).
- **DON'T:** Assume a private repository makes hardcoded secrets acceptable. Private repositories are regularly made public by mistake, forked into a personal account with different access settings, exposed through a misconfigured CI system, or accessed by more people over time (contractors, new hires, third-party integrations) than the original "this is just internal" assumption accounted for.
- **DO:** Add a git pre-commit hook or CI-integrated secret-scanning tool (scanning for API key patterns, private key headers, high-entropy strings) to catch accidental secret commits before they reach the remote repository or immediately after, rather than relying purely on developer discipline.
- **DON'T:** Paste real secrets into chat tools, ticketing systems, shared documents, or an AI coding assistant's prompt/context "just to debug" — any system that stores that conversation now holds a live credential, often with a much less carefully managed retention and access policy than the actual secret store. Redact or use a placeholder value when sharing configuration for debugging purposes.

### Secret Storage: Environment Variables and Secret Managers

- **DO:** Use a dedicated secret-management service (a cloud provider's secret manager, a self-hosted vault product, or at minimum an encrypted-at-rest configuration store with access logging) for production secrets rather than relying solely on plain environment variables passed through deployment configuration, once the system's sensitivity or team size justifies the added infrastructure.
- **DON'T:** Treat environment variables as inherently secure just because they're not hardcoded in source. Env vars are often visible to any process running as the same user, can leak through child-process inheritance, are frequently dumped by error-reporting tools' "environment" debug panels, and are usually visible in plaintext to anyone with access to the deployment platform's configuration UI — they're a meaningful improvement over hardcoding, not an endpoint of secret hygiene.
- **DO:** Restrict access to the secret manager itself with the same least-privilege discipline applied to any other sensitive system — scope IAM policies so a service can only read the specific secrets it needs, and audit-log every secret read/write so unusual access patterns (a service suddenly reading a secret it's never touched before) are visible.
- **DON'T:** Share one broad secret-manager credential or role across every service and every environment (dev, staging, production) "to keep things simple." A single compromised low-sensitivity service (say, a marketing site) should not be able to read the production database credential; separate secrets by environment and by service, and separate the credentials that unlock them accordingly.
- **DO:** Encrypt secrets at rest wherever they're stored (a secret manager, a database column holding an API key for a third-party integration a user connected, a configuration file), using envelope encryption or the platform's built-in encryption-at-rest rather than storing them as plaintext columns/files "because the disk itself is encrypted." Disk-level encryption protects against physical media theft, not against a SQL injection or a misconfigured backup export that reads the column directly.
- **DON'T:** Store secrets needed by client-side/frontend code as if they were server-side secrets. Anything shipped to a browser or mobile app bundle is extractable by the end user no matter how it's obfuscated; only ship to the client the specific, narrowly-scoped, ideally short-lived credentials meant to be public-facing (a publishable API key explicitly designed for client use), and keep the genuinely sensitive secret server-side, proxying any call that needs it.
```javascript
// BAD: a server-only secret key shipped into client-side bundle code
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY); // bundled into frontend JS

// GOOD: the secret key stays server-side; the client only ever sees a scoped publishable key
// server: uses STRIPE_SECRET_KEY to create a payment intent, returns only a client_secret
// client: uses the publicly-shareable publishable key to confirm the payment
const stripe = Stripe(PUBLISHABLE_KEY); // safe to expose
```

### Secret Rotation

- **DO:** Design every credential with rotation in mind from the start — support multiple simultaneously valid keys/versions during a rotation window (old key still works while the new one is being rolled out), so rotating a secret doesn't require a synchronized, all-at-once, downtime-inducing deployment across every consumer.
- **DON'T:** Let a secret live unrotated indefinitely simply because "it hasn't been a problem." Rotate secrets on a defined schedule appropriate to their sensitivity, and immediately and unconditionally on any suspected exposure (an accidental log, a departing employee who had access, a third-party vendor breach) rather than waiting to see if anything bad happens.
- **DO:** Automate rotation wherever the secret-management tooling supports it (many cloud secret managers can auto-rotate database credentials on a schedule and update dependent services automatically), since manual rotation processes are the ones that get skipped when things get busy.
- **DON'T:** Forget to revoke the *old* credential after rotating to a new one. Issuing a new API key without disabling the old one leaves two valid credentials where an attacker only needs to have captured either — rotation isn't complete until the previous value stops working.
- **DO:** Maintain an inventory of what has access to which secrets (which services, which people, which third-party integrations) so that when a rotation or revocation is needed, the actual scope of "everything that needs to be updated" is known rather than discovered by trial and error when something breaks.

### Keeping Secrets Out of Logs, Errors, and Debug Output

- **DO:** Explicitly redact known-sensitive fields (passwords, tokens, API keys, authorization headers, credit card numbers, session identifiers) in logging middleware/formatters before they're ever written out, using a structured-logging library's field-redaction feature or an explicit allowlist of loggable fields rather than logging the entire raw request object.
```python
# BAD: the full request, including the Authorization header, is logged as-is
logger.info(f"Incoming request: {request.headers} {request.body}")

# GOOD: sensitive headers/fields are redacted before logging
safe_headers = {k: ('***REDACTED***' if k.lower() == 'authorization' else v)
                for k, v in request.headers.items()}
logger.info("Incoming request", extra={"headers": safe_headers})
```
- **DON'T:** Log full request/response bodies indiscriminately "for debugging" on any endpoint that touches authentication, payment, or personal data. A verbose logging statement added during development is easy to forget and ship to production, where it now writes secrets into a log-aggregation system with its own separate, often broader, access policy.
- **DO:** Configure error-tracking and crash-reporting tools (Sentry-style services, APM tools) to scrub sensitive fields from captured request context, stack traces, and breadcrumbs — most of these tools support field-level redaction rules, and the defaults are rarely strict enough for fields specific to your application.
- **DON'T:** Include a secret's actual value in an exception message or a custom error thrown for debugging ("Failed to connect using key sk_live_..."). Even if the exception is caught and never shown to an end user, it can still be captured whole by an error-tracking tool, printed to a console log, or included in a crash dump.
- **DO:** Treat database connection strings, cloud credentials, and third-party tokens as sensitive in *every* context they might surface — startup logs, health-check endpoints, `/debug` or `/status` pages, verbose CLI output — not only in the main application request/response path.
- **DON'T:** Include secrets in URLs (as query-string parameters) for any authenticated API call where an alternative exists. URLs are routinely logged in full by web servers, proxies, browser history, and referrer headers, so a secret placed in a query string ends up copied into far more logs than a value sent in a request header or body ever would.

### Secrets and Version Control Hygiene

- **DO:** Commit a `.gitignore` (or equivalent) entry for `.env`, `.env.local`, and any other local secret/config file from the very first commit of a project, and provide a checked-in `.env.example`/`.env.sample` with variable names and dummy/placeholder values so contributors know what to configure without ever seeing real values.
```
# .gitignore
.env
.env.local
.env.*.local
*.pem
*.key
secrets.yml
```
- **DON'T:** Commit a real `.env` file "just this once to unblock a teammate" or check in a working example with real credentials "since it's easier to copy." Even a single commit puts the value in history permanently unless the history itself is rewritten and force-pushed everywhere, which is disruptive and easy to get wrong.
- **DO:** Run a secret-scanning tool across the *entire* git history (not just new commits) when auditing an existing repository's security posture, since a secret committed years ago and later "removed" is still present in history and still needs to be rotated if discovered.
- **DON'T:** Assume `git rm` or deleting a file in a later commit removes the sensitive content from the repository. The file's content remains fully retrievable from any earlier commit unless the history is explicitly rewritten (and even then, any existing clone, fork, or cached copy retains it) — rotating the exposed secret is the only reliable remediation once it has been committed.
- **DO:** Extend the same discipline to infrastructure-as-code files (Terraform state, Kubernetes manifests, Docker Compose files, CI pipeline YAML) — these frequently end up carrying hardcoded credentials for "convenience" and are just as much a version-control leak risk as application source code.

### Secrets in CI/CD and Build Pipelines

- **DO:** Store CI/CD secrets in the CI platform's dedicated secrets/masked-variable feature (not as plain pipeline configuration variables) so they're masked in build logs and scoped to the specific pipelines/branches that need them.
- **DON'T:** Print environment variables, full `env` dumps, or verbose command output that might echo a secret value during a CI build step for debugging purposes. Even a masked secret variable can leak in full if a build script explicitly echoes it or if it's passed through a command whose full invocation gets logged.
- **DO:** Scope CI/CD deployment credentials narrowly to what that specific pipeline needs to do (deploy to one specific environment, push to one specific registry) rather than granting a single CI credential broad account-wide or organization-wide access.
- **DON'T:** Let secrets used in CI/CD be accessible to pipelines triggered by pull requests from forks or external contributors without careful scoping. A public repository's CI configuration that exposes deployment secrets to any PR-triggered workflow can let an external contributor exfiltrate those secrets simply by adding a step to their PR's workflow that echoes the environment.
- **DO:** Rotate any secret that was ever exposed through a misconfigured CI pipeline, a public build log, or a third-party CI/CD integration immediately upon discovery, following the same "assume compromised, rotate now" rule that applies to secrets found in source control.
- **DON'T:** Grant a deployment or infrastructure-as-code service account standing write access to the secret manager itself unless the pipeline genuinely needs to create or rotate secrets as part of its job — a pipeline that only needs to *read* a secret to deploy should hold a read-only credential, not one that can also modify or delete secrets.
- **DO:** Keep a documented, tested incident-response procedure specifically for secret exposure (who to notify, which systems to rotate, how to check for misuse of the exposed credential in access logs before and after rotation) so a leaked secret is handled by a known checklist rather than improvised under pressure.

### Secrets in Containers and Infrastructure

- **DO:** Pass secrets into containers at runtime through the orchestration platform's secret-injection mechanism (mounted secret volumes, an init-container fetching from a secret manager, environment variables injected by the orchestrator from a secret store) rather than baking them into the container image itself.
- **DON'T:** `COPY` or `ADD` a file containing secrets into a Docker image, or set a secret via a Dockerfile `ENV`/`ARG` instruction, even if a later layer deletes the file — every intermediate layer of a container image remains part of the image's history and is extractable by anyone who can pull or inspect the image, regardless of what a later layer appears to remove.
```dockerfile
# BAD: the secret is baked into an image layer; deleting it in a later
# instruction does not remove it from the image's layer history
COPY secrets.env /app/secrets.env
RUN load_secrets.sh && rm /app/secrets.env

# GOOD: the secret is never part of the image; it is injected at container
# runtime from the orchestrator's secret store
# (e.g., a Kubernetes Secret mounted as a volume, or an env var sourced
# from a secret manager by the deployment platform)
```
- **DO:** Restrict which infrastructure-as-code state (Terraform state files, CloudFormation stacks) is allowed to contain plaintext secret values, and prefer resource types/data sources that reference a secret manager entry rather than embedding the raw secret value directly in the IaC definition — remember that state files themselves are a common, often under-protected place secrets end up.
- **DON'T:** Store Terraform/IaC state files (which can contain plaintext secret values pulled from providers) in a location with broad read access, such as an unencrypted, broadly shared storage bucket or a version-controlled repository. Configure remote state storage with encryption at rest and access restricted to the specific pipeline/team that needs it.

### Secret Sprawl Across Environments and Vendors

- **DO:** Track which third-party vendors, SaaS integrations, and contractors have been given credentials or API keys, and review that list periodically — pruning access for integrations that are no longer in use and rotating keys shared with a vendor relationship that has ended.
- **DON'T:** Let a decommissioned integration's credentials remain active indefinitely just because nobody explicitly revoked them. A vendor integration that was disabled on the application side but never had its corresponding API key revoked on the vendor's side leaves a live credential that grants access even though the team believes the integration is "off."
- **DO:** Use distinct secrets per environment (development, staging, production) rather than sharing one set of credentials across all of them, so a leak or misuse in a lower-security environment (a developer's laptop, a staging server with looser access controls) doesn't compromise production data or systems.
- **DON'T:** Grant a third-party vendor or contractor broader API scope/access than the specific integration requires "because it was easier to set up that way." Apply the same least-privilege discipline to external parties as to internal service accounts — an external vendor's own security posture is outside your control, and its compromise shouldn't automatically become your production compromise.

### Automated Secret Detection and Response

- **DO:** Run automated secret-scanning continuously against every new commit (not only at initial repository setup) using a tool that maintains an up-to-date set of detection patterns for common credential formats, and wire its findings into a process that actually results in rotation, not just a dashboard nobody checks.
- **DON'T:** Treat a secret-scanning tool's alert as resolved simply by removing the offending line in a follow-up commit. As covered under version-control hygiene, the historical exposure remains regardless of the follow-up fix; the scanning tool's alert should trigger rotation of the actual credential, and only then be considered closed.
- **DO:** Extend secret scanning beyond the primary application repository to any other place source code or configuration lives — internal wikis with embedded code snippets, exported Jupyter notebooks, infrastructure-as-code repositories, and CI configuration repositories, all of which have been real-world sources of leaked credentials outside the "main" codebase a team usually thinks to scan first.

### Encrypting Secrets in Transit Between Services

- **DO:** Ensure the channel a secret travels over — from a secret manager to the consuming service, from a CI pipeline to a deployment target — is itself encrypted (TLS), consistent with the Transport Security guidance elsewhere in this document, since a secret transmitted in plaintext over an internal network is exposed to anyone who can observe that network segment.
- **DON'T:** Assume an internal deployment pipeline's secret-injection step is safe from interception just because it's "internal tooling." Apply the same transport-security expectations to internal secret-distribution paths as to any other sensitive data flow.

### The "Secret Zero" Bootstrapping Problem

- **DO:** Recognize that a secret manager itself needs to authenticate whoever asks it for a secret, which means some initial credential — a "secret zero" — has to exist to bootstrap that trust, and give real thought to how that bootstrapping credential is protected, since it's the one secret that can't itself be pulled from the secret manager it unlocks.
- **DON'T:** Solve the secret-zero problem by hardcoding the secret-manager credential the same way you'd want to avoid hardcoding any other secret. Prefer platform-native identity where available (a cloud compute instance's attached identity/role, a Kubernetes service account token, a CI platform's built-in OIDC federation with the secret manager) so the bootstrapping trust is established through the infrastructure platform's own identity system rather than through a static credential that itself needs to be distributed and protected.
- **DO:** Prefer short-lived, automatically-issued credentials for the secret-zero bootstrapping step wherever the platform supports it (workload identity federation, instance metadata-based identity) over a long-lived static bootstrap credential, since a short-lived, automatically rotated bootstrap credential dramatically narrows the window in which a leak of it would matter.

### Practical Secret Manager Patterns

- **DO:** Prefer dynamic, short-lived, on-demand credentials from the secret manager (a database credential generated fresh for a short lease and automatically expired) over long-lived static secrets pulled once and cached indefinitely, where the secret manager and the target system both support it — dynamic secrets bound to a short lease dramatically reduce the value of a leaked credential, since it expires on its own shortly after issuance regardless of whether the leak is ever detected.
- **DON'T:** Cache a fetched secret in application memory indefinitely with no re-fetch/expiry logic, even when the underlying secret is a dynamic, short-lived one. A cached-forever copy of a short-lived credential defeats the point of issuing it as short-lived; respect the credential's actual lease duration and re-fetch before it expires.
- **DO:** Use a sidecar or init-container pattern (a dedicated process that fetches secrets from the manager and makes them available to the main application process via a local file or environment, without the application itself needing direct secret-manager credentials) in containerized environments, which centralizes secret-fetching logic and keeps the application code itself decoupled from the specific secret-manager API in use.

### CI/CD Pipeline Isolation

- **DO:** Isolate build/test execution for untrusted or lower-trust code (a fork-submitted pull request, a dependency's install script) from the pipeline stages that hold deployment credentials or can push to production, using separate jobs, separate runners, or separate permission scopes, so that untrusted code executed during a build cannot reach the same secrets a trusted deployment stage uses.
- **DON'T:** Run a single monolithic pipeline where the same job that builds and tests a pull request (potentially executing code from an untrusted contributor or a freshly added dependency) also holds credentials capable of deploying to production. Split the pipeline so a compromised build step's blast radius is limited to what that specific stage's scoped credentials can reach.
- **DO:** Require manual approval (or at least a distinctly separate, protected pipeline trigger) for any pipeline stage that deploys to production or touches production credentials, rather than letting every merge automatically cascade all the way to a production deployment with no human checkpoint for changes above a certain risk level.

### API Documentation and Collection Leaks

- **DO:** Treat exported API client collections (Postman/Insomnia collections), generated API documentation, and example `.http`/`curl` scripts as a potential secret-leak vector, since developers commonly save a working request — real API key, real session token, or real test credentials included — into a shared collection file for convenience and later share or commit that file without stripping the captured credentials.
- **DON'T:** Commit an exported API client collection to version control, or attach one to a shared ticket/wiki page, without first checking whether it captured any live credential in a saved header, environment variable, or example value. Treat this exactly like any other artifact scanned for secrets before sharing.
- **DO:** Use the API client's environment-variable/secret-reference feature to store credentials separately from the shared collection file itself, so a collection can be safely shared or committed without also sharing whatever credential was used to test it.

### Physical Device Security for Credentials

- **DO:** Require full-disk encryption and a strong device passcode/password on any developer or administrator machine that holds credentials, SSH keys, or cached secret-manager sessions capable of reaching production systems, and support remote wipe for lost or stolen devices where the organization's device-management tooling allows it.
- **DON'T:** Let long-lived credentials (SSH private keys, cloud CLI session tokens, secret-manager cached sessions) sit unprotected on a laptop with no passphrase on the key and no disk encryption. A lost or stolen unencrypted laptop with cached production credentials is a direct path to a production compromise, independent of anything about the application's own code-level security.

## Dependency Security

### Native and Binary Dependency Risks

- **DO:** Apply extra scrutiny to dependencies that include native/compiled binary code (a native Node/Python extension, a compiled Rust/C library wrapped for another language), since a native binary is far harder to casually audit than the equivalent pure-language source, and a malicious or compromised native extension has the same low-level access to the process (filesystem, network, memory) as any other native code — source-level review of a pure-language package's readable code doesn't apply the same way when part of the package is a prebuilt binary blob.
- **DON'T:** Trust a package's prebuilt binary distribution to match its published source code without some form of verification (a reproducible-build check, provenance attestation as covered under Dependency Security's supply-chain guidance above, or at minimum building from source when the stakes justify the extra effort). A published binary and a published source tree are, without independent verification, only related by the publisher's claim that they match.

Modern applications are assembled mostly from other people's code — direct dependencies, their transitive dependencies, container base images, CI actions, and build tooling. Every one of those is a piece of the supply chain that can introduce a vulnerability or outright malicious code without a single line of first-party code changing. Treating dependency management as a security-relevant discipline, not just a version-bumping chore, is what keeps that supply chain trustworthy.

### Keeping Dependencies Patched

- **DO:** Keep dependencies reasonably current and apply security patches promptly, especially for direct dependencies with a disclosed CVE affecting a code path the application actually exercises. The overwhelming majority of exploited dependency vulnerabilities are for known, already-patched issues that the affected application simply hadn't updated past yet.
- **DON'T:** Let dependency updates pile up indefinitely because "upgrading is risky" or "it's not broken." The longer an upgrade is deferred, the larger and riskier the eventual jump becomes (more breaking changes accumulated at once), which paradoxically makes teams even more reluctant to upgrade — small, frequent updates are safer in aggregate than large, infrequent ones.
- **DO:** Run automated dependency-vulnerability scanning (software composition analysis / SCA tooling) in CI, failing or flagging builds that introduce a dependency with a known critical/high-severity vulnerability, so vulnerable dependencies are caught before merge rather than discovered later in a periodic audit.
- **DON'T:** Treat a dependency-scanner alert as noise to be dismissed reflexively. Triage each finding on its actual relevance (is the vulnerable code path reachable from how the application uses the library? is a fix available?) and either patch, mitigate, or explicitly document an accepted-risk decision — silently ignoring every alert defeats the purpose of running the scanner at all.
- **DO:** Subscribe to or otherwise monitor security advisories for the specific frameworks, libraries, and platforms the application depends on most heavily (the language runtime, the web framework, the ORM, any cryptography library), since automated scanners can lag behind a fresh disclosure by hours to days.
- **DON'T:** Assume automated dependency bots (that open a pull request bumping a version) are safe to auto-merge blindly without CI running the full test suite against the new version. A patch release can still introduce a behavioral regression; require the same review/test gate for a dependency bump as for any other code change, while still keeping the review lightweight for routine patch-level bumps.
- **DO:** Prioritize patching based on actual exploitability and exposure — a critical vulnerability in a library that processes untrusted input on an internet-facing endpoint deserves same-day attention, while a low-severity issue in an internal build-only tool can follow the normal update cadence.

### Understanding Supply-Chain Risk

- **DO:** Recognize that a dependency's code runs with the same privileges as the application itself — a compromised or malicious package can read environment variables, access the filesystem, make network calls, and exfiltrate secrets exactly as if that code had been written in-house. "It's just a small utility library" is not a reason to skip scrutiny.
- **DON'T:** Add a new dependency without a basic sanity check on its trustworthiness — maintenance activity, download/adoption numbers relative to what it claims to do, whether the maintainer account has a history, and whether the package's actual behavior (network calls, filesystem access, install-time scripts) matches what a package with its stated purpose should need.
- **DO:** Be specifically wary of packages that run arbitrary code at install time (`postinstall` scripts, setup.py with executable logic) pulled from a public registry, since install-time hooks are a common vector for supply-chain attacks — the malicious payload runs the moment `npm install`/`pip install` executes, before the application code is ever run or reviewed.
- **DON'T:** Casually install and run CLI tools or one-off scripts from public package registries (`npx some-package`, `pip install && run` from an unfamiliar source) on a developer machine or CI runner that has access to real credentials, without first checking what the package actually is. Developer and CI environments are high-value targets precisely because they often hold broad access to source control, cloud credentials, and signing keys.
- **DO:** Consider using a private package registry / proxy (an internal mirror of the public registry) with the ability to block or quarantine newly published package versions for a short window, which mitigates a common supply-chain attack pattern where a compromised maintainer account publishes a malicious version that gets pulled by automated builds within hours before being caught and pulled from the public registry.
- **DON'T:** Grant CI/CD pipelines broader network egress or credential access than the build actually needs. A supply-chain-compromised dependency running inside a build step can only exfiltrate what that build environment can reach — restricting egress and scoping credentials limits the damage even if a malicious package does get installed.

### Typosquatting and Malicious Packages

- **DO:** Double-check package names character-by-character before adding a new dependency, especially for names that are short, common English words, or easily confused with a well-known package (a missing/extra hyphen, a swapped letter, a different pluralization). Typosquatting — publishing a malicious package under a name deliberately similar to a popular one — relies entirely on a developer or an automated tool mistyping or misreading the name once.
```
# BAD: a typosquatted package name that looks almost identical to the real one
npm install cross-env-utils     # (if the actual intended package was "cross-env")
pip install python-dateutil-    # trailing/odd characters easy to miss when copy-pasting

# GOOD: copy the exact package name from its official registry page or documentation
npm install cross-env
pip install python-dateutil
```
- **DON'T:** Copy a package installation command from an unfamiliar blog post, a search-engine result, or an AI-generated answer without cross-checking the package name against the language ecosystem's official registry search. A slightly wrong name in copied instructions is a common typosquatting delivery vector, and this applies equally to commands an AI assistant suggests — verify before running.
- **DO:** Review a new dependency's published source (or diff against the previous version on an upgrade, for anything unusually sensitive) when it is small enough to reasonably read, particularly for a package that will run in a privileged context (a build tool, a CI action, anything with network or filesystem access).
- **DON'T:** Assume a package with a high download count is automatically safe. Malicious versions have shipped through account takeovers and social-engineered maintainer handoffs on packages with millions of weekly downloads; popularity reduces but does not eliminate supply-chain risk, and a sudden, unexplained new maintainer or an unusual release right after a maintainer-account compromise report are signals worth checking.
- **DO:** Watch specifically for "dependency confusion" attacks in organizations that use both public and private/internal package registries — a public package published under the same name as an internal-only package can get pulled by a misconfigured build that resolves to the public registry instead of the internal one, silently running attacker-controlled code. Configure package managers to explicitly scope internal package names to the internal registry rather than relying on default resolution order.

### Lockfile Discipline

- **DO:** Commit the lockfile (`package-lock.json`/`yarn.lock`/`pnpm-lock.yaml`, `Pipfile.lock`/`poetry.lock`, `Gemfile.lock`, `Cargo.lock`, `go.sum`) to version control for every project, and install dependencies in CI and production using the lockfile-respecting install command (`npm ci`, not `npm install`) so builds are reproducible and every environment resolves to exactly the same dependency versions.
```
# BAD: no lockfile committed, or CI installs without honoring one
npm install    # can silently update transitive dependency versions

# GOOD: lockfile committed and CI installs strictly from it
npm ci         # fails if package.json and package-lock.json are out of sync
```
- **DON'T:** Use loose version ranges (`^`, `~`, or unpinned "latest") for dependencies in a way that allows an automatic, unreviewed upgrade to a new version between one build and the next without the lockfile pinning the actual resolved version. Without a respected lockfile, "the same command" can install different code on different days, including a newly published malicious version.
- **DO:** Treat lockfile changes in a pull request as worth actually looking at, not auto-approving — a diff that updates far more transitive dependencies than the stated change would explain is worth a second look, since it can indicate a dependency confusion issue or an unexpected transitive upgrade pulling in a compromised package.
- **DON'T:** Manually hand-edit a lockfile to "fix" a merge conflict without regenerating it properly through the package manager. A hand-edited lockfile can end up out of sync with what the manifest actually declares, silently reintroducing unpinned resolution or resolving to a version nobody intended.
- **DO:** Regenerate and commit a fresh lockfile after any manual dependency-version change in the manifest file, and verify the resulting diff only touches what's expected, rather than letting the lockfile and manifest drift out of sync.

### Avoiding Assumed or Hallucinated Packages

- **DO:** Verify that a package actually exists on the official registry, under the exact name and maintained by a plausible source, before adding it to a manifest — whether the suggestion came from documentation, a colleague, or an AI coding assistant. Never install a package purely because its name "sounds right" or matches the shape of what's needed.
- **DON'T:** Let an AI coding assistant (or any other automated suggestion) add an import or install command for a package without confirming that package is real, actively maintained, and is the one actually intended. Language models can generate plausible-sounding but non-existent package names ("hallucinated" dependencies), and attackers have specifically registered packages under names that models are known to hallucinate, anticipating exactly this workflow — a package that looks legitimate enough to install is often malicious by design in this scenario.
```
# BAD: installing a plausible-sounding package name without verifying it's the real,
# actively maintained package the developer actually intended
pip install fast-json-parser-utils   # sounds plausible, may not be the real/intended package

# GOOD: verify the package's existence, maintainer, and adoption on the official
# registry first, and prefer a name you can confirm from official documentation
pip install orjson   # a real, well-known, verifiable package for the same purpose
```
- **DO:** Prefer well-known, already-vetted libraries the team has used before, or ones explicitly named in the target ecosystem's official documentation, over a brand-new suggestion for a one-off need — especially when the suggestion comes from generated code that hasn't been independently verified.
- **DON'T:** Treat "the code imports it, so it must be a real package" as verification. A generated code sample can import a plausible but nonexistent module name with complete syntactic confidence; the only real verification is checking the package registry directly before running any install command.

### Software Bill of Materials and Ongoing Auditing

- **DO:** Generate and maintain a software bill of materials (SBOM) for production applications, listing every direct and transitive dependency with its version, which makes it possible to answer "are we affected by this newly disclosed CVE" in minutes rather than days when a new vulnerability is announced.
- **DON'T:** Treat a one-time dependency audit as sufficient. New vulnerabilities are disclosed continuously against existing, unchanged code — a dependency that was clean six months ago can have a critical CVE disclosed against it today — so dependency scanning needs to run on an ongoing/scheduled basis, not only when new dependencies are added.
- **DO:** Extend the same patching and scanning discipline to container base images, CI/CD action definitions (e.g., third-party pipeline steps pulled by reference), and infrastructure tooling, not only application-level package manifests — all of them are part of the same supply chain and can carry the same categories of vulnerability.
- **DON'T:** Pin a container base image or CI action to a mutable tag (`latest`, `main`, an unpinned major version) for anything security-sensitive. Pin to a specific, immutable version or content digest so the exact code that runs is known and reproducible, and update deliberately rather than inheriting whatever changed upstream since the last build.
```yaml
# BAD: mutable tags mean the actual code that runs can change without any local change
FROM node:latest
# uses: some-org/some-action@main

# GOOD: pinned to an explicit, immutable version (ideally a content digest)
FROM node:20.11.1-bookworm-slim
# uses: some-org/some-action@v4.1.0
```
- **DO:** Vet third-party CI/CD actions and build plugins with the same scrutiny as application dependencies before adopting them — they typically run with access to repository secrets and deployment credentials, making a compromised action an especially high-value supply-chain target.
- **DON'T:** Ignore transitive dependencies when reasoning about supply-chain exposure. A direct dependency chosen carefully can still pull in dozens or hundreds of transitive packages nobody explicitly reviewed; SCA tooling that reports vulnerabilities across the full dependency tree, not just top-level packages, is necessary to see the real exposure.
- **DO:** Prefer dependencies with a small, well-scoped footprint over a large multi-purpose package when only a fraction of its functionality is actually needed, since a smaller dependency surface means less transitive code to trust, patch, and audit.
- **DON'T:** Vendor (copy source directly into the repository) a third-party library and then never update it. Vendoring removes the dependency from automated scanning/update tooling in most setups, so a vendored copy quietly falls out of the patch cycle unless the team deliberately tracks and re-syncs it against upstream security releases.

### Container and Base Image Security

- **DO:** Choose minimal base images (a slim/distroless variant rather than a full general-purpose OS image) for production containers, which reduces the number of installed packages that can carry a vulnerability and shrinks the attack surface if a container is ever compromised.
- **DON'T:** Run application containers as the root user by default. Set an explicit non-root `USER` in the image and configure the orchestration platform to reject containers that attempt to run as root where policy allows it, so a container-breakout vulnerability doesn't automatically hand an attacker root-equivalent access to the host.
- **DO:** Scan container images for OS-package and application-dependency vulnerabilities as part of the build pipeline, the same way source-level dependencies are scanned, since a base image can introduce vulnerable system libraries entirely independent of the application's own dependency manifest.
- **DON'T:** Trust an unofficial or unverified base image from a public registry without checking its publisher and scan history. Prefer official images maintained by the language/framework project or a verified publisher, and pin to a specific digest once a version has been vetted.

### Reproducible Builds and Provenance

- **DO:** Aim for build reproducibility (the same source and lockfile always produce a bit-for-bit or verifiably equivalent artifact) where the toolchain supports it, and adopt build-provenance attestation (a cryptographically signed record of what source and process produced a given artifact) for critical software, which makes it possible to detect if a published artifact doesn't actually correspond to the source it claims to.
- **DON'T:** Treat "the package published on the registry" and "the source code in the public repository" as automatically the same thing without any verification. A compromised publishing credential can let an attacker publish a malicious package version to the registry without ever touching the actual source repository, which is why provenance attestation and reproducible builds matter as a check independent of source-code review.

### Abandoned and Unmaintained Dependencies

- **DO:** Periodically check whether critical dependencies are still actively maintained — recent commits, responsive maintainers, timely security-patch history — and have a plan (fork-and-maintain internally, migrate to an alternative) for any critical, unmaintained dependency before a vulnerability is discovered in it with no one left to fix it.
- **DON'T:** Keep adding features on top of a dependency that has shown clear signs of abandonment (no releases in years despite open vulnerability reports, an unresponsive maintainer) without a documented risk acceptance or a migration plan, since an abandoned dependency with a future vulnerability becomes the team's own problem to patch with no upstream fix ever coming.

### Build Tooling and Compiler Supply Chain

- **DO:** Apply the same scrutiny to build-time tooling — compilers, bundlers, transpilers, code generators, linters run as part of the build — as to runtime dependencies, since malicious code injected at build time by a compromised build plugin ends up in every artifact the build produces, often without leaving any trace in the application's own source code.
- **DON'T:** Grant build plugins or bundler plugins broad filesystem or network access without cause. A build-time plugin that can read arbitrary files or make network calls during the build is in a position to exfiltrate source code or secrets present in the build environment, independent of anything in the final shipped artifact.
- **DO:** Keep the build toolchain itself (not just application dependencies) on a patch cadence, since compilers and bundlers are complex software with their own vulnerability history, and a build system's supply-chain compromise can be significantly more damaging than a single library's, because it touches every artifact the pipeline produces.

### Monorepo and Internal Package Risks

- **DO:** Apply the same version pinning and review discipline to internal, first-party packages published to a private registry (a shared internal UI library, a common internal SDK) as to third-party dependencies, since an internal package with a compromised publishing pipeline is just as capable of distributing malicious code across every consuming team's application.
- **DON'T:** Assume an internal package is inherently safe simply because it was written in-house. An internal package's publishing credentials, CI pipeline, and registry access are all potential compromise points independent of the code's own origin, and dependency-confusion attacks specifically target the gap between internal and public registries for exactly this class of package.

### Ecosystem-Specific Auditing Tools

- **DO:** Run the dependency-auditing tool built into or standard for the ecosystem in use as a routine part of the development and CI workflow — most major package ecosystems now ship or have a widely adopted standard option for exactly this purpose, and running it costs little relative to the vulnerabilities it catches.
```
# Representative ecosystem-native audit commands — the specific tool and
# invocation vary by ecosystem and evolve over time, so check current
# documentation for the ecosystem actually in use:
npm audit                     # Node.js / npm
pip-audit                     # Python
bundle audit check            # Ruby
cargo audit                   # Rust
govulncheck ./...             # Go
composer audit                # PHP
dotnet list package --vulnerable   # .NET
```
- **DON'T:** Rely exclusively on one point-in-time audit run during initial project setup. Wire the audit command into CI so it runs on every build, and combine it with a dedicated software-composition-analysis (SCA) tool for broader coverage (license and transitive-dependency visibility, historical tracking) where the project's scale justifies the additional tooling investment beyond the ecosystem-native command.
- **DO:** Treat a clean audit result as a snapshot in time, not a permanent guarantee — schedule the same audit to run on a recurring basis (nightly or on every merge to the main branch, at minimum) independent of whether any dependency manifest changed, since new vulnerabilities are disclosed against unchanged code continuously.

### A Practical Checklist for Evaluating a New Dependency

- **DO:** Before adding a new dependency, check its recent commit/release activity (is it actively maintained), its issue tracker (are security reports acknowledged and addressed), its download/adoption numbers relative to its stated purpose, the number and reputation of its maintainers, and whether its declared license is compatible with the project's own licensing requirements.
- **DON'T:** Add a dependency purely because it appeared at the top of a search result or a generated suggestion without spending the few minutes this checklist takes — the cost of skipping it is paid, if at all, much later and much more expensively, when the dependency turns out to be abandoned, malicious, or license-incompatible after the codebase already depends on it in many places.
- **DO:** Prefer a dependency already used elsewhere in the codebase over adding a second, overlapping dependency for the same purpose, when a suitable one already exists — each additional dependency is additional supply-chain surface area, and consolidating on fewer, well-vetted dependencies reduces the total attack surface more than any per-dependency vetting step can.
- **DON'T:** Add a dependency solely to save writing a small amount of straightforward code that doesn't warrant an external dependency's ongoing maintenance and supply-chain exposure. Weigh a dependency's value against its cost realistically — not every few lines of logic need to become a third-party package.

### Third-Party Vendor and SaaS Security Assessment

- **DO:** Review a third-party vendor's or SaaS provider's own security posture (a current independent audit report, their data-handling and breach-notification terms, their own subprocessor list) before integrating them into a workflow that will touch sensitive data, treating vendor selection as a security-relevant decision and not purely a feature/pricing comparison.
- **DON'T:** Grant a new SaaS integration broad access (a full-account OAuth scope, an admin-level API key) when a narrower, purpose-specific scope would serve the integration's actual stated purpose — this is the same least-privilege principle applied to external vendor integrations that's covered under Authorization and Secrets Management above.
- **DO:** Maintain an inventory of which third-party vendors and integrations have access to which categories of data, and revisit that inventory periodically — an integration added for a specific short-term need that's still holding standing access long after that need ended is both unnecessary risk and, per the vendor's own future security incidents, a source of exposure the team may not even remember it has.

## Input Validation & Sanitization

### Idempotency and Duplicate Request Handling

- **DO:** Support and validate an idempotency key (a client-generated unique identifier for a specific logical operation, submitted with the request) for any endpoint that performs a non-repeatable sensitive action — charging a payment, sending a one-time notification, decrementing limited inventory — so that a network retry, a double-click, or a replayed request doesn't duplicate the effect.
```python
# GOOD: an idempotency key ties a retried request to the same stored result,
# so a network-level retry of an already-processed charge is a no-op rather
# than a second charge
def process_payment(idempotency_key, amount, account_id):
    existing = PaymentRecord.find_by(idempotency_key=idempotency_key)
    if existing:
        return existing.result   # already processed — return the same outcome
    result = charge_payment_provider(amount, account_id)
    PaymentRecord.create(idempotency_key=idempotency_key, result=result)
    return result
```
- **DON'T:** Treat idempotency purely as a reliability feature disconnected from security. A missing idempotency guard on a sensitive action is also a business-logic abuse vector, closely related to the race-condition and limited-resource-abuse concerns covered under Common Web Vulnerabilities — a client (malicious or merely buggy) that fires the same request multiple times in quick succession can trigger the same effect multiple times if there's no idempotency check to prevent it.
- **DO:** Scope an idempotency key's uniqueness to the account/session that submitted it, and validate that a replayed idempotency key matches the same request parameters as the original — an idempotency key reused with *different* parameters (a different amount, say) should be rejected as a conflict rather than silently returning the original result or silently processing the new parameters.

Every value that enters the system from outside the trust boundary — request bodies, query strings, headers, cookies, uploaded files, webhook payloads, even data read back out of a database that a lower-trust process wrote — needs to be validated before it's used. Validation and sanitization solve related but distinct problems: validation rejects input that doesn't conform to what's expected, while sanitization transforms input into a safe form for a specific context. Neither substitutes for the other, and both must happen on the server, regardless of what checks already ran on the client.

### Server-Side Validation Is Non-Negotiable

- **DO:** Re-validate every piece of input on the server, in full, even when the client (a web form, a mobile app, a single-page app) already performs its own validation. Client-side validation is a request the client happens to make honestly; nothing prevents a different client, a modified request, or a direct API call from skipping it entirely.
```javascript
// BAD: quantity bounds are enforced only in the browser form
// <input type="number" min="1" max="10" name="quantity">
app.post('/api/cart/add', (req, res) => {
  cart.add(req.body.productId, req.body.quantity); // no server-side bound check
});

// GOOD: the same bound is enforced again, authoritatively, on the server
app.post('/api/cart/add', (req, res) => {
  const quantity = Number(req.body.quantity);
  if (!Number.isInteger(quantity) || quantity < 1 || quantity > 10) {
    return res.status(400).json({ error: 'invalid quantity' });
  }
  cart.add(req.body.productId, quantity);
});
```
- **DON'T:** Treat client-side validation (HTML5 form constraints, JavaScript form checks, mobile app input masks) as a security control at all. It exists purely to give users fast feedback and reduce round trips for honest mistakes; it provides zero protection against a deliberately crafted request.
- **DO:** Validate input as close to the trust boundary as possible (the API entry point) so that everything past that boundary can be treated as already-validated, well-typed data, keeping validation logic centralized rather than re-implemented inconsistently deeper in the call stack.
- **DON'T:** Assume that because a mobile app or desktop client is "harder" to tamper with than a web browser, its requests need less server-side scrutiny. Any client is fully under the end user's control — API requests can be replayed, modified, and replayed again with a proxy tool regardless of what platform issued them originally.
- **DO:** Validate data that arrives from other internal services or from a message queue too, not only data that arrives directly from an external client. A field that was properly validated by service A when it first entered the system can still need re-validation by service B if B's assumptions about that field's shape differ, or if the value could have been mutated somewhere in between.

### Allowlisting Over Denylisting

- **DO:** Define what *valid* input looks like (an allowlist — a specific format, a specific character set, a specific enumerated set of accepted values, an explicit length range) and reject anything that doesn't match, rather than trying to enumerate every possible *invalid* pattern.
```python
# BAD: denylist tries to enumerate dangerous characters, easy to have gaps
def is_safe_username(name):
    return not any(c in name for c in ["<", ">", "'", '"', ";"])

# GOOD: allowlist defines exactly what a valid username can contain
import re
def is_valid_username(name):
    return bool(re.fullmatch(r"[a-zA-Z0-9_]{3,32}", name))
```
- **DON'T:** Rely on a denylist of "known bad" characters, keywords, or patterns as the primary input-validation strategy. Denylists require anticipating every dangerous variant in advance — different encodings, case variations, nested payloads — and attackers routinely find the gap the list's author didn't think of; a properly scoped allowlist has no equivalent gap because it only ever accepts what was explicitly enumerated as valid.
- **DO:** Validate against the narrowest reasonable definition of "valid" for each field's actual purpose — an account ID field should validate as the ID format the system actually uses (numeric, UUID, or a specific prefix pattern), not merely "any non-empty string."
- **DON'T:** Loosen validation rules reflexively the first time a legitimate edge case gets rejected, without checking whether the edge case reveals a genuinely incomplete allowlist or is actually out of scope for that field. A validation rule that's too strict is a bug to fix precisely; "just accept anything" is not the fix.
- **DO:** Validate structured formats (email addresses, phone numbers, URLs, dates, currency amounts) with a well-tested library or the platform's built-in type/format validators rather than a hand-rolled regular expression, since these formats have more edge cases (internationalized domains, valid-but-unusual email local parts) than they first appear to.

### File Upload Validation

- **DO:** Validate uploaded file type by inspecting the file's actual content (magic-byte/content sniffing, or a dedicated file-type-detection library) rather than trusting the client-supplied `Content-Type` header or the file extension alone, both of which are trivially spoofable by whoever is uploading the file.
```python
# BAD: trusts the client-supplied extension and MIME type at face value
if filename.endswith(('.jpg', '.png')) and content_type.startswith('image/'):
    save_upload(file)

# GOOD: the file's actual binary content is inspected to confirm its real type
import filetype
kind = filetype.guess(file_bytes)
if kind is None or kind.mime not in ('image/jpeg', 'image/png'):
    raise ValueError('unsupported or misidentified file type')
save_upload(file)
```
- **DON'T:** Allow uploaded files to be stored with a filename taken directly from the client, unsanitized, and served back from the same domain/origin as the main application, particularly for file types a browser might execute or render as HTML/script (SVG images, HTML files, `.js` files). Store uploads under a generated, unpredictable filename, and serve user-uploaded content from a separate origin/subdomain with no cookies or session context of its own so a malicious upload can't run in the context of an authenticated session.
- **DO:** Enforce a maximum file size limit on uploads, both at the application layer and, where available, at the web server / reverse proxy / load balancer layer before the request body is even fully read into application memory, to prevent a single oversized upload (or many concurrent ones) from exhausting memory or disk.
- **DON'T:** Store uploaded files inside the web server's document root or any directly web-accessible path without an intermediary access-control check, if those files are meant to be private or access-restricted. A file saved directly under a publicly served directory is retrievable by anyone who guesses or discovers its path, regardless of the application's intended access rules.
- **DO:** Scan uploaded files for malware where the threat model warrants it (a platform accepting uploads from untrusted or semi-trusted users, especially ones later downloaded by other users), using an antivirus/malware-scanning service integrated into the upload pipeline before the file is made available to anyone else.
- **DON'T:** Process uploaded files (image resizing, PDF parsing, document conversion, archive extraction) with a library or tool that has a history of parser vulnerabilities without keeping it patched and sandboxed. File-processing libraries are common targets precisely because they parse complex, attacker-influenced binary formats; run this processing in an isolated, resource-limited environment (a separate low-privilege process or container) so a parser exploit doesn't compromise the main application.
- **DO:** Validate archive uploads (`.zip`, `.tar`) against decompression-bomb and path-traversal risks specifically — check the uncompressed size and entry count before fully extracting, and reject any archive entry whose path attempts to escape the intended extraction directory (entries containing `../` or an absolute path).
```python
# BAD: archive entries are extracted using their raw, unchecked stored path
with zipfile.ZipFile(upload) as z:
    z.extractall(upload_dir)  # a "../../etc/cron.d/evil" entry escapes upload_dir

# GOOD: each entry's resolved path is verified to stay within the target directory
import os
with zipfile.ZipFile(upload) as z:
    for entry in z.infolist():
        target = os.path.realpath(os.path.join(upload_dir, entry.filename))
        if not target.startswith(os.path.realpath(upload_dir) + os.sep):
            raise ValueError(f"unsafe path in archive: {entry.filename}")
        z.extract(entry, upload_dir)
```
- **DO:** Set a restrictive `Content-Disposition` and `Content-Type` header (or force download rather than inline rendering) when serving user-uploaded files back, so a browser doesn't render an uploaded file as executable HTML/script content in the application's origin.

### Data Type, Length, and Format Validation

- **DO:** Validate not just format but reasonable bounds — string length maximums, numeric ranges, array/collection size limits, nesting depth for JSON payloads — on every field accepted from outside the trust boundary, since unbounded input is both a data-integrity risk and a resource-exhaustion (denial-of-service) risk.
- **DON'T:** Accept unbounded request payload sizes, unbounded array lengths in JSON bodies, or unbounded pagination parameters (`?limit=999999999`). A single request with no length cap can be used to exhaust memory, CPU, or database resources far more easily than a coordinated flood, and this class of resource-exhaustion bug rarely gets caught by testing with realistic-sized data.
- **DO:** Validate numeric input for the actual constraints of its domain (no negative quantities where negative doesn't make sense, no fractional counts where the value represents whole units) in addition to type-correctness, since a type check alone (`is this a number`) doesn't catch a semantically invalid value like `-5` items or a price of `0.00000001`.
- **DON'T:** Trust that type coercion in a dynamically typed language has quietly done the validation work already. A field accepted as `"123abc"` might coerce to `123` in one language's loose-comparison rules and to `NaN`/an error in another; validate the actual type and format explicitly rather than depending on implicit coercion behaving the way you expect.
- **DO:** Validate and normalize Unicode input deliberately when the field's meaning depends on exact string matching (usernames, filenames used as identifiers) — visually similar characters from different Unicode blocks (homoglyphs) can be used to create a username or domain that looks identical to an existing one but is a distinct string, a technique used in phishing and impersonation.

### Sanitization for the Right Context

- **DO:** Choose the sanitization approach that matches how the value will actually be used — an allowlist-based HTML sanitizer for rich text that will be rendered as markup, strict character-set validation for an identifier, canonicalization plus range checks for a numeric input — rather than applying one generic "clean this string" function everywhere.
- **DON'T:** Conflate "sanitized for storage" with "safe for every future use." A value that's safely stored in the database as an escaped string can still need output-context-specific encoding again when it's later rendered into HTML, embedded in a shell command, or included in an email — sanitize/encode at the point of use for that specific context, not only once on the way in.
- **DO:** Normalize input (trimming whitespace, normalizing Unicode forms, lowercasing where case-insensitivity is intended) *before* validating it against an allowlist, so that validation logic operates on a canonical form and can't be bypassed by an equivalent-but-differently-encoded variant of a disallowed value.
- **DON'T:** Sanitize input by silently stripping "bad" characters and continuing rather than rejecting the input outright, for anything security-relevant. Silently dropping unexpected characters can change a value's meaning without the caller realizing it, and in some cases (character removal itself) can even reintroduce an attack pattern that only appears after the offending characters are stripped out.

### Deserialization and Data-Format Risks

- **DO:** Use safe, restricted deserialization modes for any data format that supports it — JSON parsing with the platform's standard library (which does not execute arbitrary code) rather than a language-native serialization format's unrestricted deserializer for untrusted input.
- **DON'T:** Deserialize untrusted data using a format or library that can instantiate arbitrary classes or execute code as part of deserialization (native object serialization/pickling formats in several mainstream languages, unrestricted YAML loaders that support custom tag execution). These formats were designed for trusted, first-party data interchange, and deserializing attacker-controlled bytes with them is a well-documented path to remote code execution.
```python
# BAD: unrestricted deserialization of untrusted data can execute arbitrary code
import pickle
data = pickle.loads(untrusted_bytes)

# GOOD: use a data-only format with no code-execution capability for untrusted input
import json
data = json.loads(untrusted_bytes)
```
- **DO:** Use the "safe load" variant explicitly where a library offers both a safe and an unsafe deserialization mode (for example, a YAML library's safe loader versus its full loader), and confirm which mode is actually the default for the library version in use, since some libraries have changed their default over time.
- **DON'T:** Deserialize a request body directly into an internal domain/database model without an explicit schema step in between (see mass assignment below) — deserialization and validation are two different concerns, and skipping the validation step because "the deserializer already parsed it into an object" leaves type-correct but semantically invalid or unauthorized data flowing straight into business logic.

### Validation at API Boundaries and Mass Assignment

- **DO:** Define an explicit schema (a request-validation library, a typed DTO, a JSON Schema) for every API endpoint's expected input, and reject requests that don't conform — including rejecting unexpected extra fields the client shouldn't be sending, rather than silently ignoring or, worse, silently accepting them.
- **DON'T:** Bind an incoming request body directly onto a database model or ORM entity using an automatic "mass assignment" convenience feature without an explicit allowlist of which fields are bindable from client input. A user profile update endpoint that blindly assigns every JSON field onto the user record can let a client set fields it was never meant to control — `isAdmin`, `accountBalance`, `role` — simply by including them in the request body.
```javascript
// BAD: every field in the request body is assigned onto the model, including
// fields the client should never be able to set directly
const user = await User.findById(req.user.id);
Object.assign(user, req.body); // req.body could include { role: 'admin' }
await user.save();

// GOOD: only an explicit allowlist of client-editable fields is applied
const { displayName, bio } = req.body;
await User.updateOne({ _id: req.user.id }, { displayName, bio });
```
- **DO:** Keep the set of client-writable fields for any given endpoint deliberately narrow and explicitly enumerated, reviewing it whenever a new sensitive field (a role, a permission flag, a balance) is added to a model that also has a mass-updatable endpoint.
- **DON'T:** Trust a hidden form field, a disabled form field, or a value the client "shouldn't" be able to change as an implicit access control. Disabled and hidden fields are still present in the DOM and trivially re-enabled or resubmitted with a modified value by a client crafting its own request.
- **DO:** Validate query parameters, headers, and cookies with the same rigor as request bodies. It's common to see thorough body validation paired with an unvalidated `sort`, `filter`, or `X-Custom-Header` value that ends up driving a database query, a redirect, or a business decision without ever passing through the same validation layer.
- **DON'T:** Skip re-validating input on retry, webhook, or callback paths just because the original request was already validated. A payment webhook, an OAuth callback, or a retried background job receives its data over a separate channel and must be validated (and, for webhooks, authenticity-verified) independently — it did not inherit the trust of the original request that triggered it.
- **DO:** Validate the *semantic* relationship between fields, not just each field in isolation, where that relationship matters for correctness or security — a date range where "end" must not precede "start," a discount amount that cannot exceed the order total, a quantity that cannot exceed available stock. Field-level validation alone misses these cross-field invariants, which are often exactly where business-logic abuse happens.

### Webhook and Third-Party Callback Validation

- **DO:** Verify the authenticity of every inbound webhook using the sending service's documented signature-verification scheme (typically an HMAC signature over the raw request body, computed with a shared secret) before trusting or acting on its payload, since a webhook endpoint is, from the receiving server's point of view, just another public HTTP endpoint that happens to expect a particular payload shape.
```python
# BAD: the webhook payload is trusted and processed with no verification
# that it actually originated from the claimed third-party service
@app.route('/webhooks/payment', methods=['POST'])
def payment_webhook():
    event = request.get_json()
    process_payment_event(event)  # anyone who finds this URL can POST fake events

# GOOD: the signature header is verified against an HMAC computed with the
# shared webhook secret before the payload is trusted
@app.route('/webhooks/payment', methods=['POST'])
def payment_webhook():
    signature = request.headers.get('X-Signature')
    expected = hmac.new(WEBHOOK_SECRET, request.data, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(signature or '', expected):
        abort(401)
    process_payment_event(request.get_json())
```
- **DON'T:** Rely on a webhook URL being "hard to guess" as its only protection. An unguessable-looking URL is not authentication, and webhook URLs leak more easily than expected — through logs, browser history on a shared machine, or a misconfigured referrer — so signature verification is required regardless of how obscure the endpoint path is.
- **DO:** Validate that a webhook's payload content is internally consistent and within expected bounds (an amount matching what your system actually expects for that transaction, an event type from a known enumerated set) even after signature verification passes, since signature verification confirms *who* sent the payload, not that its *content* makes business sense for your system's current state.
- **DON'T:** Process a webhook payload more than once as if each delivery were guaranteed unique. Most webhook providers document at-least-once delivery semantics and can send duplicates; make webhook handlers idempotent (checking whether a given event ID has already been processed) so a duplicate delivery doesn't double-apply a sensitive effect like crediting an account twice.

### Preventing Regular-Expression Denial of Service (ReDoS)

- **DO:** Test any regular expression used to validate untrusted input against deliberately pathological inputs (long strings of repeated characters designed to trigger worst-case backtracking) as part of writing that validation rule, particularly for patterns with nested quantifiers or overlapping alternations.
- **DON'T:** Write a validation regex with nested repetition (`(a+)+`, `(a|a)*`, `([a-zA-Z]+)*`) against untrusted input without checking its worst-case time complexity. These patterns can exhibit catastrophic backtracking where a crafted input of a few dozen characters takes seconds or minutes to evaluate, tying up a request-handling thread/process and enabling a trivial denial-of-service with a single request.
- **DO:** Prefer a regex engine with guaranteed linear-time matching (available in some languages/libraries as an alternative engine) for patterns validating untrusted input at scale, or cap the input length aggressively before it ever reaches the regex engine, which bounds the worst-case cost even if the pattern itself has poor complexity characteristics.

### GraphQL and API Query Complexity Limits

- **DO:** Enforce query depth limits, complexity/cost scoring, and result-size limits on any API that lets a client shape its own query (GraphQL being the most common example, but this applies to any flexible filtering/query-building API), so a single request can't be crafted to demand an unreasonable amount of server-side computation or database work.
- **DON'T:** Expose an unrestricted, arbitrarily nestable filtering or query-building API to clients without validating the resulting query's cost before execution. A flexible query API is a convenience feature for legitimate clients and, without limits, a resource-exhaustion vector for anyone who sends a deliberately expensive query.

### Internationalization and Encoding Edge Cases

- **DO:** Decide on and enforce a canonical Unicode normalization form (typically NFC) for any input where consistent string comparison matters (usernames, search matching, deduplication), applying normalization *before* validation and storage so that visually or semantically equivalent inputs in different Unicode representations are treated consistently rather than as distinct values.
- **DON'T:** Assume input length limits measured in bytes behave the same as limits measured in characters when the input can include multi-byte Unicode characters (emoji, non-Latin scripts). A length check written assuming one byte per character can reject legitimate international input unexpectedly, or — more security-relevant — allow a stored value to exceed an intended storage column's actual byte capacity, causing truncation that changes the value's meaning after the fact.
- **DO:** Be aware of encoding-based bypass techniques against validation logic — double URL-encoding, overlong UTF-8 encodings of ASCII characters, and mixed-encoding tricks have historically been used to sneak a disallowed character past a validation filter that only checked the input's single, decoded form once. Validate after full, canonical decoding, and decode only once, explicitly, rather than leaving the number of decoding passes ambiguous.

### Numeric Input and Overflow Considerations

- **DO:** Validate numeric input against the actual representable range of the type it will be stored or computed as (a 32-bit integer column, a currency field with a defined precision), rejecting values that would overflow, underflow, or lose precision, rather than letting the underlying type silently wrap or truncate the value.
- **DON'T:** Assume a language's numeric type will raise an error on overflow by default — several mainstream languages silently wrap around on integer overflow rather than raising an exception, which can turn a large, validated-looking positive quantity into an unexpectedly negative or wrapped value at the point it's actually used, with consequences ranging from a display bug to a serious business-logic bypass (a wrapped negative "quantity" that ends up crediting an account instead of debiting it).
- **DO:** Apply extra scrutiny to numeric fields that flow into financial calculations, inventory counts, or rate-limiting counters specifically, since these are the numeric-validation failures with the most direct security/business impact if a boundary condition is missed.

### Validating Structured File Formats Beyond Uploads

- **DO:** Apply the same untrusted-input discipline to structured files the application parses even when they don't arrive via a traditional "file upload" endpoint — an imported CSV/spreadsheet, a configuration file uploaded through an admin panel, an SVG accepted as a profile image (SVG is XML and can embed scripts, making it its own XSS vector distinct from raster image formats).
- **DON'T:** Treat an SVG upload as "just an image" without recognizing it's actually an XML document capable of carrying embedded `<script>` tags and event handlers. Either rasterize SVG uploads server-side into a non-script-capable format, sanitize the SVG's XML content with a dedicated sanitizer before storage, or serve uploaded SVGs from a separate, cookie-less origin with a restrictive CSP, consistent with the general file-upload guidance above.
- **DO:** Validate CSV/spreadsheet imports against formula-injection risk ("CSV injection") — a cell value beginning with `=`, `+`, `-`, or `@` can be interpreted as a formula by spreadsheet software when the exported file is later opened, potentially triggering an external data-fetch or a malicious macro if the target application's formula execution isn't restricted. Prefix or escape leading formula-trigger characters in any user-controlled value written into a CSV/spreadsheet export.

### Schema Validation Across Languages

Defining an explicit, declarative schema for expected input — rather than validating fields with scattered, ad hoc `if` checks — makes validation rules reviewable in one place and consistently applied. Most language ecosystems have a standard or widely adopted library for exactly this purpose.

```typescript
// TypeScript: a schema library validates and types a request body in one step
const CreateOrderSchema = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().min(1).max(100),
  couponCode: z.string().max(32).optional(),
});
const input = CreateOrderSchema.parse(req.body); // throws on any invalid/extra field
```
```python
# Python: a data-validation library enforces types, bounds, and format at the boundary
class CreateOrder(BaseModel):
    product_id: UUID
    quantity: int = Field(ge=1, le=100)
    coupon_code: Optional[str] = Field(default=None, max_length=32)

order = CreateOrder.model_validate(request_json)  # raises ValidationError on bad input
```
```java
// Java: bean-validation annotations declare constraints directly on the DTO
public class CreateOrderRequest {
    @NotNull private UUID productId;
    @Min(1) @Max(100) private int quantity;
    @Size(max = 32) private String couponCode;
}
// the framework's validation layer rejects a non-conforming request automatically
```
- **DO:** Reject the entire request when schema validation fails, returning a clear (but not internals-revealing) error, rather than silently coercing an invalid field to a default value and proceeding — silent coercion can mask a client-side bug or an attacker's probing attempt as if it were valid input.
- **DON'T:** Let a schema definition drift out of sync with what the handler actually does with the data. A schema that validates a field but whose handler code still performs its own inconsistent, separate check on the same field is a sign the two have diverged — treat the schema as the single source of truth for that endpoint's input contract.

### Denial of Service via Algorithmic Complexity

- **DO:** Be aware that data structures with a documented worst-case degenerate behavior (a hash table vulnerable to "hash flooding" when an attacker can choose many keys that collide under the implementation's hash function, prior to modern languages' widespread adoption of randomized hash seeding) can turn a normally fast operation into a slow one when fed adversarially chosen input, and keep the language runtime and any custom hash-table implementation current, since this specific class of issue has largely been mitigated by default hash-randomization in modern language runtimes but can resurface in a custom or older implementation.
- **DON'T:** Build a custom hashing or indexing data structure for untrusted-keyed data without considering whether an adversary who can choose many keys could degrade it to worst-case (often quadratic) performance. Where the standard library's built-in, randomized-by-default hash table is available, prefer it over a custom implementation for exactly this reason.
- **DO:** Cap the number of distinct keys/fields accepted in a request body that will be used to populate a hash table, dictionary, or similar structure (a JSON object with attacker-controlled keys), consistent with the general input-size-bounding guidance above, which limits the practical impact of any algorithmic-complexity concern regardless of the underlying data structure's worst case.

### Second-Order Injection Illustrated

- **DO:** Trace where a value that was safely parameterized on its way *into* storage is later read back out and used to build a *new* query, command, or markup fragment, since the safety of the original write says nothing about the safety of a later read-then-reuse — this is what makes second-order injection easy to miss in review focused only on the immediately visible write path.
```python
# BAD: the display-name value was safely parameterized on the way into the
# database at signup, but is later read back and concatenated into a new,
# unparameterized query when generating an internal report — the original
# parameterization did nothing to protect this second, later use
def create_user(display_name):
    db.execute("INSERT INTO users (display_name) VALUES (%s)", (display_name,))

def generate_activity_report(user_id):
    name = db.execute("SELECT display_name FROM users WHERE id = %s", (user_id,)).fetchone()[0]
    db.execute(f"SELECT * FROM activity_log WHERE actor_name = '{name}'")  # reintroduces injection

# GOOD: the second query is parameterized too — every query gets its own
# parameterization, regardless of where the value originally came from
def generate_activity_report(user_id):
    name = db.execute("SELECT display_name FROM users WHERE id = %s", (user_id,)).fetchone()[0]
    db.execute("SELECT * FROM activity_log WHERE actor_name = %s", (name,))
```
- **DON'T:** Treat "this value was already validated/parameterized once" as a property that travels with the value forever. Safety from injection is a property of *each individual query or command construction*, not a property that a value carries with it after being safely stored once — every place a stored value is used to build new executable syntax needs its own parameterization, independent of how it was originally written.

### Envelope Encryption Worked Example

- **DO:** Understand the envelope-encryption pattern concretely: a fast, randomly generated data encryption key (DEK) encrypts the actual data locally, and that DEK is itself encrypted by a more tightly controlled key encryption key (KEK) held in a KMS/secret manager — only the encrypted DEK needs to be stored alongside the data, while the KEK never leaves the KMS boundary at all, which is both faster than encrypting large data volumes directly through a network-bound KMS call and limits what a compromised application ever has direct access to.
```python
# GOOD: envelope encryption — a per-record data key is generated locally,
# used to encrypt the data, and then itself encrypted by a KMS-held key;
# only the encrypted data key is stored, never the plaintext data key
data_key_plaintext, data_key_encrypted = kms_client.generate_data_key(key_id=KEK_ID)

ciphertext = aes_gcm_encrypt(plaintext_record, key=data_key_plaintext, nonce=os.urandom(12))
store_record(ciphertext=ciphertext, encrypted_data_key=data_key_encrypted)

# to read it back later, the KMS decrypts the stored data key, which is
# then used locally to decrypt the record — the KEK itself never leaves the KMS
data_key_plaintext = kms_client.decrypt(data_key_encrypted)
plaintext_record = aes_gcm_decrypt(ciphertext, key=data_key_plaintext)
```
- **DON'T:** Call out to the KMS/secret manager for every single record's direct encryption/decryption in a high-throughput system, when envelope encryption's local-DEK pattern would avoid that per-record network round trip while still keeping the root key (the KEK) centrally controlled and rotatable.
- **DO:** Rotate the KEK periodically without needing to re-encrypt every existing record's data — only the (small) encrypted data keys need to be re-wrapped under the new KEK, since the DEKs themselves, and the data they protect, don't change; this is one of the practical advantages envelope encryption offers over encrypting every record directly with a single long-lived key.

## Cryptography Basics

Cryptography is one of the few areas of software engineering where "good enough" intuition reliably produces broken results, because cryptographic weaknesses are often invisible under normal testing and only surface under deliberate attack. The rules here are less about specific algorithms (which shift over time) and more about a durable habit: use established, peer-reviewed libraries and configurations, and treat any custom cryptographic design as a near-automatic red flag.

### Use Vetted Libraries, Never Custom Crypto

- **DO:** Use a well-established, actively maintained cryptography library (your language ecosystem's standard TLS/crypto library, or a widely audited high-level library built on top of one) for every cryptographic operation — encryption, hashing, signing, key derivation, random generation — rather than implementing any cryptographic primitive or protocol from scratch.
- **DON'T:** Write a custom encryption scheme, a custom hashing routine, a custom "obfuscation" function pretending to be encryption, or a home-grown key-exchange protocol, no matter how simple the use case seems. Cryptography is uniquely unforgiving of subtle mistakes — small implementation errors (a reused nonce, a missing authentication tag, a non-constant-time comparison) can silently destroy the security guarantee while the code appears to work perfectly under normal testing, since broken crypto usually still produces plausible-looking ciphertext.
- **DO:** Prefer a high-level, opinionated cryptography library or API (one that makes safe choices — like an authenticated encryption mode — the default and the easy path) over a low-level primitives library that requires the caller to correctly assemble mode, padding, IV/nonce handling, and authentication by hand. The more decisions a library leaves to the caller, the more chances there are to get one of them wrong.
- **DON'T:** "Roll your own" even for something that feels too small to be real cryptography — a custom checksum used as an integrity check for security purposes, a homemade token-signing scheme, a hand-built encoding meant to hide (not just encode) a value. If the goal is confidentiality, integrity, or authenticity, it's a cryptographic problem and belongs to a vetted library, not a bespoke function.
- **DO:** Trust your cryptography library's documented, high-level API for a task (e.g., "encrypt this file," "sign this token") over composing lower-level primitives yourself, and consult the library's own recommended patterns rather than a decade-old blog post or a language model's memorized-but-possibly-outdated example when unsure which mode or parameters to use.

### Algorithm and Mode Choices

- **DO:** Use an authenticated encryption mode — AES-GCM, ChaCha20-Poly1305, or an equivalent AEAD (Authenticated Encryption with Associated Data) construction — for symmetric encryption, which provides confidentiality and integrity/authenticity together in one well-tested construction.
- **DON'T:** Use AES in ECB (Electronic Codebook) mode for anything. ECB encrypts identical plaintext blocks to identical ciphertext blocks with no chaining, which means structural patterns in the plaintext (most famously demonstrated by encrypting an image and still being able to make out its shapes in the ciphertext) remain visible; it provides essentially no real confidentiality guarantee for structured data.
```
BAD:  AES/ECB/PKCS5Padding — identical plaintext blocks produce identical
      ciphertext blocks, leaking structural patterns in the data.

GOOD: AES/GCM/NoPadding (or an equivalent AEAD mode) with a unique,
      never-reused nonce per encryption operation and an authentication tag
      verified on decryption.
```
- **DON'T:** Use unauthenticated modes like plain CBC without a separate, correctly implemented MAC. CBC without authentication is vulnerable to padding-oracle and bit-flipping attacks that let an attacker modify ciphertext in predictable, exploitable ways; if a library's only offered symmetric option is unauthenticated CBC, add a properly implemented encrypt-then-MAC construction, but prefer an AEAD mode that handles this correctly by default instead.
- **DO:** Use SHA-256, SHA-3, or another current-generation cryptographic hash function for general-purpose integrity/fingerprinting needs, and reserve slow, memory-hard KDFs (Argon2id, bcrypt, scrypt, PBKDF2 with a high iteration count) specifically for password/credential storage and key derivation from low-entropy secrets — these are different tools for different jobs and are not interchangeable.
- **DON'T:** Use MD5 or SHA-1 for any security-relevant purpose (password hashing, digital signatures, integrity verification of security-sensitive data, certificate fingerprinting). Both have practical, demonstrated collision weaknesses; MD5 in particular is thoroughly broken for anything beyond a non-security checksum, such as detecting accidental (not adversarial) data corruption.
- **DO:** Use RSA with a key size of at least 2048 bits (3072+ for longer-term protection) or, preferably, an elliptic-curve algorithm (Ed25519 for signatures, X25519 for key exchange, or NIST P-256 where ecosystem compatibility requires it) for asymmetric operations, since elliptic-curve algorithms generally offer equivalent security with smaller keys and better performance.
- **DON'T:** Use a deprecated or clearly weak cipher/algorithm because it's what a decade-old tutorial, an older training corpus, or existing legacy code demonstrates — DES, 3DES, RC4, and 1024-bit RSA are all considered inadequate for current security needs. Cryptographic recommendations shift as computing power and cryptanalysis advance; check current guidance from an authoritative, actively maintained source rather than assuming what was standard years ago still is.
- **DO:** Choose key sizes and algorithm parameters based on current guidance from an authoritative source (a national standards body, the library's own current documentation) appropriate to the data's required protection lifetime — data that must stay confidential for decades needs a more conservative parameter choice than data with a short useful life.

### Secure Random Number Generation

- **DO:** Use a cryptographically secure pseudo-random number generator (CSPRNG) — the operating system's secure random source, or the language's dedicated "secure random"/"crypto random" API — for anything security-relevant: session tokens, password-reset tokens, API keys, encryption keys/nonces/IVs, CSRF tokens.
```python
# BAD: a general-purpose PRNG is not designed to resist prediction and is
# unsuitable for anything security-sensitive
import random
token = ''.join(random.choice(chars) for _ in range(32))

# GOOD: a cryptographically secure source is used for a security-sensitive token
import secrets
token = secrets.token_urlsafe(32)
```
- **DON'T:** Use a language's default/general-purpose random number generator (`Math.random()`, `random.random()`, a basic linear congruential generator) for anything security-sensitive. General-purpose PRNGs are optimized for statistical distribution and speed, not unpredictability against an adversary — many are seeded and advance in ways that make their future or past output predictable once even a little output is observed, which is fatal for a token meant to be unguessable.
- **DO:** Generate nonces/IVs for encryption using the CSPRNG each time, and never reuse a nonce with the same key for an AEAD cipher like AES-GCM — nonce reuse under the same key catastrophically breaks both the confidentiality and the authenticity guarantees of most AEAD modes, in some cases fully revealing the authentication key.
- **DON'T:** Derive a "random-looking" value from predictable inputs (the current timestamp, a counter, a hash of the username) and treat it as unguessable. Anything derived deterministically from predictable or attacker-knowable inputs is not random in the cryptographic sense, even if it looks random to a human glancing at it.
- **DO:** Ensure sufficient entropy/length for security tokens — 128 bits of randomness is a reasonable general baseline for session tokens and similar values; a 6-digit numeric code is appropriate only for a short-lived, rate-limited, single-use context like an MFA code, never as a long-lived secret.

### Key Management Basics

- **DO:** Separate encryption keys from the data they protect — store keys in a dedicated key-management service or secret manager, never alongside the encrypted data in the same database row, the same file, or the same backup archive. Keeping the key and the ciphertext together defeats much of the purpose of encrypting in the first place, since compromising one source compromises both.
- **DON'T:** Hardcode an encryption key or a signing key in source code, a configuration file committed to version control, or a mobile/client application bundle. The same reasoning that rules out hardcoded passwords and API keys applies with even more force to cryptographic keys, since a leaked key can retroactively decrypt every message/file ever protected with it.
- **DO:** Use envelope encryption (encrypt data with a data key, then encrypt that data key with a separate, more tightly controlled master/root key managed by a KMS) for systems handling significant volumes of sensitive data, which limits how much data any single compromised key can expose and makes key rotation more practical.
- **DON'T:** Use the same key for multiple unrelated purposes (the same key for both encrypting data and signing tokens, or the same key shared across every tenant in a multi-tenant system). Key reuse across purposes and across trust boundaries increases the impact of a single key compromise and can, for some algorithm/mode combinations, actively weaken the cryptographic guarantees.
- **DO:** Plan for key rotation from the start — support decrypting data encrypted under a previous key version while new data is encrypted under the current one, using a key-version identifier stored alongside the ciphertext, so rotating a key doesn't require re-encrypting the entire existing dataset synchronously.
- **DON'T:** Leave a compromised or suspected-compromised key in active use while "figuring out the rotation plan." Revoke/rotate immediately upon suspected compromise, accepting the operational cost, since every moment a compromised key remains valid is a moment an attacker can use it.
- **DO:** Restrict access to key-management operations (create, use, rotate, delete a key) with the same least-privilege discipline as any other sensitive system, and audit-log key usage so unusual access patterns (a service suddenly decrypting far more data than its normal workload) are visible.

### Encoding Is Not Encryption

- **DO:** Understand and communicate the distinction clearly on the team: Base64, hex encoding, and URL encoding are reversible, non-secret representations meant for safely transporting binary data through text-based channels — they provide zero confidentiality. Encryption, by contrast, requires a secret key to reverse.
- **DON'T:** Base64-encode a sensitive value (a password, a secret, an ID meant to be unguessable) and treat that encoding as if it provided any security. Anyone who receives the encoded value can decode it in one line of code with no key required; encoding is not a substitute for encryption when confidentiality is the actual goal.
```
BAD:  storing a "secret" as base64("real-secret-value") and treating the
      encoding itself as protection — it decodes trivially with no key.

GOOD: encrypt the value with an AEAD cipher and a properly managed key if
      confidentiality is actually required; use base64 only afterward, purely
      to make the resulting binary ciphertext safe to transport as text.
```
- **DO:** Reserve encoding functions for their actual purpose — making binary data safe to include in text-based formats (JSON, URLs, headers) — and reach for encryption, hashing, or signing (as appropriate to the actual security goal) when the requirement is confidentiality, integrity, or authenticity.

### Digital Signatures and Message Authentication

- **DO:** Use HMAC (with a strong underlying hash like SHA-256) to verify the integrity and authenticity of a message when both parties share a secret key (webhook payload verification, signed cookies), and use asymmetric digital signatures (Ed25519, ECDSA, RSA-PSS) when the verifier should not hold the same secret the signer used.
- **DON'T:** Verify a webhook, a signed request, or any other externally supplied message by comparing a computed signature to the provided one using a standard (non-constant-time) equality check. As with password/token comparisons, use a constant-time comparison function to avoid leaking timing information that could help an attacker forge a valid signature byte-by-byte.
- **DO:** Include and verify a timestamp or nonce as part of any signed request/webhook scheme, and reject requests outside an acceptable time window, to prevent replay attacks where a captured, validly signed request is resent later to repeat its effect.

### Data at Rest and in Transit

- **DO:** Encrypt genuinely sensitive data at rest (fields like government identifiers, payment details not otherwise tokenized, health information) at the application or database-column level, in addition to whole-disk/volume encryption, when that data's sensitivity or a compliance requirement calls for defense beyond physical-media protection.
- **DON'T:** Assume "the disk is encrypted" (cloud provider default disk encryption, full-disk encryption on a server) fully addresses data-at-rest risk for highly sensitive fields. Disk-level encryption protects against physical theft of storage media; it does nothing against an application-level vulnerability (SQL injection, an overprivileged credential, a misconfigured backup export) that reads the data through the normal application/database access path.
- **DO:** Apply tokenization (replacing a sensitive value with a non-sensitive reference/token that maps back to the real value only within a tightly controlled vault) for especially high-risk data like payment card numbers, which both reduces the scope of systems that need to handle the real sensitive value and is often required by relevant compliance frameworks.
- **DON'T:** Roll a custom encryption-at-rest scheme for a specific data field "since it's simple" instead of using the database's or platform's supported column/field-level encryption feature, or a maintained library built for exactly this purpose. This is the same "no custom crypto" principle applied to a narrower, easy-to-underestimate case.
- **DO:** Distinguish deterministic encryption (same plaintext always produces the same ciphertext, useful when the field must remain searchable/joinable) from randomized encryption (the default, safer choice for most fields) and only choose deterministic encryption deliberately, since it leaks equality patterns — an attacker who sees the ciphertext can tell which rows share the same underlying value even without decrypting any of them.
- **DON'T:** Forget that backups, replicas, and data exports need the same encryption and access-control posture as the primary data store. A perfectly encrypted production database whose nightly backup is written unencrypted to a broadly accessible storage bucket has simply moved the exposure, not closed it.

### Password-Based Key Derivation

- **DO:** Use a dedicated password-based key-derivation function (Argon2id, scrypt, or PBKDF2 with a high iteration count appropriate to current hardware) when a cryptographic key must be derived from a human-chosen password or passphrase — for example, encrypting a local file with a user-supplied passphrase. This is the same class of slow, memory-hard construction used for password storage, applied here to compensate for the relatively low entropy of human-chosen passwords.
- **DON'T:** Use a fast general-purpose hash (SHA-256 alone, MD5) to turn a password directly into an encryption key. A fast hash lets an attacker who obtains the ciphertext brute-force candidate passwords at high speed, exactly as with password storage — the same slow-KDF reasoning applies whether the output is used to verify a login or to derive a symmetric key.
- **DO:** Use a properly randomly generated key (from a CSPRNG, stored via proper key management) rather than a password-derived key for any application-level encryption need where the key doesn't have to be something a human remembers — password-based key derivation is specifically for the case where a human passphrase is unavoidably the only available secret.

### Certificate and Public-Key Pinning

- **DO:** Consider certificate or public-key pinning for mobile applications or other high-value clients communicating with a known, fixed backend, which protects against a compromised or coerced certificate authority issuing a fraudulent certificate for your domain that would otherwise pass normal validation.
- **DON'T:** Pin without a rotation/update mechanism baked in from the start. A pinned certificate or key that expires or is rotated on the server side, with no corresponding update mechanism for already-deployed clients, can lock out every existing client from connecting at all — pinning failures are a well-documented cause of app-breaking outages when done without a backup pin and an update path.
- **DO:** Pin to a public key or a CA's intermediate certificate rather than to a specific leaf certificate where possible, and include at least one backup pin, so a routine certificate renewal doesn't require an emergency client update to avoid breaking connectivity.

### Common Implementation Pitfalls to Watch For

- **DO:** Double-check that an "encrypted" value is actually authenticated too (AEAD mode, or encrypt-then-MAC) rather than encryption-only, since ciphertext without integrity protection can often be modified by an attacker in ways that produce predictable, exploitable changes to the decrypted plaintext even without knowing the key.
- **DON'T:** Assume that because a value "looks encrypted" (high-entropy-looking bytes, base64 output), the implementation is actually secure. A broken implementation — a reused nonce, a static IV, a missing authentication tag, a weak key derivation — can produce output that is visually indistinguishable from a correct implementation while providing dramatically weaker real-world protection; this is exactly why a vetted library's tested implementation matters more than the output "looking right."
- **DO:** Keep cryptographic library dependencies patched with the same urgency as any other security-critical dependency — cryptographic libraries occasionally have implementation-level vulnerabilities discovered in them (a timing side channel, a flawed random-number generator) independent of the algorithm's own theoretical security.
- **DON'T:** Implement your own timing-safe comparison, padding scheme, or random-number generator "since the library's version seems slow" or "to avoid a dependency." Performance and dependency-avoidance are not good enough reasons to reimplement code whose entire value proposition is having been reviewed by cryptography specialists for exactly the kinds of subtle mistakes a well-intentioned reimplementation is likely to make.

### Code Signing and Release Integrity

- **DO:** Sign software releases and build artifacts (packages, container images, installers) with a properly managed signing key, and verify signatures on the consuming/deployment side, so that a tampered-with or substituted artifact is detectable before it's installed or deployed rather than trusted implicitly because it came from the expected-looking location.
- **DON'T:** Treat "downloaded from the official-looking URL" as equivalent to "verified authentic." URLs and even entire distribution channels can be compromised or spoofed; a cryptographic signature check against a key obtained through a trusted, separate channel is what actually establishes the artifact's authenticity, not the URL it was fetched from.
- **DO:** Protect release-signing keys with the same rigor as any other highly sensitive key — hardware security module (HSM) storage or an equivalent managed key-signing service where feasible, restricted access, and a documented, auditable signing process — since a compromised release-signing key lets an attacker distribute malicious software that appears fully legitimate to every downstream verifier.

### Random Token Format and Entropy Considerations

- **DO:** Choose a token encoding (hex, base64url, a custom alphabet) primarily for how the token will be transported (URL-safe characters for a value that appears in a URL, for instance) while keeping the underlying entropy — the actual number of random bits generated — as the real security parameter, not the encoded string's visible length.
- **DON'T:** Confuse a token's displayed character length with its actual entropy. A 32-character token encoded in a low-information-density format (e.g., only decimal digits) carries meaningfully less entropy than a 32-character token in a higher-density encoding (base64url); when a specific entropy target matters (128 bits, say), calculate the required encoded length from the target entropy and the encoding's bits-per-character, rather than picking a round character count and assuming it's sufficient.

### Cryptography and Secure Randomness Across Languages

```python
# Python: secrets module for CSPRNG tokens; cryptography library for AEAD encryption
import secrets
token = secrets.token_urlsafe(32)
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
key = AESGCM.generate_key(bit_length=256)
aesgcm = AESGCM(key)
nonce = secrets.token_bytes(12)  # unique per encryption
ciphertext = aesgcm.encrypt(nonce, plaintext, associated_data=None)
```
```javascript
// Node.js: the built-in crypto module for both CSPRNG tokens and AEAD encryption
const crypto = require('crypto');
const token = crypto.randomBytes(32).toString('base64url');
const key = crypto.randomBytes(32);
const nonce = crypto.randomBytes(12);
const cipher = crypto.createCipheriv('aes-256-gcm', key, nonce);
```
```java
// Java: SecureRandom for CSPRNG tokens; javax.crypto for AES-GCM
SecureRandom random = new SecureRandom();
byte[] tokenBytes = new byte[32];
random.nextBytes(tokenBytes);
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
cipher.init(Cipher.ENCRYPT_MODE, secretKey, new GCMParameterSpec(128, nonce));
```
```go
// Go: crypto/rand for CSPRNG tokens; crypto/aes + crypto/cipher for AES-GCM
token := make([]byte, 32)
_, err := rand.Read(token) // crypto/rand, not math/rand
block, _ := aes.NewCipher(key)
gcm, _ := cipher.NewGCM(block)
ciphertext := gcm.Seal(nil, nonce, plaintext, nil)
```
- **DO:** Notice the specific module name in each language that indicates a *cryptographically secure* random source (`secrets`/`crypto` in Python/Node, `SecureRandom` in Java, `crypto/rand` in Go) as distinct from that language's general-purpose random module (`random` in Python, `math/rand` in Go) — the naming is deliberately parallel across most ecosystems precisely to make the secure choice discoverable, but it's easy to import the wrong one by habit or autocomplete.
- **DON'T:** Let an unfamiliarity with a given language's specific cryptography API lead to skipping encryption/secure-random requirements or improvising a workaround. Every mainstream language has a standard-library or extremely widely adopted vetted option for both CSPRNG generation and authenticated encryption; look up the current recommended approach for the specific language rather than reimplementing one.

### Clarifying Encoding, Hashing, Encryption, and Signing

- **DO:** Keep these four operations conceptually distinct when choosing one for a task, since confusing them is a recurring source of security mistakes: encoding (Base64, hex, URL-encoding) is a reversible, keyless representation change with no security property at all; hashing is a one-way function producing a fixed-size fingerprint, used for integrity checks and (with a slow, salted variant) password storage; encryption is reversible only with a secret key, used for confidentiality; and signing uses a key pair to prove authenticity and integrity without providing confidentiality on its own.
- **DON'T:** Reach for whichever of the four "sounds about right" for a given requirement without checking that its actual guarantee matches the actual need. Needing confidentiality but choosing encoding provides none; needing authenticity but choosing plain hashing (with no secret key involved) provides none, since anyone can compute the same hash over a forged message; matching the operation's actual guarantee to the actual requirement is the entire exercise.
- **DO:** State explicitly, in code review of anything security-relevant, which of the four properties (confidentiality, integrity, authenticity, or none — "just a reversible representation change") a given operation is meant to provide, since naming the intended guarantee out loud is often what surfaces a mismatch between what a developer intended and what the chosen operation actually does.

### Backup and Disaster Recovery Security

- **DO:** Encrypt backups with the same rigor as the primary data store they're backing up, and store backup encryption keys separately from the backups themselves (following the same key/data separation principle covered under Cryptography Basics above), so that access to the backup storage location alone doesn't grant access to its decrypted content.
- **DON'T:** Assume backups are automatically covered by the same access controls and monitoring as production data. Backup storage is frequently a separate system (a different cloud storage bucket, a third-party backup service, offline tape) with its own access-control configuration, and it's a common gap for that configuration to be more permissive than the production system it's backing up, simply because it received less security review.
- **DO:** Maintain at least one backup copy that's immutable or offline/air-gapped (not directly writable or deletable by the same credentials that operate the production system) specifically as ransomware resilience — if an attacker who compromises production credentials can also delete or encrypt every reachable backup, backups provide no actual recovery guarantee against that specific threat.
- **DON'T:** Let backup-restoration credentials be as broadly held as day-to-day operational credentials. Restoring a backup is a powerful, high-blast-radius operation (it can overwrite current production data); scope who can trigger a restore as tightly as any other highly sensitive administrative action, and log every restore operation to the audit trail.
- **DO:** Test the actual restore process periodically, not just the backup-creation process, and treat that test as also verifying the security properties (are restored records still properly encrypted, do restored access-control settings match what's expected) rather than only verifying that data comes back.

### Compliance-Driven Cryptography Requirements

- **DO:** Identify early whether the application operates in a context with mandated cryptographic standards (a government or regulated-industry requirement for validated cryptographic modules, for instance), since meeting that requirement can mean using a specific certified build or mode of a cryptography library rather than simply "any modern, vetted library" — check the applicable requirement before assuming a general best-practice library choice automatically satisfies it.
- **DON'T:** Assume a general recommendation to "use a vetted library" automatically satisfies a specific regulatory or contractual cryptographic requirement. Some environments require cryptographic operations to run through a specifically validated module or in a specific certified mode; verify the actual requirement with whoever owns that compliance obligation rather than guessing.

### Cryptographic Agility and Forward-Looking Migration

- **DO:** Design systems for cryptographic agility — the ability to swap out an algorithm or key size in the future without a ground-up rewrite — by keeping the algorithm/parameters used for a given piece of encrypted or signed data recorded alongside it (a version/algorithm identifier stored with the ciphertext), rather than hardcoding an implicit assumption that today's algorithm choice is permanent.
- **DON'T:** Assume today's recommended algorithms and key sizes will remain adequate indefinitely. Cryptographic recommendations have shifted meaningfully over time as computing power grows and new cryptanalysis techniques emerge (including the long-term, actively researched risk that sufficiently capable quantum computers would eventually break widely deployed public-key algorithms) — data with a long required confidentiality lifetime deserves periodic reassessment against current guidance, not a one-time algorithm choice made once and never revisited.
- **DO:** Track emerging guidance on post-quantum-resistant algorithms for systems protecting data that must remain confidential for a very long time, since data encrypted today with a currently-strong but not quantum-resistant algorithm could in principle be captured and stored now for decryption later if quantum-capable cryptanalysis becomes practical — a risk specifically relevant to long-lived, highly sensitive data, not to typical short-lived application data.

## Transport Security

### Datastore and Cache Network Exposure

- **DO:** Require authentication on every network-reachable datastore and cache — the database, Redis, Elasticsearch, MongoDB, Memcached — even when it's only intended to be reached from an internal network, and bind it to a private network interface rather than a publicly routable one wherever the deployment topology allows it.
- **DON'T:** Deploy a datastore with authentication disabled "since it's only accessed internally." A striking number of real-world breaches have traced back to exactly this configuration — a database or cache left with no authentication required, reachable because a network boundary assumed to be private turned out to be reachable from the broader internet (a misconfigured cloud security group, a container port accidentally published) — and several of these datastores ship with authentication disabled by default specifically for ease of local development, a default that needs to be explicitly overridden before any non-local deployment.
- **DO:** Verify a new datastore or cache's actual network reachability after deployment (attempt a connection from outside the intended trust boundary) rather than assuming the deployment configuration achieved what was intended — configuration mistakes that leave a supposedly internal-only service internet-reachable are common precisely because the mistake produces no visible symptom until someone (hopefully the deploying team, not an attacker) actually checks.

Transport security protects data while it moves between parties — browser to server, service to service, server to third-party API. Getting this layer right is largely a matter of using TLS consistently, keeping its configuration current, and never treating a certificate-validation error as an obstacle to silence rather than a signal to investigate.

### TLS Everywhere

- **DO:** Serve every part of an application over HTTPS/TLS — not just login and payment pages — including static assets, internal admin panels, health-check endpoints, and internal service-to-service traffic wherever it crosses a network boundary that isn't fully trusted. Partial HTTPS coverage ("only the login page is encrypted") leaves session cookies and other sensitive data exposed on every other page load.
- **DON'T:** Serve any part of a production application over plain HTTP, even "temporarily" or for a supposedly low-sensitivity page. An HTTP page on the same domain as an HTTPS-protected session can still leak the session cookie (if cookie scoping isn't airtight) and gives a network attacker on the same network a foothold to inject content or redirect the user.
- **DO:** Redirect all HTTP requests to HTTPS at the server/load-balancer level (a 301/308 redirect from `http://` to `https://`) so that a user who types a bare domain or follows an old `http://` link is upgraded automatically rather than staying on an unencrypted connection.
- **DON'T:** Rely solely on an HTTP-to-HTTPS redirect as the only protection against protocol downgrade. The very first request before the redirect is sent in plaintext and can be intercepted or manipulated by a network-level attacker (a classic SSL-stripping scenario); pair the redirect with HSTS (below) so browsers stop making the initial plaintext request at all after the first visit.

### HTTP Strict Transport Security (HSTS)

- **DO:** Send the `Strict-Transport-Security` header on every HTTPS response, instructing browsers to only ever connect to the domain over HTTPS for a specified duration, which closes the SSL-stripping window left open by relying on redirects alone.
```
# GOOD: HSTS enabled with a meaningful max-age, applied to subdomains,
# and eligible for browser preload lists once verified stable
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```
- **DON'T:** Set an HSTS `max-age` so short it provides negligible protection, or omit `includeSubDomains` when every subdomain is also meant to be HTTPS-only. A very short max-age re-opens the downgrade window as soon as it expires; leaving subdomains uncovered lets an attacker who can influence DNS or a subdomain redirect a user to an unprotected HTTP endpoint on a sibling subdomain.
- **DO:** Test HSTS thoroughly in a staging environment before submitting a domain to browser preload lists, since preload inclusion is effectively permanent for a long time and difficult to reverse quickly if a legitimate need for HTTP access on some subdomain is later discovered.
- **DON'T:** Enable HSTS with `includeSubDomains` on a domain where some subdomain genuinely cannot yet serve HTTPS (a legacy system, a third-party-hosted subdomain). Doing so breaks that subdomain for every visiting browser until it also supports HTTPS; audit subdomain coverage first.

### Certificate Validation

- **DO:** Let TLS libraries perform their default, full certificate validation (chain of trust, expiration, hostname matching, revocation where checked) on every outbound HTTPS connection the application makes, including calls to internal services and third-party APIs, not only user-facing traffic.
- **DON'T:** Disable certificate verification (`verify=False`, `rejectUnauthorized: false`, `-k`/`--insecure` flags, a custom trust manager that accepts anything) to make a TLS error go away, ever, in code that will run anywhere near production — including "just for local development" code that has a track record of accidentally shipping unchanged. Disabling verification removes protection against man-in-the-middle attacks entirely; a client with verification off will happily talk to an attacker impersonating the real server.
```python
# BAD: certificate verification disabled to silence an SSL error
response = requests.get(url, verify=False)

# GOOD: the actual root cause is fixed — e.g., the correct CA bundle is supplied
# for a private/internal CA, or the client's system trust store is updated —
# instead of turning verification off
response = requests.get(url, verify="/etc/ssl/certs/internal-ca-bundle.pem")
```
- **DO:** Diagnose the actual cause of a certificate error (an expired certificate, a hostname mismatch, a missing intermediate certificate, an internal CA not present in the trust store) and fix that specific problem — renew the cert, correct the hostname, add the intermediate to the chain, or supply the internal CA bundle explicitly to the client — rather than disabling the check that surfaced it.
- **DON'T:** Accept a self-signed certificate in production code paths without pinning that exact certificate/key deliberately as a conscious architectural decision (for a closed, controlled internal system) — accepting "any" self-signed certificate is functionally identical to disabling verification, since an attacker's self-signed certificate is accepted just as readily as the legitimate one.
- **DO:** Keep the CA trust store and TLS library up to date, since expired or compromised root/intermediate certificate authorities are periodically distrusted, and an outdated trust store can either wrongly trust a revoked authority or wrongly reject a legitimately reissued certificate chain.

### Mixed Content and Certificate Configuration

- **DO:** Ensure every subresource loaded by an HTTPS page — scripts, stylesheets, images, iframes, API calls — is also loaded over HTTPS. Mixed content (an HTTPS page loading an HTTP subresource) either gets blocked by modern browsers (breaking functionality) or, for older/passive mixed content still allowed, gives a network attacker a way to tamper with page behavior despite the page itself being served over HTTPS.
- **DON'T:** Hardcode `http://` URLs for internal asset references, third-party embeds, or API base URLs. Use protocol-relative or explicitly `https://` URLs, and audit third-party embeds/widgets periodically, since a vendor can silently degrade their own endpoint from HTTPS to HTTP-only without your team noticing until a browser starts blocking it.
- **DO:** Configure the TLS server (or the reverse proxy/load balancer terminating TLS) to disable outdated protocol versions (SSLv2, SSLv3, TLS 1.0, TLS 1.1) and weak cipher suites, keeping only TLS 1.2+ with modern, forward-secrecy-providing cipher suites enabled.
- **DON'T:** Leave legacy TLS versions or export-grade/weak ciphers enabled "for compatibility with old clients" without a deliberate, time-boxed exception. Legacy protocol versions have known cryptographic weaknesses, and the compatibility need should be weighed explicitly against the security cost rather than left as an unexamined default.
- **DO:** Use mutual TLS (mTLS) for particularly sensitive service-to-service communication (internal microservices handling financial or highly regulated data) where both client and server authenticate each other with certificates, providing stronger assurance than a bearer token alone that both endpoints of the connection are legitimate.
- **DON'T:** Assume traffic that never leaves a cloud provider's internal network, or stays within a VPC/private network, needs no encryption "because it's already private." Internal network segmentation reduces exposure but does not guarantee no other tenant, compromised host, or misconfigured route can observe that traffic; encrypting internal traffic is a reasonable default for anything handling sensitive data, consistent with a zero-trust posture.
- **DO:** Verify hostnames and certificates on outbound calls made from server-side integrations, webhooks receivers verifying inbound TLS, and any custom HTTP client wrapper the team has built — a shared internal HTTP client library with verification quietly disabled deep inside it silences the protection for every caller that uses it, often without any individual call site realizing it.

### Certificate Lifecycle Management

- **DO:** Automate certificate renewal (via an ACME-based provider, a managed load-balancer certificate feature, or an internal PKI automation tool) rather than relying on a manual, calendar-reminder-driven renewal process, since an expired production certificate is one of the more common, entirely preventable causes of a full outage.
- **DON'T:** Let certificate expiration monitoring be a purely manual, easily-forgotten process. Set up automated alerting well ahead of expiration (weeks, not days) as a backstop even when renewal is automated, since automation itself can silently fail (a changed DNS record, an expired automation credential) without anyone noticing until the certificate actually expires.
- **DO:** Maintain an inventory of every certificate the organization is responsible for — including ones on internal services, IoT/embedded devices, and infrequently touched legacy systems — since an expired certificate on a rarely-visited internal tool is exactly the kind of incident that goes unnoticed until it causes a production issue.

### DNS Security Considerations

- **DO:** Publish CAA (Certification Authority Authorization) DNS records specifying which certificate authorities are permitted to issue certificates for your domain, which reduces the risk of a misissued certificate from an authority you never intended to use.
- **DON'T:** Leave DNS management access as broadly granted as general infrastructure access. DNS is a foundational trust anchor — control over DNS can be used to redirect traffic, obtain fraudulently issued TLS certificates via domain-validation bypass, and intercept mail; scope DNS management access as tightly as any other highly sensitive credential.
- **DO:** Consider DNSSEC where the domain registrar and DNS provider support it, which lets resolvers cryptographically verify that DNS responses haven't been tampered with in transit, closing a class of DNS-spoofing/cache-poisoning attacks that plain DNS has no protection against.

### Service Mesh and Internal Transport Security

- **DO:** Use a service mesh or equivalent infrastructure layer to enforce mTLS automatically between internal services at scale, when the number of services makes manually wiring up mutual TLS for each service-to-service connection impractical — this gets consistent transport security applied without relying on every individual service team to implement it correctly themselves.
- **DON'T:** Assume a service mesh's mTLS is actually enforced (rather than merely available/optional) without verifying the mesh's policy configuration explicitly requires it. Several service mesh implementations default to a permissive mode that allows both encrypted and unencrypted traffic during migration, and that permissive default is easy to leave in place indefinitely past the intended migration window.

### Email Transport and Authenticity (SPF, DKIM, DMARC)

- **DO:** Publish SPF, DKIM, and DMARC DNS records for every domain that sends email (including domains that never send legitimate mail, configured with a strict reject policy), which together let receiving mail servers verify that mail claiming to be from your domain actually originated from an authorized sender and wasn't tampered with in transit.
- **DON'T:** Leave a domain with no DMARC policy (or a policy set to monitor-only indefinitely rather than progressing to enforcement) if the domain is a plausible spoofing target. An attacker can send phishing email that appears to come from your domain to your own customers or employees when no enforced authentication policy exists to let receiving servers reject it.
- **DO:** Treat email-sending infrastructure (the credentials/API keys for a transactional email service, the domain's DNS records controlling SPF/DKIM) with the same access-control rigor as other sensitive infrastructure, since compromised email-sending capability is a direct path to highly convincing phishing against your own users, partners, or customers.

### TLS Termination Points and Internal Trust Boundaries

- **DO:** Understand and document exactly where TLS is terminated in the request path — at a CDN edge, at a load balancer, at the application server itself — and treat every hop *after* a termination point as needing its own explicit security consideration, since "the connection is HTTPS" from the end user's perspective doesn't automatically mean every internal hop behind that termination point is also encrypted.
- **DON'T:** Assume that because the public-facing URL is `https://`, every internal component involved in fulfilling that request is also communicating securely. It's common for TLS to be terminated at a load balancer or CDN with the traffic continuing on to backend servers over plain HTTP inside what's assumed to be a trusted internal network — a reasonable choice for some threat models, but one that should be a deliberate decision, not an unexamined default, particularly for sensitive data.

### Credential Transport: Headers Versus Query Strings Versus Bodies

- **DO:** Transmit bearer tokens, API keys, and session credentials in an HTTP header (`Authorization: Bearer <token>`, a custom auth header, or a cookie with proper flags) rather than in the URL path or query string, since URLs are logged in far more places by default (web server access logs, proxy logs, browser history, the `Referer` header sent to any third-party resource the page subsequently loads) than headers or request bodies typically are.
```
# BAD: the API key travels in the URL, and will be written to server access
# logs, proxy logs, browser history, and any outbound Referer header
GET /api/reports?api_key=sk_live_51H8x9K2example... HTTP/1.1

# GOOD: the credential travels in a header, which is not logged by most
# default access-log configurations and is not carried in a Referer header
GET /api/reports HTTP/1.1
Authorization: Bearer sk_live_51H8x9K2example...
```
- **DON'T:** Design a new API that expects credentials as a query-string parameter for convenience (easier to test with a pasted URL, easier to use from a context that can't easily set headers). If a specific legacy integration genuinely requires it, treat that as an explicitly accepted, documented exception rather than the general design pattern for new endpoints.
- **DO:** Recognize that even header-based credentials can still leak via server-side logging middleware that logs full request headers indiscriminately — the header being the right *transport* choice doesn't remove the need for the logging discipline covered under Security Logging above to also redact it wherever it's captured downstream.

### Server-Side Rendering and Hydration Data Exposure

- **DO:** Review exactly what data a server-side-rendering framework serializes into the page for client-side hydration (embedded JSON state, `window.__INITIAL_STATE__`-style globals, framework-specific data-fetching payloads), since this embedded data is fully visible in the page source to anyone, not only to the authenticated user the page was rendered for — it is not a private server-to-server channel.
- **DON'T:** Pass a full backend data object (fetched for server-side rendering convenience) straight into the page's hydration payload without trimming it to only the fields the client actually needs to render and interact with the page. A common mistake is fetching a complete user or resource record for server-side logic and then serializing that same complete object into the page, inadvertently exposing internal fields (other users' data pulled in for a permission check, internal flags, unrelated PII) that were never meant to reach the browser.
- **DO:** Apply the same output-shaping discipline used for REST/GraphQL API responses (returning only what's needed, covered under Common Web Vulnerabilities' over-fetching guidance) to server-side-rendered hydration data — it is, functionally, another API response, just delivered embedded in HTML instead of a separate request.

## Common Web Vulnerabilities

Beyond injection and access-control flaws, a recurring set of web-specific vulnerability classes shows up across almost every application: attacks that abuse the browser's trust in a site (CSRF, clickjacking), attacks that abuse a server's trust in its own outbound requests (SSRF), and configuration mistakes (permissive CORS, missing security headers, open redirects) that widen the attack surface for everything else. Most of these have a well-established, largely mechanical fix once recognized.

### Client-Side Storage of Tokens

- **DO:** Weigh the trade-off deliberately when deciding where a browser-based single-page application stores its access token: an `HttpOnly` cookie protects the token from being read by JavaScript (closing off theft via XSS) but requires CSRF protection for the requests that rely on it, while storing a token in `localStorage`/`sessionStorage` avoids the CSRF concern but makes the token fully readable by any script running on the page, including an XSS payload — there is no option that is unconditionally safer in every dimension, so the choice should be a deliberate one weighed against the application's actual XSS and CSRF risk profile, not a default carried over from a tutorial.
- **DON'T:** Store a long-lived, highly privileged token (a refresh token, an API key with broad scope) in `localStorage` purely because it's the most convenient API to read from client-side JavaScript. `localStorage` has no built-in protection against being read by any script that executes on the page, so any successful XSS vulnerability anywhere on the site — even in an unrelated feature — can exfiltrate every token stored there.
- **DO:** If choosing `HttpOnly` cookie storage for a single-page application's token, pair it with the CSRF defenses covered later in this section (a CSRF token or `SameSite` cookie attribute) so the trade-off is actually closed on both sides rather than only gaining the XSS protection while leaving CSRF exposure unaddressed.
- **DON'T:** Split the difference by storing a token in a cookie without `HttpOnly` "to get the best of both" — that configuration gets neither benefit: it's still readable by JavaScript (so still vulnerable to the same XSS-driven theft as `localStorage`) while also being automatically attached to cross-site requests (so still needing CSRF protection), combining both downsides rather than avoiding either.

### Cross-Site Request Forgery (CSRF)

- **DO:** Protect every state-changing request (any request that creates, updates, or deletes data, or changes account settings) with a CSRF defense — a per-session or per-request anti-CSRF token validated server-side, or reliance on `SameSite` cookies plus origin/referrer validation for APIs that don't use traditional form submissions.
```html
<!-- GOOD: a per-session CSRF token is embedded in the form and validated server-side -->
<form method="POST" action="/account/email">
  <input type="hidden" name="csrf_token" value="{{ csrf_token }}">
  <input type="email" name="new_email">
</form>
```
- **DON'T:** Rely on the request using a "non-simple" HTTP method (`PUT`/`DELETE` instead of a plain form `POST`) as the sole CSRF defense. While cross-origin `PUT`/`DELETE` requests are somewhat harder to forge from a plain HTML form, they're still forgeable via JavaScript-driven cross-origin requests in some configurations and via CORS misconfiguration on the target; use an explicit CSRF token or `SameSite` cookie protection rather than relying on method choice alone.
- **DO:** Validate the CSRF token server-side by comparing it against the value tied to the user's authenticated session (or, for the double-submit-cookie pattern, against a value delivered via a separate, non-`HttpOnly` cookie), using a constant-time comparison, and reject the request if the token is missing, malformed, or mismatched.
- **DON'T:** Disable CSRF protection framework-wide, or on a specific endpoint, to make a failing integration test or a stubborn form submission "just work." A CSRF token validation failure during development is almost always a sign the client isn't sending the token correctly, not a reason to remove the protection — fix the client to include the token rather than removing the server-side check.
- **DO:** Apply CSRF protection to any endpoint reachable via a browser session and a cookie, including ones that "just read data but also happen to trigger a side effect" (an endpoint that both returns data and increments a counter, marks something as read, or logs an action) — CSRF risk is about state changes, and even a state change that looks minor can be abused if forced silently against many victims at scale.
- **DON'T:** Assume a JSON API that doesn't use traditional `<form>` submissions is automatically immune to CSRF. If the browser will still attach cookies to a cross-origin `fetch`/`XMLHttpRequest` request to that API (which happens unless `SameSite` cookie protection or CORS is configured to prevent it), the request can still be forged from another origin; combine `SameSite` cookies with explicit origin checking for JSON APIs that rely on cookie-based auth.

### Server-Side Request Forgery (SSRF)

- **DO:** Treat any feature where the server fetches a URL supplied (directly or indirectly) by a user — webhook registration, "import from URL," link preview generation, PDF/image fetching from a remote source, an integration that calls a user-specified API endpoint — as a potential SSRF vector, since it lets an attacker make the server issue requests on the attacker's behalf, potentially reaching internal-only services the attacker could never reach directly.
- **DON'T:** Fetch a user-supplied URL from the server without restricting what that URL is allowed to point to. An unrestricted server-side fetch can be aimed at internal-only infrastructure (an internal admin panel, a cloud metadata endpoint that exposes credentials, an internal database's HTTP interface) that isn't reachable from the public internet but is reachable from inside the server's own network.
```python
# BAD: the server fetches whatever URL the user supplies, with no restriction
response = requests.get(user_supplied_url)

# GOOD: the destination is validated against an allowlist of permitted hosts/schemes,
# and requests to private/link-local address ranges are explicitly blocked
if not is_allowed_destination(user_supplied_url):
    raise ValueError("destination not permitted")
response = requests.get(user_supplied_url, allow_redirects=False, timeout=5)
```
- **DO:** Validate and restrict outbound server-side requests to an explicit allowlist of permitted hosts/domains wherever the set of legitimate destinations is known in advance (a webhook target the user registers ahead of time and that gets verified once), rather than allowing an arbitrary URL at request time.
- **DON'T:** Trust DNS resolution alone as an SSRF safeguard, and don't rely only on validating the URL's hostname string before the request is made. An attacker can use DNS rebinding (a domain that resolves to a public IP at validation time and an internal IP at request time) or an unusual IP-encoding trick to bypass a naive string-based check; validate the resolved IP address itself (rejecting private, loopback, and link-local ranges) at the point the connection is actually made, ideally by resolving once and connecting to that specific validated address.
- **DO:** Explicitly block requests to link-local and cloud-metadata address ranges (the well-known metadata IP used by major cloud providers to serve instance credentials) as part of SSRF defenses, since metadata-endpoint access via SSRF is one of the most common ways an SSRF vulnerability escalates into full cloud-account compromise.
- **DON'T:** Follow HTTP redirects blindly on a server-side fetch of a user-influenced URL. A URL that passes an allowlist check can still redirect to a disallowed internal destination; either disable automatic redirect following and re-validate each hop explicitly, or cap and validate every redirect target against the same rules as the original URL.
- **DO:** Run the component that performs server-side URL fetches (a webhook dispatcher, a link-preview generator, a file importer) with restricted network access at the infrastructure level (network policies, egress firewall rules) as a defense-in-depth layer, so that even a bypass of application-level validation is contained by network-level segmentation.
- **DON'T:** Expose internal-only services on the same network without authentication "since only the application server can reach them anyway." That assumption is exactly what SSRF breaks; internal services reachable from a server that also has an SSRF-exploitable feature should still require their own authentication, not rely purely on network position for protection.

### Clickjacking

- **DO:** Send `X-Frame-Options: DENY` (or `SAMEORIGIN` where legitimate same-origin framing is needed) and/or a `Content-Security-Policy` with a `frame-ancestors` directive on every response for pages that perform sensitive actions, preventing the page from being loaded inside a hidden or disguised iframe on an attacker's site.
```
# GOOD: modern CSP-based clickjacking protection (frame-ancestors supersedes
# X-Frame-Options in browsers that support it, but sending both is safe and
# covers older clients too)
Content-Security-Policy: frame-ancestors 'self';
X-Frame-Options: SAMEORIGIN
```
- **DON'T:** Rely on JavaScript-based "frame busting" (`if (top !== self) top.location = self.location;`) as the sole clickjacking defense. Frame-busting scripts have well-documented bypasses (via the iframe `sandbox` attribute, `X-Frame-Options` on the framed page missing entirely, or by preventing the script from executing) and are strictly weaker than a proper HTTP header enforced by the browser itself.
- **DO:** Apply frame-protection headers particularly to pages that perform a one-click sensitive action (a "confirm purchase," "delete account," "authorize this app" button), since clickjacking specifically targets tricking a user into clicking something they can see but whose actual click target — invisible, layered underneath — performs a different, attacker-chosen action.
- **DON'T:** Forget that an application intentionally offering an embeddable widget still needs `frame-ancestors` scoped to the specific, known set of domains permitted to embed it, rather than either denying all framing (breaking the legitimate widget) or allowing all framing (reopening clickjacking risk for the rest of the site if the same header applies broadly).

### Open Redirects

- **DO:** Validate any redirect target against an allowlist of known-safe destinations (an enumerated set of internal paths, or a check that the target is a relative path or matches the application's own domain) before issuing a redirect based on a user-supplied `next`/`redirect`/`return_url` parameter.
```python
# BAD: the redirect target comes straight from a query parameter with no validation
return redirect(request.args.get("next"))

# GOOD: only a relative, same-application path is accepted as a redirect target
next_url = request.args.get("next", "/")
if not next_url.startswith("/") or next_url.startswith("//"):
    next_url = "/"
return redirect(next_url)
```
- **DON'T:** Redirect to a raw, unvalidated user-supplied URL after login, logout, or any other flow, even if the parameter "usually" contains a legitimate internal path. An open redirect lets an attacker craft a link that appears to point to the trusted domain (useful for phishing) but ultimately sends the user on to an attacker-controlled site, and it can also be chained with OAuth flows to leak authorization codes/tokens.
- **DO:** Watch specifically for the `//evil.com` and `/\evil.com` style bypasses of a naive "starts with `/`" check — several browsers treat a URL beginning with `//` as protocol-relative, sending the user to a different host entirely even though the string technically "starts with a slash." Validate using a proper URL parser rather than a prefix string check.

### CORS Misconfiguration

- **DO:** Configure Cross-Origin Resource Sharing (CORS) with an explicit, narrow allowlist of trusted origins for any API that needs to be called cross-origin, and only enable `Access-Control-Allow-Credentials: true` for origins that are genuinely trusted to make authenticated, cookie-bearing requests.
- **DON'T:** Combine `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true`. Browsers actually reject this specific combination for credentialed requests, but the equivalent misconfiguration many frameworks fall into — dynamically reflecting whatever `Origin` header the request sent back as the allowed origin, while also allowing credentials — achieves the same dangerous effect: any website can make authenticated, cookie-bearing requests to the API on a logged-in victim's behalf.
```javascript
// BAD: any origin is reflected back as allowed, combined with credentialed requests —
// effectively any website can make authenticated calls to this API as the visiting user
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', req.headers.origin);
  res.header('Access-Control-Allow-Credentials', 'true');
  next();
});

// GOOD: only a specific, known, trusted origin is allowed to make credentialed requests
const ALLOWED_ORIGINS = new Set(['https://app.example.com']);
app.use((req, res, next) => {
  if (ALLOWED_ORIGINS.has(req.headers.origin)) {
    res.header('Access-Control-Allow-Origin', req.headers.origin);
    res.header('Access-Control-Allow-Credentials', 'true');
  }
  next();
});
```
- **DON'T:** Enable broad CORS as a quick fix for a browser console CORS error without understanding what the error is actually protecting against. A CORS error during development usually means a legitimate cross-origin boundary the browser is correctly enforcing; the fix is to allowlist the specific origin that legitimately needs access, not to open access to every origin.
- **DO:** Keep CORS configuration environment-aware — a permissive `*` origin may be acceptable for a genuinely public, unauthenticated API (one that serves the same non-sensitive response to any caller, with no cookies/credentials involved) but is never acceptable once the endpoint reads or relies on cookies, session state, or any per-user authorization.
- **DON'T:** Forget that CORS is a browser-enforced restriction on *client-side JavaScript* reading a cross-origin response — it does nothing to stop a direct, non-browser request (a server, a script, `curl`) from calling the API. CORS misconfiguration specifically matters because it can let malicious *browser-based* JavaScript running on another site piggyback on a logged-in user's cookies; it is not a general access-control mechanism and should never be treated as one.

### Security Headers Overview

- **DO:** Send a baseline set of security headers on every response: `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `X-Frame-Options`/`frame-ancestors`, `Strict-Transport-Security`, and a reasonable `Referrer-Policy` (such as `strict-origin-when-cross-origin`), configuring each deliberately for the application rather than copying a generic template unread.
```
# GOOD: a reasonable baseline security header set
Content-Security-Policy: default-src 'self'; object-src 'none'; base-uri 'self';
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Strict-Transport-Security: max-age=63072000; includeSubDomains
Permissions-Policy: geolocation=(), camera=(), microphone=()
```
- **DON'T:** Set `X-Content-Type-Options: nosniff` and stop there without understanding what it does, or skip it thinking it's redundant with `Content-Type`. This header prevents the browser from MIME-sniffing a response into a different, possibly executable content type than the server declared, which closes off a class of content-type-confusion attacks (for example, a file served as `text/plain` but sniffed and executed as HTML/JS by the browser).
- **DO:** Set a `Permissions-Policy` header to explicitly disable browser features/APIs (camera, microphone, geolocation, USB access) the application doesn't use, reducing what a successful script-injection attack could abuse even if it somehow ran on the page.
- **DON'T:** Set a `Referrer-Policy` that leaks the full URL (including sensitive query parameters or path segments) to third-party sites the page links to or loads resources from. Use a policy like `strict-origin-when-cross-origin` or stricter for pages with sensitive URLs, so cross-origin navigations and resource loads only leak the origin, not the full path/query.
- **DO:** Verify the actual header values sent in production, not just that the configuration file "should" set them, using a live header inspection (browser dev tools' network tab, or a header-scanning tool) after every deployment that touches server or proxy configuration — a reverse proxy or CDN layer can silently strip or override headers the application itself sets correctly.
- **DON'T:** Treat security headers as a checkbox exercise where any non-empty value "counts." A `Content-Security-Policy` with `default-src *` provides essentially no protection despite technically being present; the actual restrictiveness of the policy is what matters, not merely the header's existence.

### Rate Limiting and Denial-of-Service Considerations

- **DO:** Rate-limit authentication endpoints, password-reset requests, expensive computational endpoints, and any endpoint that triggers an outbound action (sending an email, calling a paid third-party API) per user and per IP/client, to blunt both brute-force/credential-stuffing attempts and simple resource-exhaustion abuse.
- **DON'T:** Leave expensive operations (large report generation, complex search queries, bulk export, image/video processing) completely unbounded in how often or how large a single client can request them. An attacker — or even a legitimate but misbehaving client — can degrade service for everyone by hammering an unthrottled expensive endpoint.
- **DO:** Apply rate limits at multiple layers where possible — a CDN/edge layer for coarse-grained protection against volumetric abuse, and an application layer for finer-grained, per-account or per-action limits that understand business context (e.g., "five password-reset requests per hour per account," not just "N requests per IP").
- **DON'T:** Rely on IP-based rate limiting alone as a complete defense. Attackers routinely distribute requests across many IP addresses (via botnets, proxy pools, or cloud-provider IP ranges) specifically to defeat IP-based throttling; combine IP-based limits with per-account, per-credential, and behavioral signals (a specific request pattern that indicates automated abuse) where practical.

### Business Logic and Race-Condition Vulnerabilities

- **DO:** Consider abuse cases that are logically "valid" API usage but violate the intended business rule — applying the same discount code an unlimited number of times, submitting a purchase and a refund concurrently to duplicate inventory, exploiting a workflow that assumes steps happen in a fixed order but doesn't actually enforce that order server-side.
- **DON'T:** Assume that because every individual request passes validation and authorization, the *sequence* of requests is automatically safe. Business-logic vulnerabilities exploit gaps in workflow enforcement — skipping a required step, replaying a step, or performing two supposedly-exclusive actions concurrently — rather than any single request being malformed.
- **DO:** Guard against race conditions in security-relevant, concurrency-sensitive operations (redeeming a coupon, withdrawing funds, claiming a limited resource) using database-level constraints (unique constraints, row locking, atomic increment/decrement operations, transactions with appropriate isolation levels) rather than a check-then-act pattern in application code that leaves a window between the check and the action.
```python
# BAD: a check-then-act race window lets two concurrent requests both pass the
# balance check before either has actually deducted the amount
if account.balance >= amount:
    time.sleep(0)  # any gap here is enough for a concurrent request to race
    account.balance -= amount
    account.save()

# GOOD: the check and the update happen atomically at the database level,
# so a concurrent request cannot observe a stale balance
UPDATE accounts SET balance = balance - %s
WHERE id = %s AND balance >= %s
-- application checks the affected row count to confirm the deduction succeeded
```
- **DON'T:** Trust that a limited-quantity or limited-use resource (a coupon with a redemption cap, a fixed number of event tickets, a one-time-use invite code) is actually enforced as limited unless the enforcement uses an atomic database operation. A naive read-then-write implementation is exploitable by firing concurrent requests to redeem past the intended limit — a well-documented real-world attack pattern.
- **DO:** Model and test multi-step workflows (checkout, onboarding, approval chains) for what happens if a client skips a step, repeats a step, or submits steps out of order, and enforce the required state transitions server-side (checking the record's current status before allowing the next action) rather than assuming the client-side UI's step order will always be followed.

### Host Header and Request-Smuggling Considerations

- **DO:** Validate or explicitly configure the expected `Host` header value(s) at the application or load-balancer level rather than trusting an arbitrary client-supplied `Host` header to build absolute URLs (password-reset links, redirect targets) used in server responses. A forged `Host` header can poison a password-reset email with an attacker-controlled link if the reset link is built from the request's `Host` header unchecked.
- **DON'T:** Build any security-relevant absolute URL (password-reset links, email verification links, OAuth redirect URIs) from the incoming request's `Host` header without validating it against an allowlist of the application's actual known domains.
- **DO:** Keep the chain of proxies, load balancers, and application servers configured consistently regarding how they parse ambiguous HTTP requests (conflicting `Content-Length` and `Transfer-Encoding` headers), since inconsistent parsing between two components in the chain is the root cause of HTTP request smuggling, which can let an attacker's request be interpreted differently by each hop and smuggle a second, hidden request past security controls.
- **DON'T:** Mix HTTP server/proxy software versions or configurations without verifying they agree on request-framing edge cases, especially when introducing a new reverse proxy, CDN, or load balancer in front of an existing application server — request smuggling is specifically a mismatch-between-components problem, and it's worth confirming vendor guidance on smuggling-safe configuration when changing any component in that chain.
- **DO:** Monitor for subdomain takeover risk — a DNS record (a `CNAME` pointing at a third-party service, such as a decommissioned cloud hosting endpoint or an unclaimed static-site host) left pointing at a resource that has since been deleted or deprovisioned can often be claimed by an attacker on the third-party service, letting them serve content from your own subdomain. Audit DNS records periodically for entries pointing at deprovisioned or unclaimed external resources, and remove the DNS record at the same time the underlying resource is torn down, not afterward.
- **DON'T:** Leave a decommissioned third-party integration's DNS record in place "just in case it's needed again." An unclaimed dangling DNS record is a standing invitation for takeover, and it's easy to forget about once the original integration is no longer top of mind.

### API-Specific Vulnerability Patterns

- **DO:** Treat "broken object level authorization" (a per-record authorization check missing on an API endpoint, the IDOR pattern discussed under Authorization) and "broken function level authorization" (an endpoint reachable by a role that shouldn't be able to call it at all, regardless of which record is targeted) as two distinct failure modes to check for separately on every API endpoint, since fixing one doesn't imply the other has been addressed.
- **DON'T:** Design an API's authorization purely around "is this user logged in" without a second, explicit layer checking "is this user's role/plan/permission level allowed to call this specific function." A regular user reaching an administrative endpoint simply because no function-level check exists is a distinct, very common API vulnerability separate from per-record IDOR.
- **DO:** Apply the same rate-limiting, input-validation, and authorization rigor to internal-facing or partner-facing APIs as to fully public ones. "Internal API" status is a description of the intended audience, not a security control, and internal APIs are frequently the ones with the weakest controls precisely because they were assumed to be low-risk.
- **DON'T:** Expose more data in an API response than the calling client actually needs (returning a full user object, including internal fields, when the UI only displays a name and avatar). Over-fetching in API responses is a common way sensitive fields leak to a client that was never meant to see them, discoverable simply by inspecting network traffic in browser dev tools.
- **DO:** Version APIs deliberately and retire old versions on a real schedule, since an old, forgotten API version often keeps running with none of the security fixes applied to the current version — an outdated but still-live endpoint is a common way a fixed vulnerability remains exploitable through a version nobody remembered to decommission.

### WebSocket Security

- **DO:** Authenticate and authorize WebSocket connections at the point of connection (validating a session cookie or token during the handshake) and re-verify authorization for each subsequent message that requests a sensitive action, the same way a REST endpoint would, rather than treating "the socket connected successfully" as blanket, unlimited authorization for everything sent over it afterward.
- **DON'T:** Skip origin validation on the WebSocket handshake. Unlike `fetch`/XHR, WebSocket connections are not restricted by the same-origin policy or CORS by default, so a malicious page on a different origin can open a WebSocket connection to your server using the visiting user's cookies unless the server explicitly checks the handshake's `Origin` header against an allowlist — this is effectively the WebSocket analogue of CSRF.
- **DO:** Validate and rate-limit every message received over an established WebSocket connection with the same discipline as HTTP request bodies — a long-lived connection doesn't reduce the need for per-message input validation, and in some ways increases the risk since a single connection can send many more messages than a typical HTTP request/response cycle allows.

### Cache Poisoning and Web Cache Deception

- **DO:** Be deliberate about which responses are cacheable and by what layer (browser, CDN, reverse proxy), setting explicit `Cache-Control` directives, and never let a response containing per-user or sensitive data be cached at a shared layer (a CDN or shared proxy cache) that could serve it to a different user.
- **DON'T:** Let a shared/CDN cache key be influenced by a header or parameter that isn't also part of what actually varies the response content. If a cache stores responses keyed only by URL but the actual response content depends on a cookie, an `Authorization` header, or another per-user signal, one user's personalized (or sensitive) response can be cached and served to a completely different user — a serious cross-user data leak.
- **DO:** Watch for "web cache deception" specifically: a request to a sensitive, supposedly non-cacheable authenticated endpoint with a spoofed static-looking file extension or path suffix appended (`/account/settings.css`) can, on a misconfigured cache layer that caches based on file-extension heuristics, cause that authenticated response to be cached and later served to unauthenticated or different users who request the same crafted path.
- **DON'T:** Trust that a cache layer's default configuration correctly distinguishes cacheable static assets from dynamic, per-user application responses without explicit configuration confirming it. Default heuristics (cache anything that looks like a static file extension) are a known source of this exact vulnerability class.

### Third-Party Scripts and Client-Side Supply Chain

- **DO:** Treat every third-party script loaded into a page (analytics, chat widgets, ad tech, A/B testing tools) as running with the full privileges of your page's origin — able to read cookies (unless `HttpOnly`), read form input, and modify the DOM — and vet third-party scripts accordingly before adding them, especially to pages that handle payment or account credentials.
- **DON'T:** Load a third-party script without Subresource Integrity (SRI) when the script's content should not change unexpectedly between deployments, since SRI lets the browser refuse to execute a fetched script whose content hash doesn't match what was expected, protecting against a compromised third-party CDN silently serving modified, malicious script content to every site that includes it.
```html
<!-- BAD: no integrity check — if the third-party CDN is compromised, the
     malicious replacement script executes with full page privileges -->
<script src="https://cdn.example-widget.com/widget.js"></script>

<!-- GOOD: the browser verifies the fetched script's hash before executing it,
     and refuses to run it if the content doesn't match -->
<script src="https://cdn.example-widget.com/widget.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
        crossorigin="anonymous"></script>
```
- **DO:** Restrict what third-party scripts are allowed to do via CSP (limiting which origins can be loaded as scripts at all) and, where the CSP level supports it, `require-trusted-types-for` to reduce the impact of a compromised or misbehaving third-party script even without SRI in place for scripts that legitimately need to update dynamically.
- **DON'T:** Add a checkout or payment-page third-party script (analytics, chat, marketing pixels) without specifically evaluating whether it needs to run on that exact page. Payment pages are high-value targets for exactly this kind of client-side supply-chain attack (a compromised third-party script skimming payment form data directly from the page), and the safest mitigation is often simply not loading non-essential third-party scripts on pages that handle payment details at all.

### HTTP Parameter Pollution

- **DO:** Define and enforce an explicit, single-source-of-truth behavior for how the application handles a parameter supplied more than once in the same request (`?role=user&role=admin`), since different frameworks, servers, and downstream components can disagree on which duplicate "wins" (the first, the last, or an array of all values), and an attacker can exploit that disagreement if one internal component's interpretation differs from another's.
- **DON'T:** Let a value's meaning depend on which layer of the stack happens to read a duplicated parameter first, particularly for anything security-relevant. If a WAF or reverse proxy validates the first occurrence of a parameter while the application logic reads the last occurrence (or vice versa), an attacker can smuggle a malicious value past the validating layer while the value the application actually acts on is different.

### Session Variable and State Confusion

- **DO:** Keep session/request-scoped state narrowly typed and explicitly namespaced (a dedicated, well-defined session schema) rather than storing loosely typed, ad hoc key-value data in a shared session object across multiple unrelated features, since a generic shared session store makes it easy for one feature to accidentally read or overwrite a value another feature set for a different purpose — a pattern sometimes called session puzzling, which has been used to bypass authentication or authorization checks that assumed a session variable meant one thing when another code path had set it to mean something else.
- **DON'T:** Set a security-relevant session flag (an "authenticated" or "verified" marker) using a key or pattern general enough that an unrelated, lower-trust code path could plausibly set the same key for an unrelated purpose. Namespace security-critical session state clearly and validate its expected shape/type on read, not just its presence.

### Directory Listing and Insecure Direct File Access

- **DO:** Disable directory listing on any web server or storage bucket serving files, so a request to a directory path without a specific filename doesn't return an enumerable list of every file in that directory — a surprisingly common way internal file structure, backup files, or unintentionally public documents get discovered.
- **DON'T:** Leave default or forgotten files reachable in a public web root — backup files (`.bak`, `~` swap files), version-control directories (`.git/`), configuration file templates, or old deployment artifacts. Regularly audit what's actually reachable under the public document root, since these files often contain source code, credentials, or internal configuration detail never meant to be served.

### Mobile Deep Link and Intent Validation

- **DO:** Validate the origin and structure of data delivered through a mobile app's deep link or custom URL scheme handler exactly as rigorously as data arriving over HTTP, since a deep link can be triggered by any other app or a malicious webpage, not only by a link the application itself generated.
- **DON'T:** Let a deep-link handler perform a sensitive action (logging in as a specified user, transferring funds, changing a setting) directly from an unvalidated deep-link parameter. Deep links are a common vector for parameter injection and for tricking a user into triggering an unintended in-app action via a link they were social-engineered into tapping.
- **DO:** Prefer platform-verified deep links (Android App Links / iOS Universal Links, which require proving domain ownership) over an unverified custom URL scheme where the platform supports it, since an unverified custom scheme can potentially be registered and intercepted by another, possibly malicious, app installed on the same device.
- **DON'T:** Trust data passed through an Android `Intent` from another app without validating its source and content, particularly for an exported component (an `Activity`, `Service`, or `BroadcastReceiver` marked reachable by other apps). An exported component with no validation is directly reachable by any other app on the device, effectively making it as exposed as a public network endpoint.

### Security Testing: SAST, DAST, and Penetration Testing

- **DO:** Combine multiple layers of security testing rather than relying on any single technique — static analysis (SAST) scanning source code for known-risky patterns during development, dynamic analysis (DAST) probing a running application's actual behavior, dependency/software-composition-analysis (SCA) as covered under Dependency Security, and periodic manual penetration testing or a bug-bounty program for the classes of vulnerability automated tooling reliably misses (business-logic flaws, chained multi-step exploits, authorization design gaps).
- **DON'T:** Treat a clean SAST/DAST scan as proof the application is secure. Automated tools are good at catching known patterns (injection, missing headers, outdated dependencies) and reliably poor at catching business-logic vulnerabilities, subtle authorization gaps, and anything that requires understanding what the application is actually *supposed* to do versus what it mechanically does — this is precisely why manual review and testing remain necessary even with strong automated tooling in place.
- **DO:** Run SAST tooling early and often (ideally on every pull request) so findings are cheap to fix while the relevant code is still fresh in the author's mind, rather than only running it as a pre-release gate where a finding requires re-opening work someone has already moved past.
- **DON'T:** Let a security scanning tool's findings pile up as an ever-growing backlog nobody triages. An unaddressed backlog of findings — even lower-severity ones — makes it progressively harder to spot a new, genuinely urgent finding among the noise, and signals to the team that scan results aren't actually acted on, which erodes the practice over time.
- **DO:** Commission periodic third-party penetration testing or participate in a bug-bounty program for applications handling sensitive data or significant business risk, since an external perspective — unconstrained by the assumptions the internal team has built up about how the system is "supposed" to be used — routinely finds issues internal review and automated tooling both miss.
- **DON'T:** Treat a penetration test as a one-time compliance checkbox disconnected from the regular development process. The findings from a penetration test are only valuable if they're triaged, fixed, and — where the finding reveals a systemic gap (a whole class of endpoints missing a check) — used to update the team's own review checklist and automated tooling so the same class of issue doesn't reappear in new code written afterward.

### GraphQL Introspection and Schema Exposure

- **DO:** Disable GraphQL schema introspection in production for any API not intentionally designed as a fully public, self-documenting API, since an introspectable schema hands an attacker a complete map of every type, field, query, and mutation the API exposes — including internal-sounding fields that were never meant to be discovered by trial and error.
- **DON'T:** Leave a GraphQL API's default developer-friendly settings (introspection enabled, a GraphiQL/Playground explorer UI reachable, verbose error messages including stack traces) unchanged when deploying to production. These defaults exist to make development convenient, not to be production-safe, and are a common example of exactly the "convenient scaffold default becomes the production configuration" pattern covered under AI-assistant mistakes above.
- **DO:** Apply the same rigor to GraphQL error messages as to REST error responses — a GraphQL resolver that lets an unhandled exception's message and stack trace flow through to the client response leaks the same kind of internal detail a verbose REST error page would, just wrapped in a different response format.

### Feature Flags and Incompletely Secured Features

- **DO:** Treat a feature hidden behind a feature flag as still fully reachable by anyone who can guess or discover its endpoint/route, and apply full authentication, authorization, and input validation to flagged-off features exactly as to shipped ones — a feature flag controls *visibility in the UI*, not access to the underlying code path, which typically remains deployed and callable regardless of the flag's state.
- **DON'T:** Defer implementing authorization checks on a new feature's endpoints on the reasoning that "it's behind a flag, so it's not really live yet." The endpoint is live the moment it's deployed; the flag only affects whether a UI element linking to it is shown, and a determined user (or an automated scanner) can reach a flagged-off endpoint directly with no more effort than reaching a shipped one.
- **DO:** Audit feature-flag configuration itself for who can toggle flags and whether flag state is fetched and enforced server-side (authoritative) or only client-side (bypassable) — a client-side-only flag check, like any other client-side check covered under Authorization above, is a UX convenience, not a security control.

### Vulnerability Disclosure and Responsible Reporting

- **DO:** Publish a `security.txt` file (at the well-known path defined by the relevant standard) and/or a clear vulnerability-disclosure policy describing how a security researcher can responsibly report a finding, so that someone who discovers a vulnerability has an obvious, legitimate channel to report it rather than no clear path at all, or worse, disclosing it publicly first for lack of any other option.
- **DON'T:** Leave security researchers with no clear reporting channel and then respond to a good-faith report — if one does arrive through an improvised channel like a general support inbox — with a legal threat or hostility. A punitive response to good-faith reporting discourages future responsible disclosure, both from that specific researcher and, once word spreads, from the broader research community.
- **DO:** Define and publish clear scope and safe-harbor terms for a vulnerability-disclosure or bug-bounty program (what testing is authorized, what's out of scope, a commitment not to pursue legal action against good-faith research conducted within the stated scope) so researchers can participate with confidence and the organization gets a predictable, structured inflow of findings instead of an ad hoc one.
- **DON'T:** Let a reported vulnerability sit untriaged for an extended period. Acknowledge receipt promptly and provide the reporter a realistic timeline, since a lack of response is one of the most common reasons a researcher escalates to public disclosure before a fix is ready.

### PII Minimization at the Point of Input

- **DO:** Question, at the point a new form field or data-collection point is designed, whether the data actually needs to be collected at all for the feature to work — the cheapest, most reliable way to reduce the impact of a future data breach or logging mistake is to never collect the sensitive data in the first place when it isn't genuinely required.
- **DON'T:** Collect personal data "in case it's useful later" for a feature that doesn't currently need it. Every additional field of collected personal data is additional exposure in a future breach, additional scope for a compliance obligation, and additional data that needs the same validation, encryption, and access-control discipline as everything else covered in this document — none of which is free.

### Cross-Origin Messaging with postMessage

- **DO:** Validate the `origin` property of every incoming `message` event when using `window.postMessage` for cross-origin/cross-frame communication, checking it against an explicit allowlist of expected origins before trusting or acting on the message's content, since any page — including a malicious one — can send a `postMessage` to your window if it can obtain a reference to it (via a popup, an iframe it controls, or a similar means).
```javascript
// BAD: the message handler acts on the received data without checking
// which origin actually sent it — any page that can reference this window
// can send a crafted message and have it accepted
window.addEventListener('message', (event) => {
  updateUserProfile(event.data);   // no origin check at all
});

// GOOD: the sender's origin is validated against an explicit allowlist
// before the message content is trusted
const ALLOWED_ORIGINS = new Set(['https://trusted-partner.example.com']);
window.addEventListener('message', (event) => {
  if (!ALLOWED_ORIGINS.has(event.origin)) return;
  updateUserProfile(event.data);
});
```
- **DON'T:** Send sensitive data via `postMessage` with a wildcard target origin (`*`). Specifying `*` as the target origin means the message is delivered to whatever origin the target window currently happens to be showing, which can change (via navigation) between when the reference was obtained and when the message is sent — always specify the exact expected target origin.
- **DO:** Validate the *shape and content* of `event.data` even after confirming the origin is on the allowlist, treating it with the same input-validation discipline as any other externally supplied data — a trusted origin sending unexpectedly malformed or unexpected data (due to its own bug or its own compromise) shouldn't be blindly trusted to always send well-formed messages.

## Security Logging & Error Handling

What an application logs, and what it shows a user when something goes wrong, are both security-relevant decisions. Logs need to capture enough to investigate an incident without becoming a second place sensitive data leaks from; error responses need to be helpful to legitimate users without handing an attacker a map of the system's internals.

### Avoiding Information Leakage in Error Responses

- **DO:** Show end users a generic, non-revealing error message for unexpected failures (a generic "something went wrong, please try again" with a correlation/reference ID), while capturing the full detail — stack trace, exception type, relevant context — in server-side logs and error-tracking tools that only internal engineers can access.
```javascript
// BAD: the raw exception, including internal file paths and a stack trace,
// is sent straight to the client
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.stack });
});

// GOOD: the client gets a generic message and a correlation ID; full detail
// goes to server-side logs/error tracking only
app.use((err, req, res, next) => {
  const errorId = logger.error(err, { path: req.path, userId: req.user?.id });
  res.status(500).json({ error: 'Something went wrong.', errorId });
});
```
- **DON'T:** Return a raw stack trace, an internal file path, a database error message, or a framework's default debug error page to end users in production. Stack traces reveal internal file structure, library versions (useful for an attacker looking up known vulnerabilities in those exact versions), and sometimes fragments of source code or configuration — all of it free reconnaissance handed to anyone who triggers an error.
- **DO:** Ensure debug/development error pages (a framework's verbose "whitelabel error page," an interactive debugger triggered by an unhandled exception) are explicitly disabled in production configuration, and verify this with an actual production-mode check rather than assuming the deployment process handles it — a framework's debug mode flag left on by default or by a misconfigured environment variable is a recurring, high-impact real-world mistake.
- **DON'T:** Let differing error messages between distinct failure cases leak information useful for enumeration or reconnaissance — a login form that says "no account with that email" versus "wrong password" tells an attacker which emails are registered; an API that returns a different error for "resource not found" versus "not authorized to view this resource" tells an attacker which resource IDs exist even when they can't access them.
- **DO:** Include enough context in the *server-side* log entry for an error (request path, sanitized/redacted relevant parameters, user/session identifier, a stack trace) to actually debug the issue later, since the fix for "logs aren't sensitive enough for users" is a generic user-facing message plus a rich internal log, not simply logging less everywhere.
- **DON'T:** Catch a broad exception type purely to suppress an error and return a "success" response anyway, hiding a real failure from both the user and the logs. Silently swallowing security-relevant errors (a failed authorization check that gets caught and ignored, a failed signature verification that falls through to "assume valid") is one of the more dangerous logging/error-handling anti-patterns, since it turns a defect into invisible, silent failure of a security control.

### Structured, Security-Relevant Audit Logging

- **DO:** Maintain a dedicated audit log for security-relevant and sensitive actions — authentication events (login, logout, failed login, password change, MFA changes), authorization decisions on sensitive resources, administrative actions, data exports, permission/role changes, and access to particularly sensitive records — distinct from general debug/application logs, since audit logs typically need longer retention, stricter access control, and tamper-resistance that routine debug logs don't.
- **DON'T:** Rely on general-purpose application logs (often verbose, rotated aggressively, and broadly accessible to the whole engineering team) as the system of record for security investigations. General logs are optimized for day-to-day debugging, not for the retention duration, structure, and access restrictions an incident investigation or a compliance audit actually needs.
- **DO:** Structure audit log entries consistently (a fixed schema: timestamp, actor identity, action, target resource, outcome, source IP/context) so they can be queried, correlated, and alerted on programmatically rather than only read manually as free-text lines.
```json
{
  "timestamp": "2026-09-04T14:02:11Z",
  "actor": { "userId": "usr_9f21", "ip": "203.0.113.7" },
  "action": "role.updated",
  "target": { "type": "user", "id": "usr_4412" },
  "changes": { "role": { "from": "member", "to": "admin" } },
  "outcome": "success"
}
```
- **DON'T:** Log an audit event without recording who performed the action and on whose behalf, if applicable (an admin acting on a customer's account, a service account acting for a scheduled job) — "an admin changed this setting" is far less useful during an investigation than "admin usr_9f21 changed this setting for account acc_1123 at this timestamp from this IP."
- **DO:** Protect audit logs from being modified or deleted by the same actors whose actions they record — write audit logs to an append-only store, a separate system with more restrictive write access than the application's normal database access, or a dedicated logging/SIEM pipeline, so a compromised application account can't cover its own tracks by editing the log that would reveal it.
- **DON'T:** Grant the application's normal database credential (the one used for everyday reads/writes) unrestricted delete or update access to the audit-log table/store. If that credential is ever misused (through an injection vulnerability, a compromised dependency, or an insider), the audit trail meant to help investigate the incident should not be something that same access can erase.
- **DO:** Set a log-retention policy for audit logs that satisfies both operational investigation needs and any applicable compliance/regulatory requirement, and make sure the retention is actually enforced (backups included), rather than defaulting to whatever the logging platform's default retention happens to be.

### Never Logging Sensitive Data in Plaintext

- **DO:** Maintain an explicit, reviewed list (or an automated redaction rule set) of fields that must never appear in plaintext in any log — passwords, full credit card numbers, authentication tokens/API keys, government identifiers, full session identifiers, and other regulated personal data — and enforce it at the logging-library level so individual call sites don't each need to remember to redact manually.
- **DON'T:** Log full request or response bodies as a blanket debugging convenience on any endpoint that might carry sensitive fields. It is far too easy for a body-dump logging statement, added for a specific short-term debugging need, to remain in the codebase and quietly leak sensitive fields into logs indefinitely afterward.
- **DO:** Mask or truncate sensitive identifiers when they do need to appear in logs for correlation purposes (the last four digits of a card number, a hashed or truncated version of a token) rather than either logging the full value or omitting it entirely and losing the ability to correlate related log entries.
- **DON'T:** Log a user's password, even a failed login attempt's *incorrect* password, on the reasoning that "it's wrong anyway so it doesn't matter." Users very commonly reuse similar or identical passwords across services, so a "wrong" password captured in a log is often still a real, working credential for some other account the user has, in addition to sometimes simply being a typo away from the actual correct password.
- **DO:** Extend "never log sensitive data" to third-party API request/response logging (payment processor calls, identity-provider calls) — these often carry the exact same categories of sensitive data as the application's own requests, and a generic HTTP-client logging middleware enabled for debugging a third-party integration is a common accidental leak point.
- **DON'T:** Forget that log injection/log forging is itself a risk when user-controlled input is written into logs unsanitized — a value containing newline characters or log-format-specific control sequences can be used to inject fake log entries, corrupt structured-log parsing, or (in log viewers that render log content as HTML) even achieve stored XSS against whoever views the logs. Use a structured logging format (JSON, key-value pairs with proper encoding) rather than free-text string concatenation, so user-controlled values are contained as data fields, not interpretable as log syntax.
```python
# BAD: a newline in user input can forge an additional, fake log line
logger.info(f"Login attempt for user: {username}")
# username = "admin\n[INFO] Login succeeded for user: admin" forges a second entry

# GOOD: structured logging keeps the user-controlled value contained as a
# single field value, not interpretable as additional log syntax
logger.info("login_attempt", extra={"username": username})
```

### Monitoring, Alerting, and Detection

- **DO:** Set up automated alerting on security-relevant log patterns — repeated failed logins against a single account or from a single source, a spike in authorization-denied responses, privilege-escalation events, unusual data-export volume — so an attack in progress is surfaced to a human quickly rather than discovered only in a post-incident log review.
- **DON'T:** Treat "we have logs" as equivalent to "we would notice an attack." Logs that are collected but never monitored, alerted on, or periodically reviewed provide forensic value after the fact but do nothing to shorten the time between a compromise starting and someone noticing it, which is one of the biggest factors in how much damage an incident ends up causing.
- **DO:** Centralize logs from every relevant component (application servers, load balancers/WAF, authentication service, database access logs where feasible) into one searchable system, since correlating an attack across components is far harder when each component's logs live in a separate, disconnected system with its own retention and query tooling.
- **DON'T:** Let alert fatigue from an overly noisy detection rule set cause genuinely important security alerts to be ignored or auto-triaged away. Tune detection rules to a signal-to-noise ratio the team can actually sustain reviewing, and treat a rule that fires constantly with no action taken as a rule that needs tuning, not as evidence the underlying risk doesn't matter.
- **DO:** Include enough identifying context in security alerts (which account, which resource, which source) that a responder can begin investigating immediately, rather than an alert that only says "something anomalous happened" with no actionable detail.
- **DON'T:** Log successful security-control bypasses or intentional test/debug flags without a loud, unmistakable marker distinguishing them from real events. A "test mode" flag that silently disables signature verification and logs identically to a real verified event can make an incident investigation dangerously misread a genuine bypass as routine test traffic, or vice versa.
- **DO:** Periodically review who has access to the logging and monitoring systems themselves, since logs frequently aggregate sensitive operational detail (internal hostnames, authentication flows, error content) even after redaction efforts, and the logging platform's own access control is part of the overall security posture, not an afterthought outside it.

### Debug and Diagnostic Endpoint Exposure

- **DO:** Restrict access to health-check, metrics, debug, and status endpoints (`/health`, `/metrics`, `/debug/vars`, framework-specific debug toolbars) to internal networks or authenticated internal callers, and audit what information they actually reveal — a health-check endpoint that dumps environment variables, dependency versions, or internal service topology is handing reconnaissance information to anyone who finds it.
- **DON'T:** Leave a framework's built-in debug/profiling toolbar or admin interface enabled and reachable in a production deployment. These tools are typically designed for local development, often expose significant internal detail (SQL queries with parameters, full request/response data, sometimes an interactive code-execution console), and are one of the more common "exposed by default, exploited quickly" misconfigurations.
- **DO:** Verify third-party observability integrations (APM agents, log shippers, feature-flag services) aren't themselves exposing a diagnostic or configuration endpoint publicly by default — some ship with a management UI listening on a predictable port that needs to be explicitly firewalled or disabled outside of trusted networks.

### Incident Response Readiness Through Logging

- **DO:** Ensure logs retain enough history, and are queryable quickly enough, to reconstruct the timeline of a suspected incident — "when did this account's access pattern change," "what did this compromised credential actually touch" — since the value of security logging is ultimately measured by whether it answers exactly these questions when it matters.
- **DON'T:** Discover during an actual incident that critical logs were rotated out, that a relevant component wasn't logging the field now needed, or that the team doesn't know how to query the logging system under pressure. Run periodic incident-response drills that actually exercise the logging/monitoring tooling, not just the communication plan, so gaps are found before a real incident, not during one.
- **DO:** Preserve logs relevant to a confirmed or suspected security incident beyond their normal retention window immediately upon discovery, and restrict who can modify or delete them during the investigation, since normal log-rotation policies can otherwise destroy evidence needed for root-cause analysis or a compliance-driven post-incident report.

### Compliance and Regulatory Logging Considerations

- **DO:** Identify which specific actions require audit logging to satisfy applicable regulatory or compliance obligations (data access logging for regulated personal data, financial transaction logging, administrative action logging) early in a feature's design, since retrofitting compliant audit logging onto a feature already in production is markedly more disruptive than designing it in from the start.
- **DON'T:** Over-collect and over-retain personal data inside logs "in case it's useful later." Minimizing what personal data appears in logs at all — logging a user ID rather than a name/email where the ID suffices for correlation — reduces both the compliance burden and the impact of a future log-storage compromise; log only what's actually needed for the log's purpose.

### Log Storage Security

- **DO:** Encrypt log storage at rest and in transit to the logging/aggregation system, and restrict who can read logs based on sensitivity — not every engineer needs read access to logs that may contain audit trails of other users' sensitive actions, even after redaction efforts.
- **DON'T:** Let log-shipping agents or log-aggregation pipelines transmit log data over an unencrypted channel internally "since it's just going to our own logging infrastructure." As with any other internal data flow, "internal" is not itself a security control; encrypt the transport.
- **DO:** Consider tamper-evidence for particularly sensitive audit logs (cryptographic hash chaining between consecutive entries, or writing to an append-only/immutable storage backend) so that any post-hoc modification of the log itself would be detectable, which matters most for logs that might need to serve as evidence in a compliance audit or a legal proceeding following an incident.

### Time Synchronization for Log Correlation

- **DO:** Keep clocks synchronized (via NTP or an equivalent time-sync service) across every system that produces logs, and record timestamps in a consistent timezone (UTC is the conventional choice) in every log entry, since correlating events across multiple systems during an incident investigation depends entirely on being able to trust that "this happened at 14:02:11" means the same moment in every system's logs.
- **DON'T:** Let clock drift between servers go unmonitored. A investigation trying to reconstruct "which service touched this record first" becomes far harder, and can produce actively misleading conclusions, when the systems involved don't agree closely enough on what time it currently is.

### Implementing Redaction at the Logging-Library Level

- **DO:** Implement sensitive-field redaction as a reusable processor/formatter attached to the logging library itself, applied to every log call automatically, rather than as a convention individual developers are expected to remember to apply manually at each call site.
```python
# GOOD: a logging processor redacts known-sensitive field names automatically,
# so every call site benefits without needing to remember to redact manually
SENSITIVE_FIELDS = {"password", "authorization", "token", "ssn", "credit_card"}

def redact_processor(logger, method_name, event_dict):
    for key in list(event_dict.keys()):
        if key.lower() in SENSITIVE_FIELDS:
            event_dict[key] = "***REDACTED***"
    return event_dict

structlog.configure(processors=[redact_processor, structlog.processors.JSONRenderer()])
# now every call site is covered automatically:
logger.info("user_login", username=username, password=password)  # password redacted
```
- **DON'T:** Depend on every individual developer remembering to manually redact a sensitive field before each log statement. Manual, per-call-site redaction is exactly the kind of convention that reliably gets missed under deadline pressure or by a new team member unfamiliar with the rule; a centralized processor enforces it structurally instead of relying on memory.
- **DO:** Extend the same centralized redaction approach to error-tracking and APM tool integrations, configuring their field-scrubbing rules once at the integration layer rather than trusting that every exception-handling call site remembers to scrub sensitive context before it's captured.

### Log Volume, Sampling, and Cost Trade-offs

- **DO:** Distinguish between logs that can be safely sampled or aggressively rotated for cost reasons (high-volume, low-security-relevance debug output) and logs that must be retained in full (the security-relevant audit trail covered above), applying different retention and sampling policies to each rather than a single blanket policy driven only by storage-cost concerns.
- **DON'T:** Let a cost-driven decision to reduce logging volume (sampling, shortened retention, log-level filtering) inadvertently drop or shorten retention on security-relevant audit events. When tuning logging for cost, explicitly carve out the audit-log category as exempt from sampling/aggressive-rotation policies applied to routine debug logs.

### Post-Incident Review and Blameless Postmortems

- **DO:** Conduct a blameless postmortem after any confirmed security incident, focused on what allowed the incident to happen and what systemic change (a missing control, a gap in monitoring, an unclear ownership boundary) would prevent a recurrence, rather than focused on which individual made a mistake.
- **DON'T:** Let a postmortem process that assigns individual blame become the norm. A blame-oriented postmortem culture reliably teaches people to hide near-misses and honest mistakes rather than report them, which removes exactly the early-warning signal an organization needs to catch a systemic issue before it becomes a real incident.
- **DO:** Track postmortem action items to actual completion, and treat an incident whose root-cause fix was never implemented as a live, tracked risk rather than a closed matter — the entire value of a postmortem is in the follow-through, not in the document itself.

## Common AI-Assistant Mistakes in Security-Sensitive Code

AI coding assistants are trained on a huge amount of code that itself contains security mistakes, and they optimize for producing code that looks plausible and makes the immediate error or test failure go away — which is not the same objective as producing secure code. The patterns below are specifically the ones that recur across AI-assisted development: security controls quietly weakened to unblock progress, code that is syntactically correct but semantically wrong in a security-relevant way, and confident-sounding suggestions that don't hold up to verification. Every one of these needs a human security-aware review pass, not blind trust in generated output — and the same scrutiny applies whether the code came from an AI assistant or a human under deadline pressure, since the failure mode is identical either way.

### Disabling Security Controls to Make an Error Disappear

- **DO:** Treat a security-relevant error (a TLS/certificate failure, a signature-verification failure, a CORS rejection, a permission-denied response) as a signal to find and fix the actual underlying cause, and be specifically suspicious of any suggested fix that consists of turning the failing check off.
- **DON'T:** Accept a suggested fix of the form "add `verify=False`," "catch and ignore this exception," "add `#nosec`/a lint-suppression comment," or "temporarily comment out this check" for a security-relevant failure without first understanding *why* the check is failing. An AI assistant optimizing for "the error is gone" will readily suggest disabling the exact control that was doing its job; the error disappearing is not the same thing as the underlying problem being fixed.
```python
# BAD: an AI assistant's suggested "fix" for a certificate error — the check
# is disabled rather than the actual certificate/trust-store problem being fixed
requests.get(internal_api_url, verify=False)

# GOOD: the actual cause is fixed — here, supplying the correct internal CA bundle
requests.get(internal_api_url, verify="/etc/ssl/certs/internal-ca.pem")
```
- **DO:** Ask, whenever a generated fix removes, weakens, or bypasses a check rather than addressing the condition that made the check fail: "what was this check protecting against, and does that risk still exist after this change?" If the answer is "yes, the risk is still there, we've just stopped detecting it," reject the fix.
- **DON'T:** Let a suggestion to "just make the test pass" result in the underlying application code's security control being weakened rather than the test being fixed to correctly exercise real behavior. It is common for a generated fix to a failing security test to loosen the assertion, mock out the security check entirely, or hardcode the expected result, rather than correctly implementing the behavior the test was written to verify — leaving the test green while the actual control it was meant to validate is gone.

### Hardcoding Secrets "Just for Now"

- **DO:** Push back on and reject a generated code sample that hardcodes a real-looking or placeholder API key, password, or connection string directly in application code, even when it's offered as a quick way to "get this working" before wiring up proper configuration.
- **DON'T:** Accept "hardcode it for now, move it to an environment variable later" as a reasonable intermediate step, since — as covered under Secrets Management above — that "later" step is exactly the one that reliably gets skipped, and the hardcoded value often ends up committed to version control before anyone circles back.
```javascript
// BAD: an AI assistant fills in a working example with a hardcoded-looking key
// "to demonstrate the integration," intending it to be replaced later
const client = new PaymentClient({ apiKey: 'sk_live_51H8x9K2example...' });

// GOOD: the same example is written against configuration from the start,
// with no real or real-looking value ever appearing in the code
const client = new PaymentClient({ apiKey: process.env.PAYMENT_API_KEY });
```
- **DO:** Insist that any example, demo, or scaffold code an assistant generates reads secrets from environment variables or a config object from the very first draft, so there is never a version of the code with a real credential shape embedded in it, even briefly.

### Writing Raw SQL Despite an Available ORM

- **DO:** Notice when a generated database-access suggestion reaches for raw, string-built SQL in a codebase that already has a perfectly capable ORM or query builder in use elsewhere, and redirect it to use the existing parameterized abstraction instead.
- **DON'T:** Accept a generated snippet that builds a query via f-string/template-literal interpolation "for a quick one-off query" when the same codebase already has ORM methods available that would express the same query safely. This pattern shows up often in generated code because string-built SQL is simple to produce and reads as immediately correct, without the assistant necessarily surfacing that it just reintroduced an injection vulnerability into an otherwise-safe codebase.
```python
# BAD: an ORM is used throughout the rest of the codebase, but this one
# generated "quick filter" query drops back to raw, string-built SQL
def search_users(name_query):
    sql = f"SELECT * FROM users WHERE name LIKE '%{name_query}%'"
    return db.execute(sql)

# GOOD: the same query expressed through the ORM's parameterized query builder,
# consistent with the rest of the codebase
def search_users(name_query):
    return User.query.filter(User.name.ilike(f"%{name_query}%")).all()
    # the ORM binds name_query as a parameter — it never becomes part of the SQL text
```
- **DO:** Ask explicitly, when reviewing any generated database code, "does this project already have an ORM/query-builder method for this operation, and if so, why does this suggestion bypass it?" A legitimate reason (a genuinely complex query the ORM can't express cleanly) should still use the ORM's parameterized raw-query method, not naive string interpolation.

### Subtly Broken Authentication and Authorization Checks

- **DO:** Read generated authentication/authorization code line by line for the specific comparison operators, control-flow branches, and early returns used, rather than skimming it for overall shape — this is exactly the kind of code where a single wrong operator or a missing `else` produces code that looks correct, passes a casual glance, and is completely broken.
- **DON'T:** Accept a role or permission comparison written with loose/coercive equality, or as a case-sensitive string match against a value that could plausibly arrive in different casing, without verifying the comparison is strict, type-checked, and validated against a known enumeration.
```javascript
// BAD: loose equality on a role value that ultimately originates from user-
// controllable data (e.g., deserialized from a JWT claim or a request field)
// can be satisfied by unexpected type coercion or casing the author didn't anticipate
function isAdmin(user) {
  return user.role == 'Admin';   // loose equality, and case-sensitive
}

// GOOD: strict equality against a validated, normalized enum value
const Role = Object.freeze({ ADMIN: 'admin', MEMBER: 'member' });
function isAdmin(user) {
  return user.role === Role.ADMIN;
}
```
- **DO:** Specifically check every additional code path and every early return in generated authorization logic — a second handler for the same resource added later (a bulk-update variant, a "quick admin override" branch, an alternate API version) is a common place for a generated suggestion to omit the same check enforced on the primary path, since the assistant is often working from the immediate function in view rather than the full set of routes that touch the resource.
```python
# BAD: the ownership check present on the single-item endpoint is missing
# entirely from a bulk-update variant added later
def update_task(user, task_id, data):
    task = Task.get(task_id)
    if task.owner_id != user.id:
        raise PermissionError()
    task.update(data)

def bulk_update_tasks(user, task_ids, data):     # generated later, check omitted
    for task_id in task_ids:
        Task.get(task_id).update(data)           # no ownership check at all

# GOOD: the same authorization check is applied consistently on every path
def bulk_update_tasks(user, task_ids, data):
    for task_id in task_ids:
        task = Task.get(task_id)
        if task.owner_id != user.id:
            raise PermissionError()
        task.update(data)
```
- **DON'T:** Trust a generated authorization check that reads plausibly but checks the wrong object — verifying that the *current user* is valid/authenticated without verifying that the current user is authorized for *this specific resource* (the IDOR pattern from the Authorization section above). This mistake is easy for a generated suggestion to make because "check the user is allowed to call this endpoint" and "check the user is allowed to touch this specific record" look similar in code shape but are different checks entirely.
- **DO:** Test generated authorization code explicitly with a non-privileged user attempting the privileged action, and with one user attempting to access another user's resource by ID, rather than only testing the happy path with an authorized actor — this is the single most effective way to catch the class of mistake described above before it reaches production.

### Overly Broad CORS and Network Configuration

- **DO:** Scrutinize any generated CORS configuration for the specific combination of a wildcard or reflected origin together with credentials enabled, and correct it to an explicit origin allowlist before accepting the suggestion, as covered in the Common Web Vulnerabilities section above.
- **DON'T:** Accept a generated "quick fix" for a CORS error that sets `Access-Control-Allow-Origin: '*'` (or dynamically reflects the request's `Origin` header) alongside `Access-Control-Allow-Credentials: true`, since this exact pattern is one of the most frequent security regressions introduced by an assistant trying to make a cross-origin request error disappear as fast as possible.
- **DO:** Apply the same scrutiny to generated infrastructure-as-code suggestions that open network access broadly (a security-group rule allowing inbound traffic from `0.0.0.0/0` on a database port, an IAM policy with a wildcard `*` resource/action) when a narrower, purpose-scoped rule would satisfy the actual requirement — these show up in generated Terraform/CloudFormation snippets for the same reason overly broad CORS shows up in application code: it's the fastest way to make an access-denied error go away.
```hcl
# BAD: a generated security-group rule opens the database port to the entire internet
# to unblock a connection error, rather than scoping it to what actually needs access
ingress {
  from_port   = 5432
  to_port     = 5432
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}

# GOOD: access is scoped to the specific application security group/subnet that
# actually needs to reach the database
ingress {
  from_port       = 5432
  to_port         = 5432
  protocol        = "tcp"
  security_groups = [aws_security_group.app.id]
}
```

### Recommending Outdated or Weak Cryptography

- **DO:** Cross-check any specific algorithm, key size, or cryptographic library a generated suggestion names against current guidance, since training data spans many years and can surface an approach that was reasonable when written but has since been deprecated or shown weak.
- **DON'T:** Accept a generated suggestion to use MD5 or SHA-1 for password hashing, a small RSA key size, ECB mode, or a custom "lightweight" hashing scheme "since this is just an internal tool" — internal-only status does not change the fact that a fast, unsalted hash is crackable at scale, and this is one of the more common outdated-crypto suggestions to watch for precisely because it still appears in a large amount of older reference material and legacy example code.
- **DO:** Prefer whatever the target language ecosystem's current official documentation names as the recommended approach for a given cryptographic need over a specific code sample's remembered API usage, since library APIs and their safe-default behavior do change between versions, and a generated sample may reflect an older version's defaults.
- **DON'T:** Assume a generated code sample using `Math.random()`, `random.random()`, or an equivalent general-purpose PRNG for a token, key, or nonce is safe simply because the surrounding code otherwise looks polished and idiomatic. This specific mistake is easy to miss on a quick read because the code runs correctly and produces a value that looks like a valid token — the flaw is in the token's predictability, not in anything visibly broken about the code.

### Package and Dependency Hallucination

- **DO:** Verify every package name a generated suggestion introduces against the actual language ecosystem's official package registry before installing it, exactly as covered under Dependency Security above, treating an assistant-suggested import with the same skepticism as one copied from an unfamiliar blog post.
- **DON'T:** Run a suggested install command reflexively because the accompanying code looks otherwise correct and idiomatic. A generated suggestion can produce a fully plausible, well-formatted `import`/`require` statement for a package that does not exist, has a different real name, or — in the specific case attackers have started exploiting — exists but was published maliciously by someone anticipating exactly this kind of unverified installation.
- **DO:** Prefer a package the team has used before, or one explicitly documented in the framework's own official docs, when a generated suggestion introduces an unfamiliar dependency for a need an existing, already-vetted dependency could satisfy.

### Silently Weakening a Security Control to Make Something "Work"

- **DO:** Read the full diff of any generated change that touches authentication, authorization, cryptography, input validation, or CORS/network configuration, specifically looking for anything *removed* or *loosened*, not only what was added — a security regression introduced this way is often a deletion or a narrowed-to-broadened change, which is easy to miss when review focuses mainly on new code.
- **DON'T:** Accept a generated change that quietly loosens a validation rule, widens an accepted input format, removes a length/rate limit, or downgrades a "deny by default" branch to "allow by default" as a side effect of fixing an unrelated bug or test failure. This pattern is especially easy to miss because the stated goal of the change ("fix the failing test," "resolve the type error") is unrelated to security, so the security-relevant side effect doesn't draw the reviewer's attention the way an explicitly security-labeled change would.
```python
# BAD: a generated fix for a failing test loosens the actual validation rule
# rather than fixing the test's incorrect expectation
# before: raises ValueError for any password under 12 characters
# "fix":  the assistant lowers the application's own minimum to match a test
#         that was written incorrectly, rather than correcting the test
def validate_password(password):
    if len(password) < 6:          # weakened from 12 to make a bad test pass
        raise ValueError("too short")

# GOOD: the test's incorrect expectation is fixed; the application's actual
# security requirement is left intact
def validate_password(password):
    if len(password) < 12:
        raise ValueError("too short")
```
- **DO:** Require that any change to a security-relevant constant, threshold, or default (a token expiration time, a password minimum length, a rate limit, a permitted CORS origin list) be called out explicitly in the change description and reviewed on its own merits, rather than folded silently into a larger, differently-purposed change.
- **DON'T:** Let "the tests pass now" stand in as sufficient evidence that a security-sensitive change is correct. A test suite only verifies what it was written to check; a generated change can satisfy every existing assertion while removing a protection the test suite never happened to cover, which is exactly why security-focused test cases (an unauthorized-access test, an injection-payload test) need to exist deliberately rather than being assumed to fall out of ordinary feature tests.

### General Practice: Review AI-Generated Security-Sensitive Code Like Any Other

- **DO:** Apply the same review rigor to AI-generated code touching authentication, authorization, cryptography, input handling, or infrastructure configuration as to any human-written change in those areas — a security-focused code review pass, ideally from someone other than the person who accepted the generated suggestion, checking specifically for the patterns covered throughout this document.
- **DON'T:** Treat code as more trustworthy because it was generated quickly and reads fluently. Fluent, well-formatted, idiomatic-looking code is not evidence of correctness — it's exactly the kind of output an assistant is optimized to produce, independent of whether the underlying logic is actually secure, so confidence in the code's readability should not substitute for verifying its behavior.
- **DO:** Use static analysis, dependency scanning, and security-focused linting in CI as an automated backstop specifically because it doesn't fatigue or lose focus the way a human reviewer skimming a large generated diff can, catching the mechanical instances of these patterns (raw SQL, disabled TLS verification, hardcoded-looking secrets) even when a review pass misses one.
- **DON'T:** Ask an AI assistant to "make this security check less strict so the feature works" and accept the result without independently verifying what specific protection was removed and why that's an acceptable trade-off — if it isn't an acceptable trade-off (and for most security controls, it isn't), the actual underlying blocker needs to be fixed, not the control that correctly caught it.

### Copy-Pasting Outdated Patterns From Training Data

- **DO:** Recognize that a generated code sample reflects patterns that were common across its training data, which includes a large amount of older tutorial content, Stack-Overflow-era answers, and legacy codebases written before current best practices were established — a plausible-looking sample is not automatically a current one.
- **DON'T:** Accept a generated authentication, session-management, or cryptography example uncritically just because it matches a pattern that "looks familiar" from older documentation or a widely copied tutorial. Widely copied does not mean current or correct — some of the most persistent insecure patterns online persist specifically because they were copied so many times early on that they still dominate search results and, by extension, a large share of training data.
- **DO:** Cross-check a generated suggestion for a security-sensitive task against that specific framework's or library's *current* official documentation before accepting it, particularly for anything involving password handling, session cookies, CORS, or cryptography — these are exactly the areas where "best practice" has shifted meaningfully over the past several years while a large body of older example code has not been updated to match.
- **DON'T:** Assume a generated snippet using a deprecated framework API (an old session-handling method, a superseded cookie-flag default, a hashing function the framework itself has since flagged as insecure) is safe just because it still technically compiles/runs. A deprecated API frequently still works, sometimes indefinitely, which is precisely why deprecation warnings and current documentation — not "does it run" — are the right signal to check.

### Over-Trusting Generated Tests as Proof of Security

- **DO:** Write and review security-focused test cases deliberately and separately from functional tests — an unauthorized-access attempt, an injection payload, a CSRF-forgery attempt, an expired-token rejection — rather than assuming a generated test suite's passing status says anything about these specific risks unless such cases are actually present.
- **DON'T:** Let an AI assistant's generated test suite stand in as evidence that generated application code is secure. Generated tests are often derived from the same generated implementation's assumptions, which means a flawed assumption in the implementation (a missing authorization check, a weak validation rule) can easily be mirrored rather than caught by a test generated alongside it, since both share the same blind spot.
```python
# BAD: the generated test only validates the happy path the implementation
# itself assumes, providing no evidence the endpoint is actually secure
def test_get_invoice():
    response = client.get('/api/invoices/1', headers=auth_header(user))
    assert response.status_code == 200

# GOOD: a security-focused test explicitly checks the negative case the
# implementation needs to get right — that a different user cannot access it
def test_get_invoice_denies_other_users():
    other_users_invoice_id = create_invoice(owner=other_user).id
    response = client.get(f'/api/invoices/{other_users_invoice_id}', headers=auth_header(user))
    assert response.status_code in (403, 404)
```
- **DO:** Ask explicitly, when a generated implementation and its generated tests both pass review at a glance, "what security-relevant behavior would a bug in this code fail to be caught by these tests?" — and add the missing negative-path test rather than trusting that a green test suite implies secure behavior.

### Prompt Injection and Untrusted Content in AI-Assisted and Agentic Workflows

- **DO:** Treat content an AI assistant reads from an external, untrusted source during a task — a fetched web page, a file from an untrusted upload, a third-party API response, the body of an email or support ticket the assistant is summarizing or acting on — as data to be reasoned about, never as instructions to be followed, exactly as a web application must treat user input as data rather than as trusted code or commands.
- **DON'T:** Let an AI coding assistant or agent with tool access (able to run commands, make network calls, edit files, or take actions on a user's behalf) execute instructions that appear embedded inside content it was asked to merely read or summarize — a comment in a fetched document, hidden text on a web page, or a crafted string in a file it was asked to process. This "prompt injection" pattern is a live, actively exploited technique against AI agents specifically because it looks, to the model, like part of the task rather than an attack; a well-designed agent should apply the same trust boundary a well-designed application applies to any other untrusted input.
- **DO:** Scope what actions an AI assistant/agent is allowed to take autonomously (which commands it can run, which credentials/tool access it has, whether destructive or high-privilege actions require explicit human confirmation) proportionally to how much of its input comes from untrusted or lower-trust sources, since an agent that both reads untrusted content and holds broad unsupervised tool access is the exact combination prompt-injection attacks target.
- **DON'T:** Grant an AI-assisted workflow standing access to sensitive credentials, production systems, or irreversible actions (deleting data, sending communications, making purchases, pushing to production) without a human-in-the-loop confirmation step for anything sensitive enough that a mistaken or manipulated action would cause real harm — this applies whether the mistaken action originates from a model error or from a successful prompt-injection attempt via untrusted content the agent processed.

### Confidently Wrong Explanations of Security Behavior

- **DO:** Independently verify any claim a generated explanation or code comment makes about *why* something is secure ("this is safe because the input is already sanitized upstream," "this endpoint is protected by the framework's default CSRF middleware") rather than accepting the stated justification at face value — a generated comment can assert a security property confidently and specifically without that property actually holding in the surrounding code.
- **DON'T:** Let a generated comment claiming safety substitute for tracing the actual code path yourself. It's a well-documented failure mode for generated explanations to sound authoritative and specific while being subtly or entirely wrong about the actual guarantee in place — the confidence of the explanation carries no correlation with its accuracy, and a security review should verify the claim against the real code, not the comment describing it.
```python
# BAD: the comment's safety claim is taken at face value, but the referenced
# upstream sanitization doesn't actually cover this code path
# "input is already sanitized upstream, safe to use directly in the query"
def get_report(filter_expr):
    return db.execute(f"SELECT * FROM reports WHERE {filter_expr}")

# GOOD: the actual code path is traced, the claim is found not to hold for
# this specific caller, and the fix addresses the real gap rather than
# trusting the comment
def get_report(filter_column, filter_value):
    ALLOWED_COLUMNS = {"status", "owner_id", "created_at"}
    if filter_column not in ALLOWED_COLUMNS:
        raise ValueError("invalid filter column")
    return db.execute(f"SELECT * FROM reports WHERE {filter_column} = %s", (filter_value,))
```
- **DO:** Hold a higher bar of independent verification specifically for security-relevant claims in generated documentation, code comments, and commit messages, since these are read by future developers (and future AI assistants working on the same codebase) who will reasonably treat an explicit, confidently stated security claim as trustworthy unless something prompts them to re-check it.

### Blind Trust in Generated Validation Patterns

- **DO:** Test any generated regular expression or validation function against both valid and deliberately malformed/malicious sample inputs before accepting it, since a generated validation pattern can look reasonable while having a subtle gap — an unescaped special character, a missing anchor (`^`/`$`) that lets a disallowed prefix or suffix slip through, or a case-sensitivity assumption that doesn't match how the field is actually used.
```javascript
// BAD: the generated pattern is missing start/end anchors, so it matches
// as long as the required substring appears anywhere, not the whole string
const isValidId = (s) => /[a-f0-9]{8}-[a-f0-9]{4}/.test(s);
// "'; DROP TABLE users; --12345678-1234" passes this check

// GOOD: anchored to match the entire string, not merely find a substring within it
const isValidId = (s) => /^[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}$/.test(s);
```
- **DON'T:** Accept a generated validation pattern's stated purpose ("this validates emails," "this checks for a safe filename") without independently testing it against at least a few adversarial inputs designed to probe exactly the boundary the pattern is supposed to enforce. A pattern's docstring or surrounding comment describing what it does is a claim, not a guarantee — verify it behaves as claimed.

### Assuming Framework Defaults Are Secure Without Verification

- **DO:** Verify what a framework's or library's *actual* default behavior is for a security-relevant setting (CSRF protection, session cookie flags, CORS, debug mode) in the specific version in use, rather than accepting a generated configuration's comment claiming "this is secure by default" at face value — defaults have changed across versions of most major frameworks, and a generated suggestion may reflect an older version's defaults that no longer match what's actually installed.
- **DON'T:** Skip explicitly configuring a security-relevant setting just because a generated scaffold didn't set it, on the assumption that "the framework handles this." Some frameworks are secure by default in this area; others are not, and the only way to know for a specific project is to check that framework's current documentation for that specific setting, not to infer it from how confidently a generated comment states it.

### Generating Insecure Defaults for New Projects and Scaffolds

- **DO:** Review a freshly generated project scaffold (a new service boilerplate, a starter template, a "get started" tutorial-style setup) for exactly the same security baseline expected of any other code before it becomes the foundation other code is built on — default admin credentials, permissive CORS "to make the demo work," debug mode left on, or a placeholder secret key are common in scaffold-style generated output precisely because the goal of a quick-start scaffold is usually "run immediately," not "be production-secure."
- **DON'T:** Let a convenient starting configuration meant for a first successful run silently become the production configuration because nobody revisited it. Explicitly track which scaffold-time shortcuts (a default secret key, an open CORS policy, disabled authentication on a placeholder endpoint) still need to be locked down before the project handles real data, and treat that as a concrete checklist item, not an assumption someone will remember.

### Verifying Generated Infrastructure-as-Code Against a Security Baseline

- **DO:** Run infrastructure-as-code-specific static analysis (a policy-as-code scanner for Terraform/CloudFormation/Kubernetes manifests) against any generated IaC before applying it, the same way application code is scanned, since generated infrastructure definitions are just as capable of encoding an insecure default (a public storage bucket, an overly permissive security group, a container running as root) as generated application code is of encoding a broken auth check.
- **DON'T:** Apply a generated Terraform/CloudFormation/Kubernetes change directly to a real environment without a plan/diff review step, exactly as you would review a generated application-code diff before merging it. Infrastructure changes are often higher-blast-radius than application code changes — a single generated resource definition can expose a database, widen a network boundary, or grant broad IAM access — which makes the review step, if anything, more important here, not less.
- **DO:** Ask specifically, for any generated infrastructure change, "does this change any resource's public accessibility, any IAM policy's scope, or any network boundary?" — these are the specific categories of infrastructure change most likely to introduce a security regression, and the ones most worth a deliberate, focused look rather than a quick skim of the overall diff.

### Assuming a Prior Fix Stays Fixed

- **DO:** Re-verify a previously fixed security issue after any subsequent generated change touches the same file or function, since an AI assistant working on a later, unrelated request has no persistent memory of why a particular line looks the way it does, and can plausibly "simplify" or "clean up" a deliberately defensive piece of code back toward the insecure pattern it replaced, especially if the security-motivated reasoning wasn't captured in a comment or a test.
- **DON'T:** Assume that because a security fix was made and reviewed once, the code is permanently safe from regressing on a later edit. Protect security-critical logic with an explicit, named test asserting the specific behavior the fix established (e.g., "rejects a request from a different tenant"), so that a later change reintroducing the issue fails a test immediately rather than silently reintroducing a previously fixed vulnerability.
```python
# GOOD: a regression test names and locks in the specific fix, so a later
# generated change that reintroduces the bug fails immediately rather than
# silently regressing
def test_cross_tenant_access_denied():
    """Regression test for the tenant-isolation fix — see issue #482.
    A request scoped to tenant A must never return tenant B's data,
    even when a valid record ID from tenant B is supplied directly."""
    response = client.get(f"/api/records/{tenant_b_record.id}", headers=auth_header(tenant_a_user))
    assert response.status_code in (403, 404)
```
- **DO:** Leave a brief comment at the site of a non-obvious, security-motivated piece of code explaining *why* it exists (referencing the specific risk it addresses), specifically so a future editor — human or AI — understands the constraint before "simplifying" it away.

### The Self-Review Blind Spot

- **DO:** Get security-sensitive, AI-generated code reviewed by a different reviewer (a different person, or at minimum a distinctly separate review pass with fresh context) than whoever accepted the original generated suggestion, rather than asking the same assistant session that wrote the code whether the code it just wrote is secure.
- **DON'T:** Treat an AI assistant's own affirmative answer to "is this code secure?" as independent verification of code it just generated. A model asked to evaluate its own immediately-prior output tends to explain and justify what it already produced rather than adversarially search for what might be wrong with it — the same pattern that makes self-review generally weaker than independent review for human-written code applies here as well, and arguably more so, since the assistant has no persistent memory forcing it to reconsider assumptions it made moments earlier.
- **DO:** Prefer, when using an AI assistant for a security review pass, a fresh session or an explicit instruction to review adversarially and skeptically, over continuing straight from code generation into review in the same context — a deliberately separated review step, framed as "find what's wrong with this" rather than "confirm this is fine," produces meaningfully more scrutiny than an implicit self-check folded into the same generation flow.

## Quick Checklist
**Injection**
- Every SQL query uses parameterized queries or an ORM's safe query builder — no string-concatenated SQL, ever.
- Dynamic identifiers (table/column/sort order) are mapped through a fixed allowlist, never taken directly from user input.
- Shell commands are invoked with argument arrays (`shell=False`), never through a shell string built from user input.
- Output is encoded for its actual context (HTML body, HTML attribute, JS string, URL) — not one generic "sanitize" pass.
- Rich-text/user HTML is cleaned with a vetted sanitizer library (allowlist tags/attributes), never a hand-rolled regex filter.
- Template engines only ever render fixed template files with user data bound as variables — never compile template source from user input.
- A Content-Security-Policy restricts script sources and disallows `unsafe-inline`/`unsafe-eval`.

**Authentication**
- Passwords are hashed with Argon2id/bcrypt/scrypt — never plaintext, reversible encryption, or a bare fast hash (MD5/SHA-1/SHA-256).
- MFA is available and app-based/WebAuthn is preferred over SMS-only for sensitive accounts.
- Session IDs are high-entropy random, rotated on login/privilege change, and invalidated server-side on logout.
- Session cookies set `HttpOnly`, `Secure`, and an appropriate `SameSite` value.
- JWTs pin the expected algorithm server-side and never trust an `alg` value from the token itself; `none` is rejected.
- Secret comparisons (tokens, signatures) use constant-time comparison, not `==`.
- Login/reset/MFA endpoints are rate-limited, and login failure messages don't reveal whether an account exists.
- No home-grown authentication, session, or token scheme replaces a maintained library.

**Authorization**
- Every request re-verifies that the authenticated user actually owns/may access the specific resource by ID (no IDOR).
- Authorization logic is centralized in one policy/guard layer, not duplicated ad hoc per handler.
- Every verb (GET/POST/PUT/PATCH/DELETE) and every nested/related resource has its own ownership/permission check.
- Server-side authorization is authoritative; UI-hidden buttons/routes are never treated as a security control.
- Role/permission comparisons use strict, type-safe equality against a validated enum, never loose or coerced comparison.
- Authorization checks fail closed (deny) on any unexpected error, not fail open (allow).
- Multi-tenant queries always scope by a server-derived tenant ID, never a client-supplied one.

**Secrets Management**
- No secret, API key, or credential is hardcoded in source code, "temporarily" or otherwise.
- Secrets load from environment variables or a secret manager at runtime; `.env` files are gitignored with an `.example` committed instead.
- Any secret ever committed to version control is treated as compromised and rotated immediately.
- Logs, error messages, and crash reports redact known-sensitive fields (passwords, tokens, auth headers) before writing.
- Client-side/frontend code never ships a server-only secret; only scoped, public-safe keys reach the browser.
- CI/CD secrets use the platform's masked-secret feature, scoped narrowly, never echoed in build logs.
- Secrets are rotated on a schedule and immediately on any suspected exposure, with the old value explicitly revoked.

**Dependency Security**
- Dependencies are scanned for known vulnerabilities in CI, and high/critical findings are triaged, not ignored.
- Lockfiles are committed and installs use the lockfile-strict command (`npm ci`, not `npm install`).
- New dependencies (and their maintainers/activity) are sanity-checked before adding; package names are verified character-by-character.
- Any AI-suggested or unfamiliar package is confirmed to actually exist on the official registry before installing.
- Container base images and CI actions are pinned to immutable versions/digests, not mutable tags like `latest`.
- Install-time scripts and CLI tools from public registries are treated with the same suspicion as any other untrusted code.

**Input Validation & Sanitization**
- All input is validated server-side in full, regardless of what client-side validation already ran.
- Validation uses allowlists (what's valid) rather than denylists (what's forbidden) wherever practical.
- Uploaded files are type-checked by content sniffing, not by trusting the extension or `Content-Type` header.
- Uploads are stored under generated filenames, size-limited, and served from a cookie-less, non-executing origin.
- Request bodies bind onto models through an explicit field allowlist — no mass-assignment of arbitrary client fields.
- Untrusted data is deserialized only with safe, non-code-executing parsers (e.g., JSON), never unrestricted native serialization formats.
- Field length, numeric range, array size, and JSON nesting depth all have explicit bounds.

**Cryptography**
- All cryptographic operations go through a vetted library — no custom encryption, hashing, or key-exchange logic.
- Symmetric encryption uses an authenticated mode (AES-GCM/ChaCha20-Poly1305); ECB mode is never used.
- Security-relevant randomness (tokens, keys, nonces) comes from a CSPRNG, never a general-purpose PRNG like `Math.random()`.
- Nonces/IVs are unique per encryption operation and never reused with the same key.
- Encryption keys are stored separately from the data they protect, in a secret manager/KMS, never hardcoded or co-located.
- Base64/hex/URL encoding is never mistaken for encryption or treated as providing confidentiality.

**Transport Security**
- The entire application is served over HTTPS, with HTTP requests redirected, not just the login/payment pages.
- HSTS is sent with a meaningful `max-age` (and `includeSubDomains` where appropriate).
- TLS certificate verification is never disabled to silence an error — the actual cause (expired cert, missing CA, wrong hostname) is fixed instead.
- No page loads mixed content (HTTP subresources on an HTTPS page).
- Outdated TLS versions (SSLv2/3, TLS 1.0/1.1) and weak cipher suites are disabled on the server.

**Common Web Vulnerabilities**
- Every state-changing request is protected against CSRF (anti-CSRF token and/or `SameSite` cookies).
- Server-side fetches of user-influenced URLs validate the destination (allowlist, block private/metadata IP ranges) to prevent SSRF.
- Sensitive-action pages send `X-Frame-Options`/`frame-ancestors` to prevent clickjacking.
- Redirect targets are validated against an allowlist or restricted to relative paths — no open redirects.
- CORS uses an explicit origin allowlist; `*` is never combined with credentialed (cookie-bearing) requests.
- A baseline security header set (CSP, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`) is sent on every response.
- Expensive and authentication-related endpoints are rate-limited per account and per client, not just per IP.
- Limited-use/limited-quantity operations (coupons, inventory, one-time codes) are enforced with atomic database operations, not check-then-act code.

**Security Logging & Error Handling**
- End users see a generic error message with a correlation ID; full stack traces and internals stay server-side only.
- Debug/verbose error pages are confirmed disabled in production, not merely assumed to be.
- A dedicated, append-only-where-possible audit log records authentication events, permission changes, and sensitive-data access.
- No password, full token, or other sensitive field is ever written to logs in plaintext, including failed-login attempts.
- User-controlled values written to logs are contained as structured fields, not concatenated into free-text log lines (avoiding log injection).
- Security-relevant log patterns (repeated failed logins, spikes in denied requests) trigger automated alerts, not just passive storage.

**AI-Assistant-Specific Review Points**
- Any suggestion to disable TLS verification, catch-and-ignore a security exception, or add a lint-suppression comment is treated as a signal to find the real cause, not accepted as the fix.
- Generated code is checked for hardcoded-looking secrets before being accepted, even in "temporary" or example form.
- Generated database code is checked for raw string-built SQL when the project already has a safe ORM/query-builder path.
- Every additional or newly generated code path touching a resource (bulk endpoints, alternate API versions) gets the same authorization check as the original path.
- Role/permission comparisons in generated code use strict equality against a validated enum, not loose or user-influenced string matching.
- Generated CORS and infrastructure (security group/IAM) suggestions are checked for overly broad access used only to silence an error.
- Any algorithm, library, or random-number choice in generated crypto code is cross-checked against current guidance, not assumed correct because it looks familiar.
- Every package a suggestion introduces is verified to exist on the official registry before it's installed.
- Diffs from generated changes are read for what was *removed* or *loosened*, not only what was added, especially around auth, validation, and crypto.
