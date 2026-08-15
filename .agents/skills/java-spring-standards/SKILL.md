---
name: java-spring-standards
description: "Java 21 + Spring Boot 3 synchronous REST standards using the Spring Data JPA repository pattern. Use whenever generating, refactoring, or reviewing Java code in a Spring Boot project: creating or changing JPA entities, Spring Data repositories, @Service classes, @RestController endpoints, record DTOs, @RestControllerAdvice handlers and RFC 9457 ProblemDetail error payloads, logging, Flyway migrations, configuration or endpoint security, springdoc OpenAPI documentation, or @SpringBootTest tests; adding a CRUD or a new endpoint; deciding how a feature should be structured; or checking existing code for SOLID, clean-code, and layering violations. Enforces record DTOs (Lombok banned), sequence-generated IDs, the Spring Data JPA repository pattern, explicit manual mapping (no reflection mappers), /v1/{resources} versioned URIs, constructor injection, and synchronous endpoints on virtual threads. For Spring Boot projects only — a Quarkus codebase follows java-quarkus-standards instead."
---

# Java 21 & Spring Boot 3 — Coding Standards

You are an expert Java 21+ and Spring Boot 3+ engineer. Every piece of Java code you generate, refactor, or review in
a Spring Boot codebase MUST follow the architecture below. These rules are not suggestions — do not invent alternative
layers, patterns, or libraries.

Two companion files carry the detail: `references/code-examples.md` has the canonical shape for every layer — read
the entry for the layer you are about to write — and `references/production.md` covers migrations, configuration and
access control. The rules below say what must be true; the references show the exact shape.

## Before writing code

1. **Detect the base package** from existing sources under `src/main/java/` (read the `package` declaration of any
   class, or find the `@SpringBootApplication` class — everything must live under its package so component scanning
   sees it). It may differ from the `groupId` in `pom.xml`. If the project has no Java source yet, ask the user for
   the base package before creating the first class. Never assume `com.example`.
2. **Read `pom.xml`** — take the Spring Boot version from the `spring-boot-starter-parent` (or the imported
   `spring-boot-dependencies` BOM) and the Java release from `<java.version>`. Never hardcode versions in new code or
   docs, and never pin a version the Boot BOM already manages.
3. **Check which starters are actually present** (`./mvnw dependency:tree`) — never assume the initializr defaults. A
   fresh project typically carries only `spring-boot-starter-web` and `spring-boot-starter-test`; the rest must be
   added to `pom.xml` when a feature needs them:
   - `spring-boot-starter-data-jpa` — repositories, entities, transactions
   - `spring-boot-starter-validation` — Jakarta Bean Validation. **It is not transitive through `-web`**; without it
     `@Valid` silently does nothing.
   - the JDBC driver (`org.postgresql:postgresql`, …)
   - `flyway-core` **plus the engine module** (`flyway-database-postgresql`, `flyway-mysql`, …) — Flyway 10 split
     engine support out of core, so check the resolved Flyway version in the tree before assuming core is enough
   - `springdoc-openapi-starter-webmvc-ui` — **not managed by the Boot BOM**, so it carries an explicit `<version>`;
     Spring Boot 3.x needs the 2.x line
   - `spring-boot-starter-security` — only when the project actually authenticates
4. **Verify APIs with context7** before using an annotation or method you are not certain about in the Spring Boot
   version this project resolved. Spring's own defaults move between minors (`@MockBean` → `@MockitoBean`,
   `authorizeRequests` → `authorizeHttpRequests`); do not write from memory.
5. **Reuse before creating** — check for existing shared classes (pagination wrapper, exception handler, test helpers)
   instead of duplicating them.

**On the 3.x line:** 3.5 is its final minor and open-source support for it ended in mid-2026 — a new project should
start on the current major, and an existing one carries a known upgrade debt. This skill targets 3.x because that is
what was asked for; say so once when starting greenfield work, then build what the user asked for.

## 1. Package layout

Relative to the detected base package, always use:

```
{base.package}/
├── domain/entity/   # JPA entities  (<Name>Entity)
├── repository/      # Spring Data JPA repository interfaces
├── dto/             # request/response records
├── service/         # business logic + orchestration
├── controller/      # @RestController endpoints
├── exception/       # custom exceptions and the @RestControllerAdvice
└── config/          # @Configuration classes (security, OpenAPI, @ConfigurationProperties)
```

Do not add extra layers (mappers, facades, `util` dumping grounds) beyond these.

## 2. Modern Java & DTOs

- Target **Java 21+**; prefer records, sealed types, pattern matching, and enhanced switch where they read cleanly.
- Every request and response DTO **MUST** be a `public record`.
- **Lombok is strictly prohibited.** No `@Data`, `@Builder`, `@Getter`, no Lombok dependency in `pom.xml`.
- Validate with Jakarta Bean Validation annotations placed directly on record components.

*Shape to copy: `references/code-examples.md` §2.*

## 3. Execution & threading model

- **Synchronous by default.** Do not introduce WebFlux (`Mono`/`Flux`), `CompletableFuture`, or reactive pipelines
  unless the user explicitly asks for a reactive stream. Spring MVC and Spring WebFlux are different stacks — do not
  mix `spring-boot-starter-web` and `spring-boot-starter-webflux` in one application.
- Enable virtual threads with **`spring.threads.virtual.enabled=true`** and write plain blocking code. Tomcat then
  serves each request on a virtual thread, so blocking is cheap.
- Two consequences the property does not advertise:
  - It requires Java 21 (24+ performs better, because pinning on `synchronized` was removed there). It also makes
    thread-pool sizing properties inert — there is no fixed pool left to size.
  - **It does not multiply your database connections.** The real ceiling for a JDBC endpoint is the Hikari pool
    (`spring.datasource.hikari.maximum-pool-size`), and more virtual threads simply queue for it. Size the pool
    deliberately instead of expecting virtual threads to remove the limit.

## 4. Entities — `{base.package}.domain.entity`

- Plain JPA classes named `<Name>Entity`, annotated `@Entity` + `@Table(name = "...")` with snake_case, plural table
  names.
- Primary keys **MUST** use a database sequence so Hibernate can batch inserts:
  `@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "seq_<entity>")` with a matching
  `@SequenceGenerator`. Sequence naming convention: `seq_<entity_name>` (singular, snake_case).
- **`allocationSize` must match the sequence's `INCREMENT BY` in the database.** A sequence created with the SQL
  default (`INCREMENT BY 1`) while Hibernate assumes `allocationSize = 50` hands out ids that collide. Either create
  it as `CREATE SEQUENCE seq_user INCREMENT BY 50` in the migration, or set `allocationSize = 1` and give up batching.
- All fields `private`, exposed through explicit getters/setters — written by hand, never generated by Lombok.
- Enums persisted with `@Enumerated(EnumType.STRING)`; `@ManyToOne(fetch = FetchType.LAZY)` to avoid N+1.
- **No entity ever leaves the service layer** — not as a return type, not inside another DTO. §6 governs the crossing.
- Do not implement `equals`/`hashCode` from a generated id: it is `null` before the flush, so the object changes
  identity mid-transaction and breaks any `Set` it sits in. Leave the defaults, or key on a business-unique column.

*Shape to copy: `references/code-examples.md` §4.*

## 5. Repositories — `{base.package}.repository`

- An **interface** extending `JpaRepository<XEntity, Long>` — Spring Data implements it. Never write an
  `@Repository` class wrapping an `EntityManager` for plain CRUD, and never inject an `EntityManager` outside this
  package.
- **All** JPQL, filters, sorting and pagination queries live here — never in services or controllers. Derived query
  methods (`findByEmail`) for one or two conditions; `@Query` once the method name stops reading like a sentence;
  `nativeQuery = true` only when JPQL genuinely cannot express it, with a comment saying why.
- Paginated finders take a `Pageable` and return `Page<XEntity>`; the count query comes for free. Use `Slice` when the
  total is expensive and the client only needs "is there more".
- Return `Optional<XEntity>` for single lookups — never `null`.
- A `@Query` that writes needs `@Modifying`, and it bypasses the persistence context: entities already loaded there
  keep the stale values unless you clear them.

*Shape to copy: `references/code-examples.md` §5.*

## 6. Entity ↔ DTO mapping

- **No reflection-based mappers** (ModelMapper, Dozer, and similar). They break record immutability, cost startup
  time, and fail silently when a field is renamed.
- Map explicitly: a static factory on the response record, or a private mapping method in the service.

MapStruct is compile-time and acceptable **only** if the user explicitly asks for it; otherwise write the mapping by
hand.

*Shape to copy: `references/code-examples.md` §6.*

## 7. Services — `{base.package}.service`

- `@Service`, holding all business logic and orchestration.
- **Always constructor injection** — declare collaborators `private final` and take them in the constructor (`@Autowired`
  is unnecessary with a single constructor). Field injection is not used: it hides dependencies and makes the class
  untestable without a container.
- Annotate write/update/delete methods with `@Transactional`
  (`org.springframework.transaction.annotation.Transactional`, not the Jakarta one) and read methods with
  `@Transactional(readOnly = true)`.
- **`@Transactional` is proxy-based, so it only applies on the way in.** A `public` method calling another method of
  the same class runs with the caller's transaction — the annotation on the inner method is ignored. If a step needs
  its own transaction, it belongs on another bean. `private`, `final` and `static` methods are never advised at all.
- Throw the domain exceptions from `{base.package}.exception` (§9) — never return `null` to signal "not found", and
  never build a `ResponseEntity` inside a service. Turning the exception into HTTP is the advice's job.

*Shape to copy: `references/code-examples.md` §7.*

## 8. Controllers — `{base.package}.controller`

- `@RestController` + `@RequestMapping("/v1/{resources}")` at class level, with `@GetMapping`, `@PostMapping`,
  `@PutMapping`, `@PatchMapping`, `@DeleteMapping` on the methods.
- Return **`ResponseEntity<T>`** whenever the status, headers or emptiness matter (creation, deletion); a bare DTO is
  fine for a plain 200.
- The controller **MUST NOT** touch a repository or the `EntityManager` — it delegates to the service.
- The controller **MUST NOT** accept or return JPA entities — records only, in and out.
- Return synchronous types only. No `Mono`, `Flux`, `CompletableFuture` or `DeferredResult`.
- `@Valid` on request bodies — without it, the annotations of §2 are decoration. Validating a `@PathVariable` or
  `@RequestParam` directly additionally needs `@Validated` on the class.

### URI design

- **`/v1/{resources}`** — explicit version in the path, plural nouns (`/v1/users`, `/v1/orders`). No header or
  query-parameter versioning unless explicitly requested.
- **Never put a verb in a URI.** `/v1/getUser` and `/v1/createUser` are wrong; the HTTP method is the verb. An action
  that is genuinely not CRUD becomes a sub-resource noun (`POST /v1/orders/{id}/cancellation`).
- **Nest at most two levels** (`/v1/customers/{customerId}/orders`). Anything deeper is addressed through its own
  top-level URI (`/v1/orders/{orderId}`), not `/v1/customers/1/orders/2/items/3`.
- **A published version never changes shape in place.** Once clients integrate against `/v1`, an unavoidable breaking
  change ships as `/v2` beside it — removing a field, renaming one, or tightening validation on the live version
  breaks callers who had no way to know. Retiring `/v1` afterwards is a separate, announced decision.

### Method semantics

- `GET` — read. Safe and idempotent: it never changes state, and never carries a request body.
- `POST` — create. Returns `201 Created` with a `Location` header pointing at the new resource, and the created
  payload in the body.
- `PUT` — full replacement of an existing resource, or creation when the client owns the key. Idempotent: sending it
  twice leaves the same state.
- `PATCH` — partial update; only the fields present in the payload change. The request record must distinguish an
  absent field from an explicit `null` (e.g. `Optional<String>` components), otherwise "do not touch" and "set to
  null" become the same request.
- `DELETE` — removal. Returns `204 No Content`, with no body.

### Status codes

| Code | When |
|---|---|
| 200 OK | successful read, or an update that returns the resource |
| 201 Created | successful creation (with `Location`) |
| 204 No Content | success with nothing to return (`DELETE`) |
| 400 Bad Request | malformed payload or validation failure |
| 401 / 403 | missing identity / identity without the required role (§13) |
| 404 Not Found | the entity or the URI does not exist |
| 409 Conflict | business rule violation or unique-key conflict — this is what `DomainConflictException` maps to |

Every non-2xx body is the RFC 9457 payload from §9 — controllers never format an error themselves.

### Collection contract

- Take Spring Data's **`Pageable`** as a method parameter and let the argument resolver read `page` (0-indexed),
  `size` and `sort` (`sort=createdAt,desc`) from the query string. Do not re-parse them by hand.
- Set the ceiling in configuration: `spring.data.web.pageable.max-page-size=100` and
  `spring.data.web.pageable.default-page-size=20`. The framework default maximum is 2000 — high enough for a client to
  pull most of a table in one call.
- **Validate the sort property against an allow-list** in the service. Whatever the client types goes into the query,
  and an unknown property surfaces as a `PropertyReferenceException` — a `500` for what is a client mistake. Reject it
  as `400` instead.
- **A paginated endpoint returns `PageResponse<T>`**, never Spring's `Page<T>` and never a bare `List`. Serializing
  `PageImpl` directly exposes an internal type whose JSON shape carries no stability guarantee (Spring Data logs a
  warning saying exactly that); a record you own is the contract. Map `Page<Entity>` → `PageResponse<Dto>` in the
  service.

*Shape to copy: `references/code-examples.md` §8.*

## 9. Defensive programming & global exception handling (RFC 9457) — `{base.package}.exception`

- **Input validation at the edge:** validate incoming requests with Jakarta Validation annotations on request
  `record`s. Never hand-write repetitive `if (x == null)` checks where an annotation applies.
- **Domain-specific exceptions:** create unchecked domain exceptions (extending `RuntimeException`) for expected
  business failures (`ResourceNotFoundException`, `DomainConflictException`). Services throw these — they never build
  HTTP responses themselves. Do not name yours `EntityNotFoundException`: `jakarta.persistence.EntityNotFoundException`
  already exists and Hibernate throws it, so the two get imported interchangeably and the handler catches the wrong one.
- **One global handler:** a single `@RestControllerAdvice` class in this package, **extending
  `ResponseEntityExceptionHandler`**, with one `@ExceptionHandler` method per domain exception plus a catch-all for
  `Exception`. Never a `try/catch` that formats an error inside a controller, and never a raw stack trace in a
  response.
- **Every domain exception has a handler method.** One without a handler falls into the catch-all and reaches the
  client as `500` — so introducing an exception and handling it are the same task, not two.
- **RFC 9457 Problem Details** (the RFC that obsoletes 7807; Spring's `ProblemDetail` implements it): every error
  response is a `ProblemDetail` carrying `type`, `title`, `status`, `detail`, `instance`, and a `timestamp` added as a
  property. Returning `ProblemDetail` sets `application/problem+json` automatically — do not set it by hand.
- **Validation failures:** override `handleMethodArgumentNotValid` to map each `FieldError` to a structured entry
  (`field` + `message`) under an `errors` property, and answer `400`.

Four details that decide whether this actually works:

- **Extending `ResponseEntityExceptionHandler` is what covers the framework's own exceptions** — unreadable JSON,
  wrong method, missing parameter, `MethodArgumentNotValidException`. Boot's auto-configured problem-details handler
  backs off as soon as a `ResponseEntityExceptionHandler` bean exists, so once you declare yours, yours is the only
  one; `spring.mvc.problemdetails.enabled` then changes nothing.
- `type` stays present — `about:blank` is the value the RFC prescribes when there is no documentation URI, so callers
  can rely on the field existing.
- `timestamp` and `errors` are not fields of `ProblemDetail`; add them with `setProperty(...)`, which serializes them
  at the top level. Emit the timestamp as ISO-8601 and assert the rendered shape in a test rather than trusting the
  serializer's defaults.
- **Security failures never reach the advice.** `@RestControllerAdvice` only sees exceptions raised inside the
  dispatcher, and Spring Security rejects unauthenticated requests in a filter before it. Without an
  `AuthenticationEntryPoint` and an `AccessDeniedHandler` writing the same payload, your 401 and 403 answer in a
  different shape than every other error.

Build the payload through one shared helper so every handler produces an identical envelope — the point of a global
contract is that a client parses one shape.

*Shape to copy: `references/code-examples.md` §9.*

## 10. Tests

The order of work (test first), the choice between integration and unit tests, and container/mocking setup belong to
the `pragmatic-tdd-spring` skill — read it before starting. Conventions that bind here:

- Controllers are covered by `@SpringBootTest` + `@AutoConfigureMockMvc`; every new endpoint gets at least a success
  case, a validation-failure case, and a not-found case.
- The database in tests is the real engine in a Testcontainers container, never H2.
- Mirror the production package structure under `src/test/java`; name classes `*Test` (`*IT` for failsafe runs).

## 11. Code quality, SOLID & clean code

- **Single Responsibility (SRP):** enforce strict separation of concerns across layers (Controller handles HTTP,
  Service handles domain logic, Repository handles persistence). A class that reaches across two of those
  responsibilities is split, not extended.
- **Constructor Injection (DIP):** always use constructor-based dependency injection in `@Service` and
  `@RestController` components. No `@Autowired` fields, no setter injection.
- **Cyclomatic complexity & method size:** keep methods short (under 20 lines) and focused. Use early returns (guard
  clauses) to minimize nested `if/else`.
- **Null safety & intentional returns:** never return `null` from service methods. Use `Optional<T>` for potentially
  missing single entities and empty collections (`List.of()`) when no results match.
- **Self-documenting code:** choose clear, intention-revealing names. Avoid obscure abbreviations or comments that
  restate what the code already says.

When a service method exists to serve an endpoint that must answer 404, it still throws instead of returning an empty
`Optional` — the `Optional` is for callers that can meaningfully handle absence.

*Shape to copy: `references/code-examples.md` §11.*

## 12. Observability & structured logging

- **Standard logger:** SLF4J over Logback, both already on the classpath through `spring-boot-starter`. Declare it as
  `private static final Logger log = LoggerFactory.getLogger(YourClass.class);`. Never `System.out.println`.
- **Contextual logging (no PII):** never log passwords, tokens, full request bodies, or personal identifiers. Log
  event names, resource IDs, or entity UUIDs instead.
- **Levels:**
  - `INFO` — business-relevant milestones (entity created, status updated).
  - `WARN` — handled anomalies or unexpected business conditions.
  - `ERROR` — unhandled system exceptions and infrastructure failures; always pass the `Throwable`.
  - `DEBUG` — technical execution details useful during local debugging.
- **Parametrized messaging:** use `{}` placeholders, never string concatenation:
  `log.info("Created resource with ID: {}", resourceId);`. The `Throwable` goes last, without a placeholder.

Two rules that keep the logs readable:

- **Log an exception once**, in the `@RestControllerAdvice` of §9 — at `ERROR` with the `Throwable`, or at `WARN` for
  expected business failures such as a not-found. Services and controllers do not log-and-rethrow on the way up.
- The stack trace goes to the log, never to the response body. §9 governs what the client sees; this section governs
  what the operator sees.

*Shape to copy: `references/code-examples.md` §12.*

## 13. Production concerns — schema, configuration, access

Three rules that hold everywhere; the detail, the properties and the shapes are in `references/production.md`, which
you must read before changing the schema, adding configuration, or exposing an endpoint.

- **The schema belongs to Flyway, never to Hibernate.** `spring.jpa.hibernate.ddl-auto=none`, every change shipped as
  `V<yyyyMMddHHmmss>__snake_case_description.sql`, `spring.flyway.out-of-order=true`, and an applied migration is
  never edited.
- **No secret in the repository.** Credentials and environment-specific values arrive through environment variables
  and profiles, read via a typed `@ConfigurationProperties` record.
- **Access is deny-by-default.** One `SecurityFilterChain` ending in `.anyRequest().authenticated()`, and every
  exception to that is written out explicitly.

## 14. OpenAPI & Swagger UI (springdoc)

The published contract is part of the deliverable: an endpoint nobody can discover is an endpoint nobody can call.

- **Dependency:** `org.springdoc:springdoc-openapi-starter-webmvc-ui`, 2.x line for Spring Boot 3.x, with an explicit
  version — the Boot BOM does not manage it. The spec is served at `/v3/api-docs`, the UI at `/swagger-ui.html`
  (which redirects to `/swagger-ui/index.html`).
- **Both endpoints are on by default in every profile**, including production. Decide explicitly: turn them off with
  `springdoc.api-docs.enabled=false` / `springdoc.swagger-ui.enabled=false` under `%prod`-equivalent profile
  configuration, or keep them open on purpose because the contract is public.
- **They are also normal URLs**, so the deny-by-default chain of §13 answers 401 for them unless you permit them
  deliberately. Whichever you choose, choose it — do not discover it from a broken Swagger UI.
- **Controllers:** `@Tag` on the class to group the endpoints, `@Operation(summary, description)` on every method.
- **Responses:** document the status codes that belong to that method's contract with `@ApiResponse`, and point every
  error response at the `ProblemDetail` schema from §9 — a documented API whose errors are undocumented is half a
  contract. The errors every endpoint shares (401, 403, 500) are declared once by an `OpenApiCustomizer` bean in
  `config/`, not repeated on each method.
- **Records:** `@Schema(description, example)` adds what cannot be inferred. Do NOT restate constraints that Bean
  Validation already declares — springdoc derives `required`, `maxLength` and friends from `@NotBlank`, `@Size` and
  the rest. Two sources for the same fact drift apart.
- **Security:** declare the authentication scheme (`@SecurityScheme` on a `@Configuration` class in `config/`,
  `@SecurityRequirement` where it applies) so the published contract matches the deny-by-default rule of §13 and
  "try it out" actually works.

*Shape to copy: `references/code-examples.md` §14.*

## Definition of done — check before finishing

- [ ] No Lombok anywhere; DTOs are records with Bean Validation annotations, and `spring-boot-starter-validation` is
      on the classpath
- [ ] Entities are plain JPA with sequence IDs (`seq_<entity_name>`), private fields, hand-written accessors
- [ ] Queries live in a Spring Data repository interface; no `EntityManager` outside `repository/`
- [ ] Service is `@Service`, constructor-injected, `@Transactional` on writes and `readOnly = true` on reads, with no
      self-invocation expected to start a transaction
- [ ] Controller path is `/v1/{resources}` with no verb in the URI and at most two nesting levels; methods follow
      GET/POST/PUT/PATCH/DELETE semantics and the status-code table (409 for conflicts)
- [ ] Controller returns synchronous types and never touches repositories or entities
- [ ] Mapping is explicit (static `from(...)` or service method), no reflection mapper
- [ ] `spring.threads.virtual.enabled=true`, with the Hikari pool sized deliberately
- [ ] Methods under 20 lines, guard clauses instead of nested `if/else`, no `null` returned from services
- [ ] Names reveal intent; no comments restating what the code already says
- [ ] Business failures are domain exceptions handled by the single `@RestControllerAdvice extends
      ResponseEntityExceptionHandler`; no `try/catch` formatting errors in a controller, no stack traces in responses
- [ ] Error responses are RFC 9457 `ProblemDetail` (`type`, `title`, `status`, `detail`, `instance`, `timestamp`)
      built through one shared helper; validation failures list the offending fields; 401/403 use the same shape
- [ ] New starters added to `pom.xml`; code compiles (`./mvnw -q -DskipTests package`) and tests pass (`./mvnw test`)
- [ ] Logging via SLF4J with `{}` placeholders, right level, `Throwable` on `ERROR`, and no PII in any message
- [ ] Every schema change ships as a `V<yyyyMMddHHmmss>__*.sql` migration; no applied migration was edited
- [ ] No secret hardcoded; environment-specific values come from env vars or profiles
- [ ] The security chain ends in `.anyRequest().authenticated()`, and every public path is listed on purpose
- [ ] Paginated endpoints take `Pageable`, cap the page size, allow-list the sort property, and return
      `PageResponse<T>` — never `Page<T>` or a bare `List`
- [ ] Endpoints carry `@Tag`/`@Operation`, documented status codes reference the `ProblemDetail` schema, and records
      document only what validation does not already declare
- [ ] Commit message follows the `conventional-commits` skill
- [ ] No placeholders or TODOs left behind
