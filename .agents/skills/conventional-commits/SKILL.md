---
name: conventional-commits
description: "Conventional Commits standard for every commit message. Use whenever you are about to run `git commit`, amend or reword a commit, revert one, write a squash/merge commit or a pull-request title, split staged work into commits, or review whether existing history follows the convention. Defines the `<type>(<scope>): <description>` format, the ten allowed types (feat, fix, docs, style, refactor, perf, test, chore, ci, revert), imperative-mood descriptions, `!` and `BREAKING CHANGE:` for breaking changes, and the rule of one logical change per commit. Language- and framework-agnostic."
---

# Git Commits — Conventional Commits

Every commit message MUST follow the [Conventional Commits](https://www.conventionalcommits.org/) specification.
This is a hard rule, not a preference: tooling reads these messages to derive versions and changelogs, and a
malformed one breaks that silently.

```
<type>(<optional scope>): <short description>

<optional body>

<optional footers>
```

## Before writing the message

- **Describe what is staged, not what you set out to do.** Read `git diff --staged --stat` first. If the staged set
  covers two logical changes, it is two commits, however the task was phrased.
- **Reuse the scope vocabulary already in the history** (`git log --oneline -20`). `auth` and `authentication` as two
  scopes is two vocabularies, and neither filters cleanly.
- **Write in English**, matching the rest of the history, whatever language the conversation is in.

## The subject line

- **Type** — exactly one of the following. Nothing outside this list:

  | Type | Use for |
  |---|---|
  | `feat` | a new feature for the user or the API |
  | `fix` | a bug fix |
  | `docs` | documentation only |
  | `style` | formatting, whitespace, semicolons — nothing that changes meaning |
  | `refactor` | a change that neither fixes a bug nor adds a feature |
  | `perf` | a change made for performance |
  | `test` | adding missing tests or correcting existing ones |
  | `chore` | build process, configuration, auxiliary tooling and libraries |
  | `ci` | CI configuration files and scripts |
  | `revert` | undoing an earlier commit, whose SHA goes in a footer |

  The specification itself permits other types; this standard does not. Types borrowed from other conventions map
  into the list: `build` → `chore`. `wip` is not a commit that reaches a shared branch.

- **When two types are plausible, the type describes the effect of the change, not the files it touched.** A bug
  fixed by editing only a test helper is still `fix`. A dependency bump is `chore` unless it delivers a user-visible
  fix (`fix`) or capability (`feat`). A refactor that happens to run faster is `perf` only when performance was the
  goal.
- **Scope** — optional, lowercase, in parentheses: the module, layer or area the change touches (`feat(auth):`,
  `fix(parser):`). Omit it rather than invent a vague one; `(core)` and `(misc)` say nothing.
- **Description** — imperative mood, present tense, as if completing "this commit will…": *add endpoint*, never
  *added endpoint* or *adds endpoint*. Lowercase, no trailing period, roughly 72 characters or less.
- Never restate the type in the description — `fix: fix the null check` wastes the only line most readers see.

## Body and footers

- **Body** — optional, separated from the subject by a blank line. It explains **why** the change was made, and what
  the reader would otherwise have to reconstruct: the constraint, the alternative rejected, the bug's actual cause.
  It does not narrate what changed; the diff already says that. Wrap at ~72 columns.
- **Breaking changes** — mark with a `!` immediately before the colon (`feat(api)!: change payload structure`), and
  add a `BREAKING CHANGE:` footer spelling out what consumers must now do. The `!` alone is enough only when the
  description already says what breaks; the two are normally used together, and the footer is what a reader
  migrating actually needs.
- **`BREAKING CHANGE` is uppercase**, on its own line, followed by `: `. That exact token is what tooling detects —
  `Breaking change:` or a sentence in the body is invisible to it. `BREAKING-CHANGE:` is the only accepted synonym.
- **Footers** — issue references and trailers, one per line: `Refs: #142`, `Closes #142`, `Co-Authored-By: …`. A
  footer token replaces whitespace with `-` (`Reviewed-by`, never `Reviewed by`) and is separated from its value by
  `: ` or ` #`. `BREAKING CHANGE` is the one token allowed to contain a space.

## Three rules that keep the history usable

- **One logical change per commit.** A refactor bundled into a feature commit makes the type a lie and the commit
  impossible to revert cleanly. Stage and commit them separately — if the description needs an "and", it is two
  commits.
- **Code and its tests land together.** The tests for a change belong to the `feat`/`fix` commit that needed them,
  so that every commit is independently green. `test:` is for tests added to code that already shipped.
- **`!` documents a break; it does not authorize one.** If the thing being changed carries a public version — a
  versioned URI, a published package, a stable CLI flag — a shape change belongs in a new version, not in a breaking
  commit against the current one. Reserve `!` for what consumers cannot route around: a renamed configuration key, a
  newly required environment variable, a migration with an ordering constraint.

Squash-merge titles and pull-request titles that become commits follow the same rules — a squashed branch lands as
one message, so it must name one logical change.

## Reverts, merges, and fixing a bad message

- **A revert is a commit like any other.** `git revert` proposes `Revert "<old subject>"`, which fails the standard —
  rewrite it as `revert:` carrying the subject of what was undone, with the reverted SHAs in a footer:

  ```text
  revert: feat(user): add the listing endpoint

  Refs: 8ba09f9
  ```
- **Merge commits generated by git are exempt.** Leave `Merge branch '…'` alone; tooling skips them.
- **Fix a wrong message while it is still yours** — `git commit --amend` for the last commit, `git rebase -i` for one
  further back, both only while the branch is unpushed. Never rewrite a message already on a shared branch; leave it
  and get the next one right.

## Shapes to copy

A subject line on its own is enough when the change explains itself:

```text
fix(order): reject sort fields outside the allow-list
```

A body earns its place when the reason is not visible in the diff:

```text
feat(user): return a paginated response from the listing endpoint

Clients were reading the whole table to render the first page, because a bare
list gives them no total to page against.

Refs: #142
```

A breaking change states what the consumer must now do:

```text
chore(config)!: read the datasource URL from DATABASE_URL

BREAKING CHANGE: deployments that set the datasource URL in the packaged
configuration must export DATABASE_URL instead; the property is no longer
read at startup.
```

Multi-line messages are written with a heredoc, never by concatenating `-m` flags:

```bash
git commit -m "$(cat <<'EOF'
feat(user): return a paginated response from the listing endpoint

Clients were reading the whole table to render the first page.

Refs: #142
EOF
)"
```

## Messages that fail the standard

| Message | Why it fails |
|---|---|
| `Added user endpoint` | no type, past tense |
| `feat: stuff` | the description says nothing |
| `update: bump the framework version` | `update` is not an allowed type — this is `chore` |
| `fix(user): Fix the bug.` | capitalized, trailing period, and "fix" already lives in the type |
| `feat(user): add endpoint and refactor the mapper` | two logical changes in one commit — split them |
| `feat(user): add endpoint` *(tests in a follow-up commit)* | the feature commit carries its own tests |
| `feat(api): drop the legacy field` *(field is public)* | a breaking change unmarked by `!` or a footer |
| `build(deps): bump the framework` | `build` is not an allowed type — this is `chore` |
| `feat(api)!: rename the field` + `Breaking change: …` | the footer token must be uppercase `BREAKING CHANGE:` |
| `Revert "feat(user): add endpoint"` | git's default revert message — rewrite it as `revert:` with a `Refs:` footer |

## Before committing

- [ ] The message describes what is staged, not what you intended to do
- [ ] Type is one of the ten, and honest about the effect of the diff
- [ ] Scope reuses a word already in the history, or is omitted
- [ ] Description is imperative, lowercase, no trailing period, ~72 characters or less
- [ ] The commit is one logical change, with its tests
- [ ] Body explains why, when why is not obvious from the diff
- [ ] `!` plus an uppercase `BREAKING CHANGE:` footer when consumers must react
- [ ] Footer tokens are dash-joined (`Refs:`, `Reviewed-by:`), issue reference present when one exists
