# Bash patterns for hook scripts

**Date:** 2026-04-28
**Source:** Distilled from writing `~/memory/.git/hooks/pre-commit` (see [`../git/pre-commit-hooks.md`](../git/pre-commit-hooks.md)).

---

## 1. Strict mode: `set -euo pipefail`

```bash
#!/usr/bin/env bash
set -euo pipefail
```

| Flag | Effect |
|---|---|
| `-e` | Exit immediately on any non-zero command (no silent failures) |
| `-u` | Treat unset variables as errors (catch typos like `$LOG_FIEL`) |
| `-o pipefail` | A pipeline fails if **any** stage fails (without this, only the last stage matters: `false \| true` returns 0) |

**Why for hooks:** A silently-failing security hook is worse than no hook. Strict mode makes failures loud.

---

## 2. Capture exit code from a fallible command without aborting

`set -e` aborts on any failure. But sometimes you *want* to inspect the failure (like checking if gitleaks found a leak). Pattern:

```bash
set +e                                   # disable -e temporarily
OUTPUT="$(some_command 2>&1)"            # run, capture stdout+stderr
EXIT=$?                                  # save exit code BEFORE next command
set -e                                   # re-enable -e

if [ $EXIT -ne 0 ]; then
  echo "Failed: $OUTPUT" >&2
  exit 1
fi
```

**Key rules:**
- `$?` reflects the **last command's** exit code. Capture it immediately, before any other command runs.
- `2>&1` = "send stderr (fd 2) to where stdout (fd 1) is going". Use it when you want to capture both streams in one variable.

**Alternative idiom** (no need to toggle `set -e`):
```bash
if OUTPUT="$(some_command 2>&1)"; then
  echo "ok"
else
  EXIT=$?
  echo "failed: $OUTPUT"
fi
```
The `if` syntax sets `$?` from the test, so `-e` doesn't fire.

---

## 3. Locate a binary with fallbacks

```bash
if command -v gitleaks >/dev/null 2>&1; then
  GITLEAKS="gitleaks"
elif [ -x "$HOME/.local/bin/gitleaks" ]; then
  GITLEAKS="$HOME/.local/bin/gitleaks"
else
  echo "not found" >&2
  exit 1
fi
```

| Construct | Meaning |
|---|---|
| `command -v X` | Returns 0 if X is on PATH; prints its location. **Preferred over `which`** (which is not POSIX). |
| `>/dev/null 2>&1` | Discard both stdout and stderr — we only want the exit code. |
| `[ -x "$path" ]` | True if path exists and is executable. **Always quote `"$path"`** to handle spaces. |
| `[ -f "$path" ]` | True if path exists and is a regular file. |
| `[ -d "$path" ]` | True if path exists and is a directory. |

---

## 4. Append a structured block to a log file

```bash
{
  echo "─── $(date -Iseconds) ───"
  echo "context line 1"
  some_command | sed 's/^/  /'        # indent each line by 2 spaces
  echo
} >> "$LOG_FILE"
```

- **`{ ...; }`** = command group; the redirect `>> "$LOG_FILE"` applies to the entire block. Saves repeating `>> file` on every line.
- **`>>`** = append (vs `>` truncate).
- **`sed 's/^/  /'`** = prefix every line with 2 spaces (handy for indenting nested output).
- **`date -Iseconds`** = ISO-8601 timestamp like `2026-04-28T20:24:02+05:30`. Sortable, unambiguous.

---

## 5. Echo to stderr (so users see errors regardless of redirects)

```bash
echo "❌ Something went wrong" >&2
```
- **`>&2`** redirects to fd 2 (stderr).
- Why: if the user runs `your_script > out.log`, errors still print to terminal instead of getting buried in `out.log`.

---

## 6. `bash -x` for debugging

```bash
bash -x my-script.sh
```
Prints each command (with variable expansions resolved) before running it. Lines start with `+`. The single fastest way to answer "did this line even run?".

For just one section, wrap in `set -x` / `set +x`:
```bash
set -x
risky_block
set +x
```

---

## 7. Defensive substitution for unset vars

When `-u` is on, plain `$VAR` errors if VAR is unset. To provide a fallback:

| Syntax | Meaning |
|---|---|
| `${VAR:-default}` | Use `default` if VAR is unset OR empty |
| `${VAR-default}` | Use `default` only if VAR is unset (empty stays empty) |
| `${VAR:?error msg}` | Error and exit if VAR is unset/empty (great for required args) |

Example:
```bash
USER_INPUT="${1:?usage: $0 <filename>}"
```

---

## 8. Quote everything

Always quote `"$variable"` and `"$(command)"` substitutions unless you specifically need word-splitting. Unquoted variables break on spaces, glob characters, and tabs — common cause of "works on my machine" hook bugs.
