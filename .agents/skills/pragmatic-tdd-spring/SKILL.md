---
name: pragmatic-tdd-spring
description: "Pragmatic TDD / test-first workflow and testing practices for Spring Boot 3 projects. Use whenever implementing or changing an endpoint, business service, or repository behaviour in a Spring Boot codebase; when fixing a bug; when asked to add a feature, 'write the tests', set up Testcontainers, mock a bean, or speed up a slow suite; and before declaring any implementation task complete. Defines the order of work (integration test first, then Controller/Service/Repository, then run the suite), what to test (HTTP contracts, happy path and error paths), when a unit test is worth it, and how real dependencies are provisioned with @SpringBootTest, MockMvcTester, @MockitoBean and Testcontainers @ServiceConnection. Complements the java-spring-standards skill, which defines how the code itself must be shaped. For Spring Boot projects only — a Quarkus codebase follows pragmatic-tdd instead."
---

# Development Methodology — Pragmatic TDD & Test-First (Spring Boot)

Work in this order. Do not implement first and backfill tests afterwards.

## Contract-First Testing

Before implementing any new endpoint or business service, write the integration test first (`@SpringBootTest` with
`@AutoConfigureMockMvc`, driven by `MockMvcTester`).

## Automated Verification Cycle

1. Write tests reflecting the expected HTTP status codes and response bodies (happy path & error paths).
2. Implement the required Controller, Service, and Repository components to satisfy the tests.
3. Execute tests via terminal to verify all assertions pass before marking the task complete.

## Focus on API Contracts

Prioritize black-box integration tests for REST resources over granular unit tests for internal private methods,
ensuring business contracts are protected without creating fragile test suites.

---

## Applying the cycle

**Step 1 — write the failing test.**

- One test class per controller, mirroring the production package under `src/test/java`, named `<Controller>Test`.
- Cover, at minimum: success (200/201/204), validation failure (400), and not found (404). Add the error paths the
  feature actually introduces (conflict, forbidden, etc.).
- Assert the contract, not the implementation: status code, response body fields, and headers such as `Location`.
  On error paths assert the problem-details body too (`title`, `status`, `detail`, and the offending fields on a 400),
  not just the status code — the error payload is part of the published contract.
- Run it and confirm it fails **for the intended reason** (missing endpoint, wrong status) — not because the test
  itself does not compile or a fixture is broken.

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    private MockMvcTester mvc;                       // MockMvc is auto-configured too; AssertJ picks this one

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldCreateUser() {
        assertThat(mvc.post().uri("/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {"name": "Ada", "email": "ada@example.com"}
                        """))
                .hasStatus(HttpStatus.CREATED)
                .matches(header().exists("Location"))
                .bodyJson().extractingPath("$.email").isEqualTo("ada@example.com");
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldRejectBlankName() {
        assertThat(mvc.post().uri("/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {"name": "", "email": "ada@example.com"}
                        """))
                .hasStatus(HttpStatus.BAD_REQUEST)
                .bodyJson().extractingPath("$.errors[0].field").isEqualTo("name");
    }

    @Test
    void shouldRejectAnonymousCaller() {                 // the 401 is part of the contract too
        assertThat(mvc.get().uri("/v1/users")).hasStatus(HttpStatus.UNAUTHORIZED);
    }
}
```

**Step 2 — implement.** Write the smallest Controller/Service/Repository code that satisfies the assertions, following
the layering and conventions in the `java-spring-standards` skill. No speculative endpoints, fields, or config that
no test asks for.

**Step 3 — verify in the terminal.**

```shell
./mvnw test           # unit + @SpringBootTest classes
./mvnw verify         # adds failsafe (*IT) runs when the plugin is configured
```

Report the real result. A task is complete only when the suite has actually been executed and passes.

## Choosing the kind of test

| Kind | Use it for | Cost |
|---|---|---|
| `@SpringBootTest` + `@AutoConfigureMockMvc` (**default**) | Every endpoint: status codes, payloads, validation, auth, error mapping | One context per distinct configuration, cached across classes |
| `@WebMvcTest` + `@MockitoBean` | Controller-only concerns (mapping, validation, serialization) when the service is genuinely uninteresting | Faster context, but proves nothing about persistence |
| `@DataJpaTest` + Testcontainers | Repository queries: a hand-written `@Query`, a projection, a native statement | Needs `@AutoConfigureTestDatabase(replace = NONE)` or it silently runs on an embedded database |
| Plain JUnit + Mockito (no Spring) | Pure functions and branching business rules: mappers, validators, value objects | Milliseconds — prefer it whenever no context is needed |
| `@SpringBootTest(webEnvironment = RANDOM_PORT)` (`*IT`) | Smoke-testing over a real HTTP stack, run by `./mvnw verify` | Slow; a handful of critical paths only |

A unit test that just restates the integration test with mocks is not worth writing — delete it.

## Real dependencies, never fakes

The database in tests is the **real engine production uses, in a container** — never H2 or an in-memory substitute: a
test that passes against a different engine proves nothing about the SQL that ships, and Flyway migrations written for
Postgres will not even apply. Docker must be running. `@ServiceConnection` wires the container into the context; see
`references/testing-toolbox.md` for the full setup, the dependencies to add, and the code.

External HTTP services are never called for real in tests — stub them (WireMock) or mock the client bean.

## Non-negotiables

- Never weaken, delete, or `@Disabled` a test to make a build green. Fix the code, or change the assertion
  deliberately and say why the old expectation was wrong.
- Never report an implementation as done without running the tests; if the suite cannot run, say so explicitly and why.
- A bug fix starts with a test that reproduces the bug and fails before the fix.
- Tests must be independent of execution order and of each other's leftover data — no shared mutable state, no
  "test A must run before test B".
- **Never disable security for the test profile.** Authenticate the request instead (`@WithMockUser`, `jwt()`), and
  keep one test per endpoint asserting the 401/403 — an endpoint tested without its access rules is untested.
- No `Thread.sleep` for asynchronous behaviour; poll with Awaitility or await the result deterministically.
- Do not assert on log output or on private methods.

## Reference

`references/testing-toolbox.md` — the test classpath, Testcontainers wiring via `@ServiceConnection`, transactional
rollback and data isolation, `@MockitoBean`, authenticating requests, stubbing external HTTP, and why the context
cache decides how fast the suite runs. Read it before setting up the first test class of a new kind (first DB test,
first mocked bean, first external integration).
