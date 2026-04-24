# Skills

Reusable agent skills — each one self-contained, concept-first, cross-harness where it matters.

## Layout

Each skill lives in its own folder:

```
skills/
└── <skill-name>/
    ├── README.md          # what it does, when to use it, how to install
    ├── claude/            # Claude Code implementation (SKILL.md + assets)
    └── codex/             # Codex adaptation (AGENTS.md + assets)
```

Not every skill needs both harnesses — publish what you've actually tested.

## Conventions

- **Folder name** is the canonical skill name (kebab-case).
- **Top-level `README.md`** explains the concept once, then links to each harness implementation.
- **Harness folders** hold the actual prompt/config files in whatever format that harness expects.
- **No secrets, no internal references.** Every skill must be runnable by a stranger with public tools only.
