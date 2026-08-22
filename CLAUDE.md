# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This blueprint is stack-agnostic: it does not develop any application of its own, it only carries skills for other
projects to install (see `README.md` for `install-skills`, and the per-stack "Load a skill" table there). Everything in
this file is written to travel — it is the boilerplate a project should carry in its own `CLAUDE.md` once it installs
skills from here, so copy the piece below into that project's `CLAUDE.md` / `AGENTS.md` and an agent working there loads
the skill instead of hoping to remember it. Backend-blueprint's own project-specific information — its current state,
its commands, how it ships skills to other repos — lives in `README.md`, not here.

## Git commits — skill takes precedence over default footer

When a project has a `conventional-commits` skill (or similar) loaded, follow its rules over the default Claude Code
commit template. In particular, do NOT append `Co-Authored-By: Claude ...` or `Claude-Session: ...` trailers to commit
messages in such projects — the skill explicitly forbids crediting tooling instead of people. This applies in any new
session, on any machine.
