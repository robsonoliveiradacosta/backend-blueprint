# backend-blueprint

Quarkus + Java 21 blueprint. The standards that govern this repository live as skills under `.agents/skills/`,
symlinked into `.claude/skills/` and `.cursor/skills/`. Load the one that covers what you are about to do and
follow it — those files are the source of truth, and this one only points at them.

| Load | Before |
|---|---|
| `java-quarkus-standards` | writing, refactoring or reviewing any Java code here |
| `pragmatic-tdd` | implementing or fixing behaviour, and before calling an implementation done |
| `conventional-commits` | writing any commit message |

**Load `conventional-commits` before every `git commit`.** Nothing checks the message afterwards — the
standard holds only because it was read, so read it.

## Skills shipped for other stacks

`java-spring-standards` and `pragmatic-tdd-spring` are the Spring Boot 3 counterparts of the two skills above. They
do not govern this repository — nothing here is Spring — and exist to be installed into Spring projects:

```shell
sh .agents/install-skills <target-repo> java-spring-standards pragmatic-tdd-spring conventional-commits
```

Name the skills explicitly like that. With no skill named the installer copies **all** of them, which drops the
Quarkus standards into a Spring project and the other way round.

## Commands

```shell
./mvnw quarkus:dev              # dev mode with live coding
./mvnw test                     # test suite
./mvnw -q -DskipTests package   # compile check
```
