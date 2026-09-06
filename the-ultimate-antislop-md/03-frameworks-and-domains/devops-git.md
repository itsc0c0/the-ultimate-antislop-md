# DevOps, CI/CD, Docker, Kubernetes, Infrastructure as Code & Git

## Git & Version Control

### Commit Message Conventions

- **DO:** Write commit subject lines in the imperative mood. "Add retry logic to the payment client" reads as an instruction applied to the tree, not a diary entry, matching git's own framing ("This commit will...") and keeping `git log --oneline` scannable and consistent.
- **DO:** Explain *why* a change was made in the commit body, not just *what* changed. The diff already shows the what; the body's only job is to preserve reasoning, the alternative that was rejected, or the bug being fixed, so a future reader isn't left reverse-engineering intent from code alone.
```
fix(payments): retry transient gateway timeouts

The payment gateway occasionally returns a 504 under load,
which we previously surfaced to the user as a failed charge.
Add three retries with exponential backoff before failing,
since a 504 from this gateway is almost always transient
(confirmed with their status page history over the last month).

Fixes #4213
```
- **DO:** Keep the subject line under roughly 50-72 characters and separate it from the body with a blank line. Terminals, `git log --oneline`, and PR list views all truncate long subjects, so front-load the summary and push detail into the body where it has room.
- **DO:** Adopt Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`, `test:`, `perf:`, `ci:`, `build:`) when the repo already uses them, or when the team wants automated changelog and semver generation from commit history. The type prefix lets tooling like semantic-release categorize and version a change without a human reading every diff.
```
feat(auth): add refresh-token rotation

fix(cart): prevent negative quantity on rapid double-click

docs(readme): document required environment variables
```
- **DON'T:** invent a Conventional Commits type that isn't part of the convention the repo has actually adopted. Using `update:` or `tweak:` when the team's tooling only recognizes `feat/fix/chore/docs/refactor/test/perf/ci/build/revert` silently breaks automated changelog and version-bump generation; stick to the project's declared type list.
- **DO:** Reference the issue or ticket ID in the commit message or the PR, rather than pasting the entire ticket description into the subject line. This preserves traceability between code and tracking system without turning `git log` into a support-ticket dump.
- **DON'T:** write vague, content-free commit messages as final history — "fix bug", "wip", "updates", "asdf". These convey nothing to a future `git blame`/`git log` reader; if you need to checkpoint quickly while working, squash or reword the message before the branch is merged.
- **DO:** Make each commit a single logical change. A commit that both renames a variable across the codebase and fixes an unrelated race condition is really two commits pretending to be one — hard to review, hard to selectively revert, and hard to `git bisect`.
- **DO:** Use `git commit --amend` or an interactive rebase to clean up "oops"/"typo"/"address feedback" checkpoint commits before opening a PR, as long as the branch hasn't been shared yet. Leaves a readable history for reviewers instead of forcing them to read your debugging process commit-by-commit.
- **DON'T:** rewrite commit history that has already been pushed and pulled by others, without coordinating first. Rewriting shared history forces every collaborator to manually reconcile a diverged branch, or silently lose commits that were based on the old history; discuss it and use `--force-with-lease` if a rewrite is genuinely necessary.
- **DO:** Use `git commit --fixup=<sha>` paired with `git rebase --autosquash` to address review feedback on a commit further back in the branch. This keeps the fix attached to its logical commit in the final history instead of tacking an unrelated "address review comments" commit onto the end.
```
git commit --fixup=a1b2c3d
git rebase -i --autosquash origin/main
# the fixup commit is automatically reordered and squashed
# into a1b2c3d during the interactive rebase
```
- **DO:** Sign commits (`git commit -S`, or use SSH-based signing) when the project or organization requires signed commits for supply-chain integrity. This provides cryptographic assurance of authorship on a shared, security-sensitive codebase, which matters most for infrastructure and release-critical repos.
- **DON'T:** bundle unrelated changes — a repo-wide formatting reflow plus an actual feature — into one commit. Reviewers can't distinguish the substantive diff from noise, and `git blame` becomes useless on the touched lines going forward; do a pure reformat as its own isolated commit or PR.
- **DO:** Mark breaking changes explicitly, with a `BREAKING CHANGE:` footer or a `!` after the type in Conventional Commits (`feat!:`). Automated release tooling and downstream consumers rely on this exact marker to bump a major version and to surface migration notes.
```
feat(api)!: remove deprecated v1 /users endpoint

BREAKING CHANGE: clients must migrate to /v2/users; the v1
endpoint returned inconsistent pagination and is removed.
```

### Branching Strategies

- **DO:** Choose a branching strategy — trunk-based development, GitHub Flow, or Git Flow — deliberately, based on release cadence and team size, and write it down. An undocumented, ad-hoc mix of long-lived branches and inconsistent merge habits is what actually produces slop history; the specific strategy matters less than everyone consistently following the same one.
- **DO:** Prefer trunk-based development — short-lived feature branches merged into `main` frequently, often behind feature flags — for teams doing continuous deployment. It minimizes merge conflicts and keeps `main` continuously close to what's actually running in production.
- **DO:** Reserve Git Flow's long-lived `develop`/`release`/`hotfix` branch structure for products with scheduled, versioned releases — shipped software, mobile apps with app-store review cycles, embedded firmware. Its overhead earns its keep exactly where a release train and a distinct hotfix cadence are real constraints, not on a continuously deployed web service.
- **DON'T:** let feature branches live for weeks without rebasing or merging from `main`. Long-lived branches accumulate divergence, guarantee a painful eventual merge (or worse, a silent semantic conflict that merges cleanly but breaks at runtime), and defeat the entire point of continuous integration.
- **DO:** Merge or rebase `main` into an active long-running feature branch regularly — at least daily. This keeps the eventual merge small and surfaces conflicts while the context is still fresh, instead of after weeks of parallel drift.
- **DO:** Use feature flags to decouple "merged to main" from "released to users" when a feature spans multiple PRs. This lets a team keep trunk-based development even for a large, multi-week feature, without resorting to a long-lived integration branch.
- **DON'T:** name branches uselessly — `patch`, `fix2`, `test`, `mybranch`. A branch name should describe what it does (`fix/checkout-null-pointer`, `feat/oauth-refresh-tokens`) so it's identifiable in `git branch`, in CI logs, and in the open-PR list.
- **DO:** Delete branches after merge, both locally and remotely, and configure the repository to auto-delete merged branches. An ever-growing pile of stale branches makes `git branch -a` useless and obscures which work is actually still in flight.
- **DO:** Protect `main` and release branches with required reviews, required status checks, and disallowed direct pushes.
```
# Example: configuring branch protection via GitHub CLI
gh api repos/OWNER/REPO/branches/main/protection \
  --method PUT \
  --field required_status_checks[strict]=true \
  --field required_status_checks[contexts][]=ci/test \
  --field enforce_admins=true \
  --field required_pull_request_reviews[required_approving_review_count]=1
```
- **DON'T:** rebase a branch that other people are actively pulling from, without an explicit agreement. Treat shared branches as append-only by default; rewriting one that others have based work on forces them into painful, unnecessary history reconciliation.
- **DO:** Keep `main` always in a deployable state. Trunk-based development and continuous deployment both depend on this invariant — if `main` can be broken, every feature flag and every deploy pipeline built on top of it becomes unreliable.

### Pull Request Hygiene

- **DO:** Keep PRs small and focused on one logical change. Small PRs get reviewed faster and more thoroughly, and are far easier to revert cleanly if something goes wrong; a 3,000-line PR gets rubber-stamped, not actually reviewed.
- **DO:** Write a PR description that states what changed, why, and how to verify it — including screenshots or a before/after for UI changes and explicit test steps for behavioral changes. The diff mechanically shows the "what"; the description is the only place intent and verification steps get carried forward.
```
## What
Adds retry-with-backoff to the payment gateway client.

## Why
Gateway occasionally returns transient 504s under load (see #4213).
Previously we surfaced these directly as failed charges to the user.

## How to verify
1. Run `make test-payments`
2. In staging, use the `force-504` test card to trigger the gateway's
   simulated timeout and confirm the charge succeeds on retry.

## Rollout
Behind `PAYMENTS_RETRY_ENABLED` flag, defaults to off.
```
- **DON'T:** mix a refactor with a behavior change in the same PR. A reviewer can't tell if a diff line is "moved code" or "changed logic" when both happen simultaneously; split refactor-only PRs from behavior-change PRs so each is independently reviewable and revertible.
- **DO:** Self-review your own diff before requesting review. This catches leftover debug statements, commented-out code, and obvious mistakes before spending a reviewer's time on them.
- **DON'T:** let scope creep into an in-flight PR — "while I'm here, let me also refactor X." Unrelated changes inflate the diff, slow review, and make it hard to isolate the change that caused a regression later; open a separate PR or ticket instead.
- **DO:** Link the PR to its tracking issue and fill in required template fields — testing notes, rollout plan, screenshots for UI changes. This gives reviewers and future archaeologists the context that code alone won't carry.
- **DO:** Keep the PR up to date with its target branch — merge or rebase `main` in — before requesting final review or merge. This prevents merging code that was reviewed against a now-stale base and silently reintroducing an already-fixed bug.
- **DON'T:** force a reviewer to approve a PR with failing CI "because it's probably just flaky." Fix it, or explicitly acknowledge and re-run before merge; treat a red required check as a blocking signal, not decoration to be scrolled past.
- **DO:** Mark a PR as draft while it's still in progress, and convert it to "ready for review" only once it's actually done. This signals to both reviewers and CI-resource-conscious teams what genuinely needs attention right now.
- **DO:** Respond to every review comment — with a code change, a reply, or an explicit "won't fix, because..." — rather than silently pushing new commits. Silent pushes force reviewers to re-diff the entire PR to figure out what changed, and leave feedback looking ignored.
- **DON'T:** squash-merge a PR whose individual commits told an important story — for example, a migration deliberately split across several reviewable steps — without considering whether that structure is worth keeping. Squash-merge is a reasonable default, but a deliberately structured multi-commit PR sometimes deserves a merge commit that preserves it.
- **DO:** Set a practical size budget for review — for example, aiming under roughly 400 changed lines excluding generated files and lockfiles — and split larger changes into a stacked sequence of PRs. Reviewer attention and defect-detection rate both drop sharply once a diff exceeds what a person can hold in working memory.
- **DO:** Include a rollback or rollout note in the PR description for anything touching production behavior — migrations, feature flags, config changes. This saves the on-call engineer from reverse-engineering a safe rollback path during an actual incident.
- **DON'T:** approve a PR you didn't actually read, just to unblock a teammate. A rubber-stamp approval defeats the purpose of code review entirely and puts genuinely unreviewed logic into a codebase everyone treats as "reviewed."

### Code Review Etiquette

- **DO:** Review intent and design first, then style and nits. Catching a fundamentally wrong approach on line 400 after nitpicking whitespace on lines 1-399 wastes everyone's time; skim for architecture before diving into line-level detail.
- **DO:** Phrase feedback as questions or suggestions about the code, not judgments about the author. "What happens if `items` is empty here?" lands better than "This is wrong," and keeps the review focused on the artifact, not the person who wrote it.
- **DO:** Distinguish blocking feedback from optional nits explicitly — for example, prefixing nit-level comments with `nit:` or `optional:`. This lets the author triage quickly instead of guessing which nine of your twelve comments are actually must-fix.
```
nit: could inline this variable, it's only used once
blocking: this branch never releases the lock on the error path —
  a request that throws here will deadlock the next caller
```
- **DON'T:** block a PR over a pure style preference that isn't codified in the project's linter or style guide. If it matters enough to block a merge, it belongs in the linter config applied consistently to every PR, not in ad-hoc review opinions that vary by which reviewer you happened to get.
- **DO:** Time-box review turnaround — for example, same-day for small PRs — and communicate proactively if you can't review promptly. Slow review is one of the biggest hidden costs in a team's delivery cycle; a well-written PR sitting unreviewed for three days is as costly as a genuine bug.
- **DO:** Test-drive non-trivial or risky changes locally rather than only reading the diff. Some bugs — race conditions, UX regressions, migration edge cases — are invisible in a diff view and only surface when the code actually runs.
- **DON'T:** let review turn into a design negotiation for something that should have been discussed before code was written. If the PR reveals a fundamentally different architecture is needed, take that conversation out of inline comments and into a synchronous discussion or design doc, then resume review once there's alignment.
- **DO:** Praise good decisions in review, not just flag problems. This reinforces the specific practices you want to see more of, and makes review feel like collaboration rather than a gate to get past.
- **DO:** As the author, thank reviewers for catching real issues and avoid getting defensive. Review is adversarial to bugs, not to the author — treat every caught issue as the review process working exactly as intended.
- **DON'T:** use review comments to teach unrelated lessons or link a wall of generic best-practice reading with no connection to the actual diff. Keep feedback scoped to the change under review; generic mentoring belongs in 1:1s or team docs, not buried in a PR thread.
- **DO:** Re-review promptly once the author has pushed fixes, rather than leaving a PR in limbo. An author who's addressed every comment and is waiting on re-approval is fully blocked until someone looks again.

### .gitignore Discipline

- **DO:** Commit a `.gitignore` file at the repo root before the first commit, covering build artifacts, dependency directories, and editor/OS cruft for the project's actual stack. This prevents `node_modules/`, `__pycache__/`, `.DS_Store`, and build output from ever entering history in the first place.
```
# Node
node_modules/
dist/
*.log

# Python
__pycache__/
*.pyc
.venv/

# Editors / OS
.vscode/
.idea/
.DS_Store

# Env / secrets
.env
.env.local
*.pem
```
- **DO:** Use a generated starting point — GitHub's gitignore templates, `gibo`, or an IDE-provided template — for the relevant stack, then customize it. Hand-rolling the ignore list per language reliably misses something that's already a solved problem, like `.env` or `target/`.
- **DON'T:** run `git add .` reflexively without first checking `git status` or `git diff --staged`. This is the single most common way secrets, `.env` files, IDE configs, and build artifacts sneak into a commit.
- **DO:** Ignore environment- and machine-specific files — `.env`, `*.local`, IDE workspace files — while committing a checked-in `.env.example`/`.env.sample` template with placeholder values. This documents required configuration without ever leaking a real secret.
```
# .env.example (committed)
DATABASE_URL=postgres://user:password@localhost:5432/app_dev
STRIPE_SECRET_KEY=sk_test_replace_me
JWT_SIGNING_SECRET=replace_with_a_long_random_string
```
- **DON'T:** add an overly broad ignore pattern — ignoring `*.json` or `*.yml` repo-wide — to "solve" an ignore problem for one specific file. Overbroad patterns silently swallow legitimately-needed config and data files later, and nobody notices until something is unexpectedly missing after a fresh clone.
- **DO:** Re-run `git rm -r --cached <path>` after adding a new ignore rule for something that was already tracked. Adding a pattern to `.gitignore` does not untrack files git already knows about — the file keeps being versioned until it's explicitly removed from the index.
```
echo "build/" >> .gitignore
git rm -r --cached build/
git commit -m "chore: stop tracking build output"
```
- **DO:** Keep a small, curated set of ignore patterns that reflect this repo's actual toolchain, rather than an ever-growing file copy-pasted from a dozen unrelated projects. Nobody prunes an ignore file's irrelevant cruft once it accumulates.
- **DON'T:** rely on `.gitignore` alone as your only defense against committing secrets — it only protects new commits, not history that already exists, and does nothing once a file has been added once. Pair it with pre-commit secret scanning (gitleaks, git-secrets) for real defense in depth.
- **DO:** Ignore local Terraform/IaC state and cache directories — `.terraform/`, `*.tfstate`, `*.tfstate.backup` — from version control, while still tracking the `.tf` source and, if used, a remote-backend configuration. State files frequently contain resource IDs and sometimes secrets in plaintext, and belong in a remote backend, not in git.

### Avoiding Secrets & Large Binaries

- **DON'T:** commit API keys, passwords, private keys, or tokens directly into source files or config committed to the repo. Once pushed, a secret is compromised the moment it's pushed — even a later commit that deletes it does not remove it from git history, and it must be treated as leaked and rotated.
- **DO:** Load secrets from environment variables, a secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager), or CI-injected secret variables — never from a file checked into git. This keeps the credential's lifecycle separate from the code's lifecycle, so rotating a secret never requires touching source.
```python
import os

# DO: read from the environment, fail fast if missing
stripe_key = os.environ["STRIPE_SECRET_KEY"]

# DON'T: hardcode a default that "just works" in dev
# stripe_key = os.environ.get("STRIPE_SECRET_KEY", "sk_test_51H...")
```
- **DO:** Run a pre-commit secret scanner — gitleaks, trufflehog, git-secrets — locally and again in CI. This catches accidental secret commits before they're pushed, and provides a second net for anyone who skipped or bypassed the local hook.
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
```
- **DON'T:** assume deleting a secret in a follow-up commit removes it from the repository. Git history retains every prior version; the exposed credential must be rotated regardless of what later commits do, and the history itself needs rewriting (`git filter-repo`/BFG) plus a coordinated force-push if it must actually be scrubbed.
- **DO:** If a secret is accidentally committed and pushed, rotate or revoke it immediately as the first step, before worrying about cleaning up history. History cleanup is cosmetic once a credential has been exposed to anyone with clone access; the credential itself is what's compromised.
- **DO:** Use Git LFS, or an artifact store / object storage, for large binary assets — design files, datasets, videos, compiled models — instead of committing them directly. Git's delta compression doesn't work well on binaries, so every version of a large binary permanently bloats the repository and every future clone.
```
# .gitattributes
*.psd filter=lfs diff=lfs merge=lfs -text
*.zip filter=lfs diff=lfs merge=lfs -text
datasets/**/*.csv filter=lfs diff=lfs merge=lfs -text
```
- **DON'T:** commit build artifacts, compiled binaries, or generated files — bundles, `.class`, `.pyc`, dist output — that CI regenerates from source. These belong in `.gitignore` and in build/release artifacts, not version control; committing them creates merge conflicts on generated content and hides whether the actual source changed.
- **DO:** Set repository size and file-size limits, or a pre-receive hook, to catch accidental large-file commits early. It's far cheaper to block a 200MB accidental commit at push time than to clean it out of history after ten more commits have landed on top of it.
- **DON'T:** store cloud provider credentials, SSH private keys, or `.pem`/`.pfx` files in a repo "temporarily, just for this test." "Temporarily" doesn't apply to git history — treat the repository as append-only-forever for anything that ever touches it.
- **DO:** Use short-lived, scoped credentials — OIDC federation between CI and the cloud provider, for example — instead of long-lived static keys, wherever the platform supports it. This removes an entire class of risk, since there's no long-lived secret to leak in the first place.
- **DO:** Add a `CODEOWNERS`-style rule or extra branch-protection review requirement on paths likely to contain sensitive configuration — deploy scripts, infra directories. This adds a second set of eyes specifically where a leaked secret would do the most damage.

### Rebase vs Merge

- **DO:** Understand the trade-off before choosing: merge preserves true history — including the fact that work happened in parallel — and is non-destructive; rebase produces a linear, easier-to-read history but rewrites commit SHAs. Pick a per-repo convention rather than mixing both unpredictably.
- **DO:** Rebase local, not-yet-shared feature branches onto the latest `main` before opening or updating a PR. This produces a clean, linear diff against `main` and avoids a noisy merge commit for a simple sync.
```
git fetch origin
git rebase origin/main
# resolve any conflicts, then:
git push --force-with-lease
```
- **DON'T:** rebase a branch that others have already pulled and built work on top of, without explicit coordination. Rebase rewrites SHAs; anyone with the old branch now has a diverged, conflicting copy that must be manually reconciled.
- **DO:** Use `git merge --no-ff` — or the platform's "merge commit" PR option — when you want an explicit, revertible record that "this feature landed as a unit," especially for release branches. A single merge commit is easy to `git revert` as one unit if the whole feature needs to be pulled out.
- **DO:** Use squash-merge as the default for typical feature PRs when the individual WIP commits aren't independently meaningful. This keeps `main`'s history at roughly one commit per feature or fix, which makes `git log`, `git bisect`, and `git blame` far more useful than wading through every WIP commit.
- **DON'T:** force-push over a branch others are collaborating on without warning, even with `--force-with-lease`. Coordinate first — a force-push, even a "safe" one, can still discard commits a collaborator pushed seconds earlier if you haven't fetched.
- **DO:** Prefer `git push --force-with-lease` over plain `git push --force` when you do need to force-push your own rebased branch. `--force-with-lease` refuses to overwrite if the remote has commits you haven't seen; plain `--force` clobbers them silently.
```
# safer: refuses if origin/feature-x has moved since your last fetch
git push --force-with-lease origin feature-x

# dangerous: overwrites whatever is on the remote, no check
git push --force origin feature-x
```
- **DO:** Resolve rebase conflicts commit by commit, verifying each intermediate commit still makes sense — and ideally still builds and passes tests — rather than blindly running `git rebase --continue` until it stops complaining. A rebase that "succeeds" mechanically can still leave individual commits in a broken, non-bisectable state.
- **DON'T:** use interactive rebase to silently drop a commit that fixes a real bug just because it causes a conflict, without understanding why the conflict occurred. Investigate every conflict; a "just take theirs" resolution can silently reintroduce an already-fixed bug.
- **DO:** Consider `git rebase --onto` when you need to move a branch's base without replaying commits that were only relevant to the old base. This is more surgical than a full rebase when a branch was built on top of another feature branch that's since merged.
- **DO:** Configure `git pull --rebase` — or `pull.rebase = true` — as the default for personal branches so a routine "pull latest" doesn't create pointless merge-bubble commits. This keeps a branch's history free of "Merge branch 'main' into feature-x" noise from every routine sync.

### Resolving Conflicts Safely

- **DO:** Read and understand both sides of a conflict before resolving it — don't blindly pick "ours" or "theirs." A conflict marker means two changes touched overlapping code for a reason, and understanding both intents is the only way to produce a correct merged result.
```
<<<<<<< HEAD
  return items.filter(i => i.active && i.quantity > 0);
=======
  return items.filter(i => i.quantity > 0 && !i.archived);
>>>>>>> feature/hide-archived
```
- **DON'T:** resolve a conflict by concatenating both sides without checking they compose into valid, coherent logic. Two individually-correct changes to the same function can combine into code that's syntactically valid but semantically wrong — for example, both sides adding an early return.
- **DO:** Re-run the test suite after resolving conflicts, not just after the merge or rebase "succeeds." A clean conflict resolution is a syntactic guarantee, not a semantic one — tests are what catch a merge that's wrong but plausible-looking.
- **DO:** Pull in the original authors or reviewers of both sides when a conflict is non-trivial — overlapping logic, not just adjacent lines — rather than guessing. The person who wrote the code usually resolves the ambiguity in seconds; guessing wrong can silently reintroduce a bug or drop a feature.
- **DON'T:** use `git checkout --ours`/`--theirs`, or the merge-tool equivalent, as a blanket strategy across a whole file with multiple unrelated conflicts. This resolves conflicts you didn't actually look at; go conflict by conflict even when it's slower.
- **DO:** Use a three-way merge tool — `git mergetool`, an editor's built-in merge view, `meld` — for conflicts spanning more than a few lines. Seeing base, ours, and theirs side by side is dramatically less error-prone than reading raw `<<<<<<<`/`=======`/`>>>>>>>` markers in a plain editor for anything non-trivial.
- **DO:** Commit conflict resolutions as their own clearly-marked commit — "Resolve merge conflicts in payment-client.ts" — rather than silently folding the resolution into an unrelated commit. This keeps history honest about where a manual judgment call was made, which matters if the resolution turns out wrong later.
- **DON'T:** abandon a conflicted rebase halfway with `git rebase --abort` and then just force-push over the target branch to make the conflict go away. That discards the actual work under conflict; either resolve it properly or explicitly decide, with the other author, which side wins and say so.
- **DO:** For lockfiles and generated files that conflict — `package-lock.json`, `Cargo.lock` — regenerate them after resolving the source conflict rather than hand-editing the merge markers. A hand-edited lockfile routinely produces a dependency graph that differs from what the package manager would actually resolve.
```
# after resolving package.json's conflict:
rm package-lock.json
npm install
git add package-lock.json
```

### Tagging & Release Conventions

- **DO:** Use Semantic Versioning — `MAJOR.MINOR.PATCH` — for tags on any published or consumed artifact, such as a library, container image, or CLI. Consumers rely on the major/minor/patch contract to know whether an upgrade is safe to take automatically.
```
git tag -a v2.4.1 -m "Release 2.4.1: fix null pointer in checkout flow"
git push origin v2.4.1
```
- **DO:** Use an annotated tag (`git tag -a`), not a lightweight one. Annotated tags carry a message, tagger identity, and date, and are what `git describe` and most release tooling expect.
- **DON'T:** move or retag an already-published version tag to point at a different commit. Consumers — package managers, deploy pipelines, other developers — may have already pulled or cached the artifact at that tag; moving it breaks reproducibility and trust in the version number.
- **DO:** Generate a changelog — manually curated or via Conventional Commits/semantic-release tooling — for every tagged release, listing user-facing changes, breaking changes, and fixes. This lets consumers decide whether to upgrade without reading the entire commit log.
- **DO:** Tag from a commit on the protected release branch, after CI has passed, never from an untested local commit. A release tag should point at exactly the code that was verified, not at whatever happened to be checked out locally.
- **DON'T:** skip version numbers or reuse them inconsistently — for example, releasing `2.3.0` after `2.4.0` because of a hotfix-branch mix-up. This confuses both automated tooling that compares versions and humans trying to understand release order.
- **DO:** Push tags explicitly — `git push --tags` or `git push origin <tag>` — since tags aren't included by a plain `git push`. Forgetting this step is a common reason "the tag exists locally but CI never saw it."
- **DO:** Pin CI/CD release pipelines and downstream dependents to a specific tag or commit SHA, not a moving branch reference, when reproducibility matters — a Docker build, a release artifact. This guarantees the exact same input produces the exact same release artifact if it ever needs to be rebuilt.
- **DON'T:** create a release tag without corresponding release notes on a public or shared repo. An unlabeled version bump gives downstream users no way to assess risk before upgrading; even a one-line summary is better than nothing.

## Docker

### Multi-Stage Builds

- **DO:** Use multi-stage builds to separate the build environment — compilers, dev dependencies, source — from the runtime image. The final image ships only what's needed to run the app, not the entire toolchain used to build it.
```dockerfile
# --- build stage ---
FROM golang:1.22-bookworm AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

# --- runtime stage ---
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```
- **DO:** Name build stages explicitly (`FROM golang:1.22 AS builder`) and copy only the specific build artifacts into the final stage (`COPY --from=builder /app/bin /app/bin`). Explicit naming and selective copying keep the final image free of intermediate build output, caches, and source you don't need at runtime.
- **DON'T:** install dev- or build-only dependencies — compilers, headers, test frameworks — in the same stage that ships to production. Every one of those tools is dead weight and attack surface in a runtime container that never runs `make` or `pytest` again.
- **DO:** Use a distinct, minimal base image for the final stage — `distroless`, `alpine`, or a `-slim` variant — even when the build stage uses a full-featured image. The build stage's size doesn't matter for what ships; the runtime stage's size and attack surface do.
- **DON'T:** copy the entire build context (`COPY . .` in the final stage) when only specific build outputs are needed. This re-introduces source code, `.git`, test fixtures, and build scripts into the shipped image for no benefit.
- **DO:** Cache dependency-download layers separately from source-copy layers across multi-stage builds — copy `go.mod`/`package.json` and run install before copying the rest of the source. This lets Docker reuse the dependency-install layer on every rebuild where only application code changed, not the dependency list.
- **DO:** Use multi-stage builds even for interpreted languages — Python, Node, Ruby — to strip build tools needed only for native extensions out of the runtime image. The same principle applies beyond compiled languages: anything installed just to build a wheel or native module shouldn't ship.
```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.12-slim
COPY --from=builder /root/.local /root/.local
COPY . /app
WORKDIR /app
ENV PATH=/root/.local/bin:$PATH
```
- **DON'T:** leave test execution or linting as an unconditional step baked into the same Dockerfile build path that produces the production image. Run tests in CI as a separate step, optionally against a build-stage target, rather than coupling "does it build" to "does it pass tests" inside the shippable image's build.
- **DO:** Use `docker build --target <stage>` to build and test an intermediate stage in CI independently from the final image. This lets you validate the build/test stage without needing to build, or push, the full final image every time.

### Minimal Base Images

- **DO:** Start from the smallest base image that satisfies your runtime's actual requirements — `alpine`, `-slim` variants, `distroless`, or `scratch` for static binaries. Fewer packages means fewer CVEs to patch, a smaller attack surface, and faster pulls and deploys.
- **DON'T:** default to `ubuntu:latest` or a full OS image out of habit when a slim or distroless variant would work. A general-purpose OS image ships hundreds of packages — a shell, package managers, sometimes compilers — that your app never touches but that still need patching.
- **DO:** Verify library and glibc compatibility before switching to Alpine, which uses musl libc, for languages with native extensions. Some packages behave differently or fail to build against musl; test the switch rather than assuming source compatibility.
- **DO:** Use distroless or `scratch` images for statically-linked binaries — Go, or Rust with a musl target — when you need the smallest possible attack surface. There's no shell and no package manager, so an attacker has nothing to pivot to even if the app itself has a vulnerability.
```dockerfile
FROM rust:1.78 AS builder
WORKDIR /src
COPY . .
RUN rustup target add x86_64-unknown-linux-musl && \
    cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /src/target/x86_64-unknown-linux-musl/release/app /app
ENTRYPOINT ["/app"]
```
- **DON'T:** pick a base image and never revisit it. Base images accumulate CVEs over time even without any change on your part; rebuild periodically against the latest patch of your chosen base tag rather than freezing forever on one digest.
- **DO:** Pin the base image to a specific version tag, and ideally a digest, rather than `latest`. `latest` is a moving target — the exact same Dockerfile can produce a different, untested base image tomorrow.
```dockerfile
# DON'T
FROM node:latest

# DO
FROM node:20.11.1-bookworm-slim

# stronger: pin the exact digest for full reproducibility
FROM node:20.11.1-bookworm-slim@sha256:1a2b3c...
```
- **DO:** Scan base images for known vulnerabilities — Trivy, Grype, Docker Scout — before adopting them, and on a recurring schedule afterward. This catches CVEs in the base layer itself, which is invisible if you only scan your application dependencies.
- **DON'T:** assume a smaller image is automatically more secure without verifying. A minimal image with an old, unpatched package can still be worse than a well-maintained slim image; check CVEs, not just megabytes.

### Layer Caching Order

- **DO:** Order Dockerfile instructions from least-frequently-changing to most-frequently-changing. Docker's build cache invalidates a layer, and every layer after it, the moment its inputs change, so stable inputs — OS packages, dependency manifests — belong first, and volatile ones — application source — belong last.
```dockerfile
FROM node:20.11.1-bookworm-slim
WORKDIR /app

# changes rarely -> cached almost every build
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# changes on every commit -> only this layer rebuilds normally
COPY . .
RUN npm run build
```
- **DON'T:** copy the entire application source before installing dependencies. `COPY . .` followed by `RUN npm install` invalidates the dependency-install cache on every single source change, turning a 10-second rebuild into a multi-minute one.
- **DO:** Copy only dependency manifest files — `package.json` and its lockfile, `requirements.txt`, `go.mod`/`go.sum`, `Gemfile`/lock — before running the install step, then copy the rest of the source afterward. This isolates the expensive, rarely-changing install step into its own cacheable layer.
- **DO:** Combine related `RUN` commands with `&&` into a single layer when they logically belong together, and clean up in the same layer they were created in.
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```
- **DON'T:** split a single logical operation — `apt-get update` and `apt-get install` — across separate `RUN` layers. A separate, independently-cached `apt-get update` layer can install stale or wrong package versions if that cache is reused later without re-running update.
- **DO:** Use `.dockerignore` alongside layer ordering — excluding irrelevant files reduces build context transfer time and prevents unrelated file changes from busting caches unnecessarily.
- **DO:** Leverage BuildKit cache mounts for package manager caches — pip, npm, cargo, the Go build cache — that shouldn't be baked into a layer but should persist across builds.
```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```
- **DON'T:** assume changing one line deep in the Dockerfile only affects that one layer's build time in CI — every downstream layer also rebuilds, and in CI without a warm cache, the entire image often rebuilds from scratch. Design CI caching (registry cache, BuildKit `--cache-from`) alongside Dockerfile layer order, not as an afterthought.
- **DO:** Group instructions that change at the same cadence into the same layer, and instructions that change independently into separate layers. Matching layer boundaries to actual change frequency is the entire point of caching-aware ordering.

### Running as Non-Root

- **DON'T:** run application containers as root — the default in most base images — in production. A container running as root that's compromised via an application vulnerability gives an attacker root inside the container and a meaningfully easier path to container-breakout or host compromise.
- **DO:** Create a dedicated non-root user and group in the Dockerfile and switch to it with `USER` before the final `CMD`/`ENTRYPOINT`.
```dockerfile
RUN addgroup --system app && adduser --system --ingroup app app
COPY --chown=app:app . /app
USER app
CMD ["node", "server.js"]
```
- **DO:** Set file ownership and permissions correctly for the non-root user on any directories the app needs to write to — logs, temp files, uploads. A missing `chown` is the most common regression here; test that the container actually runs, not just builds, after switching users.
- **DON'T:** use `USER root`, or omit `USER` entirely, "temporarily to get it working" and forget to switch back before the final stage. This is how "temporary" root containers end up in production indefinitely; treat a missing non-root `USER` as a build-time policy violation, not a style nit.
- **DO:** Use images that already default to a non-root user where available — many official images now ship a `node`, `nginx`, or similar unprivileged user — and select it via `USER` rather than reinventing user creation.
- **DO:** Set an explicit numeric UID, not just a username, when the container may run in Kubernetes with a `runAsNonRoot`/`runAsUser` security context that checks the numeric UID.
```dockerfile
RUN addgroup --system --gid 1001 app && \
    adduser --system --uid 1001 --ingroup app app
USER 1001:1001
```
- **DON'T:** bind to privileged ports below 1024 — such as port 80 — from a non-root process without additional configuration; this fails by design. Bind an unprivileged port inside the container, such as 8080, and map it to 80 at the host, load balancer, or ingress level instead.
- **DO:** Combine non-root execution with a read-only root filesystem — `--read-only` at runtime, or `readOnlyRootFilesystem: true` in Kubernetes — plus explicit writable volumes for the few paths that need writes. This further limits what a compromised process can tamper with, even within its own user's permissions.
- **DO:** Verify non-root execution in CI as an automated check, not a manual one-time review.
```bash
uid=$(docker run --rm myimage id -u)
if [ "$uid" = "0" ]; then
  echo "image runs as root" >&2
  exit 1
fi
```

### .dockerignore Discipline

- **DO:** Add a `.dockerignore` file to every Dockerfile-containing directory, excluding `.git`, local env files, `node_modules`/build output, and CI or editor artifacts.
```
.git
.gitignore
node_modules
dist
*.log
.env
.env.*
README.md
.github/
test/
```
- **DON'T:** let `.git` end up in the build context. Beyond the size cost, if any stage ever does something like `COPY . .`, the entire git history — including any secret that was ever committed and later "removed" — can leak into an image layer.
- **DO:** Exclude local development artifacts — `.env`, `*.log`, IDE config, a local `docker-compose.override.yml` — via `.dockerignore` so a developer's machine-specific files never accidentally end up baked into an image.
- **DO:** Keep `.dockerignore` roughly in sync with `.gitignore` for anything that's a build artifact or shouldn't exist in either place, but don't assume they should be identical — `.dockerignore` also needs to exclude things legitimately tracked in git but irrelevant to the image, such as docs and CI config.
- **DON'T:** forget `.dockerignore` in monorepo setups where a Dockerfile for one service sits in a directory that also contains sibling services' code. Without scoping the build context, unrelated services' source and dependencies get uploaded to the daemon and can leak into the image via an overly broad `COPY`.
- **DO:** Measure build context size — `docker build` prints "Sending build context to Docker daemon: X MB" — and treat an unexpectedly large number as a signal that `.dockerignore` is missing something.
- **DO:** Exclude test files, fixtures, and CI-only configuration from the build context when they aren't needed at build time, further shrinking context transfer and preventing them from being copyable into an image by an overly broad `COPY`.
- **DON'T:** rely on `COPY --exclude` as a substitute for a proper `.dockerignore` unless your BuildKit version and team tooling reliably support it. `.dockerignore` is the portable, universally understood mechanism.

### Image Size Discipline

- **DO:** Treat image size as a tracked metric, via CI reporting or registry dashboards, not an incidental side effect. Smaller images pull faster — which matters for autoscaling, CI job startup, and cold starts — reduce storage and egress cost, and typically carry a smaller CVE surface.
- **DO:** Clean up package-manager caches and temp files within the same `RUN` layer that created them. Deleting files in a later layer does not shrink the image — earlier layers are immutable and their content is still stored even if a later layer "deletes" it from the visible filesystem.
```dockerfile
# each `rm` here is in a SEPARATE later layer -> does NOT shrink the image
RUN apt-get update && apt-get install -y build-essential
RUN rm -rf /var/lib/apt/lists/*

# correct: cleanup in the SAME layer
RUN apt-get update && apt-get install -y build-essential && \
    rm -rf /var/lib/apt/lists/*
```
- **DON'T:** install recommended-but-unnecessary packages — skipping `--no-install-recommends`, or installing debugging tools "just in case" — in a production image. If debugging tools are genuinely needed, use an ephemeral debug container (`kubectl debug`) instead of shipping them permanently.
- **DO:** Use multi-stage builds as the single highest-leverage lever for image size — it structurally prevents build-only tooling from ever reaching the final layer, rather than relying on cleanup discipline within one stage.
- **DO:** Avoid installing a language's full package-manager toolchain in the runtime stage when only the runtime interpreter or VM is needed — don't ship `pip` and full Python dev headers if the app is a compiled artifact, and don't ship the full npm CLI and its own dependency tree if you only need `node` to run bundled JS.
- **DON'T:** bake datasets, ML model weights, or other large static assets into the image when they change independently of the application code. Fetch them at startup from object storage, with caching, or mount them as a volume — otherwise every asset update requires a full image rebuild and bloats every version in the registry.
- **DO:** Use `docker history <image>`, or a tool like `dive`, to inspect per-layer size contributions and find which layer is actually responsible for unexpected bloat.
```bash
docker history --no-trunc myimage:latest | head -20
```
- **DON'T:** treat "small image" as more important than correctness or maintainability. Don't strip out a package that's actually a runtime dependency just to shave megabytes, and don't hand-roll a from-scratch image so minimal it lacks CA certificates and breaks outbound TLS.

### Avoiding Secrets Baked Into Images

- **DON'T:** pass secrets as build arguments (`ARG`) or `ENV` in a Dockerfile intended to produce a shareable or pushed image. Both `ARG` values, unless using BuildKit secret mounts, and `ENV` values are visible in the image's layer history via `docker history` even if the final `CMD` never references them again.
```dockerfile
# DON'T: visible forever in `docker history`
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc && \
    npm install
```
- **DO:** Use BuildKit's `--mount=type=secret` for any credential needed only during the build, such as a private package registry token. The secret is available to that one `RUN` command's process but is never written into any image layer.
```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm install
```
```bash
docker build --secret id=npm_token,src=$HOME/.npm_token -t myapp .
```
- **DON'T:** `COPY` a `.env` file, SSH private key, or cloud credentials file into an image "just for this build" and remove it in a later layer. Removing a file in a later layer does not remove it from the image's layer history — it's still extractable from any layer that contains it.
- **DO:** Inject runtime secrets via the orchestrator — Kubernetes Secrets mounted as env vars or files, Compose's `env_file` sourced from an untracked local file, ECS task secrets — rather than baking them into the image at build time. Runtime injection means the same image is safe to push to a shared registry and reused across environments with different secrets.
- **DO:** Scan built images for embedded secrets — Trivy, gitleaks' image-scanning mode, truffleHog — as part of the CI pipeline before pushing to a registry.
- **DON'T:** assume a "private" registry makes baked-in secrets acceptable. Private just means fewer people can currently see it — anyone who ever pulls that image gets the secret, and image layers are trivially extractable with standard tools.
- **DO:** Rotate any secret that was ever baked into an image that reached a registry, the same way you'd rotate one committed to git — treat "was in an image layer" as equivalent to "was leaked," even after pushing a newer image without it.
- **DO:** Use multi-stage builds to ensure a build-time credential — used only to `git clone` a private dependency, for example — is scoped to an intermediate stage that never gets copied into or referenced by the final image.
- **DON'T:** hardcode environment-specific config that includes secrets, such as database connection strings with embedded passwords, as defaults in application config files shipped inside the image. Require them to be supplied at runtime and fail fast if they're missing, rather than silently working with a baked-in value.

### Docker Compose Conventions for Local Dev

- **DO:** Keep a `docker-compose.yml` at the repo root that brings up the full local dev stack — app plus dependent services like a DB, cache, and queue — with one command.
```yaml
services:
  app:
    build: .
    ports: ["3000:3000"]
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16.3
    environment:
      POSTGRES_PASSWORD: devpassword
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
volumes:
  db-data:
```
- **DO:** Use `docker-compose.override.yml`, loaded automatically alongside the base file, for developer-local customizations — port remaps, mounted volumes for hot-reload — and gitignore it. Keep the base `docker-compose.yml` as the shared, committed contract.
- **DON'T:** use production images or configuration unmodified in local Compose files when the local workflow needs hot-reload, debug ports, or verbose logging. Maintain a dev-oriented override, or a separate `docker-compose.dev.yml`, rather than permanently loosening the production Dockerfile or image itself.
- **DO:** Pin service image versions in Compose files — `postgres:16.3`, not `postgres:latest` — the same way you would in any other environment. A local dependency silently upgrading between two developers' machines, or between a developer's machine and CI, is a classic source of "works for me" bugs.
- **DO:** Use named volumes for stateful services so data survives `docker-compose down` and container recreation, but provide a documented `docker-compose down -v` (or an equivalent make target) for resetting to a clean state.
- **DON'T:** hardcode secrets — DB passwords, API keys — directly in a committed `docker-compose.yml`. Use a gitignored `.env` file referenced via Compose's `env_file` or variable substitution, with a committed `.env.example`.
```yaml
services:
  app:
    environment:
      DATABASE_URL: ${DATABASE_URL}
    env_file:
      - .env  # gitignored; see .env.example for required keys
```
- **DO:** Define explicit `healthcheck:` blocks for dependent services and use `depends_on: condition: service_healthy`, so the app container doesn't start racing against a database that's still initializing.
- **DO:** Document the standard Compose workflow in the README — how to run migrations, reset state, view logs for one service — and keep a `Makefile`/`justfile` wrapping the common commands if raw `docker-compose` invocations get unwieldy.
- **DON'T:** let the Compose setup and the production deployment manifests drift into defining the same service two structurally different ways with no shared source of truth. Where possible, derive both from the same environment variable names and config contract, so a config bug caught locally reflects a real production risk.
- **DO:** Use Compose profiles, or separate files, to make optional or heavy services — a full observability stack, a rarely-needed third dependent service — opt-in rather than part of the default `docker-compose up`.

## Kubernetes

### Resource Requests & Limits

- **DO:** Set both CPU and memory requests and limits for every container, based on observed usage from load testing or production metrics, not guesses. Requests drive scheduling decisions, since the scheduler packs nodes based on requests, and limits bound worst-case resource consumption; omitting either leaves the scheduler and the node's OOM killer working from bad information.
```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```
- **DON'T:** leave resource requests and limits unset "to be safe" or "to avoid throttling issues." An unset request makes Kubernetes treat the pod as `BestEffort` priority — it's the first thing evicted under node pressure, which is usually the opposite of what you want for a production workload.
- **DO:** Set memory limits equal to, or close to, memory requests, and understand that exceeding a memory limit gets a container OOM-killed, not throttled. Memory isn't compressible the way CPU is — there's no "slow it down," only "kill it," so a memory limit needs headroom based on real peak usage, not a hopeful guess.
- **DON'T:** set a CPU limit so tight that the application gets CFS-throttled during normal operation. Many teams set a CPU request but a generous or absent CPU limit precisely to avoid throttling latency-sensitive services, since Kubernetes' CPU limit enforcement is period-based and can introduce tail latency.
- **DO:** Right-size requests and limits iteratively using real metrics — via a VPA in recommendation mode, or Prometheus historical usage — rather than setting them once at service creation and never revisiting.
```bash
kubectl top pods -n payments --containers
```
- **DO:** Set namespace-level `ResourceQuota` and `LimitRange` objects so a single misconfigured deployment can't starve the rest of the namespace or cluster of resources.
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: payments
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "250m"
        memory: "256Mi"
      type: Container
```
- **DON'T:** request far more than the application typically needs "for headroom," across every service in the cluster. Over-requesting is how clusters end up needing several times more nodes than actual utilization justifies — it looks safe per-service but is expensive and wasteful in aggregate.
- **DO:** Differentiate QoS expectations deliberately: `Guaranteed`, where requests equal limits, for latency-critical services that must never be throttled or evicted ahead of others; `Burstable` for most typical workloads; and reserve `BestEffort` for genuinely non-critical or batch work.
- **DO:** Account for JVM or other managed-runtime memory behavior — configuring `-XX:MaxRAMPercentage` for Java, or the equivalent for other runtimes — so the language runtime respects the container's memory limit instead of sizing its heap off host-level memory.

### Liveness & Readiness Probes

- **DO:** Define both a liveness probe and a readiness probe for every service that receives traffic. Kubernetes uses liveness to decide when to restart a container and readiness to decide when to route traffic to it — conflating the two, or defining only one, causes either zombie pods serving errors or unnecessary restart loops.
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 2
```
- **DON'T:** point the liveness probe at an endpoint that checks downstream dependencies, such as database connectivity or a third-party API. If a downstream dependency is briefly unavailable, a liveness probe that fails on it will restart the pod repeatedly — restarting your app doesn't fix a downed database, and you'll cause a self-inflicted crash loop on top of the actual outage.
- **DO:** Make the readiness probe check what the container actually needs to serve traffic correctly — a warmed-up DB connection pool, a primed cache — so it fails during genuine unreadiness and Kubernetes removes the pod from the Service's endpoints until it recovers. This is exactly where a downstream-dependency check belongs, not in liveness.
- **DON'T:** set probe timeouts and periods so aggressive that normal startup or a brief GC pause triggers a false-positive restart. Tune `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, and `failureThreshold` against the app's actual startup time and latency profile, not framework defaults copied from an unrelated service.
- **DO:** Use a `startupProbe` for services with slow or variable startup time — JVM warm-up, a large cache preload — so the liveness probe doesn't start counting failures until startup genuinely completes.
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 5
```
- **DO:** Keep probe endpoints lightweight and fast — a cheap health check handler, not a full request pipeline. A probe that's expensive to compute adds load exactly when the system may already be under pressure.
- **DON'T:** hardcode a probe to always return 200 regardless of actual application state. This defeats the entire purpose — Kubernetes will happily keep routing traffic to, or never restart, a genuinely broken pod because the probe lies.
- **DO:** Distinguish liveness-probe failure causes from readiness-probe failure causes in logs and metrics so on-call engineers can tell "Kubernetes decided to restart this" from "Kubernetes decided to stop routing traffic to this" during an incident.
- **DO:** Test probe behavior under realistic failure conditions — kill the DB connection, saturate CPU — before relying on it in production, not just confirm it returns 200 on a happy-path pod.

### Security Context: Non-Root & Non-Privileged

- **DO:** Set `securityContext.runAsNonRoot: true` and a specific `runAsUser`/`runAsGroup` at the pod or container level, matching the non-root user baked into the image.
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  seccompProfile:
    type: RuntimeDefault
```
- **DON'T:** run containers with `privileged: true` unless there is a specific, well-understood, and unavoidable need, such as certain node-level agents or CNI plugins. A privileged container has essentially unrestricted access to the host — it defeats nearly every other container isolation guarantee Kubernetes provides.
- **DO:** Set `allowPrivilegeEscalation: false` and drop all Linux capabilities by default, adding back only the specific capability a workload genuinely needs.
```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
  readOnlyRootFilesystem: true
```
- **DO:** Set `readOnlyRootFilesystem: true` wherever the application doesn't need to write to its own container filesystem, mounting explicit `emptyDir`/volume mounts for the specific paths that do need writes.
- **DON'T:** mount the host's Docker socket, `/var/run/docker.sock`, or other sensitive host paths into application pods. This effectively grants root-on-host access to anything running in that pod — reserve it for narrowly-scoped, trusted infrastructure tooling only, if at all.
- **DO:** Enforce these security-context requirements cluster-wide via admission control — Pod Security Admission's `restricted` profile, OPA Gatekeeper, or Kyverno policies — rather than relying on every team remembering to set them per manifest.
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```
- **DON'T:** assume namespace isolation alone is a sufficient security boundary between untrusted or multi-tenant workloads. Namespaces are a logical or organizational boundary, not a hard security boundary — containers still share the host kernel by default.
- **DO:** Scan running workloads, not just images, periodically for security-context drift or misconfiguration using cluster security tools such as kube-bench, kubescape, or Polaris.
- **DO:** Apply seccomp profiles — `RuntimeDefault` at minimum — to restrict the syscalls a container can make, reducing kernel attack surface even for a compromised process running as a properly-scoped non-root user.

### ConfigMaps & Secrets

- **DO:** Externalize environment-specific configuration into ConfigMaps and mount them as environment variables or files, rather than hardcoding config values into container images or manifests' `command`/`args`.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  FEATURE_NEW_CHECKOUT: "true"
---
envFrom:
  - configMapRef:
      name: app-config
```
- **DO:** Store sensitive values — API keys, DB passwords, TLS certs — in Kubernetes Secrets, not ConfigMaps. Secrets get different handling, including RBAC-scoped access and integration points for external secret stores, that ConfigMaps don't provide, even though neither is encrypted at rest by default.
- **DON'T:** treat a Kubernetes Secret as sufficiently secure on its own. By default, Secret data is only base64-encoded, which is trivially reversible, not encrypted, and etcd itself needs encryption at rest enabled for genuine protection. Enable etcd encryption at rest and restrict RBAC access to Secrets.
- **DO:** Use an external-secrets operator or CSI secrets-store driver to sync secrets from a dedicated secrets manager into Kubernetes, rather than manually applying Secret manifests.
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: prod/db
        property: password
```
- **DON'T:** commit rendered Secret manifests, even base64-encoded ones, to a plain git repo. Base64 is encoding, not encryption — anyone with repo read access can trivially decode it. If secrets must be managed via GitOps, use a tool built for that, such as Sealed Secrets or SOPS-encrypted files.
- **DO:** Use `envFrom`/`volumeMounts` referencing a ConfigMap or Secret by name rather than duplicating individual key-value pairs across multiple manifests.
- **DON'T:** rely on containers automatically picking up ConfigMap or Secret changes without a restart, unless the app specifically watches for file changes on a volume-mounted ConfigMap, or you're using a tool that triggers a rolling restart on config change, such as Reloader.
```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```
- **DO:** Scope RBAC so that only the workloads and operators that need a given Secret can read it, using least-privilege `Role`/`RoleBinding` per namespace rather than broad cluster-wide `get secrets` permissions.
- **DO:** Rotate secrets on a defined schedule, and immediately on suspected compromise, using your secrets manager's rotation support, ensuring the app either re-reads rotated secrets or gets restarted to pick them up.

### Namespace Organization

- **DO:** Use namespaces to separate environments and/or teams and domains — for example `team-payments`, or `staging`/`production` if environments share a cluster — with consistent naming conventions across the org.
- **DON'T:** dump every workload into the `default` namespace. It becomes impossible to apply meaningful RBAC, quotas, or network policy at any granularity, and `kubectl get pods` in `default` turns into an unfilterable wall of unrelated services.
- **DO:** Apply `NetworkPolicy` resources to restrict which namespaces and pods can talk to each other, defaulting to deny-all and explicitly allowlisting required traffic.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: payments
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
```
- **DO:** Set per-namespace `ResourceQuota` and `LimitRange` so one team or namespace can't exhaust cluster resources and starve others sharing the same cluster.
- **DON'T:** use namespaces as the only separation between environments with meaningfully different risk profiles, such as production and a wide-open dev sandbox, on the same cluster without additional hardening. Consider separate clusters when the blast radius of a dev-namespace misconfiguration reaching prod-adjacent resources is unacceptable.
- **DO:** Standardize labels — `app`, `team`, `environment`, `component` — across all resources in a namespace, and use them consistently for `kubectl` selectors, dashboards, and cost allocation.
- **DO:** Document namespace ownership — which team owns it, who to page — somewhere discoverable, such as a labels/annotations convention or an internal service catalog, not just tribal knowledge.

### Avoiding the `latest` Tag in Production

- **DON'T:** deploy with `image: myapp:latest`, or any other floating tag, in a production Kubernetes manifest. `latest` is not guaranteed to mean "newest" and provides no reproducibility — the same manifest can deploy a completely different image on two different days.
- **DO:** Deploy with an immutable, specific reference — a semantic version tag, and for maximum reproducibility, the image digest.
```yaml
image: myregistry.io/myapp:1.4.2@sha256:9f8e7d6c5b4a...
imagePullPolicy: IfNotPresent
```
- **DO:** Set `imagePullPolicy: IfNotPresent`, the default for non-`latest` tags, rather than `Always`, once you're using immutable version tags. This avoids unnecessary registry pulls on every pod restart while still guaranteeing the exact version deployed.
- **DON'T:** set `imagePullPolicy: Always` as a workaround for using `latest` tags. This re-fetches whatever `latest` currently points to on every pod restart, meaning a node restarting a crashed pod can silently upgrade it to a different, untested image version mid-incident.
- **DO:** Have CI/CD tag and push a unique, traceable image reference on every build — the git SHA, or a semver bump — and have the deployment pipeline update the manifest to reference that specific tag or digest.
- **DO:** Use `latest` only for local development convenience or ephemeral CI test images that are never deployed to a real environment.
- **DON'T:** let a GitOps or Helm values file default to `latest`/`:main` for the image tag "to keep things simple" during initial setup, and then forget to pin it before the service handles real traffic. Audit image tags as a required check before an environment is considered production-ready.

### Helm Chart Conventions

- **DO:** Parameterize environment-specific values via `values.yaml`, and per-environment overrides such as `values-staging.yaml`/`values-prod.yaml`, keeping chart templates themselves environment-agnostic.
```yaml
# values.yaml (defaults)
replicaCount: 2
image:
  repository: myregistry.io/myapp
  tag: "1.4.2"
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits: { cpu: "500m", memory: "512Mi" }
```
- **DO:** Provide sensible, safe defaults in `values.yaml` — resource limits set, replicas of at least two, probes defined — so a chart deployed with no overrides is still reasonably production-safe.
- **DON'T:** template overly generic, deeply parameterized charts that try to support every conceivable configuration via dozens of interdependent conditionals. Past a certain complexity, an over-parameterized chart becomes harder to reason about than several straightforward charts.
- **DO:** Pin chart dependency versions explicitly in `Chart.yaml`'s `dependencies` block, and commit the resulting `Chart.lock`, the same way you'd pin any other dependency.
- **DO:** Use `helm template`/`helm lint`, and `helm diff` for the actual upgrade delta, in CI to validate rendered manifests before any `helm upgrade` reaches a real cluster.
```bash
helm lint ./chart
helm template ./chart -f values-prod.yaml | kubeconform -strict
```
- **DON'T:** use `helm upgrade --force` reflexively to push through a failed upgrade. `--force` deletes and recreates resources rather than patching them, which can cause unnecessary downtime and, for stateful resources, potential data loss.
- **DO:** Use Helm hooks deliberately for tasks like DB migrations, and make hook jobs idempotent and safe to retry, since Helm may re-run hooks on retried or failed releases.
- **DO:** Version charts independently from application versions using semantic versioning in `Chart.yaml`'s `version` field, separate from `appVersion` which tracks the application's own version.
- **DON'T:** hardcode secrets or environment-specific credentials directly into a chart's `values.yaml` that gets committed to a public or shared chart repo. Reference external secrets from the chart rather than baking credential values into chart source.
- **DO:** Keep charts in a versioned chart repository — an OCI registry, ChartMuseum, or a git-based repo — with releases tied to tags, so `helm install`/`upgrade` against a specific chart version is reproducible.
- **DO:** Test charts against a real, or realistic (kind/minikube), cluster in CI. `helm template` catches syntax and rendering errors, but only an actual `helm install`/`upgrade` against a live API server catches admission-webhook rejections and other runtime-only failures.

## CI/CD

### Pipeline Structure: Fast Feedback First, Fail Fast

- **DO:** Order pipeline stages so the fastest, most likely to fail checks run first — lint and type-check, then unit tests, then build, then integration tests, then deploy. A syntax error should fail in seconds, not after a ten-minute integration suite has already run.
```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [ ... ]           # ~30s
  unit-test:
    needs: lint
    steps: [ ... ]           # ~2m
  build:
    needs: unit-test
    steps: [ ... ]
  integration-test:
    needs: build
    steps: [ ... ]           # ~10m
```
- **DO:** Run independent checks in parallel — lint, unit tests, security scan — rather than serially, when they don't depend on each other's output.
```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [ ... ]
  unit-test:
    runs-on: ubuntu-latest
    steps: [ ... ]
  security-scan:
    runs-on: ubuntu-latest
    steps: [ ... ]
  # all three start simultaneously; no `needs:` between them
```
- **DON'T:** make a slow, flaky, or rarely-useful check block the entire pipeline early, when it could run later or in parallel with faster signal. A twenty-minute end-to-end suite gating even a one-line README change wastes contributor time and pipeline capacity.
- **DO:** Fail the pipeline immediately on the first failing required check rather than continuing to run subsequent stages against code already known to be broken, unless you specifically want full-matrix visibility across every failing OS/version combination at once.
- **DO:** Separate "must pass to merge" checks from "informational/advisory" checks explicitly, so contributors know which failures actually block them.
- **DON'T:** let CI pipeline structure grow organically into an unmaintainable tangle of copy-pasted steps across many jobs with no shared definition. Use reusable workflows or templates so a fix to the pipeline logic applies everywhere it's used.
```yaml
# .github/workflows/reusable-test.yml
on:
  workflow_call:
    inputs:
      node-version: { required: true, type: string }
---
# caller workflow
jobs:
  test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: "20"
```
- **DO:** Keep the "typical happy path" pipeline fast — target well under ten minutes where feasible — and push slower, more exhaustive checks to a separate, less frequently triggered pipeline: nightly, pre-release, or on-demand.
- **DO:** Surface pipeline status clearly and close to where developers work — PR status checks, chat notifications on failure — rather than requiring someone to check a separate CI dashboard.
- **DON'T:** silently swallow a step's failure by piping it through something that always exits zero, such as `command || true`, used to "keep the pipeline green" instead of fixing the underlying failure. Treat a persistently-failing step as a signal to fix, or explicitly and visibly mark as non-blocking — never silently mask it.
- **DO:** Give the pipeline a single, obvious source of truth for "did this change pass" — a required check or merge gate — even if the underlying pipeline is a DAG of many jobs.

### Caching Dependencies

- **DO:** Cache package manager dependencies — the npm/yarn/pnpm cache, pip/poetry cache, Maven/Gradle's `.m2`, the Go module cache — between CI runs, keyed on a hash of the lockfile.
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-
```
- **DO:** Key the cache on the lockfile hash rather than a static key. A static key returns stale cached dependencies even after the lockfile changes, and an unkeyed or branch-only key causes cross-contamination or unnecessary cache misses.
- **DON'T:** cache things that change every build and provide no reuse value, such as build output that's inherently unique per commit, as if they were stable dependencies. A cache that never hits, or grows unbounded because nothing is ever evicted, wastes storage and bandwidth without saving any time.
- **DO:** Set a fallback restore-key so a cache miss on the exact lockfile hash still restores the closest previous cache as a starting point, rather than starting completely cold.
- **DON'T:** rely on CI dependency caching to substitute for actually pinning dependency versions. Caching solves speed; pinning solves reproducibility — a cache eventually expires or misses, and when it does, an unpinned dependency range can resolve to a different, possibly broken, version than what was cached.
- **DO:** Cache build tool outputs where the tool supports safe incremental caching — Docker layer cache via BuildKit's `--cache-from`/registry cache, compiler caches like `ccache`/`sccache`, bundler caches.
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=registry,ref=myregistry.io/myapp:buildcache
    cache-to: type=registry,ref=myregistry.io/myapp:buildcache,mode=max
```
- **DO:** Periodically clear or rotate caches, or set an expiry, rather than letting them grow indefinitely, especially for caches keyed loosely enough to accumulate many stale variants.
- **DON'T:** cache secrets or credentials as a side effect of caching a broader directory, such as an entire home directory that happens to contain a credentials file. Scope cache paths precisely to the dependency directories that actually need caching.

### Avoiding Flaky Pipeline Steps

- **DON'T:** tolerate a known-flaky test or pipeline step by re-running it until it passes, as standard practice. Every flaky-but-ignored step erodes trust in the whole pipeline — eventually a genuine failure gets dismissed as "probably just flaky" and merged anyway.
- **DO:** Track flaky test and step failure rates, using CI-platform-native reporting, test framework reporting, or a dedicated flaky-test-detection tool, and treat flakiness as a bug to fix or a test to quarantine, not background noise.
- **DO:** Quarantine a known-flaky test — mark it skipped or non-blocking, with a tracked follow-up ticket — rather than leaving it blocking merges indefinitely while nobody fixes it.
- **DON'T:** introduce non-determinism into tests via unmocked wall-clock time, unseeded randomness, real network calls to third-party services, or reliance on test execution order. These are the most common root causes of flaky tests.
```python
# DON'T: unseeded, order-dependent, real clock
def test_discount():
    assert calculate(datetime.now(), random.random()) > 0

# DO: deterministic inputs
def test_discount():
    fixed_time = datetime(2026, 1, 1)
    assert calculate(fixed_time, seed=42) > 0
```
- **DO:** Add explicit timeouts to every CI job and step, sized realistically for the task.
```yaml
jobs:
  integration-test:
    timeout-minutes: 15
    steps:
      - run: npm run test:integration
        timeout-minutes: 10
```
- **DO:** Isolate test environments — a fresh database per test run or suite, containerized dependencies, no shared mutable state between parallel test runs — so tests don't flake due to leftover state from a previous run.
- **DON'T:** retry a failing deployment or infrastructure-provisioning step blindly without inspecting why it failed. Blind retries can paper over a real, worsening problem until it becomes much harder to recover from than if it had been investigated on the first failure.
- **DO:** Use a bounded, logged retry with backoff specifically for operations known to have transient failure modes — network calls to external services, cloud API rate limits — not as a blanket policy applied to every step regardless of whether the underlying failure is actually transient.
- **DO:** Run new or suspect tests in isolation multiple times, locally or via a CI flaky-test-detector job, before merging them, especially timing-sensitive or integration tests.

### Secrets Management in CI

- **DO:** Store CI secrets in the CI platform's dedicated secrets store — GitHub Actions Secrets, GitLab CI/CD variables marked protected and masked, CircleCI contexts — never in the pipeline YAML itself or in a committed file the pipeline reads.
- **DON'T:** echo or print secret values in pipeline logs, even "temporarily for debugging." Most CI platforms mask a known secret value if referenced through their secrets mechanism, but a secret assigned to a plain variable and then printed bypasses that masking entirely.
```yaml
# DON'T
- run: echo "Deploying with token ${{ secrets.DEPLOY_TOKEN }}"

# DO: let the tool consume it directly, never echo it
- run: ./deploy.sh
  env:
    DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```
- **DO:** Scope CI secrets as narrowly as possible — per-environment, per-job, or per-branch — rather than one shared secret available to every pipeline in the repo or org.
- **DON'T:** expose repository or organization secrets to workflows triggered by pull requests from forks without extra safeguards. Use `pull_request_target` carefully, if at all, and require maintainer approval before secret-bearing workflows run on fork PRs.
- **DO:** Prefer OIDC-based short-lived credential federation over long-lived static cloud credentials stored as CI secrets, wherever the CI platform and cloud provider support it.
```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/ci-deploy
      aws-region: us-east-1
```
- **DO:** Rotate CI secrets on a schedule, and immediately upon any suspected exposure — a departing team member who had access, a compromised runner, an accidental log leak.
- **DON'T:** grant a CI service account or deploy role more permission than the specific pipeline needs. Overly broad CI permissions turn a compromised pipeline into a compromised cloud account.
- **DO:** Use separate secrets and credentials per environment — dev, staging, prod — never a single shared production credential used across all pipeline stages.
- **DO:** Audit which workflows and jobs actually use each stored secret periodically, and remove secrets no longer referenced by any pipeline.
- **DON'T:** pass secrets to a step via command-line arguments if the platform or tool supports environment variables or files instead. Command-line arguments are often visible in process listings or shell history on the runner in ways environment variables passed directly aren't.

### Deployment Strategies: Blue-Green, Canary, Rolling

- **DO:** Choose a deployment strategy deliberately based on the service's risk tolerance and traffic patterns. Rolling updates are simple and resource-efficient but roll forward gradually with mixed old/new versions serving simultaneously; blue-green gives instant cutover and instant rollback at double the resource cost during the switch; canary gives the most gradual, metric-gated exposure but is the most operationally complex to set up well.
- **DO:** Use rolling updates as a sensible default for most stateless services, with `maxSurge`/`maxUnavailable` tuned to the service's tolerance for reduced capacity during rollout.
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```
- **DO:** Use blue-green deployment when instant, clean rollback matters more than resource efficiency — a change risky enough that you want the old version fully running and ready to receive 100% of traffic back with a single switch.
- **DON'T:** assume blue-green deployment alone solves stateful or database migration safety. If the new version requires a schema change incompatible with the old version, an instant traffic switch back to the old environment after a bad deploy can break against an already-migrated database — pair blue-green with backward-compatible migrations.
- **DO:** Use canary deployments — shifting a small percentage of real traffic to the new version, monitoring key metrics, then progressively increasing — for high-traffic or high-risk services where you want to catch a regression against real production traffic before it's fully rolled out.
```yaml
# Argo Rollouts canary example
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
```
- **DO:** Gate canary or progressive rollouts on automated metrics — error rate, latency, saturation — rather than a fixed time delay alone, using a progressive-delivery tool such as Argo Rollouts or Flagger where the scale justifies the operational complexity.
- **DON'T:** run a canary or blue-green rollout without pre-defined, automatic rollback criteria. If "watch the dashboard and decide" is the only rollback trigger, a rollout during a quiet on-call shift can sit in a degraded state far longer than an automated threshold-based rollback would allow.
- **DO:** Ensure the deployment strategy accounts for in-flight requests during a version switch — graceful shutdown handling `SIGTERM`, connection draining, readiness-probe-driven traffic removal before pod termination.
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
terminationGracePeriodSeconds: 30
```
- **DON'T:** pick the most sophisticated deployment strategy for every service regardless of actual risk and traffic. A low-traffic internal tool doesn't need the operational overhead of a full canary pipeline; match the strategy's complexity to the service's actual blast radius.
- **DO:** Document and rehearse the specific rollback procedure for whichever strategy you use, before you need it during an incident.

### Rollback Readiness

- **DO:** Make every deployment revertible with a single, fast, well-known action, and make sure whoever's on-call knows what that action is before they need it at 3am.
```bash
# Kubernetes native rollback
kubectl rollout undo deployment/myapp -n payments

# Helm rollback to the previous release
helm rollback myapp -n payments
```
- **DO:** Keep the previous several versions of deployment artifacts — images, chart releases, Terraform state history — readily available and referenced by immutable identifiers, not overwritten or garbage-collected too aggressively.
- **DON'T:** ship a database migration that's only safe to run forward, with no plan for what happens if the application needs to roll back while the migration has already applied. Prefer backward-compatible, additive migrations using an expand/contract pattern.
```
1. Expand:   add new_column (nullable)
2. Dual-write: app writes to both old_column and new_column
3. Backfill: migrate existing rows
4. Migrate reads: app reads from new_column
5. Contract: drop old_column (only once rollback is no longer a concern)
```
- **DO:** Automate rollback as a first-class pipeline action, not a manual, improvised sequence of commands someone has to reconstruct during an incident. Practice it in a non-incident context, such as a game day or staging drill.
- **DON'T:** treat "we can always roll back" as true without verifying it. Some changes are effectively one-way — a destructive data migration, an external API contract change with no backward compatibility — identify these explicitly ahead of time.
- **DO:** Pair rollback capability with feature flags for risky logic changes, so you can disable the new code path instantly without needing a full redeploy at all.
- **DO:** Keep rollback dependent only on things that are highly available and independent of the failure you're rolling back from.
- **DON'T:** define "rollback" only at the application-deployment layer while ignoring config or infrastructure changes that shipped alongside it. If a deploy also changed a ConfigMap or a Terraform-managed resource, roll that back too.
- **DO:** Set a maximum acceptable rollback time as an SLO for critical services, and validate it periodically via chaos-engineering-style rollback drills.

### Environment Parity

- **DO:** Run the exact same container image, or as close as practically achievable, across local dev, CI, staging, and production, differing only in configuration, not in the underlying artifact.
- **DON'T:** install dependencies with different tool versions across environments — a developer's locally-installed Node 18 versus CI's Node 20 versus production's Node 16. Pin runtime and tool versions explicitly and use the same pinned version everywhere.
```
# .nvmrc
20.11.1
```
```yaml
- uses: actions/setup-node@v4
  with:
    node-version-file: ".nvmrc"
```
- **DO:** Use containerized local development — Docker Compose, devcontainers — so "my machine" and CI/production share the same OS-level dependencies and library versions, not just the same application dependencies.
- **DON'T:** let staging drift from production in scale, data shape, or configuration to the point that "passed staging" stops meaningfully predicting "will work in production."
- **DO:** Generate configuration for each environment from the same templates and source — the same Helm chart with different values files, the same Terraform module with different variable sets — rather than maintaining hand-diverged, independently-edited configs per environment.
- **DO:** Test infrastructure and deployment changes, not just application code, in a lower environment before production, using the same deployment mechanism that will run in production.
- **DON'T:** let local development rely entirely on stubbed or mocked external services while production integrates with the real ones, without also having an environment or dedicated integration test suite that exercises the real integrations before release.
- **DO:** Keep a documented, versioned list of exactly what's supposed to differ between environments — replica counts, resource limits, external endpoints, feature flags — so parity doesn't mean literally identical, but deliberately and knowingly different only where it should be.
- **DO:** Version and test database schema migrations against a realistic copy, or realistic synthetic equivalent, of production data shape and volume before running them in production.

## Infrastructure as Code

### Terraform/Pulumi Conventions & State Management

- **DO:** Store Terraform or Pulumi state in a remote, shared backend — S3 with a DynamoDB lock table, Terraform Cloud, Pulumi Cloud, or GCS with locking — never rely on local state files for anything beyond a solo experiment.
```hcl
terraform {
  backend "s3" {
    bucket         = "acme-terraform-state"
    key            = "payments/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```
- **DO:** Enable state locking so concurrent `apply` runs can't race against each other and corrupt state. Without locking, two simultaneous applies can both read the same state, make conflicting changes, and leave state that doesn't match either intended outcome.
- **DON'T:** edit `.tfstate` files by hand. State files use an internal format tied to the provider's resource schema; a manual edit can silently desync state from reality in ways that only surface as confusing errors, or unintended deletions, on the next plan or apply.
- **DO:** Use `terraform state mv`/`terraform import`, or their Pulumi equivalents, for legitimate state surgery — renaming a resource in code without destroying and recreating it, or bringing an existing manually-created resource under IaC management.
```bash
terraform state mv aws_instance.old_name aws_instance.new_name
terraform import aws_s3_bucket.assets acme-assets-prod
```
- **DO:** Enable state file encryption at rest, and restrict access via IAM, since state files frequently contain sensitive data — RDS master passwords, certain provider auth details — even when secrets weren't intentionally put into Terraform config.
- **DON'T:** commit `.tfstate`/`.tfstate.backup` files to git. Beyond size and merge-conflict problems, state files routinely contain resource attribute values that are effectively secrets; treat state with the same sensitivity as a secrets file.
- **DO:** Split state per logical boundary — per environment at minimum, and often per service or component too — rather than one giant monolithic state file for the entire infrastructure. This limits blast radius and keeps plan/apply fast.
- **DON'T:** let state grow unboundedly coupled across unrelated systems just because it was convenient to add resources to an existing state file. A well-factored split makes ownership, blast radius, and apply speed all better; a monolithic state file makes every apply riskier and slower as the org grows.
- **DO:** Use workspaces deliberately, understanding that they share the same backend configuration and module code — they're for parallel instances of the same config, not a substitute for genuinely different environment-specific module structure.
- **DO:** Version-control all `.tf`/Pulumi source files, and treat that source, not any individual engineer's local plan output, as the definitive description of intended infrastructure.
- **DON'T:** run `terraform apply`, or Pulumi's equivalent, directly from a developer's laptop against shared or production infrastructure as standard practice. Route applies through CI/CD with the plan visible for review and a full audit trail.
- **DO:** Use a consistent provider version pinning strategy, with `required_providers` version constraints and a committed lockfile, so `terraform init` resolves the same provider versions across every machine and CI run.
```hcl
terraform {
  required_version = ">= 1.7.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}
```
- **DO:** Back up or enable versioning on the state backend itself — S3 bucket versioning, Terraform Cloud's state history — so a corrupted or bad state write can be rolled back to a known-good prior version.

### Avoiding Manual Console Changes & Drift

- **DON'T:** make ad-hoc changes through the cloud provider's web console or CLI to infrastructure that's managed by IaC. Manual changes create drift, where reality no longer matches what your Terraform or Pulumi source describes, and the next apply will either silently revert your manual fix or fail confusingly trying to reconcile the mismatch.
- **DO:** Route all infrastructure changes to IaC-managed resources through the IaC workflow — edit source, plan, review, apply — even for "quick" fixes, especially under incident pressure when the temptation to click a console button is highest.
- **DO:** Run periodic drift detection — `terraform plan` on a schedule, cloud-native drift detection tools, or Pulumi's `preview` against live state — and alert on any non-empty diff against a state that should be stable.
```yaml
# scheduled drift-check workflow
on:
  schedule:
    - cron: "0 6 * * *"
jobs:
  drift-check:
    steps:
      - run: terraform plan -detailed-exitcode
        continue-on-error: true
```
- **DON'T:** grant broad, standing console or CLI write access to production infrastructure to engineers who normally operate through IaC. Gate direct access behind break-glass or just-in-time access processes that are logged and time-limited.
- **DO:** Import pre-existing, manually-created resources into IaC state as soon as they're discovered, rather than leaving a permanent unmanaged exception.
- **DO:** Use policy-as-code — AWS Config rules, Azure Policy, OPA/Sentinel — to detect or even prevent configuration that deviates from what IaC would produce, as a backstop beyond human discipline.
- **DON'T:** treat drift as purely a Terraform-state problem. Investigate why the manual change happened at all — a gap in the IaC's capabilities, a process that was too slow, a genuine emergency — and fix the underlying cause.

### Module Design

- **DO:** Design modules around a single, coherent responsibility — "a VPC," "an application's full stack of compute, DB, and cache" — with a clear, minimal interface, the same way you'd design a well-scoped function or library.
```hcl
module "vpc" {
  source  = "app.terraform.io/acme/vpc/aws"
  version = "3.2.0"

  cidr_block          = "10.10.0.0/16"
  availability_zones  = ["us-east-1a", "us-east-1b"]
  enable_nat_gateway  = true
}
```
- **DON'T:** expose every possible underlying provider argument as a module variable "just in case." An over-parameterized module with dozens of optional variables is hard to use safely and hard to evolve; expose what real callers actually need.
- **DO:** Version modules — git tags on a module source repo, or a module registry with semver — and pin consumers to a specific module version, the same discipline as pinning any other dependency.
- **DO:** Write and maintain examples and documentation for each module, with a working `examples/` directory that's part of module testing, not just a comment.
- **DON'T:** create deeply nested module hierarchies, with modules calling modules calling modules many layers deep, without a strong reason. Excessive nesting makes it hard to trace which layer actually sets a given value.
- **DO:** Test modules in isolation — Terratest, or plan/apply against a throwaway environment — as part of CI for the module repo itself, not just implicitly whenever a downstream consumer happens to use it.
- **DO:** Keep environment-specific values — account IDs, specific CIDR ranges, instance counts — out of shared modules and pass them in as variables from the calling root module or environment config.
- **DON'T:** let a root module per environment become a monolithic, near-identically duplicated file across dev, staging, and prod. Factor the shared structure into versioned modules and keep the per-environment root thin.
- **DO:** Document each module's inputs, outputs, and any non-obvious side effects in a README colocated with the module source, ideally auto-generated via a tool like `terraform-docs`.

### Plan-Before-Apply Discipline

- **DO:** Always run and review `terraform plan`, or `pulumi preview`, before every apply, and require this review step to be visible in CI or a PR for any change to shared infrastructure.
```bash
terraform plan -out=tfplan
# review the plan output, then, once approved:
terraform apply tfplan
```
- **DON'T:** apply changes based on a stale plan — one generated against an older state than what's currently live, for example because someone else applied in between. Regenerate the plan immediately before apply if any time has passed.
- **DO:** Pay specific attention to any planned destroy or replace (force-new) action in the plan output, and understand exactly why it's happening before proceeding. A destroy-and-recreate on a stateful resource is often the single most consequential line in a plan.
- **DO:** Use `terraform plan -out=tfplan` to save the exact reviewed plan and apply that saved artifact, rather than re-running plan implicitly as part of apply. This guarantees the plan that was reviewed is exactly the plan that gets applied.
- **DON'T:** use `-auto-approve` for applies against shared or production infrastructure outside of a fully automated, already-plan-reviewed CI pipeline. Reserve it for genuinely low-risk, already-gated automation.
- **DO:** Post the plan output as a PR comment or CI artifact for infrastructure changes under review, so reviewers see the actual planned effect, not just the diff of `.tf` source.
```yaml
- run: terraform plan -no-color -out=tfplan | tee plan.txt
- uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        ...context.repo, issue_number: context.issue.number,
        body: "```\n" + require('fs').readFileSync('plan.txt', 'utf8') + "\n```"
      })
```
- **DO:** Treat a plan showing more changes than expected as a signal to stop and investigate, not to assume the tool knows best.
- **DON'T:** skip plan review for "small" changes on the assumption they're obviously safe. Some of the most damaging IaC incidents come from changes that looked trivial in source but produced a large, unreviewed destructive plan.

### Avoiding Hardcoded Credentials

- **DON'T:** hardcode cloud provider credentials, database passwords, or API keys directly in `.tf`/Pulumi source files. Even without committing them to a public repo, plaintext credentials end up duplicated into state files, plan output, and CI logs.
- **DO:** Source credentials from the environment — the provider's standard credential chain, such as instance profiles or workload identity, or environment variables injected by CI's secrets store — rather than provider blocks with inline access-key arguments.
```hcl
# DO: no static credentials in source at all;
# the AWS provider picks up OIDC-federated credentials
# from the environment automatically
provider "aws" {
  region = "us-east-1"
}
```
- **DO:** Mark sensitive input variables with `sensitive = true` so their values are redacted from plan and apply console output and logs.
```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```
- **DON'T:** pass a database password as a plain Terraform variable with a default value committed in `.tfvars`. Generate or manage the credential in a secrets manager, or use the provider's random-password-generation resources, and reference it.
- **DO:** Use your cloud provider's native identity federation for CI so the IaC pipeline authenticates with short-lived, scoped credentials instead of a long-lived static IAM user's access key.
- **DO:** Scope the IaC pipeline's own execution credentials to least privilege for what that specific configuration manages, not account-wide admin.
- **DON'T:** output sensitive values from a module or root config as plain, non-sensitive outputs. Mark any output that carries a secret as `sensitive`, and still treat the state file itself as sensitive regardless.
- **DO:** Rotate any credential that's ever appeared in IaC source, state, or CI logs, the same "treat exposure as compromise" discipline as with git-committed secrets.

### Idempotency

- **DO:** Design IaC configurations so running apply repeatedly against an unchanged configuration produces no further changes — a clean plan with zero diffs. This is what makes IaC trustworthy: re-running it after a partial failure should never have unintended side effects.
- **DON'T:** write provisioning logic — via `local-exec`/`null_resource` provisioners, or custom scripts — that has side effects when run more than once, such as a script that always appends to a file or always creates a resource with a timestamp-based name.
```bash
# DON'T: appends every time, not idempotent
echo "$(date): deployed" >> /var/log/deploys.log

# DO: idempotent — safe to run any number of times
grep -qxF "deployed" /var/log/deploys.log || echo "deployed" >> /var/log/deploys.log
```
- **DO:** Prefer declarative provider resources over imperative provisioners wherever a provider resource exists for the task. Provisioners sidestep the declarative model that gives Terraform its idempotency and drift-detection guarantees.
- **DO:** Make CI/CD deployment scripts themselves idempotent — a deploy script re-run against an already-deployed version should be a safe no-op or a clean re-apply, not an error or a duplicate side effect.
- **DON'T:** assume a resource's idempotency without verifying it, especially for cloud resources created via API calls with client-side-generated identifiers. Understand and handle each provider or API's actual idempotency semantics.
- **DO:** Use database migration tools that track applied migrations by identifier — Flyway, Alembic, golang-migrate — so re-running the migration step is a safe no-op for already-applied migrations.
- **DO:** Test idempotency explicitly as part of module or pipeline validation — run apply twice in a row in CI against a test environment and assert the second run shows zero planned changes.

## Common AI-Assistant Mistakes in DevOps

### Inventing CLI Flags or CI YAML Syntax That Doesn't Exist

- **DON'T:** fabricate a plausible-sounding CLI flag, Terraform argument, or CI YAML key that doesn't actually exist — inventing something like `docker build --squash-all`, a nonexistent `kubectl apply --dry-run=strict`, or a GitHub Actions key that isn't part of the schema. It looks correct, reads correctly, and fails only when someone actually runs it — the worst kind of wrong answer, because it passes a superficial read and fails only at execution.
- **DO:** Verify CLI flags and YAML schema against the actual, currently-installed tool version before presenting a command as correct, especially for less common flags or recently-changed tools such as Docker BuildKit syntax, version-dependent `kubectl` flags, or Terraform block syntax that changed across major versions. When uncertain, say so explicitly rather than presenting a guess with unearned confidence.
- **DON'T:** mix syntax from different tool versions or different-but-similar tools — blending Docker Compose v1 and v2's differing top-level key conventions, or letting GitHub Actions syntax bleed into a GitLab CI file. Each tool and version has a specific, non-interchangeable schema; a plausible-looking hybrid is still invalid and will fail to parse.
```yaml
# DON'T: GitHub Actions syntax pasted into a GitLab CI file
stages: [test]
test:
  stage: test
  # invalid here — this is a GitHub Actions concept, not GitLab CI:
  uses: actions/checkout@v4
```
- **DO:** Base generated CI pipeline YAML on the target platform's actual current schema — GitHub Actions, GitLab CI, CircleCI, and Jenkins each have meaningfully different structures — rather than a generic "CI pipeline shape" that happens to resemble valid syntax for a different platform.
```yaml
# correct GitHub Actions matrix syntax, specific to this platform's schema
strategy:
  matrix:
    node-version: [18, 20, 22]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
```
- **DON'T:** present an invented flag or config key with the same confidence as a verified one. If you're not certain a flag exists, flag that uncertainty explicitly rather than stating it as fact — an engineer who trusts a confidently-wrong flag wastes time debugging a config file that was never going to work.
- **DO:** Prefer flags and config documented in the tool's official reference over ones that "feel like they should exist" by analogy to a similar tool. Tool ecosystems are inconsistent by design — Docker's `--build-arg` and Kubernetes' secret-mounting conventions look similar but aren't interchangeable.
- **DON'T:** hallucinate a Terraform resource type, attribute, or provider that doesn't exist in the actual provider schema. Providers publish exact, versioned schemas; a resource or attribute that isn't in that schema fails at `init`, `plan`, or `validate` — but only after the user has already trusted and tried to use it.
- **DO:** When unsure whether a specific version of a tool supports a given syntax, recommend the user run the tool's built-in validation — `terraform validate`, `helm lint`, `yamllint`, the CI platform's own schema validator — rather than asserting correctness without that verification step.

### Generating Dockerfiles That Run as Root or Bake in Secrets

- **DON'T:** generate a Dockerfile that omits a `USER` directive and lets the container default to root, when asked to write a "production" or "best-practice" Dockerfile. This is one of the single most common corner-cutting mistakes in AI-generated Dockerfiles — the image builds and runs fine, so the omission is invisible until a security review or an actual compromise.
- **DO:** Include a non-root `USER` directive by default in any generated Dockerfile intended for a real service, along with the ownership and permission setup it requires — creating the user and `chown`-ing writable paths — not just the bare `USER app` line without the supporting setup.
```dockerfile
# minimum for a generated "production" Dockerfile —
# not just USER app on its own, with no user actually created
RUN addgroup --system app && adduser --system --ingroup app app
COPY --chown=app:app . /app
USER app
```
- **DON'T:** generate a Dockerfile with `ARG API_KEY` or a similar credential-shaped build argument, or a `COPY` of a credentials file, as a way to "make the example work." Even in illustrative code, this teaches — and risks getting copy-pasted into — an insecure pattern; show credential injection via BuildKit secret mounts or runtime environment variables instead.
- **DO:** Default generated Dockerfiles to multi-stage builds and a minimal, pinned base image, rather than a single-stage `FROM ubuntu:latest` with everything installed in one layer, when there's no strong reason the specific task requires otherwise.
- **DON'T:** omit `.dockerignore` guidance when generating a Dockerfile for a real project, letting a naive `COPY . .` slurp in `.git`, `.env`, and `node_modules`. Always pair a generated Dockerfile with the accompanying `.dockerignore` it depends on for safety.
- **DO:** Pin the base image tag, and mention digest-pinning as the stronger option, in generated Dockerfiles rather than defaulting to `FROM node:latest`/`FROM python:latest`. An AI-generated example that models `latest` usage normalizes exactly the practice that causes non-reproducible builds.
- **DON'T:** generate a Dockerfile that installs unnecessary packages "to be safe" — a full compiler toolchain, or `curl`/`wget`/`vim`/`ssh` in a production runtime image — without being asked for a debug or dev image specifically.
- **DO:** When a user explicitly asks for a quick, throwaway Dockerfile for local experimentation, it's fine to simplify — but say so explicitly, for example "this is a minimal example; for production, add a non-root user, pin versions, and use a multi-stage build," rather than silently presenting a corner-cut Dockerfile as if it were production-ready.

### Suggesting `--force`/`--no-verify` Instead of Fixing the Root Cause

- **DON'T:** suggest `git push --force`, `git commit --no-verify`, or similar override flags as a first response to a blocked git operation — a rejected push, a failing pre-commit hook, a failing pre-receive check. These flags exist for genuine edge cases, not as a default unblock button — reaching for them first treats a signal as an obstacle rather than as information.
- **DO:** Diagnose why a git hook, CI check, or push is failing before suggesting any bypass, and fix the underlying issue as the default recommendation. A pre-commit hook failing usually means it caught something real; skipping it with `--no-verify` doesn't fix the problem, it just hides it until later, often in CI, or worse, in production.
```bash
# DON'T lead with:
git commit --no-verify -m "fix"

# DO: find out what the hook actually caught, and fix that
npx eslint . --fix
git commit -m "fix: correct lint violations in checkout handler"
```
- **DON'T:** suggest `terraform apply -auto-approve` or `kubectl apply --force` to push through an unexpected plan or apply error without first explaining what the error means and what the forced action would actually do. Force flags in infrastructure tooling frequently mean "delete and recreate," and suggesting them as a quick fix without that context can cause real data loss or downtime.
- **DO:** When a genuine case for an override flag exists — a deliberate, coordinated force-push after an intentional, agreed-upon history rewrite — explain specifically why it's safe in this case and what could go wrong, rather than presenting the flag as a routine unblock.
- **DON'T:** suggest disabling a failing CI check — commenting it out, marking it non-required, adding `continue-on-error: true` — as a way to get a PR merged, without first determining whether the check caught a real problem. This is the CI-pipeline equivalent of `--no-verify`, and it's exactly how silently-broken pipelines accumulate over time.
- **DO:** When a check truly is a false positive — a genuinely flaky test unrelated to the change, a known tooling bug — say so explicitly and recommend the narrowly-scoped fix, such as retrying that specific job or quarantining that specific test with a tracked ticket, rather than a broad bypass that also suppresses the check for every future PR.
- **DON'T:** recommend deleting a lockfile and regenerating it, or using `--legacy-peer-deps`/similar dependency-resolution override flags, as a default response to a dependency conflict, without explaining what the conflict actually is. These can mask a genuine incompatibility that resurfaces later, in a much harder-to-diagnose form.
- **DO:** Treat every suggested override or bypass flag as something that needs its own explicit justification — what it does, why it's safe here specifically, and what the safer alternative would have looked like — rather than a terse one-line fix.

### Generating Destructive Commands Without Warning

- **DON'T:** generate a destructive command — `git push --force` to a shared branch, `git reset --hard`, `rm -rf`, `terraform destroy`, `kubectl delete namespace`, `docker volume rm`, a database `DROP TABLE`/`TRUNCATE` — without clearly flagging that it's destructive and explaining exactly what will be lost, before the user runs it. The line between helpful automation and silently deleting a week of someone's work is a single missing warning sentence.
- **DO:** State explicitly, before or alongside any destructive command, what data, state, or history will be irreversibly affected, and confirm that's actually the intended outcome. "This will permanently delete the `feature-x` branch's unmerged commits — confirm that's intended" takes one sentence and prevents an entire category of accidents.
- **DON'T:** default to the most destructive available option when a less destructive one accomplishes the same practical goal.
```bash
# risky default suggestion — discards uncommitted work permanently
git reset --hard

# safer alternative that preserves the same practical outcome
# (a clean working tree) while keeping the work recoverable
git stash push -m "wip before reset"
```
- **DO:** Distinguish, out loud, between commands that are locally reversible for a while, such as most git operations thanks to the reflog, and commands that are not, such as a force-push overwriting a remote others have already fetched from, or a dropped database table. The reversibility of the underlying operation should shape how much warning it gets.
- **DON'T:** chain a destructive command into a larger script or one-liner in a way that makes it easy to run without noticing it's there — for example burying `rm -rf ${dir}` inside a longer pipeline where an unset `${dir}` could expand to something unintended. Keep destructive operations visually distinct, and defensively guard variable expansion.
```bash
# DON'T: if $DEPLOY_DIR is ever unset, this expands to `rm -rf /`
rm -rf $DEPLOY_DIR/*

# DO: fail loudly instead of silently operating on the wrong path
: "${DEPLOY_DIR:?DEPLOY_DIR must be set}"
rm -rf "${DEPLOY_DIR:?}"/*
```
- **DO:** For infrastructure-destroying operations specifically, recommend a dry-run or plan step first — `terraform plan -destroy`, `kubectl delete --dry-run=client` — so the user sees exactly what would be removed before it actually happens.
- **DON'T:** suggest deleting a production resource — a database, a persistent volume, a cloud storage bucket — as a troubleshooting step without first suggesting a backup or snapshot, unless the user has explicitly confirmed data loss is acceptable.
- **DO:** Ask for explicit confirmation, or clearly pause for it, before generating a command that would irreversibly affect shared or production state, rather than presenting the destructive command as the natural next step in a sequence the user may run without rereading closely.
- **DON'T:** understate the blast radius of a command by describing only its direct target while omitting cascading effects — `kubectl delete namespace` also deletes every resource inside it, and a `terraform destroy` on a module can cascade to every resource that module manages, including ones that look unrelated at a glance.

### Ignoring Existing CI Conventions and Inventing a New Pipeline Structure

- **DON'T:** propose a brand-new CI pipeline structure, tool, or config format when the repository already has a working CI setup, unless the user specifically asked for a migration or a from-scratch redesign. Introducing GitHub Actions into a repo that already has a mature GitLab CI pipeline, or vice versa, fragments the team's tooling for no requested benefit.
- **DO:** Read and match the existing pipeline's conventions — job naming, stage structure, reusable workflow or template usage, existing caching strategy — before adding a new job or step, so the addition looks like it belongs and a future reader can't tell it wasn't written by the same person who wrote the rest.
- **DON'T:** reintroduce a check, tool, or convention the team has already deliberately moved away from — such as re-adding a linter config the team explicitly migrated off of — without first understanding why it was removed. Existing CI config is usually the product of accumulated lessons, not just an arbitrary starting point to improve on unprompted.
- **DO:** When existing CI config has an inconsistency or a clear improvement opportunity unrelated to the task at hand, mention it separately rather than silently "fixing" it as a side effect of an unrelated change. Bundling an unrequested pipeline refactor into an unrelated PR is the CI-config version of unrequested scope creep in application code.
- **DON'T:** assume the user's CI platform, runner environment, or available secrets and tooling match a generic default — assuming Ubuntu runners with Docker pre-installed, or assuming a specific package manager — without checking the actual existing pipeline config for what's really available.
- **DO:** Extend an existing reusable workflow, template, or orb rather than duplicating its logic inline, when the repo already has that abstraction in place for similar jobs.
- **DON'T:** silently change the meaning of an existing required status check's name — renaming a job that's referenced by branch-protection rules — without flagging that branch protection settings will also need updating. A renamed check that branch protection no longer recognizes can either block all merges or silently stop being enforced.
- **DO:** When genuinely proposing to replace or significantly restructure an existing pipeline, because the user asked or because the existing one is clearly broken, do it as a clearly-scoped, explicitly-flagged change with a migration plan, not as an incidental side effect while doing something else the user actually asked for.

### Not Pinning Dependency/Image Versions

- **DON'T:** generate Dockerfiles, CI configs, or IaC that reference `latest`, an unpinned major-version range, or a moving branch — `FROM node:latest`, `uses: actions/checkout@main`, an unconstrained Terraform provider — by default. Every one of these makes the exact same input produce a potentially different output tomorrow, the opposite of what both reproducible builds and safe automation need.
- **DO:** Pin to a specific version by default in generated examples — a semver tag for images, a release tag or commit SHA for third-party GitHub Actions, and explicit version constraints for Terraform providers, modules, and package dependencies.
```yaml
# weaker: a mutable tag the action's maintainer (or an attacker
# who compromises their account) can silently repoint
- uses: some-org/some-action@v1

# stronger: pinned to an exact, immutable commit
- uses: some-org/some-action@a1b2c3d4e5f6...  # v1.4.2
```
- **DON'T:** pin so tightly and then never revisit it, letting dependencies silently accumulate missing security patches. Pinning solves reproducibility, not staying patched — pair pinned versions with a deliberate update process, such as Dependabot or Renovate, not a "pin once and forget forever" approach.
- **DO:** When suggesting a third-party GitHub Action, Docker base image, or Terraform module, prefer well-maintained, widely-used sources and mention the supply-chain value of pinning to a commit SHA, not just a tag, for anything with meaningful permissions such as repo write access or secret handling.
- **DON'T:** leave a floating major-version range presented as sufficient version control when there's no committed, CI-respected lockfile actually being installed from. A range plus no lockfile means "whatever's newest when this happens to install" — pair ranges with a lockfile for actual reproducibility.
- **DO:** Explain the reproducibility trade-off explicitly when a user asks for the "latest" version of something on purpose, such as for local experimentation — it's fine to use `latest` there, but say so, and don't silently carry that same `latest` reference into a generated production Dockerfile or manifest elsewhere in the same response.
- **DON'T:** generate an IaC provider block or module source with no version constraint at all. An unconstrained provider can introduce breaking schema changes on a teammate's next `init`, at a time completely disconnected from when the actual config change was reviewed.
- **DO:** When updating a pinned version — bumping a base image, an action, a provider — do it as its own deliberate, reviewable change, ideally automated via Dependabot or Renovate with its own PR, rather than folding a silent version bump into an unrelated code change.

## Quick Checklist
- Write commit subjects in the imperative mood, with a body that explains why, not just what.
- Keep commits atomic — one logical change per commit — and squash "oops"/"wip" commits before opening a PR.
- Never rewrite git history that's already been pushed and pulled by others without coordinating first.
- Keep PRs small and focused on one logical change; self-review the diff before requesting review.
- Write PR descriptions that state what changed, why, and how to verify it, with a rollback note for risky changes.
- Review design and intent before nitpicking style in code review; save nits for a clearly-labeled "nit:" comment.
- Commit a `.gitignore` before the first commit; never `git add .` reflexively without checking `git status` first.
- Never commit secrets, `.env` files, or private keys; if one leaks, rotate it immediately regardless of history cleanup.
- Use Git LFS or object storage for large binaries instead of committing them directly to the repo.
- Prefer `git push --force-with-lease` over plain `--force` when force-pushing your own rebased branch.
- Resolve merge conflicts by understanding both sides, then re-run the test suite before trusting the resolution.
- Tag releases with SemVer and annotated tags; never move or retag an already-published version.
- Use multi-stage Docker builds to keep compilers and dev dependencies out of the shipped runtime image.
- Start Dockerfiles from a minimal, explicitly pinned base image; never build production images from `latest`.
- Order Dockerfile instructions from least- to most-frequently-changing to maximize layer cache hits.
- Always run application containers as a non-root `USER`, with ownership set correctly on writable paths.
- Add a `.dockerignore` that excludes `.git`, `.env`, `node_modules`, and other build-irrelevant files.
- Clean up package-manager caches in the same `RUN` layer that installed them, not a later one.
- Never bake secrets into image layers via `ARG`, `ENV`, or `COPY`; use BuildKit secret mounts or runtime injection.
- Pin service image versions in `docker-compose.yml`, and keep secrets in a gitignored `.env`, not the committed file.
- Add health checks and `depends_on: condition: service_healthy` to Compose services with startup dependencies.
- Set CPU and memory requests and limits on every Kubernetes container, based on real observed usage.
- Define both liveness and readiness probes; never point a liveness probe at a downstream dependency check.
- Set `runAsNonRoot`, drop all Linux capabilities, and use a read-only root filesystem in pod security contexts.
- Store configuration in ConfigMaps and sensitive values in Kubernetes Secrets, never hardcoded in manifests or images.
- Apply default-deny NetworkPolicies and per-namespace ResourceQuotas on any shared or multi-tenant cluster.
- Never deploy a floating `:latest` image tag to production; pin to a specific version tag or digest.
- Version Helm charts independently from application versions, and lint/template them in CI before every `helm upgrade`.
- Order CI pipeline stages fast-to-slow, and fail fast on the first failing required check.
- Run independent CI checks in parallel, and use reusable workflows or templates instead of copy-pasted job definitions.
- Cache CI dependencies keyed on a lockfile hash, with a fallback restore key for partial cache hits.
- Quarantine known-flaky tests with a tracked follow-up ticket; never silently tolerate "just re-run CI."
- Store CI secrets only in the platform's dedicated secrets store, and never echo or print them in pipeline logs.
- Prefer OIDC-based short-lived credentials over long-lived static cloud keys stored as CI secrets.
- Match deployment strategy — rolling, blue-green, or canary — to the service's actual risk tolerance and traffic pattern.
- Make every deployment revertible with a single known command, and rehearse the rollback before an incident forces it.
- Use expand/contract database migrations so an application rollback never breaks against an already-migrated schema.
- Keep dev, CI, staging, and production on the same pinned tool and runtime versions to avoid "works on my machine."
- Store Terraform or Pulumi state in a remote, locked backend; never hand-edit a `.tfstate` file.
- Never commit state files to git — they routinely contain secrets and sensitive resource attributes in plaintext.
- Split IaC state per environment and component; avoid one monolithic state file for all infrastructure.
- Never make manual console or CLI changes to IaC-managed resources; fix the source and re-apply instead.
- Run scheduled drift detection (`plan`/`preview` against live state) to catch configuration that diverged from source.
- Design IaC modules with a minimal, coherent interface, and pin every consumer to a specific module version.
- Always review `plan`/`preview` output before every `apply`, paying special attention to any destroy or replace action.
- Never use `-auto-approve` outside a fully automated, already-plan-reviewed pipeline.
- Source cloud credentials from the environment or OIDC federation; never hardcode them in `.tf`/Pulumi source files.
- Mark sensitive Terraform variables and outputs `sensitive = true`, and still treat the state file itself as sensitive.
- Design IaC configurations and deploy scripts to be idempotent — re-running against no change should be a safe no-op.
- Never invent a CLI flag, YAML key, or Terraform resource; verify against the real schema or say explicitly you're unsure.
- Default every generated Dockerfile to non-root, multi-stage, and free of baked-in secrets, unless told otherwise.
- Never suggest `--force`, `--no-verify`, or `-auto-approve` as a first fix; diagnose and fix the root cause instead.
- Always warn explicitly, in words, before generating a destructive command, stating exactly what will be lost.
- Match a repo's existing CI conventions instead of inventing a new pipeline structure or tool unasked.
- Pin every generated dependency, base image, and third-party action to a specific version — never default to `latest`/`@main`.
- Treat any secret that ever appeared in git history, an image layer, IaC state, or a CI log as compromised and rotate it.
- Prefer the narrowest, most reversible operation that accomplishes the goal over the most destructive available option.
