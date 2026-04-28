# memory

Personal knowledge base — concepts I've learned, problems I've debugged, snippets worth keeping.

## Layout

```
memory/
├── learnings/   # Evergreen, topic-organized notes (one concept per file)
│   ├── css/
│   ├── react/
│   └── ...
├── logs/        # Day-wise capture (cheap, append-only). YYYY/MM/DD.md
├── projects/    # Per-project deep dives and post-mortems
├── snippets/    # Reusable code, commands, config
└── inbox/       # Quick capture; triage into other folders later
```

## Conventions

- **Date format:** ISO 8601 — `YYYY-MM-DD` for filenames, `logs/YYYY/MM/DD.md` for daily logs.
- **One concept per file** in `learnings/`. File name = the concept (`cascade-layers.md`, not `css-stuff.md`).
- **Every file starts with a one-line summary**, then context, then details.
- **Link generously** between notes using markdown relative links.
- **Public-safe only.** Anything client/employer-specific lives in `~/private_memory/`. When in doubt, write there first and promote later.

## Workflow

1. **Capture** during the day → `logs/YYYY/MM/DD.md` or `inbox/`.
2. **Promote** durable lessons weekly → `learnings/<topic>/`.
3. **Prune** `inbox/` — if it's not worth filing in a week, delete it.

## Safety

A pre-commit hook (gitleaks / git-secrets) scans for credentials before push. Don't disable it.
