# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# backend-blueprint

Quarkus + Java 21 blueprint. The standards that govern this repository live as skills under `.agents/skills/`,
symlinked into `.claude/skills/` and `.cursor/skills/`. Load the one that covers what you are about to do and
follow it — those files are the source of truth, and this one only points at them.

| Load | Before |
|---|---|
| `java-quarkus-standards` | writing, refactoring or reviewing any Java code here |
| `pragmatic-tdd-quarkus` | implementing or fixing behaviour, and before calling an implementation done |
| `conventional-commits` | writing any commit message |

**Load `conventional-commits` before every `git commit`.** Nothing checks the message afterwards — the
standard holds only because it was read, so read it.

## Nothing has been built here yet

`src/main/java/` is empty, there is no `src/test/`, and `application.properties` is blank — `./mvnw test` passes
with no tests to run. The first feature therefore lands on bare ground: it chooses the base package (ask, do not
assume) and adds the extensions it needs, because the classpath holds only `quarkus-rest-jackson`, `quarkus-arc`
and `quarkus-junit`. Persistence, validation, OpenAPI, a JDBC driver and `rest-assured` are all absent, so even
the first integration test needs a dependency added. Let the BOM version them.

## Skills shipped for other stacks

`java-spring-standards` and `pragmatic-tdd-spring` are the Spring Boot 3 counterparts of the two skills above. They
do not govern this repository — nothing here is Spring — and exist to be installed into Spring projects:

```shell
sh .agents/install-skills <target-repo> java-spring-standards pragmatic-tdd-spring conventional-commits
```

Name the skills explicitly like that. With no skill named the installer copies **all** of them, which drops the
Quarkus standards into a Spring project and the other way round.

Edit skills in `.agents/skills/` — the two agent directories are symlinks to it, and the installer copies from it.

## Commands

```shell
./mvnw quarkus:dev                        # dev mode with live coding
./mvnw test                               # test suite
./mvnw test -Dtest=OrderResourceTest      # one test class; append '#methodName' for one test
./mvnw -q -DskipTests package             # compile check
./mvnw package -Dnative                   # native build, runs the failsafe ITs
```
