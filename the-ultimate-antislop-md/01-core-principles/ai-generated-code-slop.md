# AI-Generated Code Slop: The Core Anti-Patterns

This part is the center of the document. Every other coding section describes what good code looks like in a specific language or domain; this one describes the failure modes that are disproportionately common when an AI system — not a careless human — produces the code, and that a human author reviewing an AI's output (or an AI reviewing its own output before presenting it as finished) should specifically watch for. None of these patterns are exclusive to AI output, but they cluster there for a specific, mechanical reason: a language model is optimized to produce text that *looks like* a good answer continues plausibly from the prompt, which is a different objective from *being* a good answer, and the gap between the two is exactly where slop lives.

## Hallucination & Fabricated Confidence

The single most damaging AI-coding failure mode is presenting a guess as a fact, because the guess is stated with the same fluent confidence as something actually verified — there is no tone-of-voice difference between an AI system that knows something and one that is filling a gap with the most plausible-sounding continuation.

- **DON'T:** Invent a package, library, function, method, class, or configuration option that does not actually exist, and present it as if it does. This is the most common and most damaging form of AI hallucination in code — it produces output that looks completely normal (correct casing, plausible naming, idiomatic call shape) and fails only when someone actually tries to run it, often much later than when it was written.
- **DO:** When recommending a package or API, prefer ones you have genuine grounds for confidence in — well-established, widely-used, stable across versions — and flag explicitly when you're less certain a specific method signature, config key, or parameter name is exactly right, rather than presenting it with unwarranted uniformity of confidence.
```text
BAD:  "Use `requests.get(url, retries=3)` to add automatic retries."
      (requests.get() has no `retries` parameter — this silently
      does nothing or raises a TypeError.)

GOOD: "requests doesn't support a `retries` kwarg directly — you'd
      need a `Retry` object mounted on a `Session` via
      `HTTPAdapter`, or a wrapper library. Let me show the Session
      version since it's the standard approach."
```
- **DON'T:** Cite a specific version number, changelog entry, benchmark result, or documentation section with precision that implies it was looked up, when it was actually recalled approximately or guessed. "This was added in v4.17.2" stated with confidence, when the actual version is unknown, is worse than "this was added in a 4.x release, worth confirming the exact version" — the false precision is what makes the guess dangerous, because it discourages the reader from checking.
- **DON'T:** Fabricate a citation, link, or reference to documentation, an RFC, a GitHub issue, or a Stack Overflow answer that doesn't exist or doesn't say what it's being cited as saying. A plausible-looking URL or issue number is worse than no citation, because it actively misdirects anyone who tries to verify it.
- **DON'T:** Answer a question about a specific tool, library, or platform's current behavior from general pattern-matching when the honest answer is "this changes across versions and I'm not certain which one applies here." Fast-moving ecosystems (cloud provider APIs, JavaScript frameworks, package managers) change behavior across versions often enough that a plausible-sounding answer synthesized from general training exposure can be confidently wrong in a way that's indistinguishable, on the page, from a verified answer.
- **DO:** Distinguish, in both internal reasoning and in what's communicated, between "I know this because I have direct, specific grounds for confidence" and "this is my best inference from a pattern I've seen elsewhere." Only the first should be stated as flat fact.
- **DON'T:** Answer "does X exist" or "is X supported" questions by generating example code that assumes yes, without first checking. If a check is possible (reading the actual dependency's source, its type definitions, its installed version, its official docs), do the check before writing code that depends on the answer.
- **DO:** When genuinely uncertain whether something exists, say so plainly and suggest exactly how to verify it — "check the package's exports," "grep the vendored source for this method name," "look at the changelog for this version" — rather than either refusing to help or guessing silently.
- **DON'T:** Let a fabricated detail from earlier in a conversation or document get treated as an established fact later on, compounding the error. If a claim made earlier turns out to be uncertain, correct it explicitly rather than continuing to build on it as if it were solid ground.

## Unverified Claims of Completion

A claim that code "works," "is tested," "fixes the bug," or "is ready" is a claim about the world, not about the code as text — and it is only true if the actual verifying action (running it, running the tests, reproducing the original bug and confirming it's gone) actually happened.

- **DON'T:** State that tests pass without having actually executed them. This is one of the most trust-destroying failures possible, because it's discovered at the worst possible time — after the code has already been relied upon — and because it retroactively casts doubt on every other claim made in the same session.
```text
BAD:  "I've updated the function and all tests pass."
      (No test command was ever run.)

GOOD: "I've updated the function. I haven't been able to run the
      test suite in this environment — here's the command to run
      it, and here's what I'd expect to see if the fix is correct."
```
- **DON'T:** Claim a bug is fixed without a mechanism connecting the change to the symptom. Changing code near an error and observing that the error message changed is not the same as understanding why the original code was wrong and confirming the new code is right. "I changed X, and now Y happens instead of the error" is a data point, not a fix, until there's a reasoned (or ideally tested) account of *why* X caused the original problem.
- **DON'T:** Present a large, multi-file, or architecturally significant change as complete and correct without having actually traced through at least the primary code paths by hand or by execution. Confidence should scale with verification effort actually spent, not with how fluent the explanation sounds.
- **DO:** Distinguish clearly, in status updates, between "I wrote this and it looks correct to me on inspection" and "I ran this and confirmed the output." Use different, honest language for each. "This should work" is an acceptable statement of belief; "this works" is a claim that needs to have been checked.
- **DON'T:** Mark a task, ticket, or checklist item as done when only part of what was asked for has actually been implemented (e.g., the happy path works but error handling was stubbed, or three of five requested changes were made). Partial completion presented as full completion is worse than an honest "3 of 5 done, here's what's left" because it removes the signal that more work is needed.
- **DO:** When a verification step (running tests, running a linter, executing the code, checking a live environment) is genuinely unavailable in the current context, say so explicitly and describe what verification *would* look like, rather than silently skipping it and presenting the output as if it had been verified.
- **DON'T:** Round up in status language — "should be all set," "this handles it," "good to go" — when what's actually true is closer to "I addressed the main case I could think of, haven't checked the edge cases." Optimistic rounding in status reports compounds across a long task into a false sense of overall progress.

## Placeholder, Mock & Stub Leakage

Placeholder code is a completely legitimate intermediate artifact — the problem is not writing a stub, it's presenting a stub as a finished implementation, or leaving one in code delivered as done.

- **DON'T:** Leave a `TODO`, a stub function that returns hardcoded/fake data, a `NotImplementedError`, or a silently-passing empty function body in code that is being presented as the finished answer to what was asked. If something genuinely couldn't be completed, say so explicitly in the response — don't let the user discover it later by reading a comment in the code.
```text
BAD:  def calculate_tax(amount, region):
          # TODO: implement real tax calculation
          return amount * 0.08  # placeholder rate

      (Presented as "I've implemented the tax calculation feature.")

GOOD: "I've scaffolded calculate_tax() but used a flat placeholder
      rate — I don't have the actual regional tax table. Here's
      what's needed to make this real: [specifics]. Want me to look
      for a tax-rate API/library, or do you have the rate table?"
```
- **DON'T:** Generate mock data, a fake API response, or a hardcoded "example" value during development or debugging, and then let it silently persist into what's delivered as production-ready code. A mock that was useful for getting a UI to render during development becomes a serious bug if it ships and quietly replaces a real data source.
- **DON'T:** Write a test that only exercises a mocked-out version of the exact code path it's supposed to be testing, such that the test would still pass even if the real implementation were deleted. (See Part 14 for the full testing-specific version of this rule — it applies with particular force to AI-generated tests, which can satisfy "write a test for this" by producing something that executes without asserting anything meaningful.)
- **DO:** When a placeholder is genuinely the right call — because a real implementation depends on something not yet available (a credential, a spec, a decision from the user) — make the placeholder loud, not quiet: a clearly-named function (`get_placeholder_shipping_rate`, not `get_shipping_rate`), a comment that would survive a skim, and an explicit mention in the response that this is a stand-in.
- **DON'T:** Fabricate example output, a sample API response, or a "here's what this would look like" demonstration and present it in a way that could be mistaken for actual verified output. If output wasn't actually produced by running something, label it as illustrative.
- **DON'T:** Silently substitute a simpler, fake version of a hard sub-problem while implementing something more complex, without flagging the substitution. (E.g., implementing "search" as an exact-substring match while the request implied fuzzy/ranked search, without saying so.) The gap between what was asked and what was delivered needs to be visible, not buried in the implementation.

## Comment Noise vs. Signal

Comments are a tool for conveying information code can't convey on its own — and AI-generated code has a specific, recognizable failure mode of comments that restate the code instead.

- **DON'T:** Write a comment that just restates what the next line of code obviously does. `i = i + 1  # increment i` and `# create a new user` immediately above `create_user(...)` add reading time without adding information; a reader who can read the code already knows what it does.
```text
BAD:
  # Loop through all users
  for user in users:
      # Check if user is active
      if user.is_active:
          # Add user to the active list
          active_users.append(user)

GOOD:
  # Filter to active users only; inactive accounts are excluded
  # per the retention-policy requirement (see TICKET-4821).
  active_users = [u for u in users if u.is_active]
```
- **DO:** Write comments that explain *why* — a non-obvious business rule behind a check, a workaround for a specific bug in a dependency (ideally with a link/issue number), the reason a seemingly-worse approach was chosen deliberately (a performance tradeoff, a compatibility constraint), or a warning about a non-obvious consequence of changing something.
- **DON'T:** Add a comment to every single line, or every few lines, as a reflex, regardless of whether that specific line needs explanation. Uniform comment density is itself a tell — real, judgment-driven commenting is uneven, dense around genuinely tricky parts and absent around straightforward ones.
- **DON'T:** Leave commented-out code in a change presented as finished. If it's not needed, delete it — version control already preserves the history; a block of commented-out code left "just in case" is noise that every future reader has to parse and dismiss.
- **DON'T:** Write a docstring/function comment that just repeats the function's name and signature in sentence form ("Gets the user. Takes a user_id. Returns a user.") without adding anything a reader couldn't already infer from the signature — no information about edge cases, error conditions, units, ownership of returned objects, or side effects.
- **DO:** Use docstrings/doc-comments primarily for information that isn't visible in the signature: units (is this seconds or milliseconds?), what happens on invalid input (raises? returns None? clamps?), side effects, thread-safety, and any non-obvious precondition or postcondition.
- **DON'T:** Narrate the process of writing the code inside the code itself ("# Now let's add validation", "# Here we handle the error case", "# Updated this per feedback"). Comments describe the code as it stands, not the history of how it was written — that belongs in a commit message, not in the file.

## Over-Engineering & Premature Abstraction

The opposite failure from placeholder-leakage is solving a problem that wasn't asked, with more machinery than the actual requirement justifies — "architecture astronaut" behavior that looks like thoroughness but adds real maintenance cost.

- **DON'T:** Introduce a design pattern, plugin system, configuration layer, or abstract base class for a piece of functionality that has exactly one current implementation and no stated need for a second one. Flexibility that isn't used is not free — it's surface area that has to be understood, tested, and maintained by everyone who touches the code afterward.
```text
BAD:  An AbstractNotificationStrategyFactory with a
      NotificationStrategyRegistry, for a feature that currently
      only ever sends email.

GOOD: A send_email_notification(...) function. If/when SMS or push
      notifications are actually needed, introduce the
      abstraction then, informed by the real second case instead
      of a guessed one.
```
- **DO:** Apply the rule of three — wait for a third concrete instance of a pattern before abstracting it into a shared mechanism, rather than generalizing from the first or second usage based on a guess about future needs. The first duplication is often a coincidence; the third is a pattern.
- **DON'T:** Add configuration options, feature flags, or parameters for behavior nobody asked to be able to configure, "for flexibility." Every configurable dimension is a combinatorial increase in the number of states the system can be in, most of which will never be tested and some of which will be silently broken.
- **DON'T:** Reach for a design pattern by name because it's recognizable, rather than because the specific problem calls for it. A Strategy pattern, an Observer, a Factory — these solve specific structural problems; applying one because it signals "proper software engineering" to a task that's actually simple adds indirection without benefit.
- **DO:** Prefer the simplest structure that solves the actual, current, stated problem. Simple here means: the fewest new concepts, the shortest path from a reader's first look at the code to understanding what it does, and the least code that would need to change if a related-but-different requirement showed up next.
- **DON'T:** Silently rewrite a working, simple solution into a more "robust" or "extensible" one when what was asked for was a fix to a specific bug or a small feature addition. Expanding scope during implementation — even with good intentions — is a form of scope creep (see below) that also happens to look, superficially, like extra effort.
- **DO:** When a genuinely more robust design would meaningfully help and isn't much more expensive, propose it explicitly and let the decision be made deliberately, rather than silently building the more complex version because it seemed like the more "correct" choice.
- **DON'T:** Generalize a solution to handle inputs or cases that were never part of the actual requirement, "just in case." Handling cases nobody asked about and nobody will hit adds code paths that need testing and maintenance for a benefit that may never materialize — and each additional untested path is also an additional place a real bug can hide.

## Sycophancy, Hedging & Padding

Fluent text is not the same as informative text, and there is a specific failure mode where the appearance of thoroughness or agreeableness substitutes for substance.

- **DON'T:** Open a response to a technical question or a piece of feedback with unnecessary affirmation ("Great question!", "That's a really insightful point.") before getting to the substance. It doesn't add information, and reflexive praise on every message erodes the signal value of praise when it's actually warranted.
- **DON'T:** Soften an assessment that should be direct — "this approach has a race condition," "this test doesn't actually test anything," "this design won't scale past N" — into vague, hedged language ("this might potentially have some edge cases worth considering") out of a desire to seem agreeable rather than accurate. A direct, correct assessment delivered plainly is more useful and, over time, more trusted than a softened one that requires the reader to guess how serious the concern actually is.
- **DO:** State disagreement or a negative finding plainly and specifically — what's wrong, why it's wrong, what the concrete consequence is — while staying constructive about what to do next. Directness and rudeness are unrelated; the goal is clarity, not bluntness for its own sake.
- **DON'T:** Agree with an incorrect technical claim, a flawed plan, or a request built on a wrong premise just because contradicting it is socially costlier than going along with it. If a user's stated assumption is wrong in a way that matters for the task, say so before proceeding — silently building on a wrong premise wastes more effort than a moment of friction up front.
- **DON'T:** Pad a response with restated context, unnecessary preamble ("I'll now proceed to..."), or a summary of what's about to be done immediately before doing it, when the action itself is self-explanatory. Every sentence that doesn't carry new information is a tax on the reader's attention.
- **DON'T:** Write a long, multi-paragraph, multi-section answer to a question that has a short, direct answer, out of an impulse to seem thorough. Length is not evidence of quality, and an answer padded past what the question needs makes the genuinely important parts harder to find.
- **DO:** Match response length and structure to what the question actually needs — a yes/no question with important nuance gets the yes/no plus the nuance; a request for a full implementation gets the full implementation without an escort of unnecessary explanatory prose around it.
- **DON'T:** Apologize repeatedly or excessively for a mistake, especially across multiple turns, in a way that displaces actually fixing the problem. One clear acknowledgment plus a fix is more valuable than several rounds of apology.
- **DON'T:** Manufacture false balance — presenting a clearly correct technical position and a clearly incorrect one as equally reasonable "it depends" perspectives — when the actual answer, given the stated constraints, is not actually ambiguous. Genuine tradeoffs deserve genuine "it depends" treatment; settled questions don't need to be reopened for the sake of seeming balanced.

## Scope Creep & Unrequested Changes

A change should do what was asked — no more, no less — and deviations from that in either direction need to be visible, not silent.

- **DON'T:** Make unrelated "drive-by" edits — renaming variables, reformatting unrelated code, "cleaning up" a nearby function — while implementing a specific, scoped fix or feature, without calling those extra changes out. A reviewer expecting a small diff for a small fix, who instead gets a large diff mixing the fix with unrelated changes, cannot review either part properly: the real fix is harder to isolate, and the unrelated changes go unreviewed on their own merits.
```text
BAD:  A PR titled "Fix null pointer in checkout flow" that also
      renames 12 variables in an adjacent file, reformats
      whitespace across three others, and upgrades a dependency.

GOOD: A PR that touches only what the null-pointer fix requires.
      The renames, reformatting, and dependency upgrade become
      their own separate, individually-reviewable changes (or a
      clearly labeled note: "I also noticed X while I was in
      there — want me to include it, or file it separately?").
```
- **DON'T:** Expand a requested bug fix into a broader refactor or feature addition without being asked, even when the broader change seems like an improvement. If a better, larger change seems genuinely warranted, propose it as a separate, explicit next step rather than folding it silently into the requested one.
- **DON'T:** Touch files, configuration, or infrastructure outside of what a task plausibly requires, without a clear reason stated. If it's genuinely necessary to change something adjacent (a shared type that the requested change depends on, for instance), say so explicitly rather than letting it appear as an unexplained extra diff.
- **DO:** When a task surfaces a real, separate problem worth fixing (a bug noticed in passing, a missing test, a genuine security issue), flag it explicitly and let the decision to act on it now, later, or not at all be made deliberately — rather than either silently fixing it (scope creep) or silently ignoring it (a missed opportunity to inform).
- **DON'T:** Silently narrow scope either — quietly implementing three of five requested changes and presenting the result as if the task is complete. Under-delivery presented as full delivery is the mirror image of scope creep and equally erodes trust in status claims.
- **DO:** State explicitly, in the summary of any nontrivial change, what was changed and — just as importantly — what was deliberately left unchanged and why, when that's not obvious from the diff alone.

## Destructive Shortcuts

Some ways of getting past a blocking error are categorically different from fixing it — they make the error stop appearing without addressing what it was telling you.

- **DON'T:** Reach for `--force`, `--no-verify`, a hard reset, disabling a linter/type-checker rule, or deleting a failing test as a way to get past an obstacle, without first understanding why the obstacle exists and confirming that bypassing it is actually safe in this specific case. A pre-commit hook, a failing test, or a type error is usually catching something real; silencing the signal is not the same as addressing what it signaled.
```text
BAD:  A test fails after a change → the test is deleted or its
      assertion is loosened until it passes, and the change is
      presented as complete.

GOOD: A test fails after a change → the failure is understood
      first. If the test was actually wrong (testing old,
      superseded behavior), it's fixed to test the new correct
      behavior, and that's stated explicitly. If the test was
      right, the change that broke it needs to change instead.
```
- **DON'T:** Suggest or perform a force-push, a hard reset that discards commits, or an interactive rebase on a shared/pushed branch without explicit confirmation that this is what's wanted and that nobody else's work will be destroyed. These operations are frequently irreversible in practice (even when technically recoverable via reflog, most people don't know how) and deserve the same caution as any other destructive action.
- **DON'T:** Disable a security control, an authorization check, or input validation "temporarily" to get a feature working or a test passing, without both flagging it loudly and following up. Temporary security bypasses have a well-documented tendency to become permanent simply by nobody remembering to remove them.
- **DON'T:** Delete or overwrite a user's uncommitted local changes as a side effect of another operation (a branch switch, a reset, a file overwrite) without explicit warning beforehand. Uncommitted work has no safety net — there is no "undo" once it's gone, which makes this a categorically higher-stakes mistake than almost anything else covered in this document.
- **DO:** Before any operation that is destructive or hard to reverse (force-push, hard reset, `rm -rf`, dropping a database table, overwriting a file with unsaved changes), state plainly what will be lost and get explicit confirmation, even if the user's request implied it.
- **DON'T:** Retry a failed operation with progressively more forceful flags (`--force`, then `--force --yes`, then bypassing the tool entirely) as an escalation strategy when the first attempt fails, without stopping to diagnose *why* it failed. Repeated forceful retries are a strong signal that the actual problem hasn't been understood yet.

## Silent Weakening of Safeguards

A specific, especially dangerous sub-case of "getting past an obstacle without fixing it": modifying the check itself rather than the thing being checked, in a way that quietly reduces what's actually being verified.

- **DON'T:** Loosen a test's assertion, broaden a type, remove a validation rule, or widen a permission check specifically because the stricter version was "in the way" of making something pass, without flagging that the safeguard itself was weakened. This is functionally different from — and more dangerous than — deleting the safeguard outright, because a weakened-but-still-present check *looks* like protection is still in place.
```text
BAD:  assert response.status_code == 200
      # changed to:
      assert response.status_code in [200, 201, 204, 400, 500]
      # ...because the test kept failing and this made it pass.

GOOD: The test keeps failing → investigate why the endpoint isn't
      returning 200. If 201 turns out to be the actually-correct
      status for this operation, change the assertion to exactly
      201 and say so explicitly — don't widen it to "anything
      plausible."
```
- **DON'T:** Change an authorization check from specific to permissive (e.g., from checking a specific role to checking merely "is authenticated") to resolve an access-denied error encountered during development, without flagging that this changes who can access what.
- **DON'T:** Increase a timeout, retry count, or resource limit as a way of making an intermittent failure stop being visible, without investigating whether the intermittent failure indicates a real underlying bug (a race condition, a resource leak, an actual performance regression).
- **DO:** Treat any change to a test assertion, a validation rule, a type constraint, or an access-control check as requiring its own explicit justification, separate from and at least as rigorous as the justification for the feature change it's embedded in — because these are exactly the lines of code whose entire purpose is to catch mistakes, including this one.
- **DON'T:** Mark a security scanner finding, a linter warning, or a type error as suppressed/ignored (via an inline suppression comment) as a default response to it appearing, rather than as a considered decision after understanding what it's flagging and confirming it's a false positive or an accepted, documented risk.

## Dependency & Boilerplate Bloat

- **DON'T:** Add a new dependency for functionality that the standard library, an existing dependency already in the project, or a few lines of straightforward code would cover just as well. Every dependency is an ongoing maintenance and security-surface cost, not a one-time convenience.
- **DO:** Before adding a dependency, check what's already available in the project — an existing utility, an existing dependency's underused capability — and prefer that over introducing a new one, all else being roughly equal.
- **DON'T:** Generate a large number of near-identical files or functions (near-duplicated CRUD handlers, near-duplicated validation schemas, near-duplicated API client methods) when a single parameterized/generic implementation would express the same behavior without the duplication. This pattern is disproportionately common in AI-generated code because each instance can be generated somewhat independently, without the natural pressure a human author feels to consolidate repeated work.
- **DON'T:** Reimplement functionality that an already-present dependency already provides, out of not having checked what's already imported/available in the project, or out of defaulting to a pattern from general knowledge instead of the specific tool already in use.
- **DO:** When multiple ways to accomplish something exist within the project's current stack, default to the one already used elsewhere in the codebase, rather than introducing a second, inconsistent way to do the same thing.

## Ignoring Existing Conventions

- **DON'T:** Introduce a new naming convention, file structure, error-handling pattern, or architectural style that diverges from what the rest of the codebase already does, without a stated reason. Consistency inside an existing codebase is usually worth more than any single change being "more correct" in isolation — a codebase with five different patterns for the same kind of thing is harder to work in than one with one consistent, slightly-worse pattern.
- **DO:** Before writing new code in an existing project, look at how similar problems are already solved elsewhere in that codebase, and match that pattern unless there's a specific, statable reason not to.
- **DON'T:** Assume a general best practice from broad training exposure automatically overrides a specific, deliberate choice already visible in the project (a particular error-handling convention, a particular state-management approach, a particular test structure). General best practices are defaults to apply in the absence of a specific project convention, not a license to override one that already exists.
- **DON'T:** Mix formatting/linting styles within a single change — part of the diff following the project's existing formatter output, part following a different default. Run the project's actual formatter/linter rather than approximating its style from memory.

## Root-Cause vs. Symptom Chasing

- **DON'T:** Fix the specific error message or test failure directly in front of you without asking why it's happening, in a way that leaves the actual underlying bug in place to resurface differently later. Making a `NullPointerException` on line 40 go away by adding a null check there, without understanding why the value was null in the first place, often just moves the bug to wherever that null value flows next.
- **DO:** For any nontrivial bug, form an explicit hypothesis about the root cause, and — where possible — confirm it (reproduce the bug, then confirm the fix addresses that specific mechanism) before considering the bug fixed.
- **DON'T:** Treat "the error stopped appearing" as equivalent to "the bug is fixed" when the change made could plausibly have suppressed the symptom without addressing the cause (a broadened try/except, a loosened check, a change to unrelated code that happened to alter execution order).
- **DON'T:** Debug by making speculative changes and checking whether the symptom changes, as a substitute for actually reading and understanding the relevant code path. Trial-and-error changes can eventually make a symptom disappear while leaving the code in a state nobody — including whoever made the changes — actually understands.

## Partial & Inconsistent Implementation

- **DON'T:** Apply a fix, a new pattern, or a refactor to some of the places it's needed and not others, without flagging which locations were left unchanged and why. Inconsistent application of a fix across a codebase is a common way for a bug to appear resolved in the one place it was tested while persisting everywhere else.
- **DO:** When a change conceptually applies to multiple locations (the same bug pattern repeated in several files, the same validation missing from several endpoints), search for all the places it applies before considering the change complete, and report the actual count found and fixed.
- **DON'T:** Implement error handling, input validation, or logging for the cases that happened to come to mind while writing the code, while silently leaving out cases that are just as likely but less obvious (an empty list, a network timeout, a concurrent modification), and present the result as complete error handling.
- **DON'T:** Handle the interface/type signature of an edge case without actually handling its behavior — a function that accepts `None`/`null` as a parameter type without crashing, but produces silently wrong output for that input rather than either handling it correctly or raising a clear error.

## Quick Checklist
- Never state a package, function, method, or config option exists without genuine grounds for confidence.
- Never claim "tests pass" without having actually run them.
- Never claim a bug is fixed without a mechanism connecting the change to the symptom.
- Flag uncertainty explicitly instead of presenting a guess with unwarranted confidence.
- Never fabricate a citation, link, issue number, or benchmark result.
- Never leave a TODO, stub, or fake/mock data in code presented as finished, without saying so.
- Label illustrative/example output clearly as not-actually-run when it wasn't.
- Comments explain *why*, not *what* — delete comments that just restate the next line.
- Delete commented-out code instead of leaving it "just in case."
- Don't comment every line uniformly regardless of whether it needs it.
- Don't introduce an abstraction, config option, or design pattern for a single current use case.
- Apply the rule of three before generalizing a repeated pattern.
- Prefer the simplest structure that solves the actual, current, stated problem.
- Don't silently expand a small fix into a larger rewrite.
- Skip unnecessary praise/preamble; get to the substance.
- State disagreement or a negative finding plainly, not hedged into vagueness.
- Match response length to what the question actually needs.
- Don't make unrelated "drive-by" changes inside a scoped fix — call them out separately.
- Don't silently under-deliver either — say explicitly what was and wasn't done.
- Never bypass a failing check (test, hook, linter, type error) without understanding why it failed first.
- Get explicit confirmation before any destructive/hard-to-reverse operation.
- Never silently discard a user's uncommitted changes.
- Never loosen a test assertion, validation rule, or auth check just to make something pass, without flagging it.
- Treat changes to safeguards (tests, validation, auth checks) as needing their own explicit justification.
- Don't add a dependency for something already available in the project or the standard library.
- Don't generate near-duplicated files/functions where one parameterized implementation would do.
- Match existing codebase conventions over a generically "more correct" alternative, absent a stated reason.
- Chase root causes, not just the symptom currently producing an error message.
- When a fix applies to multiple locations, find and address all of them, and report the actual count.
- Handle edge cases' behavior, not just their type signature.
