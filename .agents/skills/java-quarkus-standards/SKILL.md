---
name: java-quarkus-standards
description: "Java 21 + Quarkus synchronous REST standards using the Panache Repository pattern. Use whenever generating, refactoring, or reviewing Java code in this project: creating or changing JPA entities, Panache repositories, services, JAX-RS resources, record DTOs, exception mappers, or QuarkusTest tests; adding a CRUD or a new endpoint; or deciding how a feature should be structured. Enforces record DTOs (Lombok banned), sequence-generated IDs, repository pattern (no active record), explicit manual mapping (no reflection mappers), /v1/{resources} versioned URIs, and synchronous endpoints with virtual threads."
---

# Java 21 & Quarkus — Coding Standards

You are an expert Java 21+ and Quarkus engineer. Every piece of Java code you generate, refactor, or review in this
codebase MUST follow the architecture below. These rules are not suggestions — do not invent alternative layers,
patterns, or libraries.

## Before writing code

1. **Detect the base package** from existing sources under `src/main/java/` (read the `package` declaration of any
   class). It may differ from the `groupId` in `pom.xml`. If the project has no Java source yet, ask the user for the
   base package before creating the first class. Never assume `com.example`.
2. **Read `pom.xml`** — take the Quarkus version from the BOM (`quarkus.platform.version`) and the Java release from
   `maven.compiler.release`. Never hardcode versions in new code or docs.
3. **Check which extensions are present.** This blueprint starts with `quarkus-rest-jackson`, `quarkus-arc` and
   `quarkus-junit` only. Persistence, validation, OpenAPI, and database drivers must be added to `pom.xml` when a
   feature needs them (e.g. `quarkus-hibernate-orm-panache`, `quarkus-hibernate-validator`, `quarkus-jdbc-postgresql`,
   `quarkus-smallrye-openapi`, `rest-assured` for tests).
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

```java
public record CreateUserRequest(
        @NotBlank(message = "name is required")
        @Size(max = 120)
        String name,

        @NotBlank @Email
        String email
) {}
```

## 3. Execution & threading model

- **Synchronous by default.** Do not introduce Mutiny (`Uni`/`Multi`), `CompletableFuture`, or reactive pipelines
  unless the user explicitly asks for a reactive stream.
- Use Java 21 virtual threads for blocking work: annotate blocking endpoints (or the blocking service method) with
  `@RunOnVirtualThread` (`io.smallrye.common.annotation.RunOnVirtualThread`) to keep throughput without reactive
  complexity.

## 4. Entities — `{base.package}.domain.entity`

- Plain JPA classes. **Never** extend `PanacheEntity`/`PanacheEntityBase` (no active record).
- `@Entity` + `@Table(name = "...")` with snake_case, plural table names.
- Primary keys **MUST** use a database sequence so Hibernate can batch inserts:

```java
@Entity
@Table(name = "users")
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "seq_user")
    @SequenceGenerator(name = "seq_user", sequenceName = "seq_user", allocationSize = 50)
    private Long id;

    @Column(nullable = false, length = 120)
    private String name;

    @Column(nullable = false, unique = true, length = 180)
    private String email;

    protected UserEntity() { // required by JPA
    }

    public UserEntity(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

- Sequence naming convention: `seq_<entity_name>` (singular, snake_case).
- All fields `private`, exposed through explicit getters/setters — written by hand, never generated by Lombok.
- Enums persisted with `@Enumerated(EnumType.STRING)`; `@ManyToOne(fetch = FetchType.LAZY)` to avoid N+1.

## 5. Repositories — `{base.package}.domain.repository`

- `@ApplicationScoped` class implementing `PanacheRepository<XEntity>`.
- **All** HQL/JPQL, filters, sorting and pagination queries live here — never in services or resources.

```java
@ApplicationScoped
public class UserRepository implements PanacheRepository<UserEntity> {

    public Optional<UserEntity> findByEmail(String email) {
        return find("email", email).firstResultOptional();
    }

    public List<UserEntity> search(String term, Page page, Sort sort) {
        return find("lower(name) like lower(?1)", sort, "%" + term + "%").page(page).list();
    }
}
```

## 6. Entity ↔ DTO mapping

- **No reflection-based mappers** (ModelMapper, Dozer, and similar). They break record immutability and GraalVM native
  image compilation.
- Map explicitly: a static factory on the response record, or a private mapping method in the service.

```java
public record UserResponse(Long id, String name, String email) {
    public static UserResponse from(UserEntity entity) {
        return new UserResponse(entity.getId(), entity.getName(), entity.getEmail());
    }
}
```

MapStruct is compile-time and acceptable **only** if the user explicitly asks for it; otherwise write the mapping by
hand.

## 7. Services — `{base.package}.service`

- `@ApplicationScoped`, holding all business logic and orchestration.
- **Constructor injection preferred** (`@Inject` on the constructor is optional with a single constructor); use field
  `@Inject` only when constructor injection is impractical.
- Annotate every write/update/delete method with `@Transactional`. Read-only methods stay untransacted.
- Throw domain exceptions from `{base.package}.exception` (or `jakarta.ws.rs.NotFoundException`) — never return `null`
  to signal "not found".

```java
@ApplicationScoped
public class UserService {

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    public UserResponse findById(Long id) {
        return repository.findByIdOptional(id)
                .map(UserResponse::from)
                .orElseThrow(() -> new NotFoundException("user %d not found".formatted(id)));
    }

    @Transactional
    public UserResponse create(CreateUserRequest request) {
        UserEntity entity = new UserEntity(request.name(), request.email());
        repository.persist(entity);
        return UserResponse.from(entity);
    }
}
```

## 8. Resources — `{base.package}.resource`

- Use the `quarkus-rest` stack (`quarkus-rest-jackson`) with standard `jakarta.ws.rs.*` annotations.
- **URI pattern: `/v1/{resources}`** — explicit version in the path, plural nouns (`/v1/users`, `/v1/orders`). No
  header or query-parameter versioning unless explicitly requested.
- Return synchronous types only: `Response`, a DTO, or `List<DTO>`. Never `Uni`, `Multi`, or `CompletableFuture`.
- The resource **MUST NOT** touch `PanacheRepository` or the `EntityManager` — it delegates to the service.
- The resource **MUST NOT** accept or return JPA entities — records only, in and out.
- `@Valid` on request bodies; `@QueryParam`/`@DefaultValue` for pagination and filters.

```java
@Path("/v1/users")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserResource {

    private final UserService service;

    public UserResource(UserService service) {
        this.service = service;
    }

    @GET
    @RunOnVirtualThread
    public List<UserResponse> list(@QueryParam("page") @DefaultValue("0") int page,
                                   @QueryParam("size") @DefaultValue("20") int size) {
        return service.list(page, size);
    }

    @POST
    @RunOnVirtualThread
    public Response create(@Valid CreateUserRequest request) {
        UserResponse created = service.create(request);
        return Response.status(Response.Status.CREATED).entity(created).build();
    }
}
```

Status codes: 200 OK, 201 Created (with `Location` when useful), 204 No Content on delete, 400 validation, 404 not
found.

## 9. Exceptions — `{base.package}.exception`

- Custom exceptions plus `@Provider ExceptionMapper<T>` implementations that translate them into a consistent error
  record (e.g. `ErrorResponse(String message, List<String> details)`).
- Never leak stack traces or entity internals in the response body.

## 10. Tests

- Integration tests for resources: `@QuarkusTest` + **RestAssured**, asserting status codes and JSON payloads.
- Unit tests for services: `@QuarkusTest` with `@InjectMock` (Quarkus + Mockito) on repositories/collaborators.
- Mirror the production package structure under `src/test/java`; name classes `*Test` (`*IT` for native/failsafe runs).
- Every new endpoint gets at least a success case, a validation-failure case, and a not-found case.

## Definition of done — check before finishing

- [ ] No Lombok anywhere; DTOs are records with Bean Validation annotations
- [ ] Entities are plain JPA with sequence IDs (`seq_<entity_name>`), private fields, hand-written accessors
- [ ] Queries live in an `@ApplicationScoped` `PanacheRepository`; no active record
- [ ] Service is `@ApplicationScoped`, constructor-injected, `@Transactional` on writes
- [ ] Resource path is `/v1/{resources}`, returns synchronous types, never touches repositories or entities
- [ ] Mapping is explicit (static `from(...)` or service method), no reflection mapper
- [ ] Blocking endpoints/services annotated with `@RunOnVirtualThread` where beneficial
- [ ] New extensions added to `pom.xml`; code compiles (`./mvnw -q -DskipTests package`) and tests pass (`./mvnw test`)
- [ ] No placeholders or TODOs left behind
