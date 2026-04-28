# Git pre-commit hooks (and wiring gitleaks for secret scanning)

**Date:** 2026-04-28
**Context:** Wanted a guard that scans `~/memory/` for secrets (API keys, tokens) before any commit lands. Decided to use a **git pre-commit hook** instead of an AI agent skill or agent hook, because the rule "no secrets in commits" must apply to *every* actor (me, AI, scripts) — not just one agent.

---

## 1. What is a git hook?

A git hook is a script git automatically executes at specific points in its lifecycle. They live in `.git/hooks/` of any repo. Git ships sample files (`*.sample`) — rename without the suffix to activate.

Key hooks:
| Hook | Fires when | Can it block? |
|---|---|---|
| `pre-commit` | Before commit message editor opens | ✅ non-zero exit aborts |
| `commit-msg` | After commit msg written, before commit finalizes | ✅ |
| `pre-push` | Before refs are pushed to remote | ✅ |
| `post-commit` | After commit is created | ❌ informational only |

**Hooks are NOT versioned.** `.git/hooks/` is local to your clone. To share hooks across machines, either commit a `hooks/` dir + symlink it, or use the [pre-commit framework](https://pre-commit.com/).

**Bypass:** `git commit --no-verify` skips all hooks. Fine for personal repos. For shared repos, add a server-side check (GitHub Actions).

---

## 2. Why a git hook (not agent skill/hook)?

| Option | Runs when | Catches commits made by |
|---|---|---|
| **git pre-commit hook** | Every `git commit` in repo | Anyone — me, AI, scripts ✅ |
| Copilot/Claude SKILL.md | Only when AI loads it during chat | Only AI, only if remembered ❌ |
| Claude Code agent hook | Only when *that agent* runs | One specific tool ❌ |

The rule is a property of the **repo**, not any one actor → enforce at the only choke point.

---

## 3. The full journey (commands + what each does)

### Step 0: Reconnaissance
```bash
command -v gitleaks && gitleaks version || echo "NOT_INSTALLED"
```
- **What:** `command -v X` returns 0 if X is on PATH; `||` runs the next clause if it fails.
- **Why:** Don't reinstall if already present.
- **Result:** Not installed.

```bash
ls -la ~/memory/.git/hooks/
```
- **What:** List existing hooks.
- **Why:** Confirm we're starting fresh (only `*.sample` files).

### Step 1: Pick the right version
```bash
curl -fsSL https://api.github.com/repos/gitleaks/gitleaks/releases/latest | grep -E '"tag_name"|"published_at"'
```
- **What:** GitHub API returns the latest release JSON; grep pulls just version + date.
- **Flags:**
  - `-f`: fail (non-zero exit) on HTTP errors instead of printing the error body
  - `-s`: silent (no progress meter)
  - `-S`: still show errors when silent
  - `-L`: follow redirects
- **Why:** **Always verify version against the registry — don't guess from training data.** I initially proposed `v8.21.2` from memory; latest was actually `v8.30.1`.

### Step 2: Download + install gitleaks
```bash
cd /tmp && \
  curl -fsSL -o gitleaks.tar.gz \
    "https://github.com/gitleaks/gitleaks/releases/download/v8.30.1/gitleaks_8.30.1_linux_x64.tar.gz" && \
  tar -xzf gitleaks.tar.gz gitleaks && \
  mv gitleaks ~/.local/bin/ && \
  rm gitleaks.tar.gz && \
  gitleaks version
```
- **`cd /tmp`**: Use a scratch dir so download artifacts don't pollute the working dir.
- **`curl -fsSL -o file URL`**: Download URL to `file`. (`-o` = output filename.)
- **`tar -xzf archive [member]`**: Extract (`x`) gzipped (`z`) file (`f`), pulling only the named member (`gitleaks` binary).
- **`mv … ~/.local/bin/`**: Place on PATH (already in `$PATH` per earlier check).
- **`gitleaks version`**: Sanity check — printed `8.30.1`. ✅

### Step 3: Write the gitleaks config
Created `~/memory/.gitleaks.toml`:
```toml
[extend]
useDefault = true              # inherit gitleaks' built-in rules (AWS, GitHub, Slack, ...)

[allowlist]
paths = ['''\.gitleaks\.toml$''', '''\.audit/''']
regexes = [
  '''AKIAIOSFODNN7EXAMPLE''',                            # AWS docs example
  '''wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY''',        # AWS docs example
]
```
- **`useDefault = true`**: Pull in 100+ built-in patterns instead of writing from scratch.
- **Allowlist `paths`**: Skip the config file itself + the audit log dir (otherwise the example regexes inside the config would trip the scanner).
- **Allowlist `regexes`**: Skip well-known docs examples used in learning notes.

### Step 4: Write the hook
Created `~/memory/.git/hooks/pre-commit` (key bits):
```bash
#!/usr/bin/env bash
set -euo pipefail

REPO_ROOT="$(git rev-parse --show-toplevel)"
AUDIT_DIR="$REPO_ROOT/.audit"
LOG_FILE="$AUDIT_DIR/blocked.log"
CONFIG="$REPO_ROOT/.gitleaks.toml"
mkdir -p "$AUDIT_DIR"

# Locate gitleaks (PATH or ~/.local/bin)
if command -v gitleaks >/dev/null 2>&1; then
  GITLEAKS="gitleaks"
elif [ -x "$HOME/.local/bin/gitleaks" ]; then
  GITLEAKS="$HOME/.local/bin/gitleaks"
else
  echo "❌ gitleaks not found." >&2; exit 1
fi

set +e
OUTPUT="$("$GITLEAKS" git --staged --redact -v --no-banner --exit-code 1 \
  --config "$CONFIG" \
  --report-path "$AUDIT_DIR/last-scan.json" --report-format json 2>&1)"
EXIT=$?
set -e

if [ $EXIT -ne 0 ]; then
  TS="$(date -Iseconds)"
  {
    echo "─── $TS ───────────"
    echo "Blocked commit attempt in $REPO_ROOT"
    git diff --cached --name-only | sed 's/^/  - /'
    echo "$OUTPUT" | sed 's/^/  /'
  } >> "$LOG_FILE"
  echo "❌ Potential secret detected. Commit blocked." >&2
  echo "$OUTPUT" >&2
  exit 1
fi
```

**Made executable:**
```bash
chmod +x ~/memory/.git/hooks/pre-commit
```
- **What:** Set the executable bit (otherwise git silently skips the hook).
- **`+x`**: Add execute permission for owner/group/other.

### Step 5: Test — first attempt, broken
```bash
echo 'AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYzFAKEKEY01"' > inbox/fake-secret-test.md
git add inbox/fake-secret-test.md
git commit -m "test: should be blocked"
```
- **Expected:** Hook blocks commit.
- **Actual:** Commit succeeded! 😱

### Step 6: Diagnose with bash trace
```bash
bash -x .git/hooks/pre-commit
```
- **`bash -x`**: Print each command before running it (`+ command...`). Best tool for "did the script even reach this line?" questions.
- **Found:** `gitleaks` ran but reported "0 commits scanned, 0 leaks". So the scanner did execute — it just scanned nothing.

### Step 7: Root cause #1 — `gitleaks protect` is deprecated
```bash
gitleaks --help 2>&1 | head -40
```
- **What:** Show available subcommands. **`2>&1`**: redirect stderr to stdout (some CLIs print help to stderr).
- **Found:** v8.30.1 has only `dir`, `git`, `stdin`, `completion`, `help`, `version`. **`protect` was removed.** Modern equivalent: `gitleaks git --staged`.

### Step 8: Root cause #2 — exit code is broken
```bash
gitleaks git --staged --redact --no-banner --config .gitleaks.toml
# Output: "leaks found: 1"
# Exit: 0   ← BUG
```
Even when leaks were found, exit was 0 → hook thought "all clear".

**Fix:** Pass `--exit-code 1` explicitly:
```bash
gitleaks git --staged --exit-code 1 --config .gitleaks.toml
# Now: exit 1 when leaks found ✅
```

### Step 9: Patched hook → end-to-end verified

```bash
# Test with a real-format GitHub PAT
printf 'token: ghp_AbCdEf1234567890AbCdEf1234567890AbCd\n' > inbox/fake.md
git add inbox/fake.md
git commit -m "should be blocked"
# Output:
# ❌ Potential secret detected. Commit blocked.
# RuleID: github-pat
# File: inbox/fake.md, Line: 1
```

**Then a clean commit:**
```bash
git rm --cached inbox/fake.md && rm inbox/fake.md
git add README.md learnings/ logs/ .gitignore .gitleaks.toml
git commit -m "Initial memory"
# ✅ committed (no secrets)
```

---

## 4. Lessons (for the index)

- ✅ Use **git hooks** for repo-wide invariants; agent skills/hooks are too narrow.
- ✅ **Verify third-party versions** against the source-of-truth registry, not memory.
- ✅ **Always make the hook executable** (`chmod +x`) — git silently skips non-executable hooks, no warning.
- ⚠️ **gitleaks v8 removed `protect`** — use `gitleaks git --staged`.
- ⚠️ **gitleaks v8.30.1 doesn't honor default `--exit-code`** — pass it explicitly.
- ⚠️ **`bash -x`** is the fastest way to debug "did my hook run?" issues.
- ⚠️ **`--no-verify` exists** — for shared repos, add a server-side scan as a second layer.

See also:
- [`gitleaks-v8-gotchas.md`](./gitleaks-v8-gotchas.md)
- [`../bash/hook-script-patterns.md`](../bash/hook-script-patterns.md)
