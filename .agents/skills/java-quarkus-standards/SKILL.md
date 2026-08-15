---
name: java-quarkus-standards
description: "Java 21 + Quarkus synchronous REST standards using the Panache Repository pattern. Use whenever generating, refactoring, or reviewing Java code in this project: creating or changing JPA entities, Panache repositories, services, JAX-RS resources, record DTOs, exception mappers and RFC 7807 error payloads, logging, Flyway migrations, configuration or endpoint security, OpenAPI documentation, or QuarkusTest tests; adding a CRUD or a new endpoint; deciding how a feature should be structured; or checking existing code for SOLID, clean-code, and layering violations. Enforces record DTOs (Lombok banned), sequence-generated IDs, repository pattern (no active record), explicit manual mapping (no reflection mappers), /v1/{resources} versioned URIs, constructor injection, and synchronous endpoints with virtual threads."
---

# Java 21 & Quarkus — Coding Standards

You are an expert Java 21+ and Quarkus engineer. Every piece of Java code you generate, refactor, or review in this
codebase MUST follow the architecture below. These rules are not suggestions — do not invent alternative layers,
patterns, or libraries.

Two companion files carry the detail: `references/code-examples.md` has the canonical shape for every layer — read
the entry for the layer you are about to write — and `references/production.md` covers migrations, configuration and
access control. The rules below say what must be true; the references show the exact shape.

## Before writing code

1. **Detect the base package** from existing sources under `src/main/java/` (read the `package` declaration of any
   class). It may differ from the `groupId` in `pom.xml`. If the project has no Java source yet, ask the user for the
   base package before creating the first class. Never assume `com.example`.
2. **Read `pom.xml`** — take the Quarkus version from the BOM (`quarkus.platform.version`) and the Java release from
   `maven.compiler.release`. Never hardcode versions in new code or docs.
3. **Check which extensions are actually present** (`./mvnw dependency:tree`, or read the `<dependencies>` block) —
   never assume the archetype defaults. A fresh blueprint typically carries only REST, Arc and the test harness;
   persistence, validation, OpenAPI, database drivers and the test DSL must be added to `pom.xml` when a feature needs
   them (e.g. `quarkus-hibernate-orm-panache`, `quarkus-hibernate-validator`, `quarkus-jdbc-<db>`,
   `quarkus-smallrye-openapi`, `rest-assured`). Let the BOM manage versions — do not pin one it already provides.
4. **Verify APIs with context7** before using an annotation or method you are not certain about in the Quarkus version
   this project resolved.
5. **Reuse before creating** — check for existing shared classes (pagination wrapper, exception mappers, test helpers)
   instead of duplicating them.

## 1. Package layout

Relative to the detected base package, always use:

```
{base.package}/
├── domain/entity/       # JPA entities  (<Name>Entity)
├── domain/repository/   # PanacheRepository implementations
├── dto/                 # request/response records
├── service/             # business logic + orchestration
├── resource/            # JAX-RS endpoints
└── exception/           # custom exceptions and ExceptionMappers
```

Do not add extra layers (mappers, facades, controllers, `util` dumping grounds) beyond these.

## 2. Modern Java & DTOs

- Target **Java 21+**; prefer records, sealed types, pattern matching, and enhanced switch where they read cleanly.
- Every request and response DTO **MUST** be a `public record`.
- **Lombok is strictly prohibited.** No `@Data`, `@Builder`, `@Getter`, no Lombok dependency in `pom.xml`.
- Validate with Jakarta Bean Validation annotations placed directly on record components.

*Shape to copy: `references/code-examples.md` §2.*

## 3. Execution & threading model

- **Synchronous by default.** Do not introduce Mutiny (`Uni`/`Multi`), `CompletableFuture`, or reactive pipelines
  unless the user explicitly asks for a reactive stream.
- Use Java 21 virtual threads for blocking work: annotate blocking endpoints (or the blocking service method) with
  `@RunOnVirtualThread` (`io.smallrye.common.annotation.RunOnVirtualThread`) to keep throughput without reactive
  complexity.

## 4. Entities — `{base.package}.domain.entity`

- Plain JPA classes. **Never** extend `PanacheEntity`/`PanacheEntityBase` (no active record).
- `@Entity` + `@Table(name = "...")` with snake_case, plural table names.
- Primary keys **MUST** use a database sequence so Hibernate can batch inserts.

- Sequence naming convention: `seq_<entity_name>` (singular, snake_case).
- **`allocationSize` must match the sequence's `INCREMENT BY` in the database.** A sequence created with the SQL
  default (`INCREMENT BY 1`) while Hibernate assumes `allocationSize = 50` hands out ids that collide. Either create it
  as `CREATE SEQUENCE seq_user INCREMENT BY 50` in the migration, or set `allocationSize = 1` and give up batching.
- All fields `private`, exposed through explicit getters/setters — written by hand, never generated by Lombok.
- Enums persisted with `@Enumerated(EnumType.STRING)`; `@ManyToOne(fetch = FetchType.LAZY)` to avoid N+1.

*Shape to copy: `references/code-examples.md` §4.*

## 5. Repositories — `{base.package}.domain.repository`

- `@ApplicationScoped` class implementing `PanacheRepository<XEntity>`.
- **All** HQL/JPQL, filters, sorting and pagination queries live here — never in services or resources.

*Shape to copy: `references/code-examples.md` §5.*

## 6. Entity ↔ DTO mapping

- **No reflection-based mappers** (ModelMapper, Dozer, and similar). They break record immutability and GraalVM native
  image compilation.
- Map explicitly: a static factory on the response record, or a private mapping method in the service.

MapStruct is compile-time and acceptable **only** if the user explicitly asks for it; otherwise write the mapping by
hand.

*Shape to copy: `references/code-examples.md` §6.*

## 7. Services — `{base.package}.service`

- `@ApplicationScoped`, holding all business logic and orchestration.
- **Always constructor injection** — declare collaborators `private final` and take them in the constructor (`@Inject`
  on the constructor is optional with a single constructor). Field injection via `@Inject` is not used.
- Annotate every write/update/delete method with `@Transactional`. Read-only methods stay untransacted.
- Throw the domain exceptions from `{base.package}.exception` (§9) — never return `null` to signal "not found", and
  never build a `Response` inside a service. Turning the exception into HTTP is the mapper's job.

*Shape to copy: `references/code-examples.md` §7.*

## 8. Resources — `{base.package}.resource`

- Use the `quarkus-rest` stack (`quarkus-rest-jackson`) with standard `jakarta.ws.rs.*` annotations.
- The resource **MUST NOT** touch `PanacheRepository` or the `EntityManager` — it delegates to the service.
- The resource **MUST NOT** accept or return JPA entities — records only, in and out.
- Return synchronous types only: `Response`, a DTO, `List<DTO>` or `PageResponse<DTO>`. Never `Uni`, `Multi`, or
  `CompletableFuture`.
- `@Valid` on request bodies; `@QueryParam`/`@DefaultValue` for pagination, sorting and filters.

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

Every non-2xx body is the RFC 7807 payload from §9 — resources never format an error themselves.

### Collection contract

- Collection endpoints accept `page` (0-indexed, default 0), `size` (default 20) and `sort` (`sort=createdAt,desc`;
  ascending when the direction is omitted). Parse `sort` in the service and translate it to Panache `Sort` — the
  resource does not build queries.
- **Validate the sort field against an allow-list** of sortable properties. A field name coming from the client goes
  straight into the query; an unknown one must answer `400`, never surface as a `500` from Hibernate.
- **A paginated endpoint returns `PageResponse<T>`**, never a bare `List` — the client cannot page without the total.
  The service asks the repository for both the page and the count; `List<DTO>` is only for collections that are
  inherently small and unpaginated.
- Clamp `size` to a maximum in the service (100 is a sane default) and floor `page` at 0, so a client cannot request
  the whole table in one call.

*Shape to copy: `references/code-examples.md` §8.*

## 9. Defensive Programming & Global Exception Handling (RFC 7807) — `{base.package}.exception`

- **Input Validation at the Edge:** Validate all incoming requests using Jakarta Validation annotations on request
  `record`s. Never manually write repetitive `if (x == null)` checks when validation annotations apply.
- **Domain-Specific Exceptions:** Create custom unchecked domain exceptions (extending `RuntimeException`) for expected
  business failures (e.g. `EntityNotFoundException`, `DomainConflictException`). Services throw these — they never
  build HTTP responses themselves.
- **Global Exception Mappers:** Implement `@Provider` classes extending `jakarta.ws.rs.ext.ExceptionMapper<T>` to catch
  domain and framework exceptions globally. Never let raw database or system stack traces leak to the REST response.
  One mapper per exception type, all living in this package — never a `try/catch` that formats an error inside a
  resource.
- **Every domain exception has a mapper.** One without a mapper falls into the catch-all and reaches the client as
  `500` — so introducing an exception and introducing its mapper are the same task, not two.
- **RFC 7807 Problem Details:** All error responses MUST return a JSON structure adhering to RFC 7807 containing at
  least `title`, `status`, `detail`, `instance` (URI) and `timestamp`.
- **Validation Failure Mapping:** Intercept `ConstraintViolationException` and return a structured list of invalid
  fields alongside user-friendly messages (HTTP 400 Bad Request).

In the validation mapper, take the field name from the tail of the violation's property path
(`create.request.email` → `email`) — it is not in the message, and the raw path must never reach the client.

Three details the checklist enforces: serve error payloads as `application/problem+json` (the media type RFC 7807
defines for them); keep `type` present — `about:blank` is the value the RFC prescribes when there is no documentation
URI, so callers can rely on the field existing; and emit `timestamp` as ISO-8601, asserting the rendered shape in the
test rather than trusting the default serializer configuration.

Factor the `Response` building into one shared helper so every mapper produces an identical envelope — the point of a
global contract is that a client parses one shape.

*Shape to copy: `references/code-examples.md` §9.*

## 10. Tests

The order of work (test first), the choice between integration and unit tests, and container/mocking setup belong to
the `pragmatic-tdd-quarkus` skill — read it before starting. Conventions that bind here:

- Resources are covered by `@QuarkusTest` + **RestAssured**; every new endpoint gets at least a success case, a
  validation-failure case, and a not-found case.
- Mirror the production package structure under `src/test/java`; name classes `*Test` (`*IT` for native/failsafe runs).

## 11. Code Quality, SOLID & Clean Code

- **Single Responsibility (SRP):** Enforce strict separation of concerns across layers (Resource handles HTTP, Service
  handles domain logic, Repository handles persistence). A class that reaches across two of those responsibilities is
  split, not extended.
- **Constructor Injection (DIP):** Always use constructor-based dependency injection in `@ApplicationScoped` beans.
  Avoid field injection via `@Inject`. Depend on the collaborator you own (the repository), never on an
  `EntityManager` smuggled into a resource.
- **Cyclomatic Complexity & Method Size:** Keep methods short (under 20 lines) and focused. Use early returns (guard
  clauses) to minimize nested `if/else` statements.
- **Null Safety & Intentional Returns:** Never return `null` from service methods. Use `Optional<T>` for potentially
  missing single entities and empty collections (`List.of()`) when no results match.
- **Self-Documenting Code:** Choose clear, intention-revealing names for methods and variables. Avoid obscure
  abbreviations or redundant inline comments that restate what the code clearly does.

When a service method exists to serve an endpoint that must answer 404, it still throws instead of returning an empty
`Optional` — the `Optional` is for callers that can meaningfully handle absence.

*Shape to copy: `references/code-examples.md` §11.*

## 12. Observability & Structured Logging

- **Standard Logger:** Always use `org.jboss.logging.Logger` — it ships with Quarkus core, so there is no
  dependency to declare. Do NOT use `System.out.println` or `java.util.logging`.
- **Field Initialization:** Declare logger instances as
  `private static final Logger LOG = Logger.getLogger(YourClass.class);`.
- **Contextual Logging (No PII):** NEVER log sensitive personal data — passwords, tokens, full request bodies, or
  personal identifiers. Log event names, resource IDs, or entity UUIDs instead.
- **Appropriate Log Levels:**
  - `INFO`: business-relevant milestones (entity created, status updated).
  - `WARN`: handled anomalies or unexpected business conditions.
  - `ERROR`: unhandled system exceptions and infrastructure failures — always pass the `Throwable` instance.
  - `DEBUG`: technical execution details useful during local debugging.
- **Parametrized Messaging:** Use placeholders instead of string concatenation:
  `LOG.infof("Created resource with ID: %s", resourceId);`. The `*f` methods take printf-style `%s`, and each level has
  a `Throwable`-first overload (`LOG.errorf(exception, "...", args)`).

Two rules that keep the logs readable:

- **Log an exception once.** The `ExceptionMapper` of §9 is where a failure is logged — at `ERROR` with the
  `Throwable`, or at `WARN` for expected business failures such as a not-found. Services and resources do not
  log-and-rethrow the same exception on the way up.
- The stack trace goes to the log, never to the response body. §9 governs what the client sees; this section governs
  what the operator sees.

*Shape to copy: `references/code-examples.md` §12.*

## 13. Production Concerns — schema, configuration, access

Three rules that hold everywhere; the detail, the properties and the shapes are in `references/production.md`, which
you must read before changing the schema, adding configuration, or exposing an endpoint.

- **The schema belongs to Flyway, never to Hibernate.** `quarkus.hibernate-orm.database.generation=none` outside dev,
  every change shipped as `V<yyyyMMddHHmmss>__snake_case_description.sql`, `quarkus.flyway.out-of-order=true`, and an
  applied migration is never edited.
- **No secret in the repository.** Credentials and environment-specific values arrive through environment variables
  and profiles, read via a typed `@ConfigMapping`.
- **Access is deny-by-default.** `quarkus.security.jaxrs.deny-unannotated-endpoints=true`, and every resource method
  carries an explicit `@RolesAllowed` or a deliberate `@PermitAll`.

## 14. OpenAPI & Swagger UI

The published contract is part of the deliverable: an endpoint nobody can discover is an endpoint nobody can call.

- **Extension:** `quarkus-smallrye-openapi`. The spec is served at `/q/openapi`, the UI at `/q/swagger-ui`.
- **Swagger UI is included in dev and test only** — that is the Quarkus default, so it needs no configuration.
  Shipping it in a deployed build requires the build-time `quarkus.swagger-ui.always-include=true`; if you do that,
  gate it deliberately (runtime `quarkus.swagger-ui.enabled` per profile, or an auth permission).
- **The UI being absent in production does not hide the spec.** `/q/openapi` is a separate, runtime-enabled endpoint
  that stays open by default in every profile. Decide explicitly: close it with
  `%prod.quarkus.smallrye-openapi.enabled=false`, or keep it open on purpose because the contract is public — and
  then protect it through `quarkus.http.auth.permission.*` if it is not.
- **Resources:** `@Tag` on the class to group the endpoints, `@Operation(summary, description)` on every method.
- **Responses:** document the status codes that belong to that method's contract with `@APIResponse`, and point every
  error response at the `ProblemDetail` schema from §9 — a documented API whose errors are undocumented is half a
  contract. Declare the responses common to the whole resource once at class level rather than repeating 400/401/500
  on each method.
- **Records:** `@Schema(description, example)` adds what cannot be inferred. Do NOT restate constraints that Bean
  Validation already declares — Quarkus derives `required`, `maxLength` and friends from `@NotBlank`, `@Size` and the
  rest. Two sources for the same fact drift apart.
- **Security:** declare the authentication scheme (`@SecurityScheme` on the application class,
  `@SecurityRequirement` where it applies) so the published contract matches the deny-by-default rule of §13 and
  "try it out" actually works.

*Shape to copy: `references/code-examples.md` §14.*

## Definition of done — check before finishing

- [ ] No Lombok anywhere; DTOs are records with Bean Validation annotations
- [ ] Entities are plain JPA with sequence IDs (`seq_<entity_name>`), private fields, hand-written accessors
- [ ] Queries live in an `@ApplicationScoped` `PanacheRepository`; no active record
- [ ] Service is `@ApplicationScoped`, constructor-injected, `@Transactional` on writes
- [ ] Resource path is `/v1/{resources}` with no verb in the URI and at most two nesting levels; methods follow
      GET/POST/PUT/PATCH/DELETE semantics and the status-code table (409 for conflicts)
- [ ] Resource returns synchronous types and never touches repositories or entities
- [ ] Mapping is explicit (static `from(...)` or service method), no reflection mapper
- [ ] Blocking endpoints/services annotated with `@RunOnVirtualThread` where beneficial
- [ ] Methods under 20 lines, guard clauses instead of nested `if/else`, no `null` returned from services
- [ ] Names reveal intent; no comments restating what the code already says
- [ ] Business failures are domain exceptions handled by an `ExceptionMapper`; no `try/catch` formatting errors in a
      resource, no stack traces in responses
- [ ] Error responses are RFC 7807 (`type`, `title`, `status`, `detail`, `instance`, `timestamp`) served as
      `application/problem+json`, built through one shared helper; validation failures list the offending fields
- [ ] New extensions added to `pom.xml`; code compiles (`./mvnw -q -DskipTests package`) and tests pass (`./mvnw test`)
- [ ] Logging via `org.jboss.logging.Logger` with parametrized messages, right level, `Throwable` on `ERROR`, and no
      PII in any message
- [ ] Every schema change ships as a `V<yyyyMMddHHmmss>__*.sql` migration; no applied migration was edited
- [ ] No secret hardcoded; environment-specific values come from env vars or profiles
- [ ] Every endpoint has an explicit `@RolesAllowed` or a deliberate `@PermitAll`
- [ ] Paginated endpoints return `PageResponse<T>` with the total, not a bare `List`
- [ ] Endpoints carry `@Tag`/`@Operation`, documented status codes reference the `ProblemDetail` schema, and records
      document only what validation does not already declare
- [ ] Commit message follows the `conventional-commits` skill
- [ ] No placeholders or TODOs left behind
