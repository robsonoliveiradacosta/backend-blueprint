# Testing Toolbox — setup, containers, mocking, isolation

Companion to `SKILL.md`. Read the section you need; do not paste all of it into a project at once.

## 1. Check the test classpath before writing the first test

Do not assume which test artifacts are available. Resolve it:

```shell
./mvnw dependency:tree -Dscope=test
./mvnw dependency:tree | grep -iE 'testcontainers|security-test|wiremock|mockito|assertj'
```

**What `spring-boot-starter-test` brings** (confirm on the version this project resolved): JUnit Jupiter, Spring Test
and Spring Boot Test, AssertJ, Hamcrest, Mockito, JSONassert, JsonPath and XMLUnit. `MockMvcTester` needs nothing
extra — it ships in `spring-test` and is auto-configured whenever AssertJ is present.

**What usually has to be declared** — add with `<scope>test</scope>` when first needed, after confirming it is absent:

| Need | Artifact |
|---|---|
| `@ServiceConnection` wiring between containers and the context | `org.springframework.boot:spring-boot-testcontainers` |
| `@Testcontainers` / `@Container` JUnit lifecycle | `org.testcontainers:junit-jupiter` |
| The database engine itself | `org.testcontainers:postgresql` (and the JDBC driver at runtime scope) |
| Authenticating requests in tests | `org.springframework.security:spring-security-test` |
| Stubbing external HTTP | `org.wiremock:wiremock-standalone` |

The Boot BOM manages Testcontainers (through the imported `testcontainers-bom`) and the Spring Security artifacts —
do not pin versions it already provides. WireMock is not managed; that one carries an explicit version.

## 2. Database container

Both options below start the real engine, and Flyway then runs your actual migrations against it — which is half the
value of the test. Pin the image tag (`postgres:16-alpine`), never `latest`.

### Option A: `@TestConfiguration` bean (preferred)

The container becomes a bean, so its lifecycle follows the application context — and the context cache means every
test class sharing this configuration shares the one container.

```java
@TestConfiguration(proxyBeanMethods = false)
public class ContainersConfiguration {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }
}
```

```java
@SpringBootTest
@AutoConfigureMockMvc
@Import(ContainersConfiguration.class)
abstract class AbstractIntegrationTest {
}
```

The same configuration can back `./mvnw spring-boot:test-run` through a `TestApplication` class
(`SpringApplication.from(Application::main).with(ContainersConfiguration.class).run(args)`), so local development and
the test suite provision the database identically.

### Option B: static `@Container` field

```java
@SpringBootTest
@Testcontainers
class UserRepositoryTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Simple, but a `static @Container` starts and stops **per test class**. Across a suite that means a container per
class. Put the field on one abstract base class every integration test extends, or enable Testcontainers reuse
(`withReuse(true)` plus `testcontainers.reuse.enable=true` in `~/.testcontainers.properties`) so the container
survives between classes and runs.

### Repository slices

`@DataJpaTest` replaces the datasource with an embedded database by default — the exact substitution this project
forbids. Always pair it with the real one:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(ContainersConfiguration.class)
class UserRepositoryTest { ... }
```

## 3. Test data isolation

Pick one strategy per test class and stick to it:

- **`@Transactional` on the test class** — the Spring TestContext framework rolls the transaction back after each
  method. It works for `@DataJpaTest` and for `@SpringBootTest` + MockMvc, because the mock request runs on the test
  thread inside that transaction.
  - It does **not** work with `webEnvironment = RANDOM_PORT`: the request is handled by the server in its own
    transaction, which commits regardless.
  - It also hides missing-flush bugs — code that works only because the test's transaction is still open. When a test
    exists to prove persistence really happened, clean explicitly instead.
- **Clean before each test** (works everywhere, explicit):

```java
@BeforeEach
void resetData() {
    userRepository.deleteAll();
    // insert only the fixtures this class needs
}
```

- `@Sql("/fixtures/users.sql")` for a fixed dataset a whole class shares.

Never rely on data created by another test class, and never assert that an auto-generated id has a specific value —
sequences do not reset between tests. Do not reach for `@DirtiesContext` to clean data: it discards the cached
context and reboots the application (§8).

## 4. Mocking

- **`@MockitoBean` / `@MockitoSpyBean`** (`org.springframework.test.context.bean.override.mockito`) replace a bean
  with a Mockito mock or wrap the real one. `@MockBean` and `@SpyBean` are deprecated — do not use them in new code.
- Mock at the boundary: external HTTP clients and third-party gateways. Do **not** mock the repository in an endpoint
  test whose job is to prove the query and the migration work.
- Every distinct set of bean-override declarations creates another entry in the context cache, i.e. another
  application boot. Group tests that need the same overrides in the same class.
- Verify observable behaviour (returned value, persisted state, response body). Reach for `verify()` only when the
  interaction *is* the contract, e.g. "the notification client was called exactly once".
- No mock of a type you own that has no logic — construct the real object instead.

## 5. Authenticating requests

With the deny-by-default chain from `java-spring-standards` §13, an unauthenticated request gets 401 and proves
nothing about the endpoint. Authenticate it — never switch security off for the test profile.

- `@WithMockUser(roles = "ADMIN")` on the method or class. `roles` prepends `ROLE_`, `authorities` does not — mixing
  them up is the usual reason a `@PreAuthorize("hasRole('ADMIN')")` test fails with 403.
- For a resource server, the request post-processors: `mvc.get().uri("/v1/users").with(jwt().authorities(...))`.
- Keep one test per endpoint asserting the 401 (anonymous) and, where roles differ, the 403 — both are contract.
- **`@WebMvcTest` does not load your `SecurityConfiguration`**: it is a `@Configuration` class, not a web-layer
  component, so the slice runs under Boot's default chain. `@Import(SecurityConfiguration.class)` if the slice test is
  meant to say anything about access rules.

## 6. External HTTP

Never let a test reach the real internet. Either stub the endpoint with WireMock and point the client's configuration
property at the stub URL, or use `MockRestServiceServer` (`RestTemplate`, `RestClient`) / the `@RestClientTest` slice.
Assert what your code does with the response — including the failure paths (timeout, 500, malformed body), which are
the ones production will hit.

## 7. Full-stack tests over real HTTP

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Import(ContainersConfiguration.class)
class UserControllerIT {

    @Autowired
    private TestRestTemplate restTemplate;
}
```

Name these `*IT` and keep them to the critical paths. Two things to know: `@Transactional` does not roll them back
(§3), and **the Spring Boot parent does not configure failsafe** — without `maven-failsafe-plugin` wired to the
`integration-test` and `verify` goals in `pom.xml`, `*IT` classes never run at all.

## 8. Making the suite fast

- **The context cache is the whole game.** Test classes with identical configuration — same annotations, same
  properties, same bean overrides — share one boot. Put the shared setup in one abstract base class and extend it
  instead of repeating annotations with small variations.
- What forks a new context: a different set of `@MockitoBean`s, a one-off `@TestPropertySource`, a different set of
  `@Import`s, `@ActiveProfiles`. Each is another application start.
- `@DirtiesContext` evicts the cache entry — use it only when a test genuinely corrupts the context.
- Prefer plain JUnit + Mockito for logic that needs no context: it costs milliseconds instead of a boot.
- One database container for the whole suite (§2), not one per class.
