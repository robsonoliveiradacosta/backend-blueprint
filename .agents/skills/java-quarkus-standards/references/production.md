# Production Concerns — schema, configuration, access

Companion to `SKILL.md` §13. Read the relevant part before touching the database schema, adding configuration, or
exposing an endpoint.

## A. Database migrations (Flyway)

- **The schema is owned by versioned migrations**, never by Hibernate. Set
  `quarkus.hibernate-orm.database.generation=none` outside dev; `update` is forbidden in any environment that holds
  data you care about.
- Scripts live in `src/main/resources/db/migration` and are named
  **`V<yyyyMMddHHmmss>__snake_case_description.sql`** (e.g. `V20260814093015__create_users_table.sql`). Take the
  timestamp from the moment you write the file — never renumber an existing one.
- **`quarkus.flyway.out-of-order=true`** is required by that naming: two branches produce interleaved timestamps, so a
  migration merged late carries a version lower than one already applied. Without out-of-order Flyway refuses it; with
  it, the migration applies on merge. The price is that a migration MUST NOT assume another branch's migration ran
  first — each one stands alone or declares its dependency in SQL.
- **Never edit a migration that has been applied anywhere.** Flyway stores a checksum; changing the file breaks
  validation for everyone. Fix forward with a new script.
- Sequences are created here, with `INCREMENT BY` matching the entity's `allocationSize` (§4). If the target engine
  has no sequences (MySQL, for instance), §4 cannot be satisfied — raise it with the user instead of silently
  falling back to `GenerationType.IDENTITY`.
- Typical settings: `migrate-at-start=true`, `baseline-on-migrate=true` on an existing database, and
  `clean-at-start` only under `%test`.

## B. Configuration & secrets

- **No secret in the repository.** Every credential, key, or external URL comes from an environment variable with, at
  most, a local-development default: `quarkus.datasource.password=${DB_PASSWORD:dev-only}`.
- Environment differences belong to profiles (`%dev`, `%test`, `%prod`) in `application.properties`, not to code
  branching on an "env" string.
- Read configuration through a typed `@ConfigMapping` interface grouped by feature, not `@ConfigProperty` fields
  scattered across beans. It fails at startup when a required key is missing, instead of at the first request.

## C. Access control is deny-by-default

- Set **`quarkus.security.jaxrs.deny-unannotated-endpoints=true`**. An endpoint that nobody annotated then answers 401
  instead of serving data — forgetting the annotation becomes a visible error rather than a silent hole.
- Every resource method carries an explicit `@RolesAllowed({...})`, or a deliberate `@PermitAll` for the few public
  ones (login, a public read endpoint). "Public because it has no annotation" is not a decision anyone made.
- This property only governs Jakarta REST endpoints. Management routes such as `/q/health` and `/q/metrics` are not
  JAX-RS resources — restrict those through `quarkus.http.auth.permission.*` (or the management interface) instead.
- `quarkus.security.deny-unannotated-members=true` extends the same default to CDI methods in classes that already
  carry security annotations.

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

```properties
# Schema owned by Flyway; timestamp-versioned scripts merged out of order
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
quarkus.flyway.out-of-order=true
quarkus.flyway.locations=classpath:db/migration
%test.quarkus.flyway.clean-at-start=true

# Secrets by environment variable, local default only
quarkus.datasource.password=${DB_PASSWORD:dev-only}

# Nothing is public unless it says so
quarkus.security.jaxrs.deny-unannotated-endpoints=true

# Swagger UI ships only in dev/test by default (build-time inclusion), but the raw spec
# endpoint is runtime-enabled and open in every profile — close it deliberately.
%prod.quarkus.smallrye-openapi.enabled=false
```

```java
@ConfigMapping(prefix = "app.billing")
public interface BillingConfig {

    URI endpoint();                       // required: startup fails if absent

    @WithDefault("5s")
    Duration timeout();

    Optional<String> apiKey();            // optional by type, not by convention
}
```

Role annotations on a resource are part of the complete shape in `code-examples.md` §8 — read for either role,
destructive operations for `ADMIN`. The only case that section does not show is the deliberate exception:

```java
@POST
@Path("/login")
@PermitAll   // the one endpoint that is public on purpose; everything else answers 401 unannotated
public TokenResponse login(@Valid LoginRequest request) {
    return authService.authenticate(request);
}
```
