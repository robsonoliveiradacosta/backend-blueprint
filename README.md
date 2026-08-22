# backend-blueprint

This project uses Quarkus, the Supersonic Subatomic Java Framework.

If you want to learn more about Quarkus, please visit its website: <https://quarkus.io/>.

## Current state

Nothing has been built here yet: `src/main/java/` is empty, there is no `src/test/`, and `application.properties`
is blank — `./mvnw test` passes with no tests to run. The first feature therefore lands on bare ground: it chooses the
base package (ask, do not assume) and adds the extensions it needs, because the classpath holds only
`quarkus-rest-jackson`, `quarkus-arc` and `quarkus-junit`. Persistence, validation, OpenAPI, a JDBC driver and
`rest-assured` are all absent, so even the first integration test needs a dependency added. Let the BOM version them.

## Commands

```shell
./mvnw quarkus:dev                        # dev mode with live coding
./mvnw test                               # test suite
./mvnw test -Dtest=OrderResourceTest      # one test class; append '#methodName' for one test
./mvnw -q -DskipTests package             # compile check
./mvnw package -Dnative                   # native build, runs the failsafe ITs
```

## Running the application in dev mode

You can run your application in dev mode that enables live coding using:

```shell script
./mvnw quarkus:dev
```

> **_NOTE:_**  Quarkus now ships with a Dev UI, which is available in dev mode only at <http://localhost:8080/q/dev/>.

## Packaging and running the application

The application can be packaged using:

```shell script
./mvnw package
```

It produces the `quarkus-run.jar` file in the `target/quarkus-app/` directory. Be aware that it’s not an _über-jar_ as
the dependencies are copied into the `target/quarkus-app/lib/` directory.

The application is now runnable using `java -jar target/quarkus-app/quarkus-run.jar`.

If you want to build an _über-jar_, execute the following command:

```shell script
./mvnw package -Dquarkus.package.jar.type=uber-jar
```

The application, packaged as an _über-jar_, is now runnable using `java -jar target/*-runner.jar`.

## Creating a native executable

You can create a native executable using:

```shell script
./mvnw package -Dnative
```

Or, if you don't have GraalVM installed, you can run the native executable build in a container using:

```shell script
./mvnw package -Dnative -Dquarkus.native.container-build=true
```

You can then execute your native executable with: `./target/backend-blueprint-1.0-SNAPSHOT-runner`

If you want to learn more about building native executables, please consult <https://quarkus.io/guides/maven-tooling>.

## Reusing the standards in another project

The skills under `.agents/skills/` are the standards this blueprint carries. To take them elsewhere:

```shell script
sh .agents/install-skills --list                       # what is available
sh .agents/install-skills ../other-project             # all of them
sh .agents/install-skills ../other-project conventional-commits
```

It copies the skill and creates the `.claude/skills/` and `.cursor/skills/` symlinks. Copies, not links to this
checkout, so the other repository stays self-contained — run it again to pick up a later fix.

### Skills shipped for other stacks

`java-spring-standards` and `pragmatic-tdd-spring` are the Spring Boot 3 counterparts of `java-quarkus-standards`
and `pragmatic-tdd-quarkus`. They do not govern this repository — nothing here is Spring — and exist to be installed
into Spring projects:

```shell
sh .agents/install-skills <target-repo> java-spring-standards pragmatic-tdd-spring conventional-commits
```

Name the skills explicitly like that. With no skill named, the installer copies **all** of them, which drops the Quarkus
standards into a Spring project and the other way round.

Edit skills in `.agents/skills/` — the `.claude/` and `.cursor/` directories are symlinks to it, and the installer
copies from it. After installing, point the target repository's `CLAUDE.md` / `AGENTS.md` at what you installed:

### Load a skill before you touch its area

| Stack       | Load                     | Before                                                                      |
|-------------|--------------------------|-----------------------------------------------------------------------------|
| Quarkus     | `java-quarkus-standards` | writing, refactoring or reviewing any Java code                             |
| Quarkus     | `pragmatic-tdd-quarkus`  | implementing or fixing behaviour, and before calling an implementation done |
| Spring Boot | `java-spring-standards`  | writing, refactoring or reviewing any Java code                             |
| Spring Boot | `pragmatic-tdd-spring`   | implementing or fixing behaviour, and before calling an implementation done |
| any stack   | `conventional-commits`   | writing any commit message                                                  |

Put only the row(s) matching the stack you installed into the target repository's `CLAUDE.md` / `AGENTS.md`, plus the
`conventional-commits` row — that one is stack-agnostic and always applies. The portable snippet for it (and for
overriding the default commit-message footer) already lives in this repository's own `CLAUDE.md` / `AGENTS.md` — copy it
over rather than retyping it.

## Commit messages

Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/); the rules live in
`.agents/skills/conventional-commits/SKILL.md`.

## Related Guides

- REST Jackson ([guide](https://quarkus.io/guides/rest#json-serialisation)): Jackson serialization support for Quarkus
  REST. This extension is not compatible with the quarkus-resteasy extension, or any of the extensions that depend on it
