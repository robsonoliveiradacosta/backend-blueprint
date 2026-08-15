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

## 5. Repositories — `{base.package}.repository`

An interface — Spring Data writes the implementation. `@Repository` is not needed on it, and the exception
translation it would enable is already applied to Spring Data proxies.

```java
public interface UserRepository extends JpaRepository<UserEntity, Long> {

    Optional<UserEntity> findByEmail(String email);

    boolean existsByEmail(String email);

    // Page and Pageable come from org.springframework.data.domain; the count query is issued for you.
    @Query("select u from UserEntity u where lower(u.name) like lower(concat('%', :term, '%'))")
    Page<UserEntity> search(@Param("term") String term, Pageable pageable);
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
@Service
public class UserService {

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public UserResponse findById(Long id) {
        return repository.findById(id)
                .map(UserResponse::from)
                .orElseThrow(() -> new ResourceNotFoundException("user %d not found".formatted(id)));
    }

    @Transactional
    public UserResponse create(CreateUserRequest request) {
        if (repository.existsByEmail(request.email())) {
            throw new DomainConflictException("email already registered");
        }
        UserEntity saved = repository.save(new UserEntity(request.name(), request.email()));
        return UserResponse.from(saved);
    }
}
```

Assembling a page — the controller hands over the `Pageable`, the service vets it and maps the result:

```java
private static final Set<String> SORTABLE = Set.of("name", "email", "createdAt");

@Transactional(readOnly = true)
public PageResponse<UserResponse> list(Pageable pageable) {
    return PageResponse.of(repository.findAll(vetSort(pageable)).map(UserResponse::from));
}

// An unknown sort property reaches Hibernate as a PropertyReferenceException — a 500 for what is a client
// mistake. InvalidSortException is a domain exception with its own handler (§9) returning 400.
private static Pageable vetSort(Pageable pageable) {
    pageable.getSort().stream()
            .map(Sort.Order::getProperty)
            .filter(property -> !SORTABLE.contains(property))
            .findFirst()
            .ifPresent(property -> {
                throw new InvalidSortException("sort property not allowed: " + property);
            });
    return pageable;
}
```

## 8. Controllers — `{base.package}.controller`

The fully dressed controller: every rule that governs this file at once — routing and status codes (§8), access
control (§13), documentation (§14) and the error contract (§9). Copy this shape whole; the examples in the other
sections are deliberately minimal and never repeat it.

```java
@RestController
@RequestMapping("/v1/users")
@Tag(name = "Users", description = "User registry")
public class UserController {

    private final UserService service;

    public UserController(UserService service) {
        this.service = service;
    }

    @GetMapping
    @PreAuthorize("hasAnyRole('USER', 'ADMIN')")          // nothing is public unless it says so
    @Operation(summary = "List users", description = "Paginated, sortable listing.")
    @ApiResponse(responseCode = "200", description = "Page of users")
    public PageResponse<UserResponse> list(@PageableDefault(size = 20, sort = "name") Pageable pageable) {
        return service.list(pageable);                    // ?page=0&size=20&sort=createdAt,desc
    }

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Create a user", description = "Registers a user and returns its Location.")
    @ApiResponse(responseCode = "201", description = "Created")
    @ApiResponse(responseCode = "409", description = "Email already registered",
            content = @Content(mediaType = "application/problem+json",
                    schema = @Schema(implementation = ProblemDetail.class)))
    public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
        UserResponse created = service.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(created.id())
                .toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Replace a user")
    @ApiResponse(responseCode = "200", description = "Replaced")
    @ApiResponse(responseCode = "404", description = "No such user",
            content = @Content(mediaType = "application/problem+json",
                    schema = @Schema(implementation = ProblemDetail.class)))
    public UserResponse replace(@PathVariable Long id, @Valid @RequestBody UpdateUserRequest request) {
        return service.replace(id, request);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Delete a user")
    @ApiResponse(responseCode = "204", description = "Deleted, no body")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

The wrapper that keeps `PageImpl` out of the contract:

```java
public record PageResponse<T>(List<T> content, int page, int size, long totalElements, int totalPages) {

    public static <T> PageResponse<T> of(Page<T> page) {
        return new PageResponse<>(page.getContent(), page.getNumber(), page.getSize(),
                page.getTotalElements(), page.getTotalPages());
    }
}
```

## 9. Defensive programming & global exception handling (RFC 9457) — `{base.package}.exception`

One advice class, one envelope. `ProblemDetail` is `org.springframework.http.ProblemDetail`; returning it directly is
supported and sets both the status and `application/problem+json` — build a `ResponseEntity` only when you also have
headers to add.

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        log.warn("Not found: {}", ex.getMessage());
        return problem(HttpStatus.NOT_FOUND, "Resource not found",
                ex.getMessage(), request.getRequestURI(), List.of());
    }

    // A business rule violation or a unique-key clash is 409, not 400.
    @ExceptionHandler(DomainConflictException.class)
    public ProblemDetail handleConflict(DomainConflictException ex, HttpServletRequest request) {
        log.warn("Conflict: {}", ex.getMessage());
        return problem(HttpStatus.CONFLICT, "Conflict",
                ex.getMessage(), request.getRequestURI(), List.of());
    }

    // Without this, the catch-all below turns a method-security denial into a 500.
    @ExceptionHandler(AccessDeniedException.class)
    public ProblemDetail handleAccessDenied(AccessDeniedException ex, HttpServletRequest request) {
        log.warn("Access denied on {}", request.getRequestURI());
        return problem(HttpStatus.FORBIDDEN, "Forbidden",
                "You are not allowed to perform this operation.", request.getRequestURI(), List.of());
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex, HttpServletRequest request) {
        // The operator gets the stack trace; the client gets a problem detail without internals.
        log.error("Unhandled failure on {}", request.getRequestURI(), ex);
        return problem(HttpStatus.INTERNAL_SERVER_ERROR, "Internal server error",
                "The request could not be completed.", request.getRequestURI(), List.of());
    }

    // Inherited from ResponseEntityExceptionHandler: @Valid failures arrive here, not in an @ExceptionHandler.
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex,
                                                                  HttpHeaders headers,
                                                                  HttpStatusCode status,
                                                                  WebRequest request) {
        List<FieldErrorDetail> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(error -> new FieldErrorDetail(error.getField(), error.getDefaultMessage()))
                .toList();
        ProblemDetail body = problem(HttpStatus.BAD_REQUEST, "Validation failed",
                "One or more fields are invalid.", path(request), errors);
        return new ResponseEntity<>(body, HttpStatus.BAD_REQUEST);
    }

    // The single envelope. `type` defaults to about:blank — set it only when a documentation URI exists.
    private static ProblemDetail problem(HttpStatus status, String title, String detail,
                                         String instance, List<FieldErrorDetail> errors) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(status, detail);
        problem.setTitle(title);
        problem.setInstance(URI.create(instance));
        problem.setProperty("timestamp", OffsetDateTime.now().toString());
        if (!errors.isEmpty()) {
            problem.setProperty("errors", errors);
        }
        return problem;
    }

    private static String path(WebRequest request) {
        return ((ServletWebRequest) request).getRequest().getRequestURI();
    }

    public record FieldErrorDetail(String field, String message) {}
}
```

The 401 and 403 raised by the security filters never reach this class — see `production.md` §C for the entry point
and access-denied handler that reuse the same envelope.

## 11. Code quality, SOLID & clean code

```java
// Guard clause instead of nesting; Optional instead of null.
@Transactional(readOnly = true)
public Optional<UserResponse> findByEmail(String email) {
    if (email == null || email.isBlank()) {
        return Optional.empty();
    }
    return repository.findByEmail(email).map(UserResponse::from);
}
```

## 12. Observability & structured logging

```java
@Service
public class UserService {

    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public UserResponse create(CreateUserRequest request) {
        UserEntity saved = repository.save(new UserEntity(request.name(), request.email()));
        log.info("User created with id {}", saved.getId());   // id, never the email or the body
        return UserResponse.from(saved);
    }
}
```

The `Throwable` is the last argument and gets no placeholder — `log.error("Unhandled failure on {}", path, ex)` in
the advice above prints the path and the full stack trace.

## 14. OpenAPI & Swagger UI

The annotated controller is the §8 shape — it already carries `@Tag`, `@Operation` and the per-method
`@ApiResponse`. What follows is what does not live on the controller class.

```java
// @Schema carries only what validation cannot express: meaning and an example.
public record CreateUserRequest(
        @NotBlank @Size(max = 120)
        @Schema(description = "Display name", example = "Ada Lovelace")
        String name,

        @NotBlank @Email
        @Schema(description = "Unique login address", example = "ada@example.com")
        String email
) {}
```

The document-wide definition and the scheme the deny-by-default rule (§13) implies, in `{base.package}.config`:

```java
@Configuration
@OpenAPIDefinition(info = @Info(title = "Users API", version = "v1"))
@SecurityScheme(name = "bearer", type = SecuritySchemeType.HTTP, scheme = "bearer", bearerFormat = "JWT")
public class OpenApiConfiguration {

    // The errors every endpoint shares, declared once instead of on every method. The $ref resolves because
    // at least one method documents ProblemDetail explicitly (§8), which is what puts it in components/schemas.
    @Bean
    OpenApiCustomizer commonErrorResponses() {
        Content problem = new Content().addMediaType("application/problem+json",
                new MediaType().schema(new Schema<>().$ref("#/components/schemas/ProblemDetail")));

        return openApi -> openApi.getPaths().values().stream()
                .flatMap(pathItem -> pathItem.readOperations().stream())
                .forEach(operation -> operation.getResponses()
                        .addApiResponse("401", new ApiResponse().description("Missing or invalid credentials"))
                        .addApiResponse("403", new ApiResponse().description("Authenticated without the required role"))
                        .addApiResponse("500", new ApiResponse().description("Unexpected failure").content(problem)));
    }
}
```

`Content`, `MediaType`, `Schema` and `ApiResponse` here are the **models** (`io.swagger.v3.oas.models.*`), not the
annotations of the same name used on the controller (`io.swagger.v3.oas.annotations.*`). Importing the wrong pair is
the usual reason this class does not compile.
