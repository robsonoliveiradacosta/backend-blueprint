# AGENTS.md

Guidance for coding agents working in this repository.

This blueprint is stack-agnostic: it does not develop any application of its own, it only carries skills for other
projects to install (see `README.md` for `install-skills`, and the per-stack "Load a skill" table there). Everything
in this file is written to travel — it is the boilerplate a project should carry in its own `AGENTS.md` once it
installs skills from here, so copy the piece below into that project's `AGENTS.md` / `CLAUDE.md` and an agent working
there loads the skill instead of hoping to remember it. Backend-blueprint's own project-specific information — its
current state, its commands, how it ships skills to other repos — lives in `README.md`, not here.

## Git commits — skill takes precedence over default footer

**Load `conventional-commits` before every `git commit`.** Nothing checks the message afterwards — the standard holds
only because it was read, so read it.

When a project has a `conventional-commits` skill (or similar) loaded, follow its rules over any tool's default commit
template. In particular, do NOT append `Co-Authored-By: ...` or session-identifying trailers to commit messages in such
projects — the skill explicitly forbids crediting tooling instead of people. This applies in any new session, on any
machine.
