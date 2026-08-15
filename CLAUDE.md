# backend-blueprint

Quarkus + Java 21 blueprint. The standards that govern this repository live as skills under `.agents/skills/`,
symlinked into `.claude/skills/` and `.cursor/skills/`. Load the one that covers what you are about to do and
follow it — those files are the source of truth, and this one only points at them.

| Load | Before |
|---|---|
| `java-quarkus-standards` | writing, refactoring or reviewing any Java code here |
| `pragmatic-tdd` | implementing or fixing behaviour, and before calling an implementation done |
| `conventional-commits` | writing any commit message |

**Load `conventional-commits` before every `git commit`.** The message format is not improvised, and
`.githooks/commit-msg` rejects a message that does not follow it — an invalid message costs a round trip.

## After cloning

Git hooks are not versioned, so any Maven build copies `.githooks/commit-msg` into `.git/hooks/`. Run
`./mvnw validate` if you need to commit before building.

## Commands

```shell
./mvnw quarkus:dev              # dev mode with live coding
./mvnw test                     # test suite
./mvnw -q -DskipTests package   # compile check
```
