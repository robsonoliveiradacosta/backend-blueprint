# Testing Toolbox — setup, containers, mocking, isolation

Companion to `SKILL.md`. Read the section you need; do not paste all of it into a project at once.

## 1. Check the test classpath before writing the first test

Do not assume which test artifacts are available — the Quarkus test harness has been split and renamed across
versions, and a project may carry `quarkus-junit`, the older `quarkus-junit5`, or neither. Resolve it:

```shell
./mvnw dependency:build-classpath -Dmdep.outputFile=/tmp/cp.txt -Dmdep.includeScope=test
tr ':' '\n' < /tmp/cp.txt | grep -iE 'quarkus-(junit|test)|rest-assured|mockito|testcontainers'
```

To find which jar owns an annotation before importing it:

```shell
for j in $(tr ':' '\n' < /tmp/cp.txt); do
  unzip -l "$j" 2>/dev/null | grep -q 'io/quarkus/test/TestTransaction.class' && echo "$j"
done
```

**What tends to be there already** (observed on Quarkus 3.38.x with the `quarkus-junit` artifact — re-check on any
other version):

| Artifact | Provides |
|---|---|
| `io.quarkus:quarkus-junit` | `io.quarkus.test.junit.QuarkusTest`, `TestProfile`, `QuarkusTestProfile`, `QuarkusIntegrationTest`, plus JUnit Jupiter |
| `io.quarkus:quarkus-test-common` (transitive) | `io.quarkus.test.common.WithTestResource`, `QuarkusTestResourceLifecycleManager`, `TestResourceScope`, and the annotations `io.quarkus.test.TestTransaction` / `io.quarkus.test.InjectMock` |

**What usually has to be declared** — add with `<scope>test</scope>` when first needed, after confirming it is absent:

| Need | Artifact |
|---|---|
| RestAssured DSL (required for the very first integration test) | `io.rest-assured:rest-assured` |
| Mockito machinery behind `@InjectMock` / `@InjectSpy` | `io.quarkus:quarkus-junit5-mockito` |
| Real database in tests | `org.testcontainers:<engine>` + the matching `quarkus-jdbc-<engine>` runtime extension |
| Stubbing external HTTP | `io.quarkiverse.wiremock:quarkus-wiremock` (or plain `org.wiremock:wiremock`) |

Versions come from the Quarkus BOM where managed — do not pin versions that the BOM already manages. Confirm an
artifact exists in the BOM before adding it (`./mvnw dependency:tree`, or grep the resolved `quarkus-bom` pom) instead
of guessing.

## 2. Database container — two options

### Option A: Dev Services (preferred when you just need "a Postgres")

With `quarkus-jdbc-postgresql` on the classpath and **no** `quarkus.datasource.jdbc.url` configured for the test
profile, Quarkus starts a PostgreSQL container automatically (Testcontainers under the hood) and wires the datasource.
Zero test code, container reused across the whole suite.

```properties
# src/main/resources/application.properties
%test.quarkus.datasource.db-kind=postgresql
# no %test jdbc.url on purpose — Dev Services provides it
```

Pin the image when reproducibility matters: `quarkus.datasource.devservices.image-name=postgres:16-alpine`.

### Option B: explicit test resource (when you need control)

Use when the container needs custom configuration, extra services, or must be shared with non-Quarkus code.
Implement `QuarkusTestResourceLifecycleManager` and return the config the application should use:

```java
public class PostgresResource implements QuarkusTestResourceLifecycleManager {

    private final PostgreSQLContainer<?> container = new PostgreSQLContainer<>("postgres:16-alpine");

    @Override
    public Map<String, String> start() {
        container.start();
        return Map.of(
                "quarkus.datasource.jdbc.url", container.getJdbcUrl(),
                "quarkus.datasource.username", container.getUsername(),
                "quarkus.datasource.password", container.getPassword());
    }

    @Override
    public void stop() {
        container.stop();
    }
}
```

Activate it with `@WithTestResource` (`io.quarkus.test.common.WithTestResource`), **not** the older
`@QuarkusTestResource`:

```java
@QuarkusTest
@WithTestResource(value = PostgresResource.class, scope = TestResourceScope.GLOBAL)
class UserResourceTest { ... }
```

`scope` (`io.quarkus.test.common.TestResourceScope`) decides how often the container starts:

- `MATCHING_RESOURCES` (default) — shared by all test classes declaring the same resources
- `GLOBAL` — one instance for the whole suite; use this for the database so it starts once
- `RESTRICTED_TO_CLASS` — its own instance for that class; forces a Quarkus restart, so use sparingly

Conflicting scopes across classes cause Quarkus restarts and a slow suite. If the suite starts crawling, this is the
first thing to check.

When Dev Services is also active you may need `quarkus.devservices.enabled=false` for the test profile so the two do
not both provision a database.

## 3. Test data isolation

Pick one strategy per test class and stick to it:

- **Clean before each test** (works everywhere, explicit):

```java
@BeforeEach
@Transactional
void resetData() {
    userRepository.deleteAll();
    // insert only the fixtures this class needs
}
```

- **`@TestTransaction`** (`io.quarkus.test.TestTransaction`) — runs the test method inside a transaction and rolls it
  back once the method completes, reverting every database change. Prefer it for tests that call repositories or
  services **directly**: no cleanup code, no leftovers. The annotation ships in `quarkus-test-common` (confirm with the
  classpath check in section 1); it needs a transaction manager at runtime, which arrives with the Hibernate ORM
  extension.

  Two limits worth knowing:
  - Plain `@Transactional` on a test does **not** roll back — those writes are persisted and will leak into the next
    test. Use it only when you deliberately want the data to survive.
  - It does not help black-box RestAssured tests: the HTTP request is handled by the server in its own transaction,
    which commits regardless. For endpoint tests, clean in `@BeforeEach` as above. The same applies to any code the
    test triggers in a separate transaction (`REQUIRES_NEW`, async work).

Never rely on data created by another test class, and never assert on auto-generated IDs having specific values —
sequences do not reset between tests.

## 4. Mocking

- `@InjectMock` (`io.quarkus.test.InjectMock`, shipped with `quarkus-test-common`) replaces a CDI
  bean with a Mockito mock for the whole test class; `@InjectSpy` wraps the real bean so unstubbed methods still run.
  Both need `quarkus-junit5-mockito` for the Mockito machinery. Do not import the legacy
  `io.quarkus.test.junit.mockito.InjectMock`.
- Mock at the boundary: repositories and REST clients, not the service under test.
- Mocking does **not** work under `@QuarkusIntegrationTest` — the app runs as a separate process there.
- Verify observable behaviour (returned value, persisted state, response body). Reach for `verify()` only when the
  interaction *is* the contract, e.g. "the notification client was called exactly once".
- No mock of a type you own that has no logic — construct the real object instead.

## 5. Alternate configuration: `@TestProfile`

When a test needs different config (a feature flag off, a shorter timeout, a stub bean), do not mutate the shared
`application.properties` — declare a profile:

```java
public class RateLimitDisabledProfile implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of("app.rate-limit.enabled", "false");
    }
}

@QuarkusTest
@TestProfile(RateLimitDisabledProfile.class)
class UserResourceRateLimitTest { ... }
```

Each distinct profile means another Quarkus boot — group tests that share a profile in the same class, and keep the
number of profiles small.

## 6. External HTTP

Never let a test reach the real internet. Either stub the endpoint with WireMock and point the REST client's config
key at the stub URL, or `@InjectMock` the `@RegisterRestClient` interface. Assert what your code does with the
response — including the failure paths (timeout, 500, malformed body), which are the ones production will hit.

## 7. Packaged-artifact tests

`@QuarkusIntegrationTest` on a `*IT` class runs the tests against the built jar or native binary and is executed by
`./mvnw verify` when the failsafe plugin is configured in `pom.xml` (the Quarkus blueprint pom wires it, with the
`native` profile enabling ITs). The
common pattern is a subclass that reuses the `@QuarkusTest` class:

```java
@QuarkusIntegrationTest
class UserResourceIT extends UserResourceTest { }
```

Keep this to the critical paths: it is slow, and mocks/`@InjectMock` do not apply.

## 8. Making the suite fast

- One boot is the goal: same profile, same test resources, `GLOBAL` scope for containers.
- Reuse fixtures via `@BeforeAll` when the data is read-only; use `@BeforeEach` cleanup only for tests that write.
- Prefer plain JUnit for logic that needs no CDI — it costs milliseconds instead of a boot.
- If a class needs a restart-inducing setup, keep it small and isolated so it does not slow the rest down.
