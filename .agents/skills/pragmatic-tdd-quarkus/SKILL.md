---
name: pragmatic-tdd-quarkus
description: "Pragmatic TDD / test-first workflow and testing practices for Quarkus projects. Use whenever implementing or changing an endpoint, business service, or repository behaviour; when fixing a bug; when asked to add a feature, 'write the tests', set up Testcontainers, mock a dependency, or speed up a slow suite; and before declaring any implementation task complete. Defines the order of work (integration test first, then Resource/Service/Repository, then run the suite), what to test (HTTP contracts, happy path and error paths), when a unit test is worth it, and how real dependencies are provisioned with QuarkusTest, RestAssured, @InjectMock and Dev Services. Complements the java-quarkus-standards skill, which defines how the code itself must be shaped. For Quarkus projects only — a Spring Boot codebase follows pragmatic-tdd-spring instead."
---

# Development Methodology — Pragmatic TDD & Test-First

Work in this order. Do not implement first and backfill tests afterwards.

## Contract-First Testing

Before implementing any new endpoint or business service, write the integration test first (`@QuarkusTest` using
RestAssured).

## Automated Verification Cycle

1. Write tests reflecting the expected HTTP status codes and response bodies (happy path & error paths).
2. Implement the required Resource, Service, and Repository components to satisfy the tests.
3. Execute tests via terminal to verify all assertions pass before marking the task complete.

## Focus on API Contracts

Prioritize black-box integration tests for REST resources over granular unit tests for internal private methods,
ensuring business contracts are protected without creating fragile test suites.

---

## Applying the cycle

**Step 1 — write the failing test.**

- One test class per resource, mirroring the production package under `src/test/java`, named `<Resource>Test`.
- Cover, at minimum: success (200/201/204), validation failure (400), and not found (404). Add the error paths the
  feature actually introduces (conflict, forbidden, etc.).
- Assert the contract, not the implementation: status code, response body fields, and headers such as `Location`.
  On error paths assert the problem-details body too (`title`, `status`, `detail`, and the offending fields on a 400),
  not just the status code — the error payload is part of the published contract.
- Run it and confirm it fails **for the intended reason** (missing endpoint, wrong status) — not because the test
  itself does not compile or a fixture is broken.

```java
@QuarkusTest
class UserResourceTest {

    @Test
    void shouldCreateUser() {
        given()
            .contentType(ContentType.JSON)
            .body(new CreateUserRequest("Ada", "ada@example.com"))
        .when()
            .post("/v1/users")
        .then()
            .statusCode(201)
            .body("id", notNullValue())
            .body("email", is("ada@example.com"));
    }

    @Test
    void shouldRejectBlankName() {
        given()
            .contentType(ContentType.JSON)
            .body(new CreateUserRequest("", "ada@example.com"))
        .when()
            .post("/v1/users")
        .then()
            .statusCode(400);
    }
}
```

**Step 2 — implement.** Write the smallest Resource/Service/Repository code that satisfies the assertions, following
the layering and conventions in the `java-quarkus-standards` skill. No speculative endpoints, fields, or config that
no test asks for.

**Step 3 — verify in the terminal.**

```shell
./mvnw test           # unit + @QuarkusTest
./mvnw verify         # adds failsafe (*IT) runs when present
```

Report the real result. A task is complete only when the suite has actually been executed and passes.

## Choosing the kind of test

| Kind | Use it for | Cost |
|---|---|---|
| `@QuarkusTest` + RestAssured (**default**) | Every endpoint: status codes, payloads, validation, auth, error mapping | Boots the app once per profile |
| `@QuarkusTest` + `@InjectMock` | Branching business rules, calculations, and edge cases awkward to reach over HTTP | Cheap once the app is booted |
| Plain JUnit (no `@QuarkusTest`) | Pure functions: mappers, validators, value objects | Milliseconds — prefer it when no CDI is needed |
| `@QuarkusIntegrationTest` (`*IT`) | Smoke-testing the packaged/native artifact, run by `./mvnw verify` | Slow; a handful of critical paths only |

A unit test that just restates the integration test with mocks is not worth writing — delete it.

## Real dependencies, never fakes

The database in tests is the **real engine production uses, in a container** — never H2 or an in-memory substitute: a
test that passes against a different engine proves nothing about the SQL that ships. Docker must be running. Two ways to get the
container, both Testcontainers underneath — see `references/testing-toolbox.md` for the full setup, dependencies to
add, and code.

External HTTP services are never called for real in tests — stub them (WireMock) or mock the REST client bean.

## Non-negotiables

- Never weaken, delete, or `@Disabled` a test to make a build green. Fix the code, or change the assertion
  deliberately and say why the old expectation was wrong.
- Never report an implementation as done without running the tests; if the suite cannot run, say so explicitly and why.
- A bug fix starts with a test that reproduces the bug and fails before the fix.
- Tests must be independent of execution order and of each other's leftover data — no shared mutable state, no
  "test A must run before test B".
- No `Thread.sleep` for asynchronous behaviour; poll with Awaitility or await the result deterministically.
- Do not assert on log output or on private methods.

## Reference

`references/testing-toolbox.md` — Testcontainers via Dev Services vs. explicit `@WithTestResource`, test data
isolation, `@TestProfile`, mocking beans and REST clients, and how to check which test dependencies are still missing.
Read it before setting up the first test class of a new kind (first DB test, first mocked service, first external
integration).
