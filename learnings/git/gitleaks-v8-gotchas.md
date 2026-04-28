# gitleaks v8.x gotchas (verified on v8.30.1, 2026-04-28)

Quick reference for surprises that cost time. See [`pre-commit-hooks.md`](./pre-commit-hooks.md) for the full story.

## 1. `protect` subcommand removed
Older guides (and older gitleaks) used:
```bash
gitleaks protect --staged
```
**v8.x has no `protect`.** Use:
```bash
gitleaks git --staged
```
Available v8.30.1 subcommands: `dir`, `git`, `stdin`, `completion`, `help`, `version`.

## 2. `--exit-code` default is broken
Documented default is `1`. **Actual behavior:** when leaks are found, exit is `0`.

**Fix:** Always pass it explicitly:
```bash
gitleaks git --staged --exit-code 1 ...
```

## 3. Default rules allowlist `EXAMPLE` strings
Test data containing `AKIAIOSFODNN7EXAMPLE` (AWS docs canonical key) won't trigger detection — gitleaks deliberately skips obvious example values.

**For testing, use a non-example pattern:**
```bash
# ✅ Triggers github-pat rule
ghp_AbCdEf1234567890AbCdEf1234567890AbCd

# ❌ Allowlisted (contains EXAMPLE)
AKIAIOSFODNN7EXAMPLE
```

GitHub PATs (`ghp_` + 36 chars) are the most reliable for hook smoke-tests because the rule is strict + format is unambiguous.

## 4. First-commit edge case
On a brand-new repo (no `HEAD` yet), `gitleaks git --staged` reports "0 commits scanned" but **still scans the staged content correctly**. So this is fine in practice — confused us at first because the log message implies it scanned nothing.

## 5. Useful flag combo for hooks
```bash
gitleaks git --staged \
  --redact \              # hide secret values in terminal output
  -v \                    # verbose: show file/line/rule (essential for hooks)
  --no-banner \           # suppress ASCII banner
  --exit-code 1 \         # see #2
  --config .gitleaks.toml \
  --report-path .audit/last-scan.json --report-format json
```

## 6. Allowlisting your own config file
The `.gitleaks.toml` itself often contains regex patterns that look like secrets to gitleaks. Allowlist it:
```toml
[allowlist]
paths = ['''\.gitleaks\.toml$''']
```

## 7. Where to add custom rules
```toml
[[rules]]
id = "internal-marker"
description = "Lines marked INTERNAL: should never be committed"
regex = '''(?i)^\s*INTERNAL:'''
tags = ["custom"]
```
Triple-quoted strings = no escaping needed in regex.
