# LearnHub Java & Spring Boot Backend — Lesson 14

<text color="secondary" size="sm">Secure Registration · Session Login · BCrypt · CSRF · Logout · Angular Authentication · PostgreSQL Tests</text>

**Stack:** Java 21 · Spring Boot 3.5.x · Spring Security 6.x · PostgreSQL · Spring Data JPA · Flyway · MockMvc · Testcontainers · Angular standalone components

**Estimated time:** 7–10 hours, including implementation, testing, and Angular integration.

In Lesson 13, we introduced Spring Security, protected course-management routes, configured BCrypt, and connected authentication to PostgreSQL.

Today we'll implement a real browser-oriented authentication flow:

**Register → Sign in → Establish a server-side session → Access protected APIs → Sign out.**

We'll preserve the public course catalog and move away from the temporary HTTP Basic workflow.

## 1. Learning outcomes

By the end of this lesson, you should be able to:

1. Register users without exposing or storing plaintext passwords.
2. Authenticate credentials using Spring Security and BCrypt.
3. Establish an authenticated server-side HTTP session.
4. Understand session fixation protection and session cookie flags.
5. Handle CSRF tokens correctly during login and logout.
6. Return safe authenticated-user DTOs.
7. Build Angular standalone registration and login forms.
8. Verify authentication using real PostgreSQL and MockMvc.
9. Distinguish `401`, `403`, and validation errors.
10. Explain why session authentication still needs resource ownership checks.

## 2. The target authentication architecture

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" padding={3} radius="lg" align="center" gap={1}>
    **Angular LearnHub**
    <text color="secondary" size="xs">Registration · Login · Profile · Logout</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring Security filter chain**
    <text color="secondary" size="xs">CSRF validation · Session authentication · Role authorization</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **AuthController + AuthService**
    <text color="secondary" size="xs">Register · Login · Current user · Logout</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box background="surface-secondary" padding={3} radius="lg" align="center" gap={1}>
    <icon name="database" size="lg"/>
    **PostgreSQL**
    <text color="secondary" size="xs">app_users · Unique email · BCrypt hash · Role · Enabled</text>
  </box>
</box>

### The HTTP contract

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
| `GET` | `/api/v1/security/csrf` | Obtain CSRF token | Public |
| `POST` | `/api/v1/auth/register` | Create student account | Public + CSRF |
| `POST` | `/api/v1/auth/login` | Authenticate and create session | Public + CSRF |
| `GET` | `/api/v1/auth/me` | Return current user | Authenticated |
| `POST` | `/api/v1/auth/logout` | Invalidate session | Authenticated + CSRF |
| `GET` | `/api/v1/courses` | Browse catalog | Public |
| `POST` | `/api/v1/courses` | Create course | Instructor/Admin + CSRF |

**Registration creates only `STUDENT` accounts.** A client must never be allowed to submit `role: "ADMIN"` or `role: "INSTRUCTOR"` and grant itself privileges.

Instructor promotion will require a trusted administrative workflow in Lesson 15.

---

## 3. Verify the Lesson 13 baseline

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info

git status
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 13 baseline verification failed."
}
```

Check the existing security files:

```powershell
Get-Content src\main\java\com\learnhub\config\SecurityConfig.java

Get-Content src\main\java\com\learnhub\security\DatabaseUserDetailsService.java

Get-ChildItem src\main\resources\db\migration |
    Sort-Object Name
```

We will build on Lesson 13 rather than creating a second security filter chain.

### Planned files

```text
learnhub-backend/
└── src/
    ├── main/
    │   ├── java/com/learnhub/
    │   │   ├── config/
    │   │   │   └── SecurityConfig.java             MODIFY
    │   │   ├── security/
    │   │   │   ├── LearnHubPrincipal.java          NEW
    │   │   │   ├── DatabaseUserDetailsService.java MODIFY
    │   │   │   ├── AuthService.java                 NEW
    │   │   │   └── AuthController.java              NEW
    │   │   ├── web/dto/
    │   │   │   ├── RegisterRequest.java             NEW
    │   │   │   ├── LoginRequest.java                NEW
    │   │   │   └── AuthUserResponse.java            NEW
    │   │   └── persistence/
    │   │       └── AppUserEntity.java               MODIFY
    │   └── resources/
    │       └── application.yml                     MODIFY
    └── test/java/com/learnhub/
        └── security/
            ├── AuthApiIntegrationTest.java         NEW
            └── SessionSecurityTest.java            NEW

learnhub/
└── src/app/
    ├── core/auth/
    │   ├── auth.models.ts                          NEW
    │   ├── auth.service.ts                         NEW
    │   └── csrf.interceptor.ts                     NEW
    └── features/auth/
        ├── login-page.component.ts                 NEW
        └── register-page.component.ts              NEW
```

We'll also configure the Angular development proxy to simplify same-origin session and CSRF behavior.

## 4. Authentication design decisions

Before writing code, establish five rules.

| Concern | Lesson 14 decision |
|---|---|
| Password storage | BCrypt hashes, never plaintext |
| Authentication | Spring Security `AuthenticationManager` |
| Login persistence | Server-side `HttpSession` |
| Browser credentials | HttpOnly session cookie |
| CSRF | Required for registration, login, logout, and course writes |

We will use a **same-origin Angular development proxy**. The browser will call `/api/v1/...` on `localhost:4200`, and Angular's development server will forward the request to Spring Boot on `localhost:8080`.

This avoids most cross-origin cookie problems and lets Angular's built-in XSRF integration work naturally.

### Important correction from Lesson 13

The previous lesson used HTTP Basic as a temporary learning mechanism. We will remove it from the final security configuration.

Also, manually authenticating with an `AuthenticationManager` does **not** automatically persist the resulting authentication in an HTTP session. We must explicitly save the `SecurityContext`.

We must also protect against session fixation by changing the session ID during successful login.

---

## 5. Create a principal containing the database user ID

In Lesson 13, `DatabaseUserDetailsService` returned Spring Security's standard `User`. That gives us the username and authorities, but not the LearnHub database user ID.

We'll need that ID for ownership authorization in Lesson 15.

**Create:** `src/main/java/com/learnhub/security/LearnHubPrincipal.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.io.Serial;
import java.io.Serializable;
import java.util.Collection;
import java.util.List;

/**
 * Immutable security identity.
 *
 * Never serialize this object to an HTTP response.
 * It contains the stored password hash.
 */
public record LearnHubPrincipal(
        Long id,
        String username,
        String password,
        SecurityRole role,
        boolean enabled
) implements UserDetails, Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    @Override
    public Collection<? extends GrantedAuthority>
    getAuthorities() {
        return List.of(
                (GrantedAuthority)
                    () -> "ROLE_" + role.name()
        );
    }

    @Override
    public boolean isEnabled() {
        return enabled;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }
}
</CodeBlock>

A `record` is useful here because the identity data is immutable after authentication.

**Production consideration:** A serialized session principal can contain the password hash. For distributed session storage, consider a minimal principal without the hash after authentication, along with a strategy for refreshing roles and disabled-account status. A stored session also does not automatically notice a later role change or account disablement.

### Replace DatabaseUserDetailsService

**File:** `src/main/java/com/learnhub/security/DatabaseUserDetailsService.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.persistence.AppUserEntity;
import com.learnhub.persistence.JpaAppUserRepository;

import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Locale;

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
        if (email == null || email.isBlank()) {
            throw new UsernameNotFoundException(
                    "Invalid credentials"
            );
        }

        String normalized = email.trim()
                .toLowerCase(Locale.ROOT);

        AppUserEntity user = users
                .findByEmailIgnoreCase(normalized)
                .orElseThrow(() ->
                        new UsernameNotFoundException(
                                "Invalid credentials"
                        )
                );

        if (user.getPasswordHash() == null
                || user.getPasswordHash().isBlank()) {
            throw new UsernameNotFoundException(
                    "Invalid credentials"
            );
        }

        return new LearnHubPrincipal(
                user.getId(),
                user.getEmail(),
                user.getPasswordHash(),
                user.getRole(),
                user.isEnabled()
        );
    }
}
</CodeBlock>

The email lookup remains case-insensitive. The database user ID becomes the stable identity used for future ownership checks.

---

## 6. Registration and login DTOs

Create the directory:

```powershell
New-Item -ItemType Directory -Force `
  src\main\java\com\learnhub\web\dto | Out-Null
```

### 6.1 RegisterRequest

**Create:** `src/main/java/com/learnhub/web/dto/RegisterRequest.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record RegisterRequest(
        @NotBlank
        @Size(max = 100)
        String displayName,

        @NotBlank
        @Email
        @Size(max = 254)
        String email,

        @NotBlank
        @Size(min = 12, max = 128)
        String password
) {
}
</CodeBlock>

The 12-character minimum is a course policy, not a universal security standard. A production system should support long passphrases, breached-password screening, and abuse prevention. BCrypt also has an effective input-length limit of 72 bytes, so password handling must explicitly address that rather than silently truncating long UTF-8 passwords. We will enforce this in the service.

### 6.2 LoginRequest

**Create:** `src/main/java/com/learnhub/web/dto/LoginRequest.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import jakarta.validation.constraints.NotBlank;

public record LoginRequest(
        @NotBlank String email,
        @NotBlank String password
) {
}
</CodeBlock>

### 6.3 AuthUserResponse

**Create:** `src/main/java/com/learnhub/web/dto/AuthUserResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.security.SecurityRole;

public record AuthUserResponse(
        Long id,
        String email,
        String displayName,
        SecurityRole role
) {
}
</CodeBlock>

Notice what is absent: `password`, `passwordHash`, `enabled` internals, and Hibernate relationships.

---

## 7. Extend AppUserEntity safely

**Modify:** `src/main/java/com/learnhub/persistence/AppUserEntity.java`

Keep the Lesson 10/13 fields, mappings, and constructors. Add this factory or adapt it to your existing constructor design:

<CodeBlock language="java" editable>
/**
 * Only the trusted registration service creates
 * new accounts through this factory.
 */
public static AppUserEntity newStudent(
        String email,
        String displayName,
        String encodedPassword
) {
    AppUserEntity user = new AppUserEntity();

    user.email = email;
    user.displayName = displayName;
    user.passwordHash = encodedPassword;
    user.role =
            com.learnhub.security.SecurityRole.STUDENT;
    user.enabled = true;

    return user;
}
</CodeBlock>

This assumes your existing entity has a protected or public no-argument constructor and fields named `email`, `displayName`, `passwordHash`, `role`, and `enabled`.

If Lesson 10 uses different names or constructors, adapt the factory without duplicating the entity.

### Repository methods

**Modify:** `src/main/java/com/learnhub/persistence/JpaAppUserRepository.java`

The interface should already extend `JpaRepository<AppUserEntity, Long>`. Ensure it contains:

<CodeBlock language="java" editable>
@org.springframework.data.jpa.repository.Query("""
        select u
        from AppUserEntity u
        where lower(u.email) = lower(:email)
        """)
java.util.Optional<AppUserEntity> findByEmailIgnoreCase(
        @org.springframework.data.repository.query.Param("email")
        String email
);
</CodeBlock>

Do not add a second method with the same signature if Lesson 13 already introduced it.

---

## 8. Registration service

Registration must be transactional and must never accept a role from the request.

**Create:** `src/main/java/com/learnhub/security/AuthService.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.persistence.AppUserEntity;
import com.learnhub.persistence.JpaAppUserRepository;
import com.learnhub.web.dto.AuthUserResponse;
import com.learnhub.web.dto.RegisterRequest;

import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.nio.charset.StandardCharsets;
import java.util.Locale;

@Service
public class AuthService {

    private final JpaAppUserRepository users;
    private final PasswordEncoder passwordEncoder;

    public AuthService(
            JpaAppUserRepository users,
            PasswordEncoder passwordEncoder
    ) {
        this.users = users;
        this.passwordEncoder = passwordEncoder;
    }

    @Transactional
    public AuthUserResponse register(RegisterRequest request) {
        String email = request.email()
                .trim()
                .toLowerCase(Locale.ROOT);

        String displayName = request.displayName().trim();
        String password = request.password();

        if (displayName.isBlank()) {
            throw new IllegalArgumentException(
                    "Display name cannot be blank"
            );
        }

        // BCrypt's effective input limit is 72 bytes.
        int bytes = password.getBytes(
                StandardCharsets.UTF_8
        ).length;

        if (bytes > 72) {
            throw new IllegalArgumentException(
                    "Password exceeds the supported byte limit"
            );
        }

        if (users.findByEmailIgnoreCase(email).isPresent()) {
            throw new DuplicateEmailException(
                    "An account with this email already exists"
            );
        }

        AppUserEntity user = AppUserEntity.newStudent(
                email,
                displayName,
                passwordEncoder.encode(password)
        );

        // flush forces a uniqueness violation to surface
        // before returning from the service method.
        AppUserEntity saved = users.saveAndFlush(user);

        return new AuthUserResponse(
                saved.getId(),
                saved.getEmail(),
                saved.getDisplayName(),
                saved.getRole()
        );
    }
}
</CodeBlock>

**Create:** `src/main/java/com/learnhub/security/DuplicateEmailException.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

public class DuplicateEmailException
        extends RuntimeException {

    public DuplicateEmailException(String message) {
        super(message);
    }
}
</CodeBlock>

### Why check uniqueness twice?

The service checks whether an email exists to provide a useful error.

But concurrent requests can both pass that check.

The database's unique index on `lower(email)` is the authoritative protection.

We will map its specific unique-constraint violation to HTTP `409 Conflict`.

Do not catch every `DataIntegrityViolationException` and call it a duplicate email: unrelated foreign-key or check-constraint failures should not be misreported.

### Password normalization warning

We normalize emails, **not passwords**. Trimming, lowercasing, or Unicode-normalizing passwords without a carefully defined policy changes the user's secret.

---

## 9. Configure session-based Spring Security

This is the central backend change.

**Replace:** `src/main/java/com/learnhub/config/SecurityConfig.java`

<CodeBlock language="java" editable>
package com.learnhub.config;

import com.learnhub.security.DatabaseUserDetailsService;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
import org.springframework.security.web.context.SecurityContextRepository;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

import static org.springframework.security.config.Customizer.withDefaults;

@Configuration
public class SecurityConfig {

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    AuthenticationManager authenticationManager(
            DatabaseUserDetailsService users,
            PasswordEncoder encoder
    ) {
        DaoAuthenticationProvider provider =
                new DaoAuthenticationProvider(users);

        provider.setPasswordEncoder(encoder);

        return new ProviderManager(provider);
    }

    @Bean
    SecurityContextRepository securityContextRepository() {
        return new HttpSessionSecurityContextRepository();
    }

    @Bean
    SecurityFilterChain securityFilterChain(
            HttpSecurity http,
            SecurityContextRepository contextRepository
    ) throws Exception {

        http
            .securityContext(context ->
                context.securityContextRepository(
                        contextRepository
                )
            )

            .cors(withDefaults())

            .csrf(csrf ->
                csrf.csrfTokenRepository(
                    CookieCsrfTokenRepository
                        .withHttpOnlyFalse()
                )
            )

            .sessionManagement(session -> session
                .sessionCreationPolicy(
                    SessionCreationPolicy.IF_REQUIRED
                )
                .sessionFixation(fixation ->
                    fixation.changeSessionId()
                )
            )

            .authorizeHttpRequests(auth -> auth

                .requestMatchers(
                    HttpMethod.GET,
                    "/api/v1/security/csrf",
                    "/api/v1/courses",
                    "/api/v1/courses/*",
                    "/api/v1/courses/categories",
                    "/api/v1/course-feed",
                    "/actuator/health"
                ).permitAll()

                .requestMatchers(
                    HttpMethod.POST,
                    "/api/v1/auth/register",
                    "/api/v1/auth/login"
                ).permitAll()

                .requestMatchers(
                    HttpMethod.GET,
                    "/api/v1/auth/me"
                ).authenticated()

                .requestMatchers(
                    HttpMethod.POST,
                    "/api/v1/auth/logout"
                ).authenticated()

                .requestMatchers(
                    HttpMethod.POST,
                    "/api/v1/courses",
                    "/api/v1/courses/*/publish",
                    "/api/v1/courses/*/archive"
                ).hasAnyRole("INSTRUCTOR", "ADMIN")

                .requestMatchers(
                    HttpMethod.PATCH,
                    "/api/v1/courses/*/publish",
                    "/api/v1/courses/*/archive"
                ).hasAnyRole("INSTRUCTOR", "ADMIN")

                .requestMatchers(
                    HttpMethod.PUT,
                    "/api/v1/courses/*"
                ).hasAnyRole("INSTRUCTOR", "ADMIN")

                .requestMatchers(
                    HttpMethod.DELETE,
                    "/api/v1/courses/*"
                ).hasAnyRole("INSTRUCTOR", "ADMIN")

                .anyRequest().denyAll()
            )

            // No HTTP Basic or default HTML form login.
            .httpBasic(basic -> basic.disable())
            .formLogin(form -> form.disable())

            // Our controller performs logout explicitly.
            // Disable the framework's competing /logout route.
            .logout(logout -> logout.disable())

            .exceptionHandling(errors -> errors
                .authenticationEntryPoint(
                    (request, response, exception) ->
                        response.sendError(401)
                )
                .accessDeniedHandler(
                    (request, response, exception) ->
                        response.sendError(403)
                )
            );

        return http.build();
    }
}
</CodeBlock>

### Why `denyAll()` at the end?

Unlike Lesson 13's `authenticated()` fallback, `denyAll()` forces us to explicitly approve new endpoints.

This is useful for a backend that is gradually introducing privileged workflows.

**Check the actual HTTP methods in your Lesson 11 controller.** Earlier lessons used `PATCH` for publishing and archiving; this configuration explicitly handles both `POST` and `PATCH` to prevent accidental authorization gaps. Remove the unused method once your API contract is settled.

### Important: public catalog routes

If your Lesson 11 catalog has additional public endpoints, such as `/api/v1/courses/categories`, ensure they are explicitly permitted.

Security rules must match the actual controller routes. A route that is accidentally omitted should fail closed, not become public.

### CSRF bootstrap

Keep Lesson 13's `SecuritySessionController` and its `/api/v1/security/csrf` endpoint.

For Angular's same-origin XSRF integration, we need a reliable way to obtain the raw CSRF token and refresh it after authentication or logout. The implementation below returns the token in JSON and makes sure the corresponding cookie is written.

---

## 10. Implement registration, login, current user, and logout

**Create:** `src/main/java/com/learnhub/security/AuthController.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.persistence.AppUserEntity;
import com.learnhub.persistence.JpaAppUserRepository;
import com.learnhub.web.dto.AuthUserResponse;
import com.learnhub.web.dto.LoginRequest;
import com.learnhub.web.dto.RegisterRequest;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;
import jakarta.validation.Valid;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.context.SecurityContextRepository;
import org.springframework.security.web.csrf.CsrfToken;
import org.springframework.security.web.csrf.CsrfTokenRepository;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/auth")
public class AuthController {

    private final AuthService authService;
    private final AuthenticationManager authenticationManager;
    private final SecurityContextRepository contextRepository;
    private final JpaAppUserRepository users;
    private final CsrfTokenRepository csrfRepository;

    public AuthController(
            AuthService authService,
            AuthenticationManager authenticationManager,
            SecurityContextRepository contextRepository,
            JpaAppUserRepository users
    ) {
        this.authService = authService;
        this.authenticationManager = authenticationManager;
        this.contextRepository = contextRepository;
        this.users = users;
        this.csrfRepository =
                org.springframework.security.web.csrf
                    .CookieCsrfTokenRepository
                    .withHttpOnlyFalse();
    }

    @PostMapping("/register")
    public ResponseEntity<AuthUserResponse> register(
            @Valid @RequestBody RegisterRequest request
    ) {
        AuthUserResponse created = authService.register(request);

        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(created);
    }

    @PostMapping("/login")
    public AuthUserResponse login(
            @Valid @RequestBody LoginRequest request,
            HttpServletRequest servletRequest,
            HttpServletResponse servletResponse
    ) {
        Authentication authentication =
                authenticationManager.authenticate(
                    UsernamePasswordAuthenticationToken
                        .unauthenticated(
                            request.email(),
                            request.password()
                        )
                );

        // Manual authentication must explicitly protect
        // against session fixation.
        HttpSession existing =
                servletRequest.getSession(false);

        if (existing != null) {
            servletRequest.changeSessionId();
        } else {
            servletRequest.getSession(true);
        }

        SecurityContext context =
                SecurityContextHolder.createEmptyContext();

        context.setAuthentication(authentication);
        SecurityContextHolder.setContext(context);

        // Required for authentication to survive
        // the next HTTP request.
        contextRepository.saveContext(
                context,
                servletRequest,
                servletResponse
        );

        // Invalidate any pre-login CSRF token.
        csrfRepository.saveToken(
                null,
                servletRequest,
                servletResponse
        );

        LearnHubPrincipal principal =
                (LearnHubPrincipal) authentication.getPrincipal();

        return currentUser(principal);
    }

    @GetMapping("/me")
    public AuthUserResponse me(Authentication authentication) {
        LearnHubPrincipal principal =
                (LearnHubPrincipal) authentication.getPrincipal();

        return currentUser(principal);
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(
            HttpServletRequest request,
            HttpServletResponse response
    ) {
        HttpSession session = request.getSession(false);

        if (session != null) {
            session.invalidate();
        }

        SecurityContextHolder.clearContext();

        csrfRepository.saveToken(
                null,
                request,
                response
        );

        return ResponseEntity.noContent().build();
    }

    private AuthUserResponse currentUser(
            LearnHubPrincipal principal
    ) {
        AppUserEntity user = users.findById(principal.id())
                .orElseThrow(() ->
                    new IllegalStateException(
                        "Authenticated account no longer exists"
                    )
                );

        return new AuthUserResponse(
                user.getId(),
                user.getEmail(),
                user.getDisplayName(),
                user.getRole()
        );
    }
}
</CodeBlock>

### Session fixation protection

A session fixation attack attempts to make a victim authenticate using a session ID chosen or known by an attacker.

On successful login, we call:

```java
servletRequest.changeSessionId();
```

If a session exists, this changes its identifier while retaining appropriate session attributes.

If no session exists, a new one is created.

Spring Security's configured `sessionFixation().changeSessionId()` applies to authentication flows managed through its normal session-authentication strategy. Because this lesson uses a custom JSON login controller, we perform the session ID change explicitly.

### Important CSRF lifecycle detail

After successful login, the old CSRF token is cleared. Angular must call `GET /api/v1/security/csrf` again before its next state-changing request.

The same applies after logout.

A session cookie is an authentication credential; the XSRF token is not. Do not use the XSRF cookie as proof that someone is logged in.

### Two production refinements

The controller currently retrieves the user again to construct the safe response. In a production application, that lookup should live in a dedicated read-only application service rather than directly in the controller.

Also, for full session invalidation semantics, remember-me support, distributed sessions, and session concurrency limits, use Spring Security's session-management infrastructure rather than accumulating custom session behavior.

---

## 11. Fix the CSRF token bootstrap

Lesson 13's CSRF endpoint returned the token. For Angular, we'll keep that endpoint and ensure token materialization.

**Replace:** `src/main/java/com/learnhub/web/SecuritySessionController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import org.springframework.security.web.csrf.CsrfToken;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/security")
public class SecuritySessionController {

    @GetMapping("/csrf")
    public CsrfResponse csrf(CsrfToken token) {
        // Accessing the token forces deferred token generation.
        return new CsrfResponse(
                token.getHeaderName(),
                token.getToken()
        );
    }

    public record CsrfResponse(
            String headerName,
            String token
    ) {
    }
}
</CodeBlock>

### Spring Security 6 CSRF cookie compatibility

Spring Security 6 uses BREACH-resistant token handling. With cookie-based CSRF tokens, the framework must resolve the submitted header token correctly.

The reference setup uses Spring Security's `CookieCsrfTokenRepository`. For Angular's XSRF cookie/header integration, configure a request handler that accepts the raw header token while retaining XOR protection for rendered request attributes.

Add this class:

**Create:** `src/main/java/com/learnhub/security/SpaCsrfTokenRequestHandler.java`

<CodeBlock language="java" editable>
package com.learnhub.security;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import org.springframework.security.web.csrf.CsrfToken;
import org.springframework.security.web.csrf.CsrfTokenRequestAttributeHandler;
import org.springframework.security.web.csrf.CsrfTokenRequestHandler;
import org.springframework.security.web.csrf.XorCsrfTokenRequestAttributeHandler;

import java.util.function.Supplier;

/**
 * Resolves raw X-XSRF-TOKEN headers from an SPA while
 * retaining XOR token handling for request attributes.
 */
public final class SpaCsrfTokenRequestHandler
        implements CsrfTokenRequestHandler {

    private final XorCsrfTokenRequestAttributeHandler xor =
            new XorCsrfTokenRequestAttributeHandler();

    private final CsrfTokenRequestAttributeHandler plain =
            new CsrfTokenRequestAttributeHandler();

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            Supplier<CsrfToken> csrfToken
    ) {
        xor.handle(request, response, csrfToken);

        // Ensure the token is loaded, so the cookie
        // repository can write a token when necessary.
        csrfToken.get();
    }

    @Override
    public String resolveCsrfTokenValue(
            HttpServletRequest request,
            CsrfToken csrfToken
    ) {
        String header = request.getHeader(
                csrfToken.getHeaderName()
        );

        return header != null
                ? plain.resolveCsrfTokenValue(
                    request, csrfToken
                )
                : xor.resolveCsrfTokenValue(
                    request, csrfToken
                );
    }
}
</CodeBlock>

In `SecurityConfig`, replace the `.csrf(...)` section with:

<CodeBlock language="java" editable>
.csrf(csrf -> csrf
    .csrfTokenRepository(
        CookieCsrfTokenRepository.withHttpOnlyFalse()
    )
    .csrfTokenRequestHandler(
        new com.learnhub.security
            .SpaCsrfTokenRequestHandler()
    )
)
</CodeBlock>

This is a deliberate integration point: CSRF token representation and lifecycle must be tested through actual HTTP requests, not assumed from cookie names.

---

## 12. Configure secure session cookies

**Modify:** `src/main/resources/application.yml`

Merge these settings with the existing datasource, Flyway, and JPA configuration:

<CodeBlock language="yaml" editable>
server:
  servlet:
    session:
      timeout: 30m
      cookie:
        http-only: true
        same-site: lax
        secure: true

spring:
  jpa:
    open-in-view: false
</CodeBlock>

### Local HTTP development

A secure cookie should be transmitted only over HTTPS. For a local development environment using plain `http://localhost`, override this setting in a dedicated development profile.

**Create or modify:** `src/main/resources/application-dev.yml`

<CodeBlock language="yaml" editable>
server:
  servlet:
    session:
      cookie:
        secure: false
        http-only: true
        same-site: lax
</CodeBlock>

**Never enable the development cookie override in production.**

For production, use HTTPS end to end or a correctly configured trusted reverse proxy, and ensure Spring Boot understands forwarded headers.

### Cookie comparison

| Cookie | Purpose | HttpOnly | Secure in production |
|---|---|---|---|
| `JSESSIONID` | Authenticated session | Yes | Yes |
| `XSRF-TOKEN` | CSRF token for SPA header | No | Yes, under HTTPS |

The `XSRF-TOKEN` cookie is intentionally readable by Angular. That does not make it an authentication cookie.

**Deployment note:** The secure flag for the CSRF cookie is configured separately from the session cookie. Ensure the `CookieCsrfTokenRepository` cookie attributes are appropriate for HTTPS production deployment; do not assume `server.servlet.session.cookie.secure` configures every cookie emitted by the application.

---

## 13. Map authentication errors to safe HTTP responses

Authentication failures must not reveal whether a particular email address exists.

For login:

- Wrong email → `401`
- Wrong password → `401`
- Disabled account → `401`
- Valid credentials → `200`

For registration:

- Invalid input → `400`
- Duplicate email → `409`
- Successful registration → `201`

### Add exception handling

**Modify:** `src/main/java/com/learnhub/web/error/GlobalExceptionHandler.java`

Add these handlers to the existing `@RestControllerAdvice`:

<CodeBlock language="java" editable>
@org.springframework.web.bind.annotation.ExceptionHandler(
        org.springframework.security.core.AuthenticationException.class
)
public org.springframework.http.ResponseEntity<
        org.springframework.http.ProblemDetail>
authenticationFailed(
        org.springframework.security.core.AuthenticationException ex
) {
    var problem = org.springframework.http.ProblemDetail
            .forStatusAndDetail(
                org.springframework.http.HttpStatus.UNAUTHORIZED,
                "Invalid email or password."
            );

    problem.setTitle("Authentication failed");

    return org.springframework.http.ResponseEntity
            .status(401)
            .body(problem);
}

@org.springframework.web.bind.annotation.ExceptionHandler(
        com.learnhub.security.DuplicateEmailException.class
)
public org.springframework.http.ResponseEntity<
        org.springframework.http.ProblemDetail>
duplicateEmail(
        com.learnhub.security.DuplicateEmailException ex
) {
    var problem = org.springframework.http.ProblemDetail
            .forStatusAndDetail(
                org.springframework.http.HttpStatus.CONFLICT,
                "An account with this email already exists."
            );

    problem.setTitle("Duplicate account");

    return org.springframework.http.ResponseEntity
            .status(409)
            .body(problem);
}
</CodeBlock>

**Important:** The database unique index can still raise a `DataIntegrityViolationException` during a concurrent registration. To return `409` reliably, identify the specific `ux_app_users_email_lower` constraint in a narrowly scoped exception translator outside the failed transaction. Do not convert every database integrity error into a duplicate-account response.

Also verify that your Lesson 8 advice does not have an overly broad `RuntimeException` handler that converts authentication errors into `500`.

---

## 14. PostgreSQL integration tests

We'll use the existing `PostgresIntegrationTest` base with PostgreSQL Testcontainers.

The tests below exercise real database persistence, BCrypt verification, Spring Security, cookies, and CSRF.

### 14.1 Create the test class

**Create:** `src/test/java/com/learnhub/security/AuthApiIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.mock.web.MockHttpSession;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class AuthApiIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired
    MockMvc mvc;

    @Autowired
    ObjectMapper json;

    private static final String EMAIL =
            "lesson14@example.test";

    private static final String PASSWORD =
            "LongLocalPassword123!";

    private void register() throws Exception {
        mvc.perform(post("/api/v1/auth/register")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "displayName": "Lesson 14 Student",
                      "email": "lesson14@example.test",
                      "password": "LongLocalPassword123!"
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.role").value("STUDENT"));
    }

    @Test
    void registrationCreatesStudentAccount()
            throws Exception {
        register();
    }

    @Test
    void registrationDoesNotReturnPasswordHash()
            throws Exception {
        mvc.perform(post("/api/v1/auth/register")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "displayName": "Safe Response",
                      "email": "safe-response@example.test",
                      "password": "LongLocalPassword123!"
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.password").doesNotExist())
            .andExpect(jsonPath("$.passwordHash").doesNotExist());
    }

    @Test
    void duplicateEmailReturnsConflict()
            throws Exception {
        register();

        mvc.perform(post("/api/v1/auth/register")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "displayName": "Another User",
                      "email": "LESSON14@example.test",
                      "password": "AnotherLongPassword123!"
                    }
                    """))
            .andExpect(status().isConflict());
    }

    @Test
    void invalidPasswordIsRejected()
            throws Exception {
        register();

        mvc.perform(post("/api/v1/auth/login")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "email": "lesson14@example.test",
                      "password": "IncorrectPassword123!"
                    }
                    """))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void loginPersistsAuthenticationInSession()
            throws Exception {
        register();

        var login = mvc.perform(post("/api/v1/auth/login")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "email": "lesson14@example.test",
                      "password": "LongLocalPassword123!"
                    }
                    """))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.email").value(EMAIL))
            .andReturn();

        MockHttpSession session =
                (MockHttpSession) login.getRequest()
                        .getSession(false);

        org.junit.jupiter.api.Assertions
                .assertNotNull(session);

        mvc.perform(get("/api/v1/auth/me")
                .session(session))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.role").value("STUDENT"));
    }

    @Test
    void anonymousMeReturnsUnauthorized()
            throws Exception {
        mvc.perform(get("/api/v1/auth/me"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void registrationWithoutCsrfIsRejected()
            throws Exception {
        mvc.perform(post("/api/v1/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isForbidden());
    }
}
</CodeBlock>

These seven tests verify registration, safe DTOs, duplicate emails, invalid login, session persistence, protected user details, and CSRF.

### Test transaction warning

These tests use `@Transactional`, which rolls back writes after each test. For synchronous MockMvc requests, the application usually participates in the test-managed transaction.

That makes this approach convenient for basic API contract tests, but it can hide the behavior of independent database transactions.

For true concurrency, committed-session behavior, or multiple simultaneous requests, use tests without a test-level transaction and clean up fixtures explicitly.

### 14.2 Test session fixation and logout

**Create:** `src/test/java/com/learnhub/security/SessionSecurityTest.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.mock.web.MockHttpSession;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class SessionSecurityTest extends PostgresIntegrationTest {

    @Autowired
    MockMvc mvc;

    @Test
    void loginChangesExistingSessionId() throws Exception {
        mvc.perform(post("/api/v1/auth/register")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "displayName": "Session Test",
                      "email": "session-test@example.test",
                      "password": "LongLocalPassword123!"
                    }
                    """))
            .andExpect(status().isCreated());

        MockHttpSession session = new MockHttpSession();
        String before = session.getId();

        mvc.perform(post("/api/v1/auth/login")
                .session(session)
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "email": "session-test@example.test",
                      "password": "LongLocalPassword123!"
                    }
                    """))
            .andExpect(status().isOk());

        assertNotEquals(before, session.getId());
    }

    @Test
    void logoutInvalidatesAuthenticatedSession()
            throws Exception {
        mvc.perform(post("/api/v1/auth/register")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "displayName": "Logout Test",
                      "email": "logout-test@example.test",
                      "password": "LongLocalPassword123!"
                    }
                    """))
            .andExpect(status().isCreated());

        var result = mvc.perform(post("/api/v1/auth/login")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "email": "logout-test@example.test",
                      "password": "LongLocalPassword123!"
                    }
                    """))
            .andExpect(status().isOk())
            .andReturn();

        MockHttpSession session =
                (MockHttpSession) result.getRequest()
                        .getSession(false);

        assertNotNull(session);

        mvc.perform(post("/api/v1/auth/logout")
                .session(session)
                .with(csrf()))
            .andExpect(status().isNoContent());

        mvc.perform(get("/api/v1/auth/me"))
            .andExpect(status().isUnauthorized());
    }
}
</CodeBlock>

**Test-strengthening exercise:** The last assertion makes an anonymous request, so it does not by itself prove the original session was invalidated. Add a test that reuses the old session cookie and verifies that authentication is no longer accepted. Also inspect `session.isInvalid()` in MockMvc.

### Run the focused tests

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=AuthApiIntegrationTest,SessionSecurityTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 14 authentication tests failed."
}
```

**Expected new tests:** nine, plus any exercises you implement.

These are reference tests; they have not been executed against your local project.

---

## 15. Configure the Angular development proxy

The simplest browser integration is to make Angular call relative API URLs.

### Create proxy configuration

**File:** `learnhub/proxy.conf.json`

<CodeBlock language="json" editable>
{
  "/api": {
    "target": "http://localhost:8080",
    "secure": false,
    "changeOrigin": false
  }
}
</CodeBlock>

Start Angular with:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd serve --proxy-config proxy.conf.json
```

The browser requests:

```text
http://localhost:4200/api/v1/auth/login
```

Angular's dev server forwards that request to:

```text
http://localhost:8080/api/v1/auth/login
```

The browser sees a same-origin request, simplifying cookie and XSRF behavior.

**Important:** Existing Angular API services that hardcode `http://localhost:8080` must switch to relative `/api/v1/...` URLs for this proxy approach.

---

## 16. Angular authentication models

**Create:** `learnhub/src/app/core/auth/auth.models.ts`

<CodeBlock language="typescript" editable>
export type UserRole =
  | 'STUDENT'
  | 'INSTRUCTOR'
  | 'ADMIN';

export interface AuthUser {
  id: number;
  email: string;
  displayName: string;
  role: UserRole;
}

export interface RegisterRequest {
  displayName: string;
  email: string;
  password: string;
}

export interface LoginRequest {
  email: string;
  password: string;
}

export interface CsrfResponse {
  headerName: string;
  token: string;
}
</CodeBlock>

These models intentionally contain no password hash or session ID.

---

## 17. Configure Angular XSRF support

Angular's `HttpClient` can read the `XSRF-TOKEN` cookie and send its value in the `X-XSRF-TOKEN` header for eligible same-origin mutating requests.

**Modify:** `learnhub/src/app/app.config.ts`

Merge the following into your existing providers:

<CodeBlock language="typescript" editable>
import {
  provideHttpClient,
  withXsrfConfiguration,
} from '@angular/common/http';

// Inside your existing ApplicationConfig.providers:
provideHttpClient(
  withXsrfConfiguration({
    cookieName: 'XSRF-TOKEN',
    headerName: 'X-XSRF-TOKEN',
  })
),
</CodeBlock>

Do not add a second `provideHttpClient()` if one already exists; consolidate the options.

### Why we don't need a custom interceptor yet

With same-origin relative URLs and Angular's built-in XSRF support, a custom interceptor is unnecessary for the basic flow.

We still need to fetch a CSRF token before the first state-changing request and refresh it after login/logout.

### An important first-request detail

When Angular starts, the XSRF cookie may not exist.

Before calling `POST /auth/register` or `POST /auth/login`, request:

```text
GET /api/v1/security/csrf
```

That establishes the token cookie.

After login, Spring Security clears the old CSRF token. Request the bootstrap endpoint again before the next write.

---

## 18. Angular AuthService

**Create:** `learnhub/src/app/core/auth/auth.service.ts`

<CodeBlock language="typescript" editable>
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';

import {
  Observable,
  switchMap,
  tap,
  map,
  catchError,
  of,
} from 'rxjs';

import {
  AuthUser,
  CsrfResponse,
  LoginRequest,
  RegisterRequest,
} from './auth.models';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly http = inject(HttpClient);

  private readonly base = '/api/v1';

  readonly currentUser = signal<AuthUser | null>(null);

  /**
   * Ensures the server issues the XSRF-TOKEN cookie.
   */
  private csrf(): Observable<void> {
    return this.http
      .get<CsrfResponse>(
        `${this.base}/security/csrf`
      )
      .pipe(map(() => void 0));
  }

  register(
    request: RegisterRequest
  ): Observable<AuthUser> {
    return this.csrf().pipe(
      switchMap(() =>
        this.http.post<AuthUser>(
          `${this.base}/auth/register`,
          request
        )
      )
    );
  }

  login(
    request: LoginRequest
  ): Observable<AuthUser> {
    return this.csrf().pipe(
      switchMap(() =>
        this.http.post<AuthUser>(
          `${this.base}/auth/login`,
          request
        )
      ),
      switchMap(user =>
        // The server invalidates the pre-login token.
        this.csrf().pipe(
          map(() => user)
        )
      ),
      tap(user => this.currentUser.set(user))
    );
  }

  loadCurrentUser(): Observable<AuthUser | null> {
    return this.http.get<AuthUser>(
      `${this.base}/auth/me`
    ).pipe(
      tap(user => this.currentUser.set(user)),
      catchError(error => {
        if (error.status === 401) {
          this.currentUser.set(null);
          return of(null);
        }

        throw error;
      })
    );
  }

  logout(): Observable<void> {
    return this.csrf().pipe(
      switchMap(() =>
        this.http.post<void>(
          `${this.base}/auth/logout`,
          {}
        )
      ),
      switchMap(() => this.csrf()),
      tap(() => this.currentUser.set(null))
    );
  }
}
</CodeBlock>

### State management

The `currentUser` signal is a convenient frontend representation of the session's user.

It is **not** the source of authorization.

The backend must continue to enforce roles and ownership regardless of what Angular displays.

### Browser refresh

After a page refresh, the Angular signal starts as `null` even if the session cookie remains valid.

Call `loadCurrentUser()` during application initialization or in a suitable auth-aware shell to restore the UI state.

Avoid treating `null` as proof of being logged out until that initial request finishes. A production auth state model usually distinguishes `loading`, `authenticated`, and `anonymous`.

---

## 19. Angular standalone login component

**Create:** `learnhub/src/app/features/auth/login-page.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  inject,
  signal,
} from '@angular/core';

import { ReactiveFormsModule, FormBuilder, Validators }
  from '@angular/forms';

import { Router, RouterLink } from '@angular/router';
import { finalize } from 'rxjs';

import { AuthService } from '../../core/auth/auth.service';

@Component({
  selector: 'app-login-page',
  standalone: true,
  imports: [ReactiveFormsModule, RouterLink],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main class="auth-page">
      <h1>Sign in to LearnHub</h1>

      <form [formGroup]="form" (ngSubmit)="submit()">
        <label for="email">Email</label>
        <input
          id="email"
          type="email"
          autocomplete="username"
          formControlName="email"
        />

        <label for="password">Password</label>
        <input
          id="password"
          type="password"
          autocomplete="current-password"
          formControlName="password"
        />

        @if (error()) {
          <p role="alert">{{ error() }}</p>
        }

        <button
          type="submit"
          [disabled]="form.invalid || loading()"
        >
          {{ loading() ? 'Signing in...' : 'Sign in' }}
        </button>
      </form>

      <p>
        New to LearnHub?
        <a routerLink="/register">Create an account</a>
      </p>
    </main>
  `,
})
export class LoginPageComponent {
  private readonly fb = inject(FormBuilder);
  private readonly auth = inject(AuthService);
  private readonly router = inject(Router);

  readonly loading = signal(false);
  readonly error = signal<string | null>(null);

  readonly form = this.fb.nonNullable.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required]],
  });

  submit(): void {
    if (this.form.invalid || this.loading()) {
      return;
    }

    this.loading.set(true);
    this.error.set(null);

    this.auth.login(this.form.getRawValue()).pipe(
      finalize(() => this.loading.set(false))
    ).subscribe({
      next: () => {
        void this.router.navigateByUrl('/');
      },
      error: () => {
        this.error.set(
          'Unable to sign in. Check your credentials and try again.'
        );
      },
    });
  }
}
</CodeBlock>

The error message is intentionally generic so the frontend does not reveal whether an account exists.

---

## 20. Angular standalone registration component

**Create:** `learnhub/src/app/features/auth/register-page.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  inject,
  signal,
} from '@angular/core';

import { ReactiveFormsModule, FormBuilder, Validators }
  from '@angular/forms';

import { Router, RouterLink } from '@angular/router';
import { finalize } from 'rxjs';

import { AuthService } from '../../core/auth/auth.service';

@Component({
  selector: 'app-register-page',
  standalone: true,
  imports: [ReactiveFormsModule, RouterLink],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main class="auth-page">
      <h1>Create your LearnHub account</h1>

      <form [formGroup]="form" (ngSubmit)="submit()">
        <label for="name">Display name</label>
        <input
          id="name"
          formControlName="displayName"
          autocomplete="name"
        />

        <label for="email">Email</label>
        <input
          id="email"
          type="email"
          formControlName="email"
          autocomplete="email"
        />

        <label for="password">Password</label>
        <input
          id="password"
          type="password"
          formControlName="password"
          autocomplete="new-password"
        />

        <p>Use at least 12 characters.</p>

        @if (error()) {
          <p role="alert">{{ error() }}</p>
        }

        @if (success()) {
          <p role="status">
            Account created. Redirecting to sign in...
          </p>
        }

        <button
          type="submit"
          [disabled]="form.invalid || loading()"
        >
          {{ loading() ? 'Creating...' : 'Create account' }}
        </button>
      </form>

      <a routerLink="/login">Already have an account?</a>
    </main>
  `,
})
export class RegisterPageComponent {
  private readonly fb = inject(FormBuilder);
  private readonly auth = inject(AuthService);
  private readonly router = inject(Router);

  readonly loading = signal(false);
  readonly error = signal<string | null>(null);
  readonly success = signal(false);

  readonly form = this.fb.nonNullable.group({
    displayName: ['', [
      Validators.required,
      Validators.maxLength(100),
    ]],
    email: ['', [
      Validators.required,
      Validators.email,
      Validators.maxLength(254),
    ]],
    password: ['', [
      Validators.required,
      Validators.minLength(12),
      Validators.maxLength(128),
    ]],
  });

  submit(): void {
    if (this.form.invalid || this.loading()) {
      return;
    }

    this.loading.set(true);
    this.error.set(null);

    this.auth.register(this.form.getRawValue()).pipe(
      finalize(() => this.loading.set(false))
    ).subscribe({
      next: () => {
        this.success.set(true);
        void this.router.navigateByUrl('/login');
      },
      error: error => {
        this.error.set(
          error.status === 409
            ? 'An account with this email already exists.'
            : 'Registration failed. Please check your details.'
        );
      },
    });
  }
}
</CodeBlock>
The browser validates length and basic email formatting for user experience, but the backend remains authoritative.

### Register the routes

**Modify:** `learnhub/src/app/app.routes.ts`

Add these routes to the existing array without duplicating paths:

<CodeBlock language="typescript" editable>
{
  path: 'login',
  loadComponent: () =>
    import('./features/auth/login-page.component')
      .then(m => m.LoginPageComponent),
},
{
  path: 'register',
  loadComponent: () =>
    import('./features/auth/register-page.component')
      .then(m => m.RegisterPageComponent),
},
</CodeBlock>

Keep the existing course catalog, course details, and other Angular routes.

---

## 21. Verify the complete browser workflow

Start Spring Boot:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

In another terminal, start Angular:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd serve --proxy-config proxy.conf.json
```

Open `http://localhost:4200/register`.

### Manual acceptance scenario

1. Register a new student with a unique email.
2. Verify the API returns `201 Created` and role `STUDENT`.
3. Open the login page and sign in.
4. Inspect browser DevTools → Application → Cookies.
5. Confirm a `JSESSIONID` cookie exists and is HttpOnly.
6. Refresh the page and verify `/api/v1/auth/me` still recognizes the session.
7. Attempt an instructor-only course write as a student and verify `403 Forbidden`.
8. Sign out and verify `/api/v1/auth/me` returns `401 Unauthorized`.
9. Verify the public course catalog still works anonymously.

### PowerShell API verification

For an end-to-end script, use a single web session to retain cookies:

```powershell
$base = "http://localhost:8080"
$webSession = New-Object Microsoft.PowerShell.Commands.WebRequestSession

$csrf = Invoke-RestMethod `
  "$base/api/v1/security/csrf" `
  -WebSession $webSession

$headers = @{
    "X-XSRF-TOKEN" = $csrf.token
}

$email = "lesson14-manual-$([guid]::NewGuid().ToString('N'))@example.test"

$registerBody = @{
    displayName = "Manual Test Student"
    email       = $email
    password    = "LongLocalPassword123!"
} | ConvertTo-Json

$registered = Invoke-RestMethod `
  -Method Post `
  -Uri "$base/api/v1/auth/register" `
  -WebSession $webSession `
  -Headers $headers `
  -ContentType "application/json" `
  -Body $registerBody

$registered | Format-List
```

Login using the same web session:

```powershell
$loginBody = @{
    email    = $email
    password = "LongLocalPassword123!"
} | ConvertTo-Json

$loggedIn = Invoke-RestMethod `
  -Method Post `
  -Uri "$base/api/v1/auth/login" `
  -WebSession $webSession `
  -Headers $headers `
  -ContentType "application/json" `
  -Body $loginBody

$loggedIn | Format-List
```

After login, obtain a fresh CSRF token:

```powershell
$csrf = Invoke-RestMethod `
  "$base/api/v1/security/csrf" `
  -WebSession $webSession

$headers = @{
    "X-XSRF-TOKEN" = $csrf.token
}
```

Verify the session:

```powershell
Invoke-RestMethod `
  "$base/api/v1/auth/me" `
  -WebSession $webSession
```

Logout:

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "$base/api/v1/auth/logout" `
  -WebSession $webSession `
  -Headers $headers
```

The example uses a fixed development-only test password in a local script. Do not reuse it for a real account or commit scripts containing real credentials.

### Expected response examples

Successful login:

```json
{
  "id": 42,
  "email": "student@example.test",
  "displayName": "Manual Test Student",
  "role": "STUDENT"
}
```

Invalid credentials:

```json
{
  "type": "about:blank",
  "title": "Authentication failed",
  "status": 401,
  "detail": "Invalid email or password."
}
```

The ID is illustrative; PostgreSQL generates the actual value.

---

## 22. Regression checks from Lessons 11–13

Security changes often expose earlier assumptions. Review these before treating the lesson as complete.

| Earlier behavior | Lesson 14 adjustment |
|---|---|
| HTTP Basic authentication | Removed; replace related tests with session login |
| `@WithMockUser` role tests | Still valid for authorization-only tests |
| Public catalog GET | Must remain explicitly permitted |
| Instructor CRUD tests | Must supply authenticated role and CSRF |
| Angular absolute backend URL | Change to relative `/api/v1/...` |
| CSRF token obtained before login | Refresh after successful login |
| Logout test | Verify old session is actually unusable |
| Legacy instructor without password | Cannot log in until credentials are provisioned |
| Instructor ownership | Still not implemented |

### A subtle but important issue: transactional tests

A test-managed transaction can make registration and login work together inside MockMvc even though the registration transaction hasn't committed independently.

Add at least one **non-transactional integration test** that registers an account, logs in through a separate HTTP request, verifies `/me`, logs out, and checks that the original session no longer authenticates. Use a unique email and explicit database cleanup.

That test is especially valuable before Lesson 15.

### Production gaps to keep visible

This lesson does not yet implement login throttling, email verification, password reset, account lockout policy, multi-device session management, or trusted instructor promotion.

Do not deploy public registration without appropriate abuse controls and operational monitoring.

---

## 23. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="shield-check" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise A — Real session lifecycle**

      Write a non-transactional Testcontainers test that registers, logs in, accesses `/me`, logs out, and verifies the previous session is rejected.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="key-round" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise B — Password boundary tests**

      Test passwords containing multibyte Unicode characters. Verify that the 72-byte BCrypt limit is enforced and that the API returns a useful validation response rather than silently truncating the password.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="database" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise C — Concurrent registration**

      Simulate two registrations with the same normalized email. Verify that the database unique index prevents duplicate accounts and that the API maps the expected constraint violation to `409 Conflict`.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="cookie" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise D — Session fixation**

      Establish a pre-login session, authenticate, and verify the session ID changes. Then test that the authenticated session remains usable.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="monitor-smartphone" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise E — Angular auth state**

      Introduce an explicit `loading | anonymous | authenticated` auth state and display a loading placeholder while `/auth/me` restores the browser session after refresh.
    </box>
  </row>
</box>

## 24. Knowledge quiz

{@body const qs=[{q:"What must happen after a custom JSON login successfully authenticates credentials?",a:["Return a JWT even though the application uses sessions","Save the SecurityContext in a session-backed repository","Store the plaintext password in a cookie"],c:1,why:"Manual AuthenticationManager authentication does not automatically persist the SecurityContext for subsequent requests."},{q:"Why should a session ID change during successful login?",a:["To prevent session fixation","To make BCrypt hashes unique","To disable CSRF"],c:0,why:"Rotating the identifier prevents an attacker from continuing to use a known pre-authentication session ID."},{q:"What role should a public registration endpoint assign?",a:["The role submitted in JSON","ADMIN for the first user","STUDENT from trusted server-side logic"],c:2,why:"Clients must not be able to grant themselves elevated privileges."},{q:"Why must Angular refresh its CSRF token after login?",a:["The server clears the pre-login CSRF token","The password changes after login","The catalog requires a new database index"],c:0,why:"The authentication transition invalidates the previous CSRF token, so the next unsafe request needs a fresh token."},{q:"Does an authenticated INSTRUCTOR role prove ownership of a specific course?",a:["Yes","No"],c:1,why:"Role-based authorization and resource ownership are separate checks."}]}
{@body const [a,setA]=DIL.useState({})}
{@body const [show,setShow]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Lesson 14 — Authentication and sessions**
    <text color="secondary" size="xs">5 questions</text>
  </row>
  {#each qs as q,i}
    <box key={i} gap={2}>
      **{i+1}. {q.q}**
      <radio-group direction="col" value={a[i]===undefined?"":String(a[i])} onChange={v=>{setA(old=>({...old,[i]:Number(v)}));setShow(false);}}>
        {#each q.a as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if show}
        <row align="start" gap={2}>
          <icon name={a[i]===q.c?"check-circle":"info"} color={a[i]===q.c?"success":"secondary"}/>
          <box flex="1" gap={1}>
            <text weight="medium" size="sm">{a[i]===q.c?"Correct":"Correct answer: "+q.a[q.c]}</text>
            <text color="secondary" size="sm">{q.why}</text>
          </box>
        </row>
      {/if}
    </box>
    {#if i<qs.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if show}
    <title size="lg">{qs.filter((q,i)=>a[i]===q.c).length} / 5 correct</title>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(a).length!==qs.length} onClick={()=>setShow(true)}>Check answers</button>
    <button color="secondary" variant="outline" onClick={()=>{setA({});setShow(false);}}>Reset</button>
  </row>
</box>

## 25. Completion checklist

{@body const groups=[{name:"Backend authentication",items:["Verified Lesson 13 baseline","Created LearnHubPrincipal","Updated DatabaseUserDetailsService","Created registration and login DTOs","Added student registration service","Configured session-backed SecurityContextRepository","Implemented JSON login and /me","Implemented logout and session invalidation","Added authentication error mapping"]},{name:"Session and CSRF security",items:["Removed temporary HTTP Basic authentication","Enabled session fixation protection","Kept CSRF enabled for unsafe requests","Configured SPA CSRF request handling","Verified CSRF refresh after login and logout","Configured session cookie flags","Verified instructor routes remain protected"]},{name:"PostgreSQL and tests",items:["Verified unique normalized email index","Tested BCrypt authentication","Added registration integration tests","Added session security tests","Tested duplicate registration","Added a non-transactional session lifecycle test","Ran Maven clean verify"]},{name:"Angular and completion",items:["Configured same-origin development proxy","Configured Angular XSRF support","Created AuthService and models","Created standalone login component","Created standalone registration component","Restored user state after refresh","Ran Angular build","Completed two exercises","Completed quiz","Reviewed and committed changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((n,g)=>n+g.items.length,0)}
{@body const complete=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 14 progress**
      <text color="secondary" size="xs">Mark items only after verifying them locally.</text>
    </box>
    <title size="lg" tabularNums>{complete}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${complete/total*100}%`} radius="full"/>
  </box>
  {#each groups as group,i}
    <box key={i} gap={2}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!done[`${i}-${j}`]} onChange={v=>setDone(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if complete===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 14 checklist completed.**
    </row>
  {/if}
</box>

## 26. Final verification and Git checkpoint

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=AuthApiIntegrationTest,SessionSecurityTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Authentication tests failed."
}

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Full backend verification failed."
}

git diff --check
git status
git diff --stat
```

Angular:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}

git diff --check
git status
```

Commit the backend after reviewing staged changes:

```powershell
cd $HOME\angular-learning\learnhub-backend

git add pom.xml src\main src\test
git diff --cached --stat
git commit -m "Implement secure session registration and login"
```

Commit the frontend:

```powershell
cd $HOME\angular-learning\learnhub

git add src\app proxy.conf.json
git diff --cached --stat
git commit -m "Integrate Angular registration and session login"
```

Never commit real passwords, database credentials, production cookie secrets, or local environment files.

### Final engineering review

| Capability | Lesson 14 implementation |
|---|---|
| Registration | Validated student account creation |
| Password storage | BCrypt |
| Authentication | PostgreSQL-backed `AuthenticationManager` |
| Session | Explicitly persisted `SecurityContext` |
| Session fixation | Session ID rotation |
| CSRF | Cookie token with SPA-compatible handling |
| Logout | Session invalidation and context clearing |
| Current user | Safe `/auth/me` DTO |
| Angular | Standalone login and registration |
| Tests | MockMvc + PostgreSQL Testcontainers |

The code is a reference implementation continuing your Lesson 13 architecture. It has not been compiled or tested against your local workspace here. Reconcile the actual `AppUserEntity` constructor, security configuration, previous migration schema, and Angular routes before considering it verified.

## 27. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="shield-check" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 15 — Roles, Instructor Ownership, and Resource Authorization**

    We'll move beyond role checks and enforce who actually owns each course. This includes instructor onboarding, safe role assignment, ownership-aware queries, `@PreAuthorize`, service-level authorization, 403 versus 404 decisions, and PostgreSQL integration tests.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 15: role-based and resource-level authorization, instructor onboarding, trusted role assignment, authenticated principal IDs, course ownership checks, @PreAuthorize, service-level authorization, safe 403 versus 404 behavior, PostgreSQL Testcontainers and MockMvc tests, Angular instructor route guards, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 14 with experienced-Java pacing.")}>Start Lesson 15 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 14 deliverable:** A LearnHub backend and Angular frontend prepared for real session-based registration, login, current-user retrieval, and logout, with BCrypt credentials, CSRF protection, and database-backed authentication tests.

The central lesson is that **a successful login must establish a durable, protected server-side security context—not merely return a successful HTTP response**.