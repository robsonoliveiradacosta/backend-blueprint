# Production Concerns — schema, configuration, access

Companion to `SKILL.md` §13. Read the relevant part before touching the database schema, adding configuration, or
exposing an endpoint.

## A. Database migrations (Flyway)

- **The schema is owned by versioned migrations**, never by Hibernate. Set `spring.jpa.hibernate.ddl-auto=none`
  explicitly — the default depends on whether the datasource looks embedded, which is not a decision to leave to
  detection. `update` is forbidden in any environment that holds data you care about.
- Scripts live in `src/main/resources/db/migration` and are named
  **`V<yyyyMMddHHmmss>__snake_case_description.sql`** (e.g. `V20260814093015__create_users_table.sql`). Take the
  timestamp from the moment you write the file — never renumber an existing one.
- **`spring.flyway.out-of-order=true`** is required by that naming: two branches produce interleaved timestamps, so a
  migration merged late carries a version lower than one already applied. Without out-of-order Flyway refuses it; with
  it, the migration applies on merge. The price is that a migration MUST NOT assume another branch's migration ran
  first — each one stands alone or declares its dependency in SQL.
- **Never edit a migration that has been applied anywhere.** Flyway stores a checksum; changing the file breaks
  validation for everyone. Fix forward with a new script.
- Sequences are created here, with `INCREMENT BY` matching the entity's `allocationSize` (§4). If the target engine
  has no sequences (MySQL, for instance), §4 cannot be satisfied — raise it with the user instead of silently
  falling back to `GenerationType.IDENTITY`.
- Flyway runs before the `EntityManagerFactory` is built, so `spring.jpa.hibernate.ddl-auto=validate` is a genuine
  extra check in CI: it fails the boot when the entities and the migrated schema disagree.

## B. Configuration & secrets

- **No secret in the repository.** Every credential, key, or external URL comes from an environment variable with, at
  most, a local-development default: `spring.datasource.password=${DB_PASSWORD:dev-only}`.
- Environment differences belong to profiles — `application-prod.properties`, or a
  `spring.config.activate.on-profile` document in the main file — never to code branching on an "env" string.
- Read configuration through a **typed `@ConfigurationProperties` record** grouped by feature, not `@Value` fields
  scattered across beans. Add `@Validated` so a missing or malformed key fails at startup instead of at the first
  request, and register it with `@ConfigurationPropertiesScan` on the application class (or
  `@EnableConfigurationProperties`).

## C. Access control is deny-by-default

- One `SecurityFilterChain` bean, ending in **`.anyRequest().authenticated()`**. Every public path is listed above
  that line on purpose: an endpoint nobody thought about answers 401 instead of serving data.
- `@EnableMethodSecurity` turns on the `@PreAuthorize` annotations the controllers of §8 carry. Belt and braces: the
  chain decides who may reach the URI, the annotation decides who may run the method.
- **The security filters run before the DispatcherServlet, so the `@RestControllerAdvice` of §9 never sees their
  failures.** Register an `AuthenticationEntryPoint` (401) and an `AccessDeniedHandler` (403) that write the same
  `ProblemDetail` envelope, or your API answers two different error shapes depending on where the request died.
- Actuator is not a REST controller either. Expose deliberately
  (`management.endpoints.web.exposure.include=health,info`) and secure the rest of it in the chain.
- If Swagger UI stays enabled (§14), its paths (`/swagger-ui/**`, `/v3/api-docs/**`) need an explicit rule — permitted
  on purpose, or authenticated on purpose.

## D. Shapes to copy

Migration file — `src/main/resources/db/migration/V20260814093015__create_users_table.sql`. The sequence increment
matches the entity's `allocationSize`:

```sql
CREATE SEQUENCE seq_user INCREMENT BY 50;

CREATE TABLE users (
    id     BIGINT       PRIMARY KEY,
    name   VARCHAR(120) NOT NULL,
    email  VARCHAR(180) NOT NULL UNIQUE
);

CREATE INDEX idx_user_email ON users (email);
```

`src/main/resources/application.properties`:

```properties
# Schema owned by Flyway; timestamp-versioned scripts merged out of order
spring.jpa.hibernate.ddl-auto=none
spring.flyway.locations=classpath:db/migration
spring.flyway.out-of-order=true
spring.flyway.baseline-on-migrate=true

# Blocking code on virtual threads (Java 21+) — the connection pool is the real concurrency ceiling
spring.threads.virtual.enabled=true
spring.datasource.hikari.maximum-pool-size=20

# No lazy loading after the service returns; inserts batched to match allocationSize (§4)
spring.jpa.open-in-view=false
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true

# Paging bounded by configuration, not by hope (the framework default maximum is 2000)
spring.data.web.pageable.default-page-size=20
spring.data.web.pageable.max-page-size=100

# Secrets by environment variable, local default only
spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5432/app}
spring.datasource.username=${DB_USER:app}
spring.datasource.password=${DB_PASSWORD:dev-only}
```

`src/main/resources/application-prod.properties` — the contract and the management surface, closed on purpose:

```properties
springdoc.api-docs.enabled=false
springdoc.swagger-ui.enabled=false
management.endpoints.web.exposure.include=health,info
```

The chain, in `{base.package}.config`:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                       // enables the @PreAuthorize of §8
public class SecurityConfiguration {

    @Bean
    SecurityFilterChain apiFilterChain(HttpSecurity http,
                                       AuthenticationEntryPoint entryPoint,
                                       AccessDeniedHandler accessDeniedHandler) throws Exception {
        return http
                // A stateless API authenticated by a bearer token has no session cookie to forge.
                .csrf(AbstractHttpConfigurer::disable)
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/actuator/health/**").permitAll()
                        .anyRequest().authenticated())          // everything else, on purpose
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
                .exceptionHandling(handling -> handling
                        .authenticationEntryPoint(entryPoint)
                        .accessDeniedHandler(accessDeniedHandler))
                .build();
    }
}
```

The 401 that the advice of §9 cannot reach — the 403 handler is the same shape with `AccessDeniedHandler`:

```java
@Component
public class ProblemAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    public ProblemAuthenticationEntryPoint(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException exception) throws IOException {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.UNAUTHORIZED, "Authentication is required to access this resource.");
        problem.setTitle("Unauthorized");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("timestamp", OffsetDateTime.now().toString());

        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
        objectMapper.writeValue(response.getWriter(), problem);
    }
}
```

Typed configuration — required by absence of a default, optional by type:

```java
@ConfigurationProperties(prefix = "app.billing")
@Validated
public record BillingProperties(
        @NotNull URI endpoint,                        // startup fails if absent
        @DefaultValue("5s") Duration timeout,
        Optional<String> apiKey) {
}
```
