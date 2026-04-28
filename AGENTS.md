# yaswanth's learning notes — AI agent guide

If you are an AI coding assistant (Claude Code, Codex CLI, Cursor, Continue, etc.) working in this repo, read [`.github/copilot-instructions.md`](.github/copilot-instructions.md) for full conventions.

**TL;DR:**
- This is a personal knowledge base. Topics live under `learnings/<topic>/`, daily logs under `logs/YYYY/MM/DD.md`.
- The user disambiguates: **"memory repo"** = this repo, **"memory tool"** = agent's built-in memory scope. If they just say "memory" without context, ask.
- A pre-commit hook runs `gitleaks` to block accidental secret commits. If your learning note legitimately contains an example secret pattern, allowlist it in `.gitleaks.toml` in the same commit.
- Never edit `.audit/`.
- Group new entries by topic; cross-link related ones.
