# LearnHub Java & Spring Boot Backend — Lesson 13

<text color="secondary" size="sm">Spring Security · Authentication vs. Authorization · BCrypt · SecurityFilterChain · CSRF · PostgreSQL Users · Integration Testing</text>

**Stack:** Java 21 · Spring Boot 3.5.x · Spring Security 6.x · PostgreSQL · Spring Data JPA · Flyway · JUnit 5 · MockMvc · Testcontainers · Angular

**Estimated time:** 6–8 hours, including implementation, tests, and troubleshooting.

In Lesson 12, we improved the LearnHub catalog with PostgreSQL search indexes, dynamic filtering, query-plan analysis, and Angular search integration.

But we still have a serious gap: **any HTTP client can potentially create, edit, publish, archive, or delete courses.**

Today we close that gap with Spring Security.

We will keep the catalog public while protecting all course management operations. We'll also prepare a PostgreSQL-backed user model for real registration and login in Lesson 14.

## 1. What you'll learn

By the end of this lesson, you'll understand:

- The difference between authentication and authorization.
- How Spring Security's servlet filter chain works.
- How to configure `SecurityFilterChain` using modern Spring Security APIs.
- How BCrypt password hashing and verification work.
- How to model users and roles in PostgreSQL.
- How to protect course write endpoints while preserving public catalog reads.
- Why CSRF protection matters for browser sessions.
- How to test security behavior with MockMvc and PostgreSQL Testcontainers.
- How Angular should handle protected APIs without storing passwords or session secrets.

### Security behavior we're targeting

| Request | Anonymous | Authenticated student | Instructor | Admin |
|---|---|---|---|---|
| `GET /api/v1/courses` | Allowed | Allowed | Allowed | Allowed |
| `GET /api/v1/courses/{id}` | Allowed | Allowed | Allowed | Allowed |
| `GET /api/v1/course-feed` | Allowed | Allowed | Allowed | Allowed |
| `POST /api/v1/courses` | Denied | Denied | Allowed | Allowed |
| `PUT /api/v1/courses/{id}` | Denied | Denied | Allowed* | Allowed |
| `DELETE /api/v1/courses/{id}` | Denied | Denied | Allowed* | Allowed |
| `POST /api/v1/courses/{id}/publish` | Denied | Denied | Allowed* | Allowed |
| `POST /api/v1/courses/{id}/archive` | Denied | Denied | Allowed* | Allowed |

*For Lesson 13, instructor authorization is role-based only. Actual **course ownership checks** will be implemented in Lesson 15. Until then, an instructor with the correct role could act on another instructor's course. Keep the protected write API out of public production use.*

## 2. Authentication versus authorization

**Authentication** answers: *Who are you?*

**Authorization** answers: *What are you allowed to do?*

For example, an instructor logs in as `alex@example.com`. Authentication verifies the supplied credentials and establishes an authenticated principal.

Authorization then determines whether that principal can create a course, publish a course, or access an administrative endpoint.

Being logged in does not automatically grant permission to edit every course.

### How a Spring Security request flows

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    **Browser or API client**
    <text color="secondary" size="xs">GET /api/v1/courses or POST /api/v1/courses</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring Security FilterChainProxy**
    <text color="secondary" size="xs">Session · CSRF · Authentication · Authorization</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **SecurityFilterChain rules**
    <text color="secondary" size="xs">Public GET? Authenticated? Required role?</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring MVC CourseController**
    <text color="secondary" size="xs">Validation · Service calls · Response DTOs</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    **Transactional services and PostgreSQL**
    <text color="secondary" size="xs">Course lifecycle · Data integrity · Future ownership checks</text>
  </box>
</box>

Security is enforced before a request reaches your controller. This is why CORS configuration is not a substitute for authentication or authorization.

---

## 3. Verify the Lesson 12 baseline

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info

git status
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 12 baseline verification failed."
}
```

Check the existing database and JPA user model:

```powershell
Get-ChildItem src\main\java\com\learnhub -Recurse -Filter *.java |
    Select-String "class AppUserEntity|enum UserRole|interface JpaAppUserRepository"

Get-ChildItem src\main\resources\db\migration |
    Sort-Object Name
```

Lesson 10 introduced instructor accounts, so your database may already have an `app_users` table.

**Do not create a second user table.** We will extend the existing one.

### Planned files

```text
src/main/java/com/learnhub/
├── config/
│   └── SecurityConfig.java
├── security/
│   ├── DatabaseUserDetailsService.java
│   ├── SecurityProblemHandler.java
│   └── SecurityRole.java
├── persistence/
│   ├── AppUserEntity.java           MODIFY
│   └── JpaAppUserRepository.java    MODIFY
└── web/
    └── SecuritySessionController.java

src/main/resources/
└── db/migration/
    └── V6__user_security_credentials.sql

src/test/java/com/learnhub/
├── security/
│   ├── SecurityRulesTest.java
│   ├── PasswordEncodingTest.java
│   └── DatabaseUserDetailsServiceTest.java
└── web/
    └── SecurityHttpContractTest.java
```

We'll add the security dependency, database migration, role mapping, user lookup, filter chain, and tests.

## 4. Add Spring Security dependencies

**File:** `learnhub-backend/pom.xml`

Inside `<dependencies>`, add:

<CodeBlock language="xml" editable>
<!-- Spring Security: authentication and authorization -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- MockMvc security helpers: @WithMockUser, csrf(), etc. -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
</CodeBlock>

Keep the existing web, validation, JPA, PostgreSQL, Flyway, and Testcontainers dependencies.

Spring Boot manages compatible dependency versions through its parent POM; you normally should not add a separate Spring Security version.

Check dependency resolution:

```powershell
mvn dependency:tree "-Dincludes=org.springframework.security"

if ($LASTEXITCODE -ne 0) {
    throw "Spring Security dependency resolution failed."
}
```

### What happens if you run the application now?

Spring Boot's security auto-configuration can secure endpoints by default and provide a generated development password when no application authentication configuration exists.

We will replace that default behavior with our own explicit `SecurityFilterChain`.

---

## 5. Prepare PostgreSQL user credentials

We need a single authoritative user table that supports:

- Unique normalized email addresses.
- A password hash, never a plaintext password.
- A role such as `STUDENT`, `INSTRUCTOR`, or `ADMIN`.
- An enabled/disabled flag.
- The instructor relationship already used by courses.

### 5.1 Inspect the existing schema first

From the backend directory:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "\d app_users"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT column_name, data_type, is_nullable FROM information_schema.columns WHERE table_schema = 'public' AND table_name = 'app_users' ORDER BY ordinal_position;"
```

Adjust the PostgreSQL installation path if your version differs.

For the reference implementation, we assume the existing table has:

```text
app_users
  id              BIGINT PRIMARY KEY
  email           VARCHAR(...)
  display_name    VARCHAR(...)
```

and that `courses.instructor_user_id` references `app_users.id`.

### 5.2 Create Flyway V6

**File:** `src/main/resources/db/migration/V6__user_security_credentials.sql`

Use `V6` only if your previous applied migration is V5.

<CodeBlock language="sql" editable>
-- Lesson 13: extend the existing app_users table.
-- PostgreSQL reference schema.
--
-- No default password hashes are generated.
-- Existing users cannot authenticate until their
-- credentials are provisioned deliberately.

ALTER TABLE app_users
    ADD COLUMN IF NOT EXISTS password_hash VARCHAR(255);

ALTER TABLE app_users
    ADD COLUMN IF NOT EXISTS role VARCHAR(20);

ALTER TABLE app_users
    ADD COLUMN IF NOT EXISTS enabled BOOLEAN NOT NULL DEFAULT TRUE;

-- Existing accounts start as students unless
-- previously assigned a role.
UPDATE app_users
SET role = 'STUDENT'
WHERE role IS NULL;

ALTER TABLE app_users
    ALTER COLUMN role SET NOT NULL;

ALTER TABLE app_users
    ADD CONSTRAINT ck_app_users_role
    CHECK (role IN ('STUDENT', 'INSTRUCTOR', 'ADMIN'));

-- Preserve legacy accounts without passwords.
-- New registration will require a password hash.
CREATE UNIQUE INDEX IF NOT EXISTS ux_app_users_email_lower
    ON app_users (lower(email));
</CodeBlock>

### Two important migration checks

**First:** The constraint name `ck_app_users_role` must not already exist. If Lesson 9 or 10 created it, reuse the existing constraint rather than attempting to add a duplicate.

**Second:** Creating the case-insensitive unique index fails if existing accounts have duplicate emails differing only by case.

Check first:

```sql
SELECT lower(email), COUNT(*)
FROM app_users
GROUP BY lower(email)
HAVING COUNT(*) > 1;
```

Resolve any duplicates intentionally before applying the migration.

### Why is `password_hash` nullable?

Existing instructor accounts may not have passwords yet. A non-null constraint would break the migration or tempt us to assign a shared dummy password.

Instead, legacy accounts remain unable to authenticate until they complete a deliberate credential-provisioning process.

**Never put a shared plaintext password or precomputed demo hash into a production Flyway migration.**

### Verify

After starting Spring Boot and allowing Flyway to run:

```powershell
& $psql -U learnhub_app -h localhost -d learnhub `
  -c "\d app_users"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT installed_rank, version, description, success FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 5;"
```

If your table already has `password_hash`, `role`, or `enabled`, adapt the migration to preserve those existing columns and values.

---

## 6. Model security roles

**File:** `src/main/java/com/learnhub/security/SecurityRole.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

/**
 * Application-wide roles.
 *
 * Stored without ROLE_ in PostgreSQL.
 * Spring Security authorities use ROLE_ prefixes.
 */
public enum SecurityRole {
    STUDENT,
    INSTRUCTOR,
    ADMIN
}
</CodeBlock>

Spring Security distinguishes authorities from roles.

For example:

```text
Database role: INSTRUCTOR
GrantedAuthority: ROLE_INSTRUCTOR
```

Then:

```java
.hasRole("INSTRUCTOR")
```

checks for the authority `ROLE_INSTRUCTOR`.

Do not store `ROLE_ROLE_INSTRUCTOR` or inconsistently mix prefixed and unprefixed values.

## 7. Extend the existing AppUserEntity

**File:** `src/main/java/com/learnhub/persistence/AppUserEntity.java`

Add these fields inside the existing entity, preserving its ID, email, display name, constructors, and instructor relationships:

<CodeBlock language="java" editable>
@jakarta.persistence.Column(
        name = "password_hash",
        length = 255
)
private String passwordHash;

@jakarta.persistence.Enumerated(
        jakarta.persistence.EnumType.STRING
)
@jakarta.persistence.Column(
        name = "role",
        nullable = false,
        length = 20
)
private com.learnhub.security.SecurityRole role =
        com.learnhub.security.SecurityRole.STUDENT;

@jakarta.persistence.Column(
        name = "enabled",
        nullable = false
)
private boolean enabled = true;

public String getPasswordHash() {
    return passwordHash;
}

public com.learnhub.security.SecurityRole getRole() {
    return role;
}

public boolean isEnabled() {
    return enabled;
}

/**
 * Credential provisioning belongs to trusted
 * application code, never a public entity setter.
 */
public void setPasswordHash(String passwordHash) {
    this.passwordHash = passwordHash;
}

public void setRole(
        com.learnhub.security.SecurityRole role
) {
    this.role = java.util.Objects.requireNonNull(role);
}

public void setEnabled(boolean enabled) {
    this.enabled = enabled;
}
</CodeBlock>

These are additions to the existing class, not a second entity definition.

### Important ownership preparation

The role tells us that a user is an instructor.

It does **not** tell us which courses they own.

That later check will compare the authenticated principal's user ID with the course's `instructorAccount.id`.

## 8. Extend the user repository

**File:** `src/main/java/com/learnhub/persistence/JpaAppUserRepository.java`

Preserve existing repository methods and add:

<CodeBlock language="java" editable>
@org.springframework.data.jpa.repository.Query("""
        SELECT u
        FROM AppUserEntity u
        WHERE lower(u.email) = lower(:email)
        """)
java.util.Optional<AppUserEntity> findByEmailIgnoreCase(
        @org.springframework.data.repository.query.Param("email")
        String email
);
</CodeBlock>

This supports case-insensitive login lookup.

It should be backed by the `lower(email)` index added by the migration.

**Do not return password hashes from any REST response DTO.**

## 9. Implement database-backed authentication

Spring Security uses a `UserDetailsService` to load user information by username.

For LearnHub, the username will be the email address.

**File:** `src/main/java/com/learnhub/security/DatabaseUserDetailsService.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.persistence.AppUserEntity;
import com.learnhub.persistence.JpaAppUserRepository;

import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Locale;

/**
 * Loads credentials from PostgreSQL.
 *
 * Authentication never exposes password hashes
 * through a controller or response DTO.
 */
@Service
public class DatabaseUserDetailsService
        implements UserDetailsService {

    private final JpaAppUserRepository users;

    public DatabaseUserDetailsService(
            JpaAppUserRepository users
    ) {
        this.users = users;
    }

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String email) {
        String normalized = email.trim()
                .toLowerCase(Locale.ROOT);

        AppUserEntity user = users
                .findByEmailIgnoreCase(normalized)
                .orElseThrow(() ->
                        new UsernameNotFoundException(
                                "Invalid credentials."
                        )
                );

        // Legacy accounts without provisioned credentials
        // are not permitted to authenticate.
        if (user.getPasswordHash() == null
                || user.getPasswordHash().isBlank()) {
            throw new UsernameNotFoundException(
                    "Invalid credentials."
            );
        }

        return User.withUsername(user.getEmail())
                .password(user.getPasswordHash())
                .authorities(
                        "ROLE_" + user.getRole().name()
                )
                .disabled(!user.isEnabled())
                .build();
    }
}
</CodeBlock>

### Why return Spring Security's `UserDetails`?

It is a security-specific representation containing credentials and authorities.

Your JPA entity remains part of the persistence layer.

That separation avoids making every persistence relationship or mutable entity property part of the authentication principal.

In Lesson 15, we'll introduce a principal that also carries the immutable database user ID for ownership checks.

### Password hash verification

The `UserDetailsService` does not compare plaintext passwords itself.

Spring Security's authentication provider obtains the stored hash and calls a `PasswordEncoder` to verify the supplied password.

---

## 10. Configure BCrypt

BCrypt is an adaptive password hashing algorithm. It generates a salt and includes the algorithm parameters in the stored encoded string.

For example, a BCrypt hash typically begins with a marker such as `$2a$`, `$2b$`, or `$2y$`.

We should never encrypt passwords with a reversible application key or hash them using unsalted SHA-256.

**File:** `src/main/java/com/learnhub/config/SecurityConfig.java`

We'll create the full configuration shortly. Its password encoder bean is:

<CodeBlock language="java" editable>
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);
}
</CodeBlock>

The cost factor `12` is a starting point for this course, not a universal production recommendation. Measure hashing latency on the deployment hardware and adjust it based on your threat model and capacity.

### How verification works

```java
String encoded = passwordEncoder.encode("ExamplePassword123!");

boolean matches = passwordEncoder.matches(
        "ExamplePassword123!",
        encoded
);
```

The encoded value is stored in PostgreSQL.

The original password is never stored.

### Password handling rules

- Never log passwords.
- Never return password hashes through APIs.
- Never place production passwords in source code.
- Never email existing passwords.
- Rate-limit login attempts when we add the login endpoint.
- Use TLS in production.
- Never use the user's email address as their password.

---

## 11. Configure the SecurityFilterChain

We'll use one filter chain with explicit public and protected routes.

For Lesson 13, we'll enable **HTTP Basic authentication as a temporary local testing mechanism**. It lets us verify real PostgreSQL credentials before building the session login workflow in Lesson 14.

We'll keep CSRF enabled and prepare the application for browser sessions.

**File:** `src/main/java/com/learnhub/config/SecurityConfig.java`

<CodeBlock language="java" editable>
package com.learnhub.config;

import com.learnhub.security.DatabaseUserDetailsService;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import org.springframework.http.HttpMethod;

import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;

import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;

import org.springframework.security.core.userdetails.UserDetailsService;

import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public AuthenticationProvider authenticationProvider(
            DatabaseUserDetailsService users,
            PasswordEncoder passwordEncoder
    ) {
        DaoAuthenticationProvider provider =
                new DaoAuthenticationProvider(users);

        provider.setPasswordEncoder(passwordEncoder);

        return provider;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http,
            AuthenticationProvider authenticationProvider
    ) throws Exception {

        http
            .authenticationProvider(authenticationProvider)

            .authorizeHttpRequests(auth -> auth

                // Public catalog and course details.
                .requestMatchers(
                        HttpMethod.GET,
                        "/api/v1/courses",
                        "/api/v1/courses/*",
                        "/api/v1/course-feed",
                        "/api/v1/course-summaries"
                ).permitAll()

                // Public health endpoint.
                .requestMatchers(
                        HttpMethod.GET,
                        "/actuator/health"
                ).permitAll()

                // Instructor/admin course management.
                .requestMatchers(
                        HttpMethod.POST,
                        "/api/v1/courses"
                ).hasAnyRole("INSTRUCTOR", "ADMIN")

                .requestMatchers(
                        HttpMethod.PUT,
                        "/api/v1/courses/*"
                ).hasAnyRole("INSTRUCTOR", "ADMIN")

                .requestMatchers(
                        HttpMethod.DELETE,
                        "/api/v1/courses/*"
                ).hasAnyRole("INSTRUCTOR", "ADMIN")

                .requestMatchers(
                        HttpMethod.POST,
                        "/api/v1/courses/*/publish",
                        "/api/v1/courses/*/archive"
                ).hasAnyRole("INSTRUCTOR", "ADMIN")

                // Do not accidentally expose new endpoints.
                .anyRequest().authenticated()
            )

            // Keep CSRF enabled for browser-session safety.
            .csrf(csrf -> csrf
                .csrfTokenRepository(
                    CookieCsrfTokenRepository.withHttpOnlyFalse()
                )
            )

            // Prepare for session-based login in Lesson 14.
            .sessionManagement(session ->
                session.sessionCreationPolicy(
                    SessionCreationPolicy.IF_REQUIRED
                )
            )

            // Temporary development authentication.
            // Replace with a proper login flow in Lesson 14.
            .httpBasic(Customizer.withDefaults())

            // No default HTML login page for this API.
            .formLogin(form -> form.disable());

        return http.build();
    }
}
</CodeBlock>

### Spring Security version compatibility

The reference code targets the Spring Security APIs supplied by Spring Boot 3.5.x. If your project is on a different major version, check the actual `DaoAuthenticationProvider` constructor and configuration APIs before copying it.

### A deliberate security rule

Notice the final rule:

```java
.anyRequest().authenticated()
```

This is a safer default than:

```java
.anyRequest().permitAll()
```

New endpoints require authentication unless we explicitly mark them public.

However, this fallback grants access to **any authenticated role**. Sensitive new routes must receive their own authorization rules before deployment.

### Method matching matters

A public `GET /api/v1/courses/{id}` does not imply that `PUT` or `DELETE` to the same path is public.

Spring Security evaluates both HTTP method and path.

### Security filter order

The security chain runs before Spring MVC. A request can fail due to authentication, authorization, or CSRF before the controller's validation code runs.

This matters when writing integration tests.

## 12. Understand CSRF before testing writes

CSRF stands for **Cross-Site Request Forgery**.

Suppose an instructor is logged into LearnHub and the browser automatically sends their session cookie with requests. A malicious website may attempt to trigger an unwanted state-changing request using that browser session.

CSRF protection requires an additional unpredictable token for protected state-changing requests.

### Authentication and CSRF solve different problems

| Protection | Question |
|---|---|
| Authentication | Who is making this request? |
| Authorization | Is this principal allowed to perform the action? |
| CSRF | Did a browser-session state-changing request originate through an authorized interaction? |
| CORS | Can browser JavaScript from this origin read or send certain cross-origin requests? |

CORS does not replace CSRF protection.

### Why keep CSRF enabled?

In Lesson 14, we will use browser sessions. Disabling CSRF today would create a dangerous default for that future architecture.

The `CookieCsrfTokenRepository` stores a token in a cookie, typically named `XSRF-TOKEN`. The browser client sends the token in a header such as `X-XSRF-TOKEN`.

The cookie is intentionally readable by same-origin JavaScript when using `withHttpOnlyFalse()`. **The authentication session cookie should remain HttpOnly.**

Spring Security 6 also uses deferred CSRF tokens and BREACH protection. A cookie's raw token and the token exposed as a request attribute may be represented differently; use the framework's supported CSRF integration rather than inventing token transformations.

## 13. Add a CSRF bootstrap endpoint

We need a way for a client to request a CSRF token before making a state-changing call.

**File:** `src/main/java/com/learnhub/web/SecuritySessionController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import org.springframework.security.web.csrf.CsrfToken;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * CSRF token bootstrap for API clients.
 *
 * This is not a login endpoint.
 */
@RestController
@RequestMapping("/api/v1/security")
public class SecuritySessionController {

    @GetMapping("/csrf")
    public CsrfResponse csrf(CsrfToken token) {
        return new CsrfResponse(
                token.getHeaderName(),
                token.getParameterName(),
                token.getToken()
        );
    }

    public record CsrfResponse(
            String headerName,
            String parameterName,
            String token
    ) {
    }
}
</CodeBlock>

Add a public matcher **before** `.anyRequest().authenticated()` in `SecurityConfig`:

```java
.requestMatchers(
        HttpMethod.GET,
        "/api/v1/security/csrf"
).permitAll()
```

This endpoint is safe to call anonymously because a CSRF token is not proof of identity.

The client must still authenticate and have the required role.

### PowerShell: obtain a token

```powershell
$base = "http://localhost:8080"

$session = New-Object Microsoft.PowerShell.Commands.WebRequestSession

$csrf = Invoke-RestMethod `
  -Uri "$base/api/v1/security/csrf" `
  -WebSession $session

$csrf | Format-List
```

For subsequent state-changing requests, send the token and reuse the same web session.

The token is not a replacement for authentication credentials.

---

## 14. Configure safe development CORS

Lesson 11 and 12 introduced development CORS for Angular.

Do **not** leave two competing `WebMvcConfigurer` classes with overlapping CORS rules. Retain one authoritative configuration.

When Spring Security is enabled, also integrate CORS into the security chain:

```java
.cors(Customizer.withDefaults())
```

Add that call in the `http` configuration chain before `.csrf(...)`.

For an Angular dev server on `http://localhost:4200`, a single explicit CORS configuration can be:

**File:** `src/main/java/com/learnhub/config/DevCorsConfig.java`

<CodeBlock language="java" editable>
package com.learnhub.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

/**
 * Local development only.
 *
 * Never allow arbitrary origins with credentials.
 */
@Configuration
@Profile("dev")
public class DevCorsConfig {

    @Bean
    public UrlBasedCorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        config.setAllowedOrigins(
                List.of("http://localhost:4200")
        );

        config.setAllowedMethods(
                List.of(
                        "GET",
                        "POST",
                        "PUT",
                        "DELETE",
                        "OPTIONS"
                )
        );

        config.setAllowedHeaders(
                List.of(
                        "Content-Type",
                        "Authorization",
                        "X-XSRF-TOKEN"
                )
        );

        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source =
                new UrlBasedCorsConfigurationSource();

        source.registerCorsConfiguration(
                "/api/**",
                config
        );

        return source;
    }
}
</CodeBlock>

Remove or disable the older overlapping development CORS configuration only after consolidating its required settings.

### Angular session preparation

For cross-origin development, requests carrying session cookies will need:

```typescript
{ withCredentials: true }
```

But that alone is insufficient for CSRF: Angular's built-in XSRF behavior has specific same-origin assumptions, so a cross-origin dev setup may require a custom interceptor or, preferably, an Angular development proxy that makes API calls appear same-origin.

We'll implement the full browser login and CSRF flow in Lesson 14.

---

## 15. Security tests with MockMvc

We'll start by testing authorization rules independently of the database user lookup.

These tests answer:

- Can anonymous users browse courses?
- Are anonymous writes denied?
- Are students denied course management?
- Are instructors and admins allowed through the security layer?
- Does CSRF protection reject unsafe requests without a token?

### 15.1 Create SecurityRulesTest

**File:** `src/test/java/com/learnhub/security/SecurityRulesTest.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;

import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * Security rules tested through the servlet filter chain.
 * * @WithMockUser tests role authorization; it does not
 * verify database-backed password authentication.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class SecurityRulesTest extends PostgresIntegrationTest {

    @Autowired
    private MockMvc mvc;

    private static final String VALID_COURSE = """
            {
              "title": "Security Test Course",
              "description": "Spring Security authorization tests.",
              "instructor": "Alex Morgan",
              "category": "Web Development",
              "price": 19.99,
              "isFree": false,
              "level": "BEGINNER",
              "rating": 4.5
            }
            """;

    @Test
    void anonymousCanBrowseCatalog() throws Exception {
        mvc.perform(get("/api/v1/courses"))
                .andExpect(status().isOk());
    }

    @Test
    void anonymousCanGetCsrfToken() throws Exception {
        mvc.perform(get("/api/v1/security/csrf"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.token").isString())
                .andExpect(jsonPath("$.headerName")
                        .value("X-XSRF-TOKEN"));
    }

    @Test
    void anonymousCannotCreateCourse() throws Exception {
        mvc.perform(post("/api/v1/courses")
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(VALID_COURSE))
                .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "STUDENT")
    void studentCannotCreateCourse() throws Exception {
        mvc.perform(post("/api/v1/courses")
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(VALID_COURSE))
                .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "INSTRUCTOR")
    void instructorCanCreateCourse() throws Exception {
        mvc.perform(post("/api/v1/courses")
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(VALID_COURSE))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.status").value("DRAFT"));
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanCreateCourse() throws Exception {
        mvc.perform(post("/api/v1/courses")
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(VALID_COURSE))
                .andExpect(status().isCreated());
    }

    @Test
    @WithMockUser(roles = "INSTRUCTOR")
    void missingCsrfTokenIsRejected() throws Exception {
        mvc.perform(post("/api/v1/courses")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(VALID_COURSE))
                .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "STUDENT")
    void studentCannotPublishCourse() throws Exception {
        mvc.perform(post("/api/v1/courses/1/publish")
                        .with(csrf()))
                .andExpect(status().isForbidden());
    }
}
</CodeBlock>

**Expected new tests:** eight.

### Why use `@WithMockUser`?

It creates an authenticated principal with specified authorities without requiring a database user or password.

This is useful for testing **authorization rules**.

It does not prove that BCrypt authentication works or that the database user lookup is correct. We'll test those separately.

### A likely regression from earlier lessons

Lesson 11 and 12 HTTP contract tests may POST without CSRF tokens.

After adding Spring Security, those requests can return `403 Forbidden` before reaching Bean Validation.

For tests that are intended to verify request validation, add:

```java
.with(csrf())
.with(user("test-instructor").roles("INSTRUCTOR"))
```

Imports:

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
```

Keep separate tests that intentionally omit CSRF tokens to verify rejection.

Do not disable security filters across the entire test suite just to restore previous test expectations.

---

## 16. Test BCrypt password encoding

**File:** `src/test/java/com/learnhub/security/PasswordEncodingTest.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import org.junit.jupiter.api.Test;

import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

import static org.junit.jupiter.api.Assertions.*;

class PasswordEncodingTest {

    private final PasswordEncoder encoder =
            new BCryptPasswordEncoder(12);

    @Test
    void encodedPasswordIsNotPlaintext() {
        String raw = "ExamplePassword123!";
        String encoded = encoder.encode(raw);

        assertNotEquals(raw, encoded);
        assertTrue(encoder.matches(raw, encoded));
    }

    @Test
    void wrongPasswordDoesNotMatch() {
        String encoded =
                encoder.encode("CorrectPassword123!");

        assertFalse(encoder.matches(
                "WrongPassword123!",
                encoded
        ));
    }

    @Test
    void samePasswordProducesDifferentSaltedHashes() {
        String raw = "ExamplePassword123!";

        String first = encoder.encode(raw);
        String second = encoder.encode(raw);

        assertNotEquals(first, second);

        assertTrue(encoder.matches(raw, first));
        assertTrue(encoder.matches(raw, second));
    }
}
</CodeBlock>

These three tests verify the essential behavior without asserting a specific BCrypt hash string.

---

## 17. PostgreSQL-backed authentication integration test

Now we'll verify that real database users can authenticate.

We will create a user in the Testcontainers database with a BCrypt password hash and send a real HTTP Basic authorization header through MockMvc.

### Create DatabaseAuthenticationIntegrationTest

**File:** `src/test/java/com/learnhub/security/DatabaseAuthenticationIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;

import org.springframework.http.MediaType;
import org.springframework.jdbc.core.JdbcTemplate;

import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.httpBasic;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * Tests real PostgreSQL credentials and BCrypt.
 *
 * Uses JDBC fixture insertion so the test does not
 * depend on AppUserEntity constructor signatures.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class DatabaseAuthenticationIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired
    private MockMvc mvc;

    @Autowired
    private JdbcTemplate jdbc;

    @Autowired
    private PasswordEncoder encoder;

    private static final String EMAIL =
            "lesson13-instructor@example.test";

    private static final String PASSWORD =
            "LocalTestPassword123!";

    @BeforeEach
    void createInstructor() {
        jdbc.update("""
                INSERT INTO app_users (
                    email,
                    display_name,
                    password_hash,
                    role,
                    enabled
                )
                VALUES (?, ?, ?, ?, ?)
                """,
                EMAIL,
                "Lesson 13 Instructor",
                encoder.encode(PASSWORD),
                "INSTRUCTOR",
                true
        );
    }

    @Test
    void correctDatabaseCredentialsAllowInstructorWrite()
            throws Exception {

        String course = """
                {
                  "title": "Authenticated API Course",
                  "description": "Database authentication test.",
                  "instructor": "Lesson 13 Instructor",
                  "category": "Programming",
                  "price": 29.99,
                  "isFree": false,
                  "level": "BEGINNER",
                  "rating": 4.5
                }
                """;

        mvc.perform(post("/api/v1/courses")
                        .with(httpBasic(EMAIL, PASSWORD))
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(course))
                .andExpect(status().isCreated());
    }

    @Test
    void incorrectPasswordIsRejected() throws Exception {
        mvc.perform(post("/api/v1/courses")
                        .with(httpBasic(
                                EMAIL,
                                "IncorrectPassword"
                        ))
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{}"))
                .andExpect(status().isUnauthorized());
    }

    @Test
    void unknownUserIsRejected() throws Exception {
        mvc.perform(post("/api/v1/courses")
                        .with(httpBasic(
                                "unknown@example.test",
                                PASSWORD
                        ))
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{}"))
                .andExpect(status().isUnauthorized());
    }
}
</CodeBlock>

These three tests verify the real authentication path:

<box border radius="lg" padding={3} gap={2}>
  <row align="center" gap={2}>
    <icon name="user-round" color="secondary"/>
    **HTTP Basic credentials**
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row align="center" gap={2}>
    <icon name="shield-check" color="secondary"/>
    **Spring Security authentication provider**
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row align="center" gap={2}>
    <icon name="database" color="secondary"/>
    **PostgreSQL user lookup**
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row align="center" gap={2}>
    <icon name="key-round" color="secondary"/>
    **BCrypt password verification**
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row align="center" gap={2}>
    <icon name="lock-keyhole" color="secondary"/>
    **ROLE_INSTRUCTOR authorization**
  </row>
</box>

### Test fixture assumptions

This test expects `app_users.id` to be database-generated and the remaining columns to be nullable or defaulted appropriately.

If your Lesson 10 user schema has additional required fields, add them to the fixture.

Also, if your earlier integration tests use a shared PostgreSQL container and parallel execution, use unique test identities or disable parallel execution for database-mutating test classes.

### Test count

| Test class | New tests |
|---|---:|
| `SecurityRulesTest` | 8 |
| `PasswordEncodingTest` | 3 |
| `DatabaseAuthenticationIntegrationTest` | 3 |
| **Total** | **14** |

These are expected additions, not a claim that the tests have already passed.

## 18. Test the security boundary with PowerShell

Start the application:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

Open a second PowerShell terminal:

```powershell
$base = "http://localhost:8080"
```

### 18.1 Public catalog remains accessible

```powershell
Invoke-RestMethod "$base/api/v1/courses?page=0&size=5"
```

Expected: `200 OK`.

### 18.2 Public health endpoint

```powershell
Invoke-RestMethod "$base/actuator/health"
```

Expected: `200 OK`, assuming the application is healthy.

### 18.3 Anonymous course creation is rejected

First obtain a CSRF token, so we can distinguish authentication failure from CSRF failure:

```powershell
$session = New-Object Microsoft.PowerShell.Commands.WebRequestSession

$csrf = Invoke-RestMethod `
  "$base/api/v1/security/csrf" `
  -WebSession $session

$body = @{
    title       = "Unauthorized Course"
    description = "Should not be created."
    instructor  = "Unknown"
    category    = "Programming"
    price       = 0
    isFree      = $true
    level       = "BEGINNER"
    rating      = 4.5
} | ConvertTo-Json

try {
    Invoke-RestMethod `
      -Method Post `
      -Uri "$base/api/v1/courses" `
      -WebSession $session `
      -Headers @{ $csrf.headerName = $csrf.token } `
      -ContentType "application/json" `
      -Body $body
}
catch {
    Write-Host "Expected authentication rejection:"
    $_.Exception.Message
}
```

Expected: `401 Unauthorized` for an anonymous request with a valid CSRF token under the HTTP Basic configuration.

### 18.4 Missing CSRF is rejected

```powershell
try {
    Invoke-RestMethod `
      -Method Post `
      -Uri "$base/api/v1/courses" `
      -ContentType "application/json" `
      -Body $body
}
catch {
    Write-Host "Expected CSRF rejection:"
    $_.Exception.Message
}
```

Expected: `403 Forbidden` because the request lacks a valid CSRF token.

### 18.5 Test a real instructor

To exercise successful HTTP Basic authentication manually, you'll need a local development account with a BCrypt hash and `INSTRUCTOR` role.

The integration test in Section 17 creates such an account safely inside its isolated Testcontainers database.

For the development database, provision a dedicated test account through trusted local code that calls `PasswordEncoder.encode(...)`. Do not paste plaintext passwords into Flyway migrations or store them in your repository.

Once an account is provisioned, PowerShell can send credentials using a secure prompt:

```powershell
$credential = Get-Credential
```

For a controlled local-only HTTP Basic exercise, you can construct the authorization header from the entered credential. **Avoid this workflow over unencrypted networks**; use HTTPS for any nonlocal deployment.

The more important Lesson 13 verification is that the automated integration tests prove the database credential path and role restrictions.

---

## 19. Prepare Angular for authenticated APIs

Our Angular frontend currently uses the public catalog.

That should continue to work without credentials.

For future authenticated instructor operations, use an API client that:

- Never stores a password.
- Never stores a BCrypt hash.
- Does not store authentication secrets in `localStorage`.
- Sends the session cookie through the browser when appropriate.
- Sends CSRF tokens on state-changing requests.
- Handles `401 Unauthorized` and `403 Forbidden` differently.

### HTTP status behavior

| Status | Angular behavior |
|---|---|
| `200 OK` | Render data |
| `400 Bad Request` | Show field or query validation errors |
| `401 Unauthorized` | Prompt the user to sign in |
| `403 Forbidden` | Show insufficient permission or CSRF failure |
| `404 Not Found` | Show missing or unpublished course |
| `409 Conflict` | Show course lifecycle conflict |

### Example Angular error handling

In an Angular service or component:

<CodeBlock language="typescript" editable>
import { HttpErrorResponse } from '@angular/common/http';

export function courseApiErrorMessage(
  error: unknown
): string {
  if (!(error instanceof HttpErrorResponse)) {
    return 'An unexpected error occurred.';
  }

  switch (error.status) {
    case 0:
      return 'Cannot reach the LearnHub server.';

    case 400:
      return 'Please check the submitted course data.';

    case 401:
      return 'Please sign in to continue.';

    case 403:
      return 'You do not have permission to perform this action.';

    case 404:
      return 'This course could not be found.';

    case 409:
      return 'The course has changed state. Refresh and try again.';

    default:
      return 'The request could not be completed.';
  }
}
</CodeBlock>

**File:** `learnhub/src/app/core/utils/course-api-error.ts`

This is a reusable error mapper, not a complete login implementation.

### Why we won't build Angular login today

A browser session login involves more than sending credentials:

- Session creation and regeneration.
- Session cookie settings.
- Login and logout endpoints.
- CSRF token rotation.
- Cross-origin development behavior.
- Login rate limiting.
- Remembering authenticated user state.
- Session expiration and reauthentication.

We'll implement those together in Lesson 14 rather than introducing an incomplete or insecure frontend login.

---

## 20. Common security mistakes

| Mistake | Risk | Correct approach |
|---|---|---|
| `.anyRequest().permitAll()` | New endpoints may be exposed | Default to authentication |
| Disabling CSRF globally | Browser session requests become vulnerable | Keep CSRF for cookie-based auth |
| Treating CORS as security | Non-browser clients bypass CORS | Authenticate and authorize requests |
| Storing plaintext passwords | Credential disclosure | Adaptive password hashing |
| Using client-supplied role as proof | Privilege escalation | Load roles from trusted persistence |
| Checking only `ROLE_INSTRUCTOR` | Instructors can modify each other's courses | Add ownership checks |
| Returning `AppUserEntity` as JSON | Credential and relationship leakage | Explicit safe DTOs |
| Testing only `@WithMockUser` | Database authentication bugs go unnoticed | Add real credential integration tests |
| Disabling security filters in API tests | False confidence in endpoint safety | Test through the actual filter chain |
| Using HTTP Basic in production without TLS | Credentials exposed in transit | Use HTTPS and appropriate session login |

### Additional production requirements

Role-based route rules are only one layer. A production system also needs:

- Resource-level authorization.
- Secure session management.
- Login throttling and monitoring.
- Credential reset and account recovery.
- Audit logging for privileged actions.
- Safe exception messages.
- Secret management.
- HTTPS and secure cookie settings.
- Explicit CORS and CSRF configuration.

We will build these incrementally.

---

## 21. Hands-on exercises

<box gap={3}>
  <row gap={3} align="start">
    <icon name="shield-check" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise A — Authorization matrix**

      Add tests for `PUT`, `DELETE`, `publish`, and `archive`. Verify anonymous users and students cannot perform them, while instructors and admins pass the security layer. Use an existing eligible course fixture when testing successful operations.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="key-round" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise B — Disabled accounts**

      Add a PostgreSQL integration test for a user with `enabled = false`. Confirm that correct credentials still cannot authenticate.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="database" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise C — Case-insensitive email**

      Create a user with a mixed-case email address. Authenticate using a differently cased version and verify the unique lower-email index prevents duplicates.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="lock" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise D — CSRF contract**

      Test the same authenticated POST with and without `.with(csrf())`. Explain why the controller is never reached in the rejected case.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="users" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise E — Ownership design**

      Write pseudocode for an instructor ownership check using the authenticated user ID and `course.instructorAccount.id`. Explain why the instructor display name is not a trustworthy ownership key.
    </box>
  </row>
</box>

## 22. Knowledge quiz

{@body const questions=[{q:"What is the difference between authentication and authorization?",opts:["Authentication verifies identity; authorization checks permissions.","Authentication checks permissions; authorization hashes passwords.","Both mean exactly the same thing."],answer:0,why:"Authentication establishes the principal. Authorization determines what that principal may do."},{q:"Why is BCrypt preferred over a plain SHA-256 hash for password storage?",opts:["It makes passwords reversible.","It uses salted, adaptive password hashing.","It removes the need for TLS."],answer:1,why:"BCrypt uses salts and an adjustable work factor, making large-scale password guessing more expensive."},{q:"Which HTTP status is appropriate for an authenticated student denied instructor-only access?",opts:["200 OK","401 Unauthorized","403 Forbidden"],answer:2,why:"The student is authenticated but lacks the required authority, so 403 is appropriate."},{q:"Does CORS replace CSRF protection for cookie-authenticated browser sessions?",opts:["Yes","No"],answer:1,why:"CORS and CSRF address different browser security concerns."},{q:"Is ROLE_INSTRUCTOR alone enough to ensure an instructor owns a course?",opts:["Yes","No"],answer:1,why:"Role checks grant general capabilities; ownership requires resource-specific authorization."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [revealed,setRevealed]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Lesson 13 — Security fundamentals**
    <text color="secondary" size="xs">5 questions</text>
  </row>
  {#each questions as q,i}
    <box key={i} gap={2}>
      <text weight="medium">{i+1}. {q.q}</text>
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setRevealed(false);}}>
        {#each q.opts as opt,j}
          <radio value={String(j)}>{opt}</radio>
        {/each}
      </radio-group>
      {#if revealed}
        <row align="start" gap={2}>
          <icon name={answers[i]===q.answer?"check-circle":"info"} color={answers[i]===q.answer?"success":"secondary"}/>
          <box flex="1" gap={1}>
            <text weight="medium" size="sm">{answers[i]===q.answer?"Correct":"Correct answer: "+q.opts[q.answer]}</text>
            <text color="secondary" size="sm">{q.why}</text>
          </box>
        </row>
      {/if}
    </box>
    {#if i<questions.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if revealed}
    <box gap={1}>
      <title size="lg" tabularNums>{questions.filter((q,i)=>answers[i]===q.answer).length} / {questions.length} correct</title>
      <text color="secondary" size="xs">Review the explanations and retry any missed questions.</text>
    </box>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==questions.length} onClick={()=>setRevealed(true)}>Check answers</button>
    <button color="secondary" variant="outline" onClick={()=>{setAnswers({});setRevealed(false);}}>Reset</button>
  </row>
</box>

## 23. Lesson 13 completion checklist

{@body const sections=[{name:"Dependencies and database",items:["Verified Lesson 12 baseline","Added Spring Security and security-test dependencies","Inspected existing app_users schema","Applied the next Flyway security migration","Verified normalized email uniqueness"]},{name:"Security implementation",items:["Added SecurityRole","Extended AppUserEntity and repository","Implemented DatabaseUserDetailsService","Configured BCrypt PasswordEncoder","Configured SecurityFilterChain","Protected course write endpoints","Kept catalog reads public","Kept CSRF protection enabled","Added CSRF bootstrap endpoint","Consolidated development CORS"]},{name:"Testing and integration",items:["Added MockMvc authorization tests","Added BCrypt unit tests","Added PostgreSQL authentication tests","Updated older HTTP tests for security","Verified anonymous GET and rejected POST","Ran complete Maven verification","Reviewed Angular 401 and 403 handling"]},{name:"Completion",items:["Completed at least two exercises","Completed the knowledge quiz","Reviewed Git changes and committed"]}]}
{@body const [checked,setChecked]=DIL.useState({})}
{@body const total=sections.reduce((n,s)=>n+s.items.length,0)}
{@body const count=Object.values(checked).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 13 progress**
      <text size="xs" color="secondary">Mark items after local verification.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${count/total*100}%`} radius="full"/>
  </box>
  {#each sections as section,i}
    <box key={i} gap={2}>
      **{section.name}**
      {#each section.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!checked[`${i}-${j}`]} onChange={v=>setChecked(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if count===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 13 checklist completed.**
    </row>
  {/if}
</box>

## 24. Final verification and Git checkpoint

Run the focused tests:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=SecurityRulesTest,PasswordEncodingTest,DatabaseAuthenticationIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 13 security tests failed."
}
```

Then the full backend build:

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 13 backend verification failed."
}

git diff --check
git status
git diff --stat
```

If the Angular error helper was added:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}
```

Commit only after reviewing changes.

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

git add pom.xml src\main src\test
git diff --cached --stat

git commit -m "Secure LearnHub course management with Spring Security"
```

Frontend:

```powershell
cd $HOME\angular-learning\learnhub

git add src\app
git diff --cached --stat

git commit -m "Prepare Angular API error handling for authentication"
```

### Important final security boundary

Lesson 13 establishes **role-based access control**, not complete production authorization.

Until Lesson 15 adds resource ownership checks, a user with `ROLE_INSTRUCTOR` can potentially edit another instructor's course. Additionally, HTTP Basic is included here only for controlled development and authentication testing; the browser session workflow is deferred to Lesson 14.

The examples have not been run against your local repository. In particular, reconcile the existing `AppUserEntity` schema, migration version, Spring Security dependency APIs, and prior HTTP tests before treating the implementation as verified.

---

## 25. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="log-in" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 14 — Registration, Login, and Secure Sessions**

    We'll implement registration with validation and BCrypt hashing, database-backed login, session creation and regeneration, logout, CSRF token handling, safe session cookies, authenticated user DTOs, Angular login forms, and end-to-end authentication tests.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 14: secure user registration and login, BCrypt password hashing, database-backed authentication, session cookies, session fixation protection, CSRF token handling, logout, authenticated user DTOs, Angular standalone login integration, PostgreSQL Testcontainers and MockMvc tests, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 13 with experienced-Java pacing.")}>Start Lesson 14 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 13 deliverable:** A LearnHub backend with public catalog access, protected course management routes, PostgreSQL-backed user authentication, BCrypt password verification, enabled CSRF protection, and a security test suite.

The key engineering distinction is that **authentication establishes identity, authorization grants permissions, and ownership checks restrict access to specific resources**. All three are needed before LearnHub's instructor features are ready for public deployment.