# Code Examples — canonical shapes per section

Companion to `SKILL.md`. Each heading matches the numbered section of the skill that mandates it. Copy the shape,
rename to the entity at hand, and keep every rule from the skill — these examples are not a licence to improvise.

## 2. Modern Java & DTOs

```java
public record CreateUserRequest(
        @NotBlank(message = "name is required")
        @Size(max = 120)
        String name,

        @NotBlank @Email
        String email
) {}
```

## 4. Entities — `{base.package}.domain.entity`

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

## 5. Repositories — `{base.package}.domain.repository`

```java
@ApplicationScoped
public class UserRepository implements PanacheRepository<UserEntity> {

    public Optional<UserEntity> findByEmail(String email) {
        return find("email", email).firstResultOptional();
    }

    // Page and Sort come from io.quarkus.panache.common — not from Spring Data.
    public List<UserEntity> search(String term, Page page, Sort sort) {
        return find("lower(name) like lower(?1)", sort, "%" + term + "%").page(page).list();
    }
}
```

## 6. Entity ↔ DTO mapping

```java
public record UserResponse(Long id, String name, String email) {
    public static UserResponse from(UserEntity entity) {
        return new UserResponse(entity.getId(), entity.getName(), entity.getEmail());
    }
}
```

## 7. Services — `{base.package}.service`

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
                .orElseThrow(() -> new EntityNotFoundException("user %d not found".formatted(id)));
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
        return Response.created(URI.create("/v1/users/" + created.id()))
                .entity(created)
                .build();
    }
}
```

## 9. Defensive Programming & Global Exception Handling (RFC 7807) — `{base.package}.exception`

```java
public record ProblemDetail(
        String type,        // "about:blank" unless a documentation URI exists
        String title,
        int status,
        String detail,
        String instance,
        OffsetDateTime timestamp,
        List<FieldError> errors) {   // populated only for validation failures

    public record FieldError(String field, String message) {}
}

@Provider
public class EntityNotFoundExceptionMapper implements ExceptionMapper<EntityNotFoundException> {

    @Context   // JAX-RS context injection, not CDI field injection — this is the idiom for providers
    UriInfo uriInfo;

    @Override
    public Response toResponse(EntityNotFoundException exception) {
        ProblemDetail problem = new ProblemDetail(
                "about:blank",
                "Resource not found",
                Response.Status.NOT_FOUND.getStatusCode(),
                exception.getMessage(),
                uriInfo.getRequestUri().getPath(),
                OffsetDateTime.now(),
                List.of());
        return problemResponse(problem);
    }
}
```

```java
@Provider
public class ConstraintViolationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(ConstraintViolationException exception) {
        List<FieldError> errors = exception.getConstraintViolations().stream()
                .map(violation -> new FieldError(lastNode(violation.getPropertyPath()), violation.getMessage()))
                .toList();
        // ... build the ProblemDetail with status 400, title "Validation failed" and these errors
    }

    private static String lastNode(Path path) {
        String full = path.toString();
        return full.substring(full.lastIndexOf('.') + 1);
    }
}
```

## 11. Code Quality, SOLID & Clean Code

```java
// Guard clause instead of nesting; Optional instead of null.
public Optional<UserResponse> findByEmail(String email) {
    if (email == null || email.isBlank()) {
        return Optional.empty();
    }
    return repository.findByEmail(email).map(UserResponse::from);
}
```

## 12. Observability & Structured Logging

```java
@ApplicationScoped
public class UserService {

    private static final Logger LOG = Logger.getLogger(UserService.class);

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public UserResponse create(CreateUserRequest request) {
        UserEntity entity = new UserEntity(request.name(), request.email());
        repository.persist(entity);
        LOG.infof("User created with id %s", entity.getId());   // id, never the email or the body
        return UserResponse.from(entity);
    }
}
```

```java
@Provider
public class UnhandledExceptionMapper implements ExceptionMapper<Exception> {

    private static final Logger LOG = Logger.getLogger(UnhandledExceptionMapper.class);

    @Context
    UriInfo uriInfo;

    @Override
    public Response toResponse(Exception exception) {
        // The operator gets the stack trace; the client gets a problem detail without internals.
        LOG.errorf(exception, "Unhandled failure on %s", uriInfo.getRequestUri().getPath());
        ProblemDetail problem = new ProblemDetail(
                "about:blank",
                "Internal server error",
                Response.Status.INTERNAL_SERVER_ERROR.getStatusCode(),
                "The request could not be completed.",
                uriInfo.getRequestUri().getPath(),
                OffsetDateTime.now(),
                List.of());
        return problemResponse(problem);
    }
}
```

