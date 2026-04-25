# Warga.jp - Agent Guide

## ⚠ Hard Rules (apply every session, no exceptions)

- **Never run `git commit`, `git add`, `git push`, or any git write command** unless the user explicitly says to commit (e.g. "commit this", "commit changes"). Always stop after making file changes and wait.

## Files

- **docs/CONTEXT.md** - # Warga.jp - Project Context
- **docs/rules/** - Conventions split by topic (auth, forms, admin, features, content, infrastructure). Start at `docs/rules/README.md` for the index and "if you're working on X, read Y" lookup.
- **docs/SKILLS.md** - Step-by-step procedures: add feature, run migrations, user/admin workflows
- **docs/EXAMPLES.md** - Code examples for model, repository, handler, viewmodel, template, migration
- **docs/TOOLS.md** - CLI commands for running the app and code generation
- **docs/SECURITY_TODO.md** - Outstanding security fixes. Read before auth-adjacent work or before re-enabling the user signup/login handler (currently disabled in router/router.go).

## Model preference

Run `/model opusplan` at the start of each session. This built-in Claude Code preset automatically uses **Opus in plan mode** and **Sonnet otherwise** — matching the preferred workflow for this project (Opus for exploration/design, Sonnet for writing code).
