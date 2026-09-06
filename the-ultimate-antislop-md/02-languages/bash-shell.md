# Bash/Shell

## Quoting Discipline

- **DO:** Treat quoting as the default and unquoted expansion as the rare, deliberate exception that needs a comment explaining why — flip the mental default from "quote when I remember to" to "unquoted needs a reason," since the cost of forgetting to quote is a real, exploitable bug while the cost of an unnecessary quote is approximately zero.
- **DO:** Use `set -- "${args[@]}"` (or an equivalent) to reset a function's or script's positional parameters from an array explicitly and safely when needed, rather than relying on implicit re-splitting of a concatenated string.
- **DO:** Double-quote every variable expansion (`"$var"`, `"${array[@]}"`, `"$(command)"`) unless you have a specific, deliberate reason to allow word-splitting and globbing. An unquoted variable is silently split on whitespace and glob-expanded, which is almost never what's intended and is the single most common source of shell scripting bugs.
  ```bash
  # BAD — breaks on any filename containing a space, and globs unexpectedly
  for file in $(ls *.txt); do
      rm $file
  done

  # GOOD — quoted expansion, safe with spaces/globs in filenames
  for file in *.txt; do
      rm -- "$file"
  done
  ```
- **DO:** Quote command substitutions (`"$(command)"`), not just simple variables — an unquoted `$(...)` undergoes the exact same word-splitting and glob-expansion as an unquoted variable.
- **DO:** Use `"${array[@]}"` (quoted, with `@` not `*`) to expand an array into separate, correctly-quoted words — `"${array[*]}"` joins all elements into a single string using the first character of `IFS`, which is a different (and usually wrong) operation.
- **DON'T:** Rely on the historical convention of leaving variables unquoted "because it usually works." It usually works only because most test inputs don't contain spaces, globs, or empty values — the failure mode shows up later, in production, on exactly the input nobody tested with (a filename with a space, an empty variable that vanishes into nothing instead of an empty string).
- **DO:** Quote the right-hand side of a `[[ ]]` test when comparing against a variable that could contain glob characters or be empty (`[[ "$name" == "$pattern" ]]`), and understand that inside `[[ ]]` unquoted variables are safer than inside `[` `]` (POSIX test) but quoting is still the unambiguous, portable habit.
- **DO:** Use `printf '%s\n' "$var"` instead of `echo $var` when the value might start with a dash (which `echo` can misinterpret as an option) or contain backslash escapes whose interpretation varies by shell/`echo` implementation.
- **DON'T:** Build a command by concatenating unquoted, unvalidated strings into a variable and then executing that variable directly (`cmd=$1 args=$2; $cmd $args`). Beyond word-splitting bugs, this pattern is a step away from arbitrary command injection if any of the inputs are attacker-influenced.
- **DO:** Use arrays to build up a command line with a variable number of arguments, rather than concatenating them into a single string — arrays preserve each argument as a distinct word regardless of embedded spaces.
  ```bash
  # BAD — a filename with a space breaks into two arguments
  args="-l -a $target_dir"
  ls $args

  # GOOD — each element stays a distinct argument even with spaces
  args=(-l -a "$target_dir")
  ls "${args[@]}"
  ```
- **DO:** Quote glob patterns you don't want expanded by the shell before a command sees them (e.g., passing a literal `*` to `grep` or `find -name`), since an unquoted glob is expanded by the shell itself before the command ever runs, which is rarely the intended behavior for a search pattern.
- **DON'T:** Use single quotes when a variable inside the string needs to expand, or double quotes when it must not — know which quoting style you need per string: single quotes suppress all expansion (including `$variable`), double quotes allow variable/command substitution but suppress globbing and word-splitting.
- **DO:** Quote `$@` as `"$@"` specifically (not `"$*"` or unquoted `$@`) when forwarding a script's or function's positional arguments to another command, since `"$@"` is the one form that preserves each original argument as a distinct word, spaces and all.
  ```bash
  # GOOD — forwards each argument to the wrapped command unchanged,
  # correctly, even if some contain spaces
  run_with_logging() {
      echo "Running: $*" >&2
      real_command "$@"
  }
  ```
- **DO:** Quote command arguments that come from a config file, environment variable, or any other external source exactly as carefully as ones from direct user input — "internal" data sources are just as capable of containing spaces or special characters as a user-typed value.
- **DON'T:** Assume a variable is never empty and therefore safe to leave unquoted. An unquoted empty variable disappears entirely from the command line (`cmd $empty_var extra` becomes `cmd extra`, silently dropping an argument position), which can shift positional arguments in a way that's very hard to debug.
  ```bash
  # BAD — if $flag is empty, this becomes `cp file` with no destination
  cp $file $flag $dest

  # GOOD — quoted, so an empty variable stays a distinct (empty) argument
  cp "$file" "$flag" "$dest"
  ```
- **DO:** Use `printf -v var 'format' args` to build a formatted string into a variable safely, instead of concatenating with `var="$var$something"` in a loop when formatting (not just concatenation) is involved.
- **DO:** Quote here-string and here-document delimiters correctly — an unquoted heredoc delimiter (`<<EOF`) allows variable expansion inside the block, while a quoted one (`<<'EOF'`) suppresses it — and choose deliberately based on whether the heredoc's content should have variables substituted.
  ```bash
  # Quoted delimiter: $HOME is printed literally, not expanded
  cat <<'EOF'
  Your home directory is $HOME
  EOF
  ```
- **DO:** Quote the pattern side of a `case` statement's branches when a variable (not a literal glob) is being matched, since an unquoted variable there is still subject to the same word-splitting concerns as anywhere else, even though `case` patterns support globbing.
- **DON'T:** Interpolate a variable directly into a regular expression passed to `grep`/`sed`/`awk` without considering that the variable's content might itself contain regex metacharacters (`.`, `*`, `[`, `$`) that get interpreted rather than matched literally — use each tool's fixed-string mode (`grep -F`, `sed`'s escaped literal matching) when the variable's content should be treated as a literal string, not a pattern.
  ```bash
  # BAD — if $search contains "a.b", the "." matches any character,
  # not a literal dot, silently over-matching
  grep "$search" file.txt

  # GOOD — -F treats the pattern as a fixed literal string
  grep -F "$search" file.txt
  ```
- **DO:** Quote the target of a redirection (`> "$output_file"`) exactly as carefully as any other variable use — an unquoted redirection target is just as susceptible to word-splitting as an unquoted command argument, and a filename with a space silently redirects to the wrong (truncated) path.
- **DO:** Use `"${var:-default}"` (parameter expansion with a default) for a quoted, inline fallback value instead of a separate `if [ -z "$var" ]; then var=default; fi` block, when the fallback is a simple literal and the extra verbosity of a full conditional isn't warranted.
  ```bash
  # Quoted parameter expansion: falls back to "production" if $ENV is unset/empty
  environment="${ENV:-production}"
  ```
- **DO:** Use associative arrays (`declare -A config; config[timeout]=30`) for structured key-value script configuration instead of a series of loosely related individually-named variables, when a script's Bash version target (4.0+) supports them — an associative array groups related settings under one name and can be iterated over generically, where a pile of separately named variables can't.
- **DO:** Quote here-string input (`command <<< "$variable"`) exactly as carefully as any other variable use — a here-string is still subject to the same word-splitting-adjacent concerns as far as what gets passed to the command's stdin if the quoting is dropped.
- **DO:** Use `${var//pattern/replacement}` (Bash's built-in string substitution) for simple in-shell text substitution instead of shelling out to `sed`/`awk` for a substitution the shell itself can already do — this avoids an unnecessary subprocess for a simple, common operation.
  ```bash
  path="/usr/local/bin"
  echo "${path//\//_}"   # => _usr_local_bin, no external sed process needed
  ```

## Error Handling

- **DO:** Trap signals explicitly (`trap 'handle_sigterm' TERM`, `trap 'handle_sigint' INT`) for a long-running script (a daemon-like loop, a deploy script that should clean up on interruption) so the script can shut down gracefully — release locks, kill child processes, remove temp files — instead of being killed abruptly mid-operation with no chance to clean up.
  ```bash
  cleanup() { echo "Shutting down gracefully..." >&2; kill "$child_pid" 2>/dev/null; }
  trap cleanup TERM INT
  long_running_command &
  child_pid=$!
  wait "$child_pid"
  ```
- **DO:** Never hardcode secrets (API keys, passwords, tokens) directly in a script's source — read them from environment variables, a secrets manager, or a permission-restricted file the script reads at runtime, since a script's source is far more likely to be committed to version control, logged, or shared than a runtime-only secret store.
- **DON'T:** Echo or log a secret value while debugging (`echo "token=$API_TOKEN"`, or `set -x` left enabled around a block handling secrets) — `set -x` in particular prints every expanded command including secret values to stderr, which commonly ends up in a CI log that isn't as access-restricted as intended.
- **DO:** Use `wait` with a specific PID (or PIDs) to wait for background jobs launched with `&`, checking each one's individual exit status via `wait "$pid"; status=$?` when a script needs to know which specific background job failed, rather than a bare `wait` that only reports the last job's status.
- **DO:** Start every non-trivial script with `set -euo pipefail`: `-e` exits immediately on any command's non-zero exit status, `-u` treats an unset variable reference as an error instead of silently expanding to an empty string, and `-o pipefail` makes a pipeline's exit status reflect the last command that actually failed, not just the final command in the pipe.
  ```bash
  #!/usr/bin/env bash
  set -euo pipefail
  IFS=$'\n\t'
  ```
- **DON'T:** Assume `set -e` catches every failure — it does not trigger inside a condition being tested (`if some_command; then`), inside a pipeline's non-last command without `pipefail`, inside a command whose result feeds `&&`/`||`, or inside a function called as part of a condition. Understand these gaps rather than treating `set -e` as a blanket safety net.
- **DO:** Check the exit status of critical commands explicitly when the failure needs specific handling (a custom error message, a cleanup step, a retry) rather than relying solely on `set -e` to abort the whole script generically.
  ```bash
  if ! curl -fsSL "$url" -o "$dest"; then
      echo "ERROR: failed to download $url" >&2
      exit 1
  fi
  ```
- **DO:** Set `IFS=$'\n\t'` (or otherwise deliberately control the internal field separator) near the top of scripts that do any word-splitting on purpose, so splitting happens only on newlines and tabs, not on any whitespace including spaces inside filenames.
- **DO:** Use `trap 'cleanup' EXIT` (and optionally `ERR`, `INT`, `TERM`) to guarantee cleanup code (removing a temp file, releasing a lock) runs whether the script exits normally, via an error, or via a signal — this is the shell equivalent of `finally`/`ensure`/`defer`.
  ```bash
  tmpfile=$(mktemp)
  trap 'rm -f "$tmpfile"' EXIT
  ```
- **DO:** Send error and diagnostic messages to stderr (`echo "error: ..." >&2`), not stdout, so a script's actual output stream stays clean for piping while error messages remain visible on the terminal and in logs.
- **DO:** Use meaningful, distinct exit codes for different failure categories in scripts whose exit status is consumed programmatically (by CI, by another script), documenting what each non-zero code means, rather than always exiting `1` for every possible failure.
- **DON'T:** Silently discard a command's failure with `command || true` or `command 2>/dev/null` as a default habit to "make the script keep going." Reserve this for cases where the failure is genuinely expected and safe to ignore (checking whether an optional tool exists), and comment why — an unexplained swallowed failure is indistinguishable from a bug at review time.
- **DO:** Validate script arguments and required environment variables explicitly at the top of the script, failing fast with a clear usage message, rather than letting a missing argument surface as a confusing failure many lines later.
  ```bash
  if [[ $# -lt 1 ]]; then
      echo "Usage: $0 <target-directory>" >&2
      exit 64
  fi
  ```
- **DO:** Prefer `command -v tool` over `which tool` to check whether a command exists (`which` is not POSIX-specified, its output format varies by system, and it's often just a wrapper around a shell builtin that duplicates `command -v`'s job less reliably).
- **DO:** Use `set -x` (or `bash -x script.sh`) temporarily during debugging to print every command as it executes with its expanded arguments, which is often the fastest way to see exactly where a quoting or logic error actually diverges from expectation.
- **DO:** Write functions with a `local` declaration for every variable that shouldn't leak into the caller's scope (`local result; result=$(compute)`), since Bash variables are global by default unless explicitly scoped, and an unscoped variable inside a function can silently shadow or overwrite a caller's variable of the same name.
  ```bash
  # BAD — count leaks into the global scope and can clobber a
  # caller's own variable named `count`
  process_items() {
      count=0
      for item in "$@"; do count=$((count + 1)); done
      echo "$count"
  }

  # GOOD — scoped to the function, no risk of collision
  process_items() {
      local count=0
      for item in "$@"; do count=$((count + 1)); done
      echo "$count"
  }
  ```
- **DO:** Check a command's actual exit status via `$?` (or, better, an `if`/`&&`/`||` directly on the command) immediately after the command, since `$?` reflects only the most recently executed command and is overwritten by the very next command, including an unrelated one run to check or log something.
- **DON'T:** Assume a function's `return` code communicates the same thing as its stdout output — `return` in Bash sets only a small integer exit status (0-255), it does not return arbitrary data. Use `echo`/`printf` plus command substitution (`result=$(my_func)`) to return actual data, and reserve `return`'s exit code for success/failure signaling.
- **DO:** Use `getopts` (POSIX-compatible) or a manual `while`/`case` argument parser for scripts accepting command-line flags, rather than positional-only argument handling that becomes unreadable once a script accepts more than one or two options.
  ```bash
  while getopts "vo:" opt; do
      case "$opt" in
          v) verbose=1 ;;
          o) outfile="$OPTARG" ;;
          *) echo "Usage: $0 [-v] [-o outfile]" >&2; exit 64 ;;
      esac
  done
  ```

## Avoiding Parsing `ls`

- **DON'T:** Parse the output of `ls` to iterate over files or extract file attributes. `ls`'s output format is meant for human reading, not machine parsing — it can be affected by locale settings, column formatting, filenames containing newlines or unusual characters, and aliases/wrapper scripts that change its default flags.
  ```bash
  # BAD — breaks on filenames with spaces or newlines, and depends
  # on ls's column formatting rather than a stable machine format
  for f in $(ls *.log); do
      echo "$f"
  done

  # GOOD — the shell's own glob expansion, no external parsing needed
  for f in *.log; do
      [[ -e "$f" ]] || continue   # guard against a glob matching nothing
      echo "$f"
  done
  ```
- **DO:** Use the shell's native globbing (`for f in *.txt`) for straightforward file iteration — it's built into the shell, correctly handles spaces and special characters in filenames when the loop variable is quoted, and needs no external process.
- **DO:** Use `find ... -print0 | xargs -0 ...` (or `find ... -exec ... +`) when you need recursive traversal or filtering by attribute (age, size, permissions) — the null-byte-delimited (`-print0`/`-0`) pairing is immune to filenames containing spaces or even newlines, unlike newline-delimited output.
  ```bash
  # GOOD — null-delimited, safe with any filename including embedded newlines
  find . -name '*.tmp' -print0 | xargs -0 rm -f --
  ```
- **DON'T:** Assume filenames never contain spaces, newlines, or leading dashes in a script meant to run against real-world or user-supplied data — a script that only handles "normal" filenames will eventually be run against one that isn't, and the null-byte-delimited idiom exists specifically to handle that case correctly.
- **DO:** Use `[[ -e "$f" ]]`, `[[ -d "$f" ]]`, `[[ -r "$f" ]]`, etc. (the shell's own file-test operators) to check file existence/type/permissions directly, rather than parsing `ls -l`'s permission-string column or `stat`'s human-readable output.
- **DO:** Enable `shopt -s nullglob` (bash) when a glob pattern that matches nothing should expand to zero arguments instead of the literal unexpanded pattern string — otherwise a `for f in *.txt` loop over an empty directory silently iterates once over the literal string `"*.txt"`.
- **DO:** Use `stat --format` (GNU) or `stat -f` (BSD/macOS, different format string syntax) when a script genuinely needs a file's size, modification time, or owner programmatically — the two `stat` implementations are not command-line compatible, so scripts intended to run on both need to detect or special-case the platform, or avoid `stat` in favor of a portable alternative.
- **DO:** Use `mapfile`/`readarray` (Bash 4+) to read a command's newline-delimited output into an array in one step, rather than a manual `while read` loop appending to an array element-by-element, when the source data is well-behaved newline-delimited output (not filenames, which need the null-delimited approach instead).
  ```bash
  mapfile -t lines < <(grep -c "ERROR" *.log)
  ```
- **DON'T:** Pipe a command's output into a `while read` loop when the loop body needs to set variables that must persist after the loop — a pipeline runs its last stage in a subshell in most shells, so variables set inside `cmd | while read line; do var=$line; done` don't survive past the loop. Use process substitution (`while read line; do ...; done < <(cmd)`) instead, which avoids the subshell.
  ```bash
  # BAD — count is set inside a subshell and lost after the pipe
  count=0
  find . -name '*.txt' | while read -r f; do count=$((count + 1)); done
  echo "$count"   # prints 0, not the real count

  # GOOD — process substitution avoids the subshell
  count=0
  while read -r f; do count=$((count + 1)); done < <(find . -name '*.txt')
  echo "$count"
  ```
- **DO:** Use `read -r` (the `-r` flag) whenever reading a line into a variable, so backslashes in the input are treated literally rather than as line-continuation/escape characters — omitting `-r` is a subtle, common bug when processing filenames or data containing backslashes.
- **DO:** Use `find ... -maxdepth N` to bound recursive traversal depth deliberately when a script only needs to look one or two levels deep, rather than letting an unbounded recursive `find` walk an entire (possibly enormous) directory tree it didn't need to touch.
- **DON'T:** Use `ls -1 | wc -l` to count files in a directory — this still parses `ls` output and misbehaves on filenames containing newlines; use `find . -maxdepth 1 -type f -print0 | grep -zc ''` or a shell glob array's length (`files=(*); echo "${#files[@]}"`) instead.
- **DO:** Prefer glob qualifiers (Bash's `shopt -s extglob`/`globstar`, or simpler direct patterns like `*.txt`) over piping `ls` through `grep`/`awk` to filter files by name pattern, since the shell's own glob matching avoids the whole "parse text output" problem `ls`-piping introduces.

## Portability (POSIX sh vs. Bash-isms)

- **DO:** Decide explicitly whether a script targets POSIX `sh` or Bash specifically, and set the shebang accordingly (`#!/bin/sh` for POSIX-only, `#!/usr/bin/env bash` for Bash-specific features) — this single decision determines which features (arrays, `[[ ]]`, `local`, string manipulation operators) are actually safe to use.
- **DON'T:** Use Bash-only features (`[[ ]]`, arrays, `local`, `((...))` arithmetic, `${var,,}`/`${var^^}` case conversion, `<<<` here-strings, process substitution `<(...)`) in a script with a `#!/bin/sh` shebang or one explicitly documented as POSIX-compliant. On systems where `/bin/sh` is `dash` or another minimal shell (common on Debian/Ubuntu and many containers), these constructs fail outright rather than silently degrading.
  ```bash
  # BAD in a #!/bin/sh script — [[ ]] and arrays are bash-only
  #!/bin/sh
  arr=(a b c)
  if [[ "$1" == "start" ]]; then ...

  # GOOD — POSIX-compatible equivalents
  #!/bin/sh
  set -- a b c
  if [ "$1" = "start" ]; then ...
  ```
- **DO:** Use `[ ]` (POSIX test) with `=` for string comparison in POSIX `sh` scripts, since `==` inside `[ ]` is a non-portable Bash/ksh extension even though many Bash installations silently accept it.
- **DO:** Use `$(...)` for command substitution over backticks (`` `...` ``) in both POSIX and Bash scripts — `$(...)` nests cleanly without escaping and is POSIX-standard, while backticks require awkward backslash-escaping when nested and are harder to read.
- **DON'T:** Assume `bash` is present at `/bin/bash` on every target system — some minimal containers and embedded systems ship without Bash at all, only a POSIX-compliant `sh` (often `dash` or `busybox sh`). Check the actual deployment target before choosing Bash-specific syntax, especially for scripts meant to run inside Docker `RUN` layers, init scripts, or cross-platform tooling.
- **DO:** Test POSIX-targeted scripts against `dash` specifically (`dash script.sh`, or `checkbash -p`-style tooling), not just against Bash running in POSIX-compatibility mode, since Bash's POSIX mode doesn't perfectly replicate every real POSIX shell's behavior.
- **DO:** When a script genuinely needs Bash-only features (associative arrays, robust string manipulation, `mapfile`), just require Bash explicitly via the shebang and document the requirement, rather than attempting a half-POSIX/half-Bash script that's fragile in both directions.
- **DON'T:** Rely on GNU-specific flags for standard Unix utilities (`sed -i` without a backup-suffix argument, `date -d`, certain `grep -P`/Perl-regex support) in a script meant to be portable to BSD/macOS systems, where the same tools' default builds behave differently or lack the flag entirely. Either detect the platform and branch, or use flags common to both implementations.
  ```bash
  # GNU sed: sed -i 's/foo/bar/' file.txt
  # BSD/macOS sed requires an explicit (even if empty) backup suffix:
  # sed -i '' 's/foo/bar/' file.txt
  # Portable-ish approach: write to a temp file and move it into place.
  sed 's/foo/bar/' file.txt > file.txt.tmp && mv file.txt.tmp file.txt
  ```
- **DO:** Use `#!/usr/bin/env bash` rather than a hardcoded `#!/bin/bash` shebang for portability across systems where Bash isn't installed at exactly `/bin/bash` (some BSD/macOS setups, some Linux distributions using an alternate install path) — `env` looks up `bash` via `PATH` instead of assuming a fixed location.
- **DON'T:** Assume Bash's associative arrays (`declare -A`), available since Bash 4.0, are available everywhere — macOS shipped Bash 3.2 (an old, pre-GPLv3 version) as its default `/bin/bash` for a very long time due to licensing, so a script relying on Bash 4+ features needs either a Homebrew-installed newer Bash explicitly on `PATH`, or an associative-array-free fallback, if it must run unmodified on stock macOS.
- **DO:** Test scripts against the actual minimum Bash/shell version the project claims to support, not just whatever version happens to be installed on the developer's own machine, since Bash version differences are a common, easy-to-miss source of "works on my machine" script failures.
- **DO:** Use `[ -n "$var" ]`/`[ -z "$var" ]` (POSIX-portable) rather than Bash's `[[ -n $var ]]` inside a script that must run under `sh`, and understand that `[[ ]]` itself is simply unavailable (a syntax error) under a strict POSIX `sh`.
- **DO:** Use `printf` for portable, predictable output formatting instead of `echo`, whose handling of backslash escapes and flags (`-e`, `-n`) differs across shells and even across different builds of the same shell — `printf`'s behavior is consistent and POSIX-specified.
- **DON'T:** Assume `local` is available in every POSIX `sh` implementation — it's a near-universal extension supported by Bash, dash, and most other shells in practice, but it's technically not required by the POSIX standard itself, so a script targeting maximum strict portability should verify it against the actual minimum shell the project commits to supporting.
- **DO:** Use POSIX-compliant arithmetic (`$(( ))`, supported broadly including in POSIX `sh`) rather than the external `expr` command for arithmetic, since `$(( ))` is both more efficient (no subprocess) and less error-prone than `expr`'s comparatively fragile argument-based syntax.

## Shellcheck Conventions

- **DO:** Run `shellcheck` on every shell script as a standard part of the development workflow (locally via editor integration, and in CI as a gate on merges) — it catches an enormous class of real bugs (unquoted expansions, unreachable code after unconditional exits, misused test operators) that are easy to miss by eye.
- **DO:** Address a shellcheck warning by fixing the underlying issue in the vast majority of cases, rather than reaching immediately for a suppression directive — most shellcheck warnings (like `SC2086`, unquoted variable) point at a genuine, reproducible bug, not a false positive.
- **DO:** When a specific shellcheck warning is a deliberate, understood false positive for a specific line, suppress only that one warning on that one line with a directive comment explaining why (`# shellcheck disable=SC2034  # used indirectly via eval below`), rather than disabling the check globally for the whole file.
  ```bash
  # GOOD — narrowly scoped suppression with a reason, not a blanket disable
  # shellcheck disable=SC2064  # intentional early expansion of $tmpfile
  trap "rm -f $tmpfile" EXIT
  ```
- **DON'T:** Add a file-wide or project-wide shellcheck disable comment for a whole category of check (e.g., disabling all quoting warnings) just to make the linter stop complaining across many lines — this defeats the purpose of running the linter at all and hides genuinely new bugs behind the same blanket suppression.
- **DO:** Pin the shellcheck version used in CI (via a container image tag or explicit installed version) so warnings don't silently appear or disappear across unrelated commits purely because the linter itself was upgraded.
- **DO:** Treat shellcheck's "info"-level style suggestions (preferring `$(...)` over backticks, suggesting `printf` over `echo` in specific cases) as worth adopting for consistency even when they're not bug-level severity, since they steer the codebase toward the more robust idiom by default.
- **DON'T:** Ignore shellcheck's warnings about unquoted variables specifically (`SC2086`) as "probably fine" — this is the single check most directly tied to the most common real-world shell scripting bug (word-splitting/globbing on an unquoted expansion), and it should essentially never be suppressed without a very specific, documented reason.
- **DO:** Run shellcheck against a script's declared shell dialect specifically (`shellcheck -s sh` for a POSIX-targeted script, `shellcheck -s bash` for Bash), since shellcheck's warnings differ based on which dialect it's checking against — running it in the wrong mode can either miss genuine POSIX-portability violations or produce false-positive warnings about legitimate Bash-only syntax.
- **DO:** Fix shellcheck warnings incrementally when adopting it on a large pre-existing codebase (using its baseline/ignore-file support to suppress only pre-existing warnings while enforcing a clean bill of health on new and changed code) rather than either fixing nothing or attempting a single enormous fix-everything commit that's hard to review.
- **DO:** Treat a shellcheck failure in CI as a hard merge-blocking gate, the same as a failing test, rather than a warning that's easy to ignore in a PR review — shell scripts often run with elevated privileges (deploy scripts, CI itself, infrastructure automation), which raises the cost of an unquoted-variable bug reaching production well above the equivalent bug in typical application code.
- **DO:** Enable shellcheck integration directly in the editor (via an LSP plugin) so warnings surface while writing the script, rather than only being discovered at the next CI run — catching a quoting bug at write-time is strictly cheaper than catching it after a commit and a CI round-trip.
- **DO:** Pair shellcheck with `shfmt` (or an equivalent formatter) for consistent indentation and formatting, since shellcheck focuses on correctness issues and doesn't enforce a specific formatting style — the two tools are complementary, not overlapping.
- **DON'T:** Interpret a shellcheck "info"-level or "style"-level finding as automatically lower priority than a "warning"/"error"-level one without reading what it actually says — some info-level suggestions (like preferring `$(...)` over backticks, or using `printf` instead of `echo` for certain edge cases) prevent real, if less common, bugs, even though shellcheck's severity classification puts them at a lower default level.
- **DO:** Read shellcheck's own wiki page for a specific warning code (`https://www.shellcheck.net/wiki/SC2086`, for instance) when its exact meaning or the right fix isn't immediately obvious from the one-line message — each code has a dedicated explanation with examples that clarify edge cases the terse inline message can't cover.
- **DO:** Run shellcheck against generated or templated shell scripts (a script assembled by another tool, or one generated from a template with variable substitution) after generation, not just against hand-written scripts, since generated scripts are just as capable of containing quoting and portability bugs as hand-written ones.

## Common AI-Assistant Mistakes

- **DON'T:** Generate shell code with unquoted variable expansions as a default habit (`rm $file`, `cp $src $dst`, `if [ $x = $y ]`). This is the most common Bash anti-pattern in AI-generated scripts and breaks the moment a value contains a space, is empty, or contains a glob character; quote every expansion by default and only omit quotes with a specific, understood reason.
- **DON'T:** Omit `set -euo pipefail` from generated scripts, or add it but not actually verify the script behaves correctly under it (some patterns, like `grep` returning non-zero on no-match inside a normal control flow, need explicit handling once `-e` is active). Add the safety flags and then actually reason through what happens on each command's failure.
- **DON'T:** Use `eval` on any string built from external or user-supplied input. `eval` executes its argument as shell code, and building that argument from untrusted input is a direct path to arbitrary command execution — find a non-`eval` way to accomplish the goal (arrays, `"${!varname}"` indirect expansion, a `case` statement) in nearly every real scenario.
  ```bash
  # DANGEROUS — if $user_input is "; rm -rf /", it just runs that
  eval "process_item $user_input"

  # SAFE — no shell re-interpretation of the value
  process_item "$user_input"
  ```
- **DON'T:** Generate code that parses `ls` output for file iteration or attribute extraction — reach for native globbing or `find -print0`/`xargs -0` as the default pattern instead, as covered above.
- **DON'T:** Ignore filenames containing spaces when generating file-handling loops and commands. Test the mental model against a filename like `"my file.txt"` before considering the script correct — a huge fraction of generated shell scripts silently assume space-free filenames.
- **DON'T:** Generate Bash-specific syntax (`[[ ]]`, arrays, `local`) in a script whose shebang is `#!/bin/sh`, or generate a script targeting portability without checking what shell/platform it will actually run on. Ask or infer the target shell from context (existing scripts in the repo, a CI config, a Dockerfile's base image) rather than defaulting to whichever style is most familiar.
- **DON'T:** Write destructive commands (`rm -rf`, mass file moves/overwrites, `dd`) without adding a safety check first (confirming the target path is what's expected, checking it's non-empty before recursive delete, or requiring an explicit `--force`/confirmation flag) — generated scripts that blindly execute a destructive operation on a computed path are a well-known way to cause real data loss from a single bad input.
- **DON'T:** Assume every target system has GNU coreutils with GNU-specific flags available. A script generated and tested only against a Linux/GNU environment can fail outright on macOS/BSD tools with the same names but different flag sets — flag portability differences (`sed -i`, `date`, `readlink -f`) are a common, easy-to-miss gap in generated scripts meant for cross-platform use.
- **DON'T:** Leave variables unscoped inside generated shell functions (omitting `local`) as a default habit — this is a common source of a generated script's function accidentally clobbering a caller's variable of the same name, especially once a script grows past a handful of functions.
- **DON'T:** Generate a pipeline whose loop body sets variables the surrounding script depends on after the loop (`cmd | while read ...; do var=$x; done`), which silently loses those variable assignments to a subshell — use process substitution instead whenever a `while read` loop's body needs to affect the calling shell's state.
- **DON'T:** Hardcode `#!/bin/bash` when generating a script meant to be portable, or hardcode `#!/bin/sh` while then using Bash-only syntax inside it — pick one deliberately, match the shebang to the syntax actually used, and prefer `#!/usr/bin/env bash` when Bash features are genuinely needed.
- **DON'T:** Generate scripts that silently continue after a critical command fails just because `set -e` wasn't added, or that add `set -e` without accounting for its known gaps (conditions, pipeline non-last commands without `pipefail`) — verify the actual failure-handling behavior of every critical command path, not just the presence of the flag.
- **DON'T:** Generate an `ls -1 | wc -l`-style file-count pattern, or any other `ls`-piped-into-a-text-processing-tool pattern, as a default habit — reach for `find`, shell globbing, or an array's length instead.
- **DON'T:** Use `echo` with escape sequences or flags in generated code intended to be portable — use `printf`, whose behavior doesn't vary across shells the way `echo -e`/`echo -n` support does.
- **DON'T:** Generate a script that reads a command's output into a `while read` pipeline and then relies on variables set inside that loop being visible afterward — use process substitution instead, as this is one of the most common subshell-scoping mistakes in generated shell code.

## Quick Checklist
- Double-quote every variable expansion, command substitution, and array expansion (`"$var"`, `"$(cmd)"`, `"${arr[@]}"`) by default.
- Use arrays to build multi-argument command lines instead of concatenating into one string.
- Start every non-trivial script with `set -euo pipefail`; understand its gaps (conditions, non-last pipeline commands without `pipefail`).
- Use `trap ... EXIT` for guaranteed cleanup instead of manual cleanup calls before every exit point.
- Send error/diagnostic output to stderr (`>&2`), keep stdout clean for piping.
- Validate arguments and required environment variables at the top of the script with a clear usage message.
- Never parse `ls` output for iteration or attributes; use native globbing or `find -print0 | xargs -0`.
- Guard globs that might match nothing (`shopt -s nullglob`, or an explicit existence check inside the loop).
- Use shell file-test operators (`[[ -e ]]`, `[[ -d ]]`) instead of parsing `ls -l`/`stat` output.
- Decide POSIX `sh` vs. Bash explicitly via the shebang, and don't use Bash-only syntax in a `#!/bin/sh` script.
- Use `$(...)` command substitution over backticks; use `[ "$a" = "$b" ]` (not `==`) in POSIX scripts.
- Don't assume `/bin/bash` exists on every target system; verify before relying on Bash-only features.
- Watch for GNU-vs-BSD flag differences (`sed -i`, `date -d`) in cross-platform scripts.
- Run `shellcheck` on every script, locally and in CI; fix the underlying issue rather than suppressing by default.
- Suppress a specific shellcheck warning narrowly, with a comment explaining why — never a blanket file-wide disable.
- Never use `eval` on a string built from external or user-supplied input.
- Test file-handling logic against filenames containing spaces before considering it correct.
- Add a safety check before any destructive command (`rm -rf`, mass overwrite) — confirm the target path first.
- Pin the shellcheck version in CI so warnings don't drift purely from linter upgrades.
- Prefer `command -v tool` over `which tool` to check for a command's existence.
- Quote `"$@"` (not `"$*"` or bare `$@`) when forwarding positional arguments.
- Never assume a variable is non-empty and safe to leave unquoted — an empty unquoted variable silently drops an argument position.
- Declare every function-local variable with `local` to avoid leaking into or clobbering the caller's scope.
- Use process substitution (`< <(cmd)`) instead of piping into `while read` when the loop must set variables the caller needs afterward.
- Always use `read -r` to avoid backslash-escape surprises when reading lines.
- Use `getopts` (or a manual `while`/`case` parser) for scripts accepting more than one or two flags.
- Use `#!/usr/bin/env bash`, not a hardcoded `#!/bin/bash` path, for portability.
- Don't assume Bash 4+ features (associative arrays) are available on every target — verify the actual minimum supported version.
- Run shellcheck against the script's actual declared dialect (`-s sh` vs. `-s bash`).
- Treat a shellcheck failure in CI as a hard, merge-blocking gate for any script with real-world privileges.
