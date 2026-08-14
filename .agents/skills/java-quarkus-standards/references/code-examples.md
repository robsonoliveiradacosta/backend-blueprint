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

// A full replacement (PUT) carries every mutable field; a PATCH record would use Optional<T> components.
public record UpdateUserRequest(
        @NotBlank @Size(max = 120) String name,
        @NotBlank @Email String email
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
    public List<UserEntity> findPage(Page page, Sort sort) {
        return findAll(sort).page(page).list();
    }

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

Assembling a page — the clamp lives in the service, the counting in the repository:

```java
private static final Set<String> SORTABLE = Set.of("name", "email", "createdAt");

public PageResponse<UserResponse> list(int page, int size, String sort) {
    int safeSize = Math.clamp(size, 1, 100);
    int safePage = Math.max(page, 0);

    List<UserResponse> content = repository.findPage(Page.of(safePage, safeSize), parseSort(sort))
            .stream()
            .map(UserResponse::from)
            .toList();

    return PageResponse.of(content, safePage, safeSize, repository.count());
}

// "createdAt,desc" -> Sort. InvalidSortException is a domain exception with its own mapper (§9) returning 400 —
// an unknown field must never surface as a 500 from the query.
private static Sort parseSort(String sort) {
    String[] parts = sort.split(",", 2);
    String field = parts[0].trim();
    if (!SORTABLE.contains(field)) {
        throw new InvalidSortException("sort field not allowed: " + field);
    }
    return parts.length > 1 && "desc".equalsIgnoreCase(parts[1].trim())
            ? Sort.by(field).descending()
            : Sort.by(field).ascending();
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
    @RolesAllowed({"USER", "ADMIN"})   // deny-by-default is on (§13): no method goes unannotated
    @RunOnVirtualThread
    public PageResponse<UserResponse> list(@QueryParam("page") @DefaultValue("0") int page,
                                           @QueryParam("size") @DefaultValue("20") int size,
                                           @QueryParam("sort") @DefaultValue("name") String sort) {
        return service.list(page, size, sort);   // "name" or "createdAt,desc" — parsed in the service
    }

    @POST
    @RolesAllowed("ADMIN")
    @RunOnVirtualThread
    public Response create(@Valid CreateUserRequest request) {
        UserResponse created = service.create(request);
        return Response.created(URI.create("/v1/users/" + created.id()))
                .entity(created)
                .build();
    }

    @PUT
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    @RunOnVirtualThread
    public UserResponse replace(@PathParam("id") Long id, @Valid UpdateUserRequest request) {
        return service.replace(id, request);   // 200 with the resulting resource
    }

    @DELETE
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    @RunOnVirtualThread
    public Response delete(@PathParam("id") Long id) {
        service.delete(id);
        return Response.noContent().build();   // 204, no body
    }
}
```

```java
public record PageResponse<T>(List<T> content, int page, int size, long totalElements, int totalPages) {

    public static <T> PageResponse<T> of(List<T> content, int page, int size, long totalElements) {
        int totalPages = size == 0 ? 0 : (int) Math.ceil((double) totalElements / size);
        return new PageResponse<>(content, page, size, totalElements, totalPages);
    }
}
```

## 9. Defensive Programming & Global Exception Handling (RFC 7807) — `{base.package}.exception`

The payload and the single shared envelope every mapper reuses:

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

public abstract class AbstractProblemMapper {

    @Context   // JAX-RS context injection, not CDI field injection — this is the idiom for providers
    UriInfo uriInfo;

    protected Response problem(Response.Status status, String title, String detail,
                               List<ProblemDetail.FieldError> errors) {
        ProblemDetail problem = new ProblemDetail(
                "about:blank",
                title,
                status.getStatusCode(),
                detail,
                uriInfo.getRequestUri().getPath(),
                OffsetDateTime.now(),
                errors);
        return Response.status(status)
                .type("application/problem+json")
                .entity(problem)
                .build();
    }
}
```

A business failure the API expects: `WARN`, no stack trace, message safe to expose.

```java
@Provider
public class EntityNotFoundExceptionMapper extends AbstractProblemMapper
        implements ExceptionMapper<EntityNotFoundException> {

    private static final Logger LOG = Logger.getLogger(EntityNotFoundExceptionMapper.class);

    @Override
    public Response toResponse(EntityNotFoundException exception) {
        LOG.warnf("Not found: %s", exception.getMessage());
        return problem(Response.Status.NOT_FOUND, "Resource not found", exception.getMessage(), List.of());
    }
}
```

A business rule violation or a unique-key clash is `409`, not `400`:

```java
@Provider
public class DomainConflictExceptionMapper extends AbstractProblemMapper
        implements ExceptionMapper<DomainConflictException> {

    private static final Logger LOG = Logger.getLogger(DomainConflictExceptionMapper.class);

    @Override
    public Response toResponse(DomainConflictException exception) {
        LOG.warnf("Conflict: %s", exception.getMessage());
        return problem(Response.Status.CONFLICT, "Conflict", exception.getMessage(), List.of());
    }
}
```

Validation: the field name is the tail of the violation's property path, never the raw path.

```java
@Provider
public class ConstraintViolationExceptionMapper extends AbstractProblemMapper
        implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(ConstraintViolationException exception) {
        List<ProblemDetail.FieldError> errors = exception.getConstraintViolations().stream()
                .map(violation -> new ProblemDetail.FieldError(
                        lastNode(violation.getPropertyPath()), violation.getMessage()))
                .toList();
        return problem(Response.Status.BAD_REQUEST, "Validation failed",
                "One or more fields are invalid.", errors);
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
public class UnhandledExceptionMapper extends AbstractProblemMapper implements ExceptionMapper<Exception> {

    private static final Logger LOG = Logger.getLogger(UnhandledExceptionMapper.class);

    @Override
    public Response toResponse(Exception exception) {
        // The operator gets the stack trace; the client gets a problem detail without internals.
        LOG.errorf(exception, "Unhandled failure on %s", uriInfo.getRequestUri().getPath());
        return problem(Response.Status.INTERNAL_SERVER_ERROR, "Internal server error",
                "The request could not be completed.", List.of());
    }
}
```
