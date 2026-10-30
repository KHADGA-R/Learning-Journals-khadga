# LearnHub Java & Spring Boot — Lesson 15

<text color="secondary" size="sm">Roles · Instructor Onboarding · Course Ownership · Method Security · PostgreSQL Integration Tests · Angular Guards</text>

**Stack:** Java 21 · Spring Boot 3.5.x · Spring Security 6.x · PostgreSQL · Spring Data JPA · Flyway · JUnit 5 · MockMvc · Testcontainers · Angular standalone

**Estimated time:** 7–9 hours, including implementation and exercises.

## 1. The security problem we're solving

In Lesson 14, LearnHub gained registration, login, authenticated sessions, logout, and a principal containing the database user ID.

But there's still a critical authorization gap.

Suppose two instructors are logged in:

- Alex owns *Angular Complete Guide*.
- Sarah owns *Python for Beginners*.

Both have `ROLE_INSTRUCTOR`.

Without ownership checks, Alex might be able to edit or delete Sarah's course simply by changing the course ID in an HTTP request.

**Lesson 15 changes the authorization rule from “is this user an instructor?” to “is this user allowed to modify this particular course?”**

We'll implement defense in depth:

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" padding={3} radius="lg" align="center" gap={1}>
    **Authenticated HTTP request**
    <text color="secondary" size="xs">Session cookie + CSRF token</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **SecurityFilterChain**
    <text color="secondary" size="xs">Instructor or Admin role required</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **@PreAuthorize**
    <text color="secondary" size="xs">Method-level role restriction</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Transactional course service**
    <text color="secondary" size="xs">Owner ID must match authenticated user ID</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" padding={3} radius="lg" align="center" gap={1}>
    **PostgreSQL**
    <text color="secondary" size="xs">courses.instructor_user_id → app_users.id</text>
  </box>
</box>

### What you'll build

| Capability | Target behavior |
|---|---|
| Public catalog | Anyone can browse published courses |
| Registration | Creates students only |
| Instructor onboarding | Trusted admin promotes a student |
| Course creation | Automatically assigns authenticated instructor as owner |
| Course update | Owner or admin only |
| Publish/archive | Owner or admin only |
| Course deletion | Owner or admin only, subject to lifecycle rules |
| Unauthorized modification | `403 Forbidden` |
| Missing or hidden course | `404 Not Found` where appropriate |
| Angular instructor pages | Protected navigation, with backend enforcement |

**Important:** Angular route guards improve the user experience. They do not provide security. Every authorization decision must remain enforced on the backend.

---

## 2. Verify your Lesson 14 baseline

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info

git status
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 14 baseline is not passing."
}
```

Inspect your existing model and migration history:

```powershell
Get-ChildItem src\main\resources\db\migration |
    Sort-Object Name

Get-Content src\main\java\com\learnhub\security\LearnHubPrincipal.java

Get-Content src\main\java\com\learnhub\persistence\CourseEntity.java

Get-Content src\main\java\com\learnhub\persistence\AppUserEntity.java

Get-Content src\main\java\com\learnhub\service\CourseManagementService.java
```

We will **extend these existing classes**, not create competing entities or services.

### Critical compatibility checks

Earlier lessons introduced two overlapping representations of course ownership: a human-readable instructor name and a foreign key to `app_users`.

For security, the foreign key is authoritative:

```text
courses.instructor_user_id
```

The text field:

```text
courses.instructor
```

is display data only. It must never be trusted as proof of ownership.

Also, the Lesson 14 controller used a custom JSON login flow. Its `LearnHubPrincipal.id()` is the authenticated user ID we'll use here.

---

## 3. Database ownership model

Lesson 9 created the ownership column and foreign key:

```sql
courses.instructor_user_id
    REFERENCES app_users(id)
```

Lesson 10 introduced JPA relationships.

Our target structure is:

```text
app_users
---------
id                  BIGINT PK
email               VARCHAR UNIQUE (case-insensitive index)
display_name        VARCHAR
password_hash       VARCHAR
role                VARCHAR
enabled             BOOLEAN

courses
-------
id                  BIGINT PK
title               VARCHAR
instructor          VARCHAR
instructor_user_id  BIGINT FK → app_users.id
status              VARCHAR
...
```

A course can have one owner. An instructor can own many courses.

### Why ownership must be stored in the database

A malicious client can modify JSON fields such as:

```json
{
  "instructor": "Sarah Lee"
}
```

or invent:

```json
{
  "instructorUserId": 42
}
```

Neither should change the course owner.

**The backend must derive ownership from the authenticated principal.**

---

## 4. Flyway migration: prepare ownership enforcement

Existing seeded courses may have `instructor_user_id = NULL`.

We must not make the column `NOT NULL` until those legacy records have been assigned to real accounts.

Create the next unused Flyway migration. The filename below assumes Lesson 14 did not add a migration after V6.

**File:** `src/main/resources/db/migration/V7__course_owner_index.sql`

<CodeBlock language="sql" editable>
-- Lesson 15: prepare efficient ownership queries.
--
-- The instructor_user_id column and foreign key
-- already exist from the Lesson 9 schema.
--
-- Do not make it NOT NULL yet: legacy seeded courses
-- may still have no associated instructor account.

CREATE INDEX IF NOT EXISTS
    idx_courses_owner_status_id
ON courses (
    instructor_user_id,
    status,
    id
);
</CodeBlock>

The index supports instructor-specific management queries.

### Inspect legacy courses

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT id, title, instructor, instructor_user_id, status FROM courses ORDER BY id;"
```

For any legacy course with a null owner, choose a deliberate backfill strategy.

Do not automatically match owners by display name: names are neither unique nor authoritative.

**Lesson 15 policy:** Unowned legacy courses are editable only by admins until ownership is explicitly assigned.

---

## 5. Role model and trusted instructor onboarding

We'll preserve the existing roles:

```java
public enum SecurityRole {
    STUDENT,
    INSTRUCTOR,
    ADMIN
}
```

### Authorization matrix

| Operation | Student | Instructor | Admin |
|---|---|---|---|
| Browse published courses | Yes | Yes | Yes |
| Register as student | Yes | Yes | Yes |
| Become instructor | No self-promotion | Already instructor | Admin approval |
| Create course | No | Yes | Yes |
| Edit owned course | No | Yes | Yes |
| Edit another instructor's course | No | No | Yes |
| Publish owned draft | No | Yes | Yes |
| Archive owned published course | No | Yes | Yes |
| Delete owned draft | No | Yes | Yes |
| Assign legacy course owner | No | No | Yes |

We will create an admin-only instructor promotion endpoint.

### Why not let registration accept a role?

Because a client-controlled role is a privilege escalation vulnerability.

This is unsafe:

```java
user.setRole(request.role());
```

The registration service from Lesson 14 correctly assigns `STUDENT` server-side.

### A note on administrative bootstrap

Do not create a publicly accessible “first user becomes admin” endpoint.

For the learning environment, admin accounts should be provisioned through a trusted, local-only administrative process or isolated Testcontainers fixtures.

In production, use a controlled operational workflow with audit logging and appropriate approval.

## 6. Add method security

**Modify:** `src/main/java/com/learnhub/config/SecurityConfig.java`

Add `@EnableMethodSecurity` to the existing configuration class:

```java
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@Configuration
@EnableMethodSecurity
public class SecurityConfig {
    // Keep the Lesson 14 beans and filter chain.
}
```

Do not create another `SecurityFilterChain`.

`@EnableMethodSecurity` activates annotations such as `@PreAuthorize` on Spring-managed beans.

For example:

```java
@PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
```

restricts a method to instructors and administrators.

**Important distinction:** `@PreAuthorize` checks permissions at method invocation time. We will still enforce course ownership inside the transactional service, using a database-backed owner ID.

This keeps the authorization check close to the state-changing operation and avoids trusting a stale UI or a controller-supplied user ID.

## 7. Add an authorization exception

**Create:** `src/main/java/com/learnhub/security/CourseAccessDeniedException.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import org.springframework.security.access.AccessDeniedException;

/**
 * Raised when an authenticated principal lacks
 * permission to modify a specific course.
 */
public final class CourseAccessDeniedException
        extends AccessDeniedException {

    public CourseAccessDeniedException() {
        super("You are not allowed to modify this course.");
    }
}
</CodeBlock>

Why extend Spring Security's `AccessDeniedException`?

It expresses the correct security semantics and allows the application to consistently map denied operations to HTTP `403`.

### Add a safe HTTP error mapping

The Lesson 14 `SecurityFilterChain` already handles many access denials at the filter level.

For service-layer denials, add a handler to the existing `GlobalExceptionHandler`.

**Modify:** `src/main/java/com/learnhub/web/error/GlobalExceptionHandler.java`

<CodeBlock language="java" editable>
@org.springframework.web.bind.annotation.ExceptionHandler(
    org.springframework.security.access.AccessDeniedException.class
)
public org.springframework.http.ResponseEntity<
        org.springframework.http.ProblemDetail> accessDenied(
        org.springframework.security.access.AccessDeniedException ex
) {
    var problem =
        org.springframework.http.ProblemDetail.forStatusAndDetail(
            org.springframework.http.HttpStatus.FORBIDDEN,
            "You do not have permission to perform this action."
        );

    problem.setTitle("Access denied");

    return org.springframework.http.ResponseEntity
        .status(org.springframework.http.HttpStatus.FORBIDDEN)
        .body(problem);
}
</CodeBlock>

Keep the error message generic. Do not expose another instructor's user ID, email, or private course data.

---

## 8. Create a reusable authenticated-user accessor

Instead of passing user IDs from Angular, retrieve the current principal from Spring Security.

**Create:** `src/main/java/com/learnhub/security/CurrentActor.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import org.springframework.security.authentication.AnonymousAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.authentication.AuthenticationCredentialsNotFoundException;
import org.springframework.stereotype.Component;

/**
 * Provides the authenticated database user ID and role.
 *
 * Never accepts an identity from HTTP request JSON.
 */
@Component
public class CurrentActor {

    public LearnHubPrincipal require() {
        Authentication authentication =
                SecurityContextHolder.getContext()
                        .getAuthentication();

        if (authentication == null
                || !authentication.isAuthenticated()
                || authentication instanceof
                        AnonymousAuthenticationToken
                || !(authentication.getPrincipal()
                        instanceof LearnHubPrincipal principal)) {
            throw new AuthenticationCredentialsNotFoundException(
                    "Authentication is required."
            );
        }

        return principal;
    }

    public long id() {
        return require().id();
    }

    public boolean isAdmin() {
        return require().role() == SecurityRole.ADMIN;
    }
}
</CodeBlock>

The explicit principal type check is intentional.

It prevents a mocked or unexpected authentication object from silently being interpreted as a real LearnHub account.

For tests that exercise these service methods, use real `LearnHubPrincipal` objects or authenticate through the database. A plain `@WithMockUser` creates a different principal type.

---

## 9. Extend the course JPA repository

**Modify:** `src/main/java/com/learnhub/persistence/JpaCourseRepository.java`

Keep all existing methods. Add the following:

<CodeBlock language="java" editable>
/**
 * Checks ownership without loading the full entity.
 *
 * This is useful for read-only permission checks,
 * but mutation services should still load and
 * authorize the entity in their transaction.
 */
boolean existsByIdAndInstructorAccount_Id(
        Long courseId,
        Long instructorId
);

/**
 * Instructor dashboard query.
 *
 * The method name assumes CourseEntity has:
 *
 * @ManyToOne(fetch = FetchType.LAZY)
 * @JoinColumn(name = "instructor_user_id")
 * private AppUserEntity instructorAccount;
 */
org.springframework.data.domain.Page<CourseEntity>
findByInstructorAccount_Id(
        Long instructorId,
        org.springframework.data.domain.Pageable pageable
);
</CodeBlock>

If your Lesson 10 association has a different Java property name, adjust the repository method names to match it.

### Why use a database user ID?

An instructor can change their display name. Multiple users can have the same display name.

A foreign key to `app_users.id` is stable and referentially enforced.

### Transactional mutation strategy

For ownership-sensitive mutations, we'll load the managed entity, check its owner, change the entity, and let Hibernate flush at transaction commit.

For a more advanced concurrency design, we could use a conditional update such as:

```sql
UPDATE courses
SET ...
WHERE id = :courseId
  AND instructor_user_id = :actorId
```

That can make ownership and mutation part of one database statement.

For this lesson, we'll use a clear transactional service and discuss concurrency boundaries explicitly.

---

## 10. Implement ownership checks

**Create:** `src/main/java/com/learnhub/security/CourseOwnershipPolicy.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.persistence.CourseEntity;

import org.springframework.stereotype.Component;

/**
 * Resource-level authorization.
 *
 * Admins may manage any course.
 * Instructors may manage only their own courses.
 * Legacy courses without an owner are admin-only.
 */
@Component
public class CourseOwnershipPolicy {

    public void requireCanManage(
            LearnHubPrincipal actor,
            CourseEntity course
    ) {
        if (actor.role() == SecurityRole.ADMIN) {
            return;
        }

        if (actor.role() != SecurityRole.INSTRUCTOR) {
            throw new CourseAccessDeniedException();
        }

        if (course.getInstructorAccount() == null) {
            throw new CourseAccessDeniedException();
        }

        Long ownerId =
                course.getInstructorAccount().getId();

        if (!actor.id().equals(ownerId)) {
            throw new CourseAccessDeniedException();
        }
    }
}
</CodeBlock>

This is the central ownership rule.

A course's title, instructor display name, and client-supplied fields have no authority over ownership.

### Avoid unnecessary lazy loading

Calling `course.getInstructorAccount().getId()` on a Hibernate lazy association generally allows access to the identifier without fully initializing the associated entity.

However, do not build security correctness around avoiding a particular SQL query. Verify the actual Hibernate behavior in integration tests, and keep the operation inside a transaction.

---

## 11. Create a transactional ownership-aware course service

We'll introduce a dedicated service for secure mutations. It can call the existing domain lifecycle logic or update the managed entity directly.

To avoid duplicating your Lesson 11 validation and mapping rules, we'll first implement the ownership boundary and lifecycle operations, then connect draft creation and editing.

**Create:** `src/main/java/com/learnhub/service/AuthorizedCourseService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.security.CourseOwnershipPolicy;
import com.learnhub.security.CurrentActor;
import com.learnhub.security.LearnHubPrincipal;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.function.Consumer;

/**
 * Ownership-aware course lifecycle operations.
 *
 * All mutations run inside a transaction.
 */
@Service
public class AuthorizedCourseService {

    private final JpaCourseRepository courses;
    private final CurrentActor currentActor;
    private final CourseOwnershipPolicy ownership;

    public AuthorizedCourseService(
            JpaCourseRepository courses,
            CurrentActor currentActor,
            CourseOwnershipPolicy ownership
    ) {
        this.courses = courses;
        this.currentActor = currentActor;
        this.ownership = ownership;
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional
    public void publish(long courseId) {
        mutate(courseId, course -> {
            if (course.getStatus() != CourseStatus.DRAFT) {
                throw new IllegalStateException(
                    "Only draft courses can be published."
                );
            }

            course.changeStatus(CourseStatus.PUBLISHED);
        });
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional
    public void archive(long courseId) {
        mutate(courseId, course -> {
            if (course.getStatus() != CourseStatus.PUBLISHED) {
                throw new IllegalStateException(
                    "Only published courses can be archived."
                );
            }

            course.changeStatus(CourseStatus.ARCHIVED);
        });
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional
    public void delete(long courseId) {
        LearnHubPrincipal actor = currentActor.require();

        CourseEntity course = courses.findById(courseId)
                .orElseThrow(() ->
                    new com.learnhub.service
                        .CourseNotFoundException(courseId)
                );

        ownership.requireCanManage(actor, course);

        if (course.getStatus() == CourseStatus.PUBLISHED) {
            throw new IllegalStateException(
                "Published courses cannot be deleted."
            );
        }

        courses.delete(course);
        courses.flush();
    }

    private void mutate(
            long courseId,
            Consumer<CourseEntity> change
    ) {
        LearnHubPrincipal actor = currentActor.require();

        CourseEntity course = courses.findById(courseId)
                .orElseThrow(() ->
                    new com.learnhub.service
                        .CourseNotFoundException(courseId)
                );

        ownership.requireCanManage(actor, course);

        change.accept(course);

        // Managed JPA entity: Hibernate flushes changes
        // when the transaction commits.
    }
}
</CodeBlock>

**Compatibility adjustment:** The earlier course exception may be located at `com.learnhub.exception.CourseNotFoundException` rather than `com.learnhub.service.CourseNotFoundException`, and its constructor may accept a message rather than a numeric ID. Use your existing exception type and constructor.

Similarly, the reference assumes `CourseEntity` has `getStatus()` and `changeStatus(CourseStatus)`, as introduced earlier.

### A deliberate lifecycle rule

We check ownership **before** validating whether a course can be published, archived, or deleted.

This avoids revealing lifecycle details to a caller who has no right to manage the course.

### Why call `flush()` on deletion?

A course may have enrollments or other dependent records protected by foreign keys.

Flushing inside the transaction surfaces a database constraint violation before the method returns.

For production, translate known dependency conflicts into an appropriate `409 Conflict` without exposing SQL internals.

---

## 12. Secure course creation and editing

The most important rule for creation is:

**The owner comes from the session, not the request body.**

The Lesson 14 `LearnHubPrincipal` already has the authenticated user ID.

### 12.1 Add owner assignment to CourseEntity

**Modify:** `src/main/java/com/learnhub/persistence/CourseEntity.java`

Ensure the association exists:

<CodeBlock language="java" editable>
@ManyToOne(fetch = FetchType.LAZY, optional = true)
@JoinColumn(name = "instructor_user_id")
private AppUserEntity instructorAccount;

public AppUserEntity getInstructorAccount() {
    return instructorAccount;
}

/**
 * Called only by trusted application services.
 */
public void assignInstructorAccount(
        AppUserEntity instructorAccount
) {
    this.instructorAccount =
            java.util.Objects.requireNonNull(
                instructorAccount
            );
}
</CodeBlock>

Use these imports if not already present:

```java
import jakarta.persistence.ManyToOne;
import jakarta.persistence.FetchType;
import jakarta.persistence.JoinColumn;
```

Do not duplicate the association if Lesson 10 already created it.

### 12.2 Add owner-aware draft creation

**Modify:** `src/main/java/com/learnhub/service/AuthorizedCourseService.java`

Add a dependency:

```java
private final com.learnhub.persistence.JpaAppUserRepository users;
```

Add it to the constructor and assign it.

Then add:

<CodeBlock language="java" editable>
@PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
@Transactional
public CourseEntity createDraft(
        com.learnhub.application.CourseUpsertCommand command
) {
    LearnHubPrincipal actor = currentActor.require();

    AppUserEntity owner = users.findById(actor.id())
            .orElseThrow(() ->
                new IllegalStateException(
                    "Authenticated user no longer exists."
                )
            );

    // Reuse the existing Lesson 8 entity factory.
    CourseEntity course = CourseEntity.newDraft(
            command.title(),
            command.description(),
            owner.getDisplayName(),
            command.category(),
            command.price(),
            command.free(),
            command.level(),
            command.rating(),
            command.thumbnail(),
            command.durationHours()
    );

    // Never accept ownerId from client JSON.
    course.assignInstructorAccount(owner);

    return courses.saveAndFlush(course);
}
</CodeBlock>

Add:

```java
import com.learnhub.persistence.AppUserEntity;
```

**Important:** This reference uses the `CourseEntity.newDraft(...)` factory shape introduced in Lesson 7. If Lesson 11 changed that factory, use its current signature.

For this lesson, even an admin creating a course becomes its owner. A future administrative endpoint may explicitly assign an instructor, but it must be a separate privileged operation.

### 12.3 Add owner-aware draft editing

Add to the same service:

<CodeBlock language="java" editable>
@PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
@Transactional
public CourseEntity updateDraft(
        long courseId,
        com.learnhub.application.CourseUpsertCommand command
) {
    LearnHubPrincipal actor = currentActor.require();

    CourseEntity course = courses.findById(courseId)
            .orElseThrow(() ->
                new com.learnhub.exception
                    .CourseNotFoundException(courseId)
            );

    ownership.requireCanManage(actor, course);

    if (course.getStatus() != CourseStatus.DRAFT) {
        throw new IllegalStateException(
            "Only draft courses can be edited."
        );
    }

    // Reuse the existing entity update method.
    // Do not change instructorAccount.
    course.replaceDraftFields(
            command.title(),
            command.description(),
            command.instructor(),
            command.category(),
            command.price(),
            command.free(),
            command.level(),
            command.rating(),
            command.thumbnail(),
            command.durationHours()
    );

    return course;
}
</CodeBlock>

**Adapt the exception package and `replaceDraftFields` signature to the actual Lesson 11 implementation.** In particular, if your entity update method expects a date or omits `instructor`, preserve its existing contract.

### Security refinement: don't trust the instructor display field

The earlier `CourseUpsertCommand` includes an `instructor` string because the original LearnHub model used it for display.

From this lesson forward, that string should be **server-controlled** for instructor-owned courses.

For new drafts, use `owner.getDisplayName()`.

For updates, preserve the existing instructor display value or derive it from the owner record. Do not allow a client to impersonate another instructor by modifying the display field.

A safe update call therefore passes:

```java
course.getInstructor()
```

instead of:

```java
command.instructor()
```

if your existing entity method still requires that argument.

Eventually, the response DTO can derive the display name directly from the owner association, with a fallback for legacy courses.

---

## 13. Connect the controller to the secure service

**Modify:** `src/main/java/com/learnhub/web/CourseController.java`

Inject `AuthorizedCourseService` alongside the existing read-only catalog service and DTO mapping components.

Replace the **write-path service calls** with the secure equivalents.

The route names and HTTP methods must remain consistent with your Lesson 11 API contract.

### Creation

<CodeBlock language="java" editable>
@PostMapping
public ResponseEntity<CourseResponse> create(
        @Valid @RequestBody CourseWriteRequest request
) {
    CourseEntity saved =
            authorizedCourseService.createDraft(
                request.toCommand()
            );

    CourseResponse response =
            courseResponseMapper.fromEntity(saved);

    return ResponseEntity
        .created(
            URI.create("/api/v1/courses/" + saved.getId())
        )
        .body(response);
}
</CodeBlock>

Here `courseResponseMapper.fromEntity(...)` represents your existing Lesson 11 DTO mapping. **Do not create an undefined mapper bean:** use the mapper already present in your project, or your existing `CourseResponse.from(...)` conversion with the metadata it requires.

### Update

```java
@PutMapping("/{id}")
public CourseResponse update(
        @PathVariable long id,
        @Valid @RequestBody CourseWriteRequest request
) {
    CourseEntity updated =
            authorizedCourseService.updateDraft(
                id,
                request.toCommand()
            );

    return courseResponseMapper.fromEntity(updated);
}
```

### Publish, archive, and delete

```java
@PatchMapping("/{id}/publish")
public ResponseEntity<Void> publish(
        @PathVariable long id
) {
    authorizedCourseService.publish(id);
    return ResponseEntity.noContent().build();
}

@PatchMapping("/{id}/archive")
public ResponseEntity<Void> archive(
        @PathVariable long id
) {
    authorizedCourseService.archive(id);
    return ResponseEntity.noContent().build();
}

@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(
        @PathVariable long id
) {
    authorizedCourseService.delete(id);
    return ResponseEntity.noContent().build();
}
```

These snippets are **replacement methods**, not additional routes. Remove or modify the old controller methods so each route has exactly one handler.

### Why this controller refactor is necessary

Protecting only a new `AuthorizedCourseService` does nothing if the controller still calls the older, unprotected `CourseManagementService`.

Every externally reachable course mutation must pass through the ownership-aware path.

Also inspect other services, scheduled jobs, and admin endpoints for calls to the old mutation methods. Internal operations should use explicit trusted workflows rather than accidentally bypassing authorization.

---

## 14. Trusted instructor onboarding

We will implement a small administrative service that promotes an existing student to instructor.

The public registration endpoint remains unchanged.

### 14.1 Create InstructorOnboardingService

**Create:** `src/main/java/com/learnhub/security/InstructorOnboardingService.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.persistence.AppUserEntity;
import com.learnhub.persistence.JpaAppUserRepository;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class InstructorOnboardingService {

    private final JpaAppUserRepository users;

    public InstructorOnboardingService(
            JpaAppUserRepository users    ) {
        this.users = users;
    }

    /**
     * Trusted role transition.
     *
     * Students can become instructors.
     * Admin accounts are never demoted here.
     */
    @PreAuthorize("hasRole('ADMIN')")
    @Transactional
    public void approveInstructor(long userId) {
        AppUserEntity user = users.findById(userId)
                .orElseThrow(() ->
                    new IllegalArgumentException(
                        "User does not exist."
                    )
                );

        if (user.getRole() == SecurityRole.ADMIN) {
            throw new IllegalStateException(
                "Cannot modify an admin using this operation."
            );
        }

        if (!user.isEnabled()) {
            throw new IllegalStateException(
                "Disabled accounts cannot be promoted."
            );
        }

        user.setRole(SecurityRole.INSTRUCTOR);
    }
}
</CodeBlock>

### 14.2 Create the admin controller

**Create:** `src/main/java/com/learnhub/web/AdminInstructorController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.security.InstructorOnboardingService;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/admin/instructors")
public class AdminInstructorController {

    private final InstructorOnboardingService onboarding;

    public AdminInstructorController(
            InstructorOnboardingService onboarding
    ) {
        this.onboarding = onboarding;
    }

    @PostMapping("/{userId}/approve")
    public ResponseEntity<Void> approve(
            @PathVariable long userId
    ) {
        onboarding.approveInstructor(userId);
        return ResponseEntity.noContent().build();
    }
}
</CodeBlock>

### 14.3 Add the admin security rule

**Modify:** `src/main/java/com/learnhub/config/SecurityConfig.java`

Before `.anyRequest().denyAll()`, add:

```java
.requestMatchers(
    HttpMethod.POST,
    "/api/v1/admin/instructors/*/approve"
).hasRole("ADMIN")
```

CSRF remains required.

### Session role refresh

After promotion, an already logged-in student may still have `ROLE_STUDENT` in their existing session principal.

That's expected: session authorities do not automatically update when a database role changes.

For this lesson, require the promoted user to **sign out and sign back in**.

Production systems may need session revocation or a role-version mechanism for immediate privilege changes, especially for demotions.

### Production onboarding is more than role assignment

Before public deployment, add an approval record, audit trail, eligibility checks, and administrative identity verification. The endpoint above is a minimal trusted-role transition, not a complete instructor application system.

---

## 15. PostgreSQL ownership integration tests

We'll use the real PostgreSQL schema through Testcontainers and exercise the HTTP security chain with MockMvc.

The tests need two instructors, one student, and courses with explicit owners.

### Test fixture design

<box border radius="lg" padding={3} gap={2}>
  <grid columns={2} gap={2}>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={3} gap={1}>
        **Alex — Instructor**
        <text color="secondary" size="xs">Owns Course A</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={3} gap={1}>
        **Sarah — Instructor**
        <text color="secondary" size="xs">Owns Course B</text>
      </box>
    </grid-item>
  </grid>
  <box border radius="md" padding={3} gap={1}>
    **Maya — Student**
    <text color="secondary" size="xs">Owns no courses; cannot perform instructor operations</text>
  </box>
</box>

### 15.1 Create CourseOwnershipIntegrationTest

**Create:** `src/test/java/com/learnhub/security/CourseOwnershipIntegrationTest.java`

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
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests ownership against actual PostgreSQL rows.
 *
 * Spring Security's user() helper supplies a
 * LearnHubPrincipal; the ownership service then
 * checks the database foreign key.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class CourseOwnershipIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired MockMvc mvc;
    @Autowired JdbcTemplate jdbc;
    @Autowired PasswordEncoder encoder;

    private long alexId;
    private long sarahId;
    private long studentId;

    private long alexCourseId;
    private long sarahCourseId;

    @BeforeEach
    void setup() {
        alexId = insertUser(
                "alex-owner@example.test",
                "Alex",
                "INSTRUCTOR"
        );

        sarahId = insertUser(
                "sarah-owner@example.test",
                "Sarah",
                "INSTRUCTOR"
        );

        studentId = insertUser(
                "maya-owner@example.test",
                "Maya",
                "STUDENT"
        );

        alexCourseId = insertCourse(
                "Alex's Ownership Course",
                alexId
        );

        sarahCourseId = insertCourse(
                "Sarah's Ownership Course",
                sarahId
        );
    }

    private long insertUser(
            String email,
            String name,
            String role
    ) {
        return jdbc.queryForObject("""
                INSERT INTO app_users (
                    email, display_name,
                    password_hash, role, enabled
                )
                VALUES (?, ?, ?, ?, true)
                RETURNING id
                """,
                Long.class,
                email,
                name,
                encoder.encode("TestPassword123!"),
                role
        );
    }

    private long insertCourse(
            String title,
            long ownerId
    ) {
        return jdbc.queryForObject("""
                INSERT INTO courses (
                    title, description, instructor,
                    category, price, is_free,
                    level, rating, status,
                    instructor_user_id
                )
                VALUES (
                    ?, 'Ownership integration fixture',
                    'Fixture Instructor',
                    'Programming', 0.00, true,
                    'BEGINNER', 4.5, 'DRAFT', ?
                )
                RETURNING id
                """,
                Long.class,
                title,
                ownerId
        );
    }

    private LearnHubPrincipal principal(
            long id,
            String email,
            SecurityRole role
    ) {
        return new LearnHubPrincipal(
                id,
                email,
                encoder.encode("TestPassword123!"),
                role,
                true
        );
    }

    @Test
    void ownerCanPublishOwnDraft() throws Exception {
        mvc.perform(patch(
                    "/api/v1/courses/{id}/publish",
                    alexCourseId
                )
                .with(user(principal(
                    alexId,
                    "alex-owner@example.test",
                    SecurityRole.INSTRUCTOR
                )))
                .with(csrf()))
            .andExpect(status().isNoContent());

        String status = jdbc.queryForObject(
                "SELECT status FROM courses WHERE id = ?",
                String.class,
                alexCourseId
        );

        assertEquals("PUBLISHED", status);
    }

    @Test
    void instructorCannotPublishAnotherOwnersDraft()
            throws Exception {
        mvc.perform(patch(
                    "/api/v1/courses/{id}/publish",
                    sarahCourseId
                )
                .with(user(principal(
                    alexId,
                    "alex-owner@example.test",
                    SecurityRole.INSTRUCTOR
                )))
                .with(csrf()))
            .andExpect(status().isForbidden());

        String status = jdbc.queryForObject(
                "SELECT status FROM courses WHERE id = ?",
                String.class,
                sarahCourseId
        );

        assertEquals("DRAFT", status);
    }

    @Test
    void adminCanPublishAnotherOwnersDraft()
            throws Exception {
        long adminId = insertUser(
                "admin-owner@example.test",
                "Admin",
                "ADMIN"
        );

        mvc.perform(patch(
                    "/api/v1/courses/{id}/publish",
                    sarahCourseId
                )
                .with(user(principal(
                    adminId,
                    "admin-owner@example.test",
                    SecurityRole.ADMIN
                )))
                .with(csrf()))
            .andExpect(status().isNoContent());
    }

    @Test
    void studentCannotPublishAnyCourse()
            throws Exception {
        mvc.perform(patch(
                    "/api/v1/courses/{id}/publish",
                    alexCourseId
                )
                .with(user(principal(
                    studentId,
                    "maya-owner@example.test",
                    SecurityRole.STUDENT
                )))
                .with(csrf()))
            .andExpect(status().isForbidden());
    }

    @Test
    void ownerCannotPublishWithoutCsrf()
            throws Exception {
        mvc.perform(patch(
                    "/api/v1/courses/{id}/publish",
                    alexCourseId
                )
                .with(user(principal(
                    alexId,
                    "alex-owner@example.test",
                    SecurityRole.INSTRUCTOR
                ))))
            .andExpect(status().isForbidden());
    }

    @Test
    void anonymousCannotPublishCourse()
            throws Exception {
        mvc.perform(patch(
                    "/api/v1/courses/{id}/publish",
                    alexCourseId
                )
                .with(csrf()))
            .andExpect(status().isUnauthorized());
    }
}
</CodeBlock>

This test class verifies six important security cases.

### Test prerequisites

The tests assume the existing migration creates `courses` with defaults for `last_updated` and nullable metadata, and that the controller's publish route returns `204 No Content`.

If Lesson 11's publish endpoint still returns a course DTO with `200 OK`, update the success assertions to match that contract. Do not weaken the ownership assertions.

### A valuable extra assertion

In the denied-owner test, we verify the course remains `DRAFT`.

That's important: testing only the HTTP status does not prove the database mutation was prevented.

---

## 16. Test instructor promotion

**Create:** `src/test/java/com/learnhub/security/InstructorOnboardingIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.security;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class InstructorOnboardingIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired MockMvc mvc;
    @Autowired JdbcTemplate jdbc;

    @Test
    void adminCanPromoteStudent() throws Exception {
        Long studentId = jdbc.queryForObject("""
                INSERT INTO app_users (
                    email, display_name, role, enabled
                )
                VALUES (
                    'promote-student@example.test',
                    'Student',
                    'STUDENT',
                    true
                )
                RETURNING id
                """,
                Long.class
        );

        mvc.perform(post(
                    "/api/v1/admin/instructors/{id}/approve",
                    studentId
                )
                .with(user("admin").roles("ADMIN"))
                .with(csrf()))
            .andExpect(status().isNoContent());

        String role = jdbc.queryForObject(
                "SELECT role FROM app_users WHERE id = ?",
                String.class,
                studentId
        );

        assertEquals("INSTRUCTOR", role);
    }

    @Test
    void studentCannotPromoteSelf() throws Exception {
        Long studentId = jdbc.queryForObject("""
                INSERT INTO app_users (
                    email, display_name, role, enabled
                )
                VALUES (
                    'self-promote@example.test',
                    'Student',
                    'STUDENT',
                    true
                )
                RETURNING id
                """,
                Long.class
        );

        mvc.perform(post(
                    "/api/v1/admin/instructors/{id}/approve",
                    studentId
                )
                .with(user("student").roles("STUDENT"))
                .with(csrf()))
            .andExpect(status().isForbidden());

        String role = jdbc.queryForObject(
                "SELECT role FROM app_users WHERE id = ?",
                String.class,
                studentId
        );

        assertEquals("STUDENT", role);
    }
}
</CodeBlock>

These two tests verify that role assignment is controlled by a trusted admin path.

### Security testing depth

The onboarding tests use `@WithMockUser`-style request principals to verify role authorization.

The ownership tests use a real `LearnHubPrincipal` with a database user ID to verify resource authorization.

In a later end-to-end test, we'll authenticate through the Lesson 14 login endpoint and reuse the resulting session for a course mutation. That tests the entire chain, including the database credential lookup.

---

## 17. Safe 403 versus 404 decisions

Authorization failures need consistent semantics.

### Recommended LearnHub policy

| Scenario | Response |
|---|---|
| Anonymous request to protected write API | `401 Unauthorized` |
| Student attempts instructor-only operation | `403 Forbidden` |
| Instructor attempts to edit another instructor's known course | `403 Forbidden` |
| Course ID does not exist | `404 Not Found` |
| Anonymous user requests unpublished course | `404 Not Found` |
| Instructor tries to publish an already published course | `409 Conflict` |
| Instructor tries to delete an enrolled course | `409 Conflict`, if deletion is blocked |

### When should a forbidden course return 404 instead?

For sensitive resources, returning `404` can reduce information leakage about whether a record exists.

But consistency matters.

If the application exposes public course IDs and the course is already published, a `403` for unauthorized modification can be reasonable.

For unpublished drafts, you may choose to conceal existence and return `404` to non-owners.

**Lesson 15 implementation uses `403` for existing courses that the caller cannot manage.** A future hardened API can adopt concealment consistently across all relevant endpoints.

### Preventing information leakage

Do not return:

```text
You cannot edit Sarah Lee's course owned by user 27.
```

Return a generic denial.

Also avoid exposing draft details through a public `GET` endpoint. Public catalog queries must continue filtering by `PUBLISHED` status.

---

## 18. Angular instructor route guards

Angular should prevent students from navigating into instructor-only pages.

But the backend remains the source of truth.

### 18.1 Create the guard

**Create:** `learnhub/src/app/core/auth/instructor.guard.ts`

<CodeBlock language="typescript" editable>
import { inject } from '@angular/core';

import {
  CanActivateFn,
  Router,
} from '@angular/router';

import {
  catchError,
  map,
  of,
  switchMap,
} from 'rxjs';

import { AuthService } from './auth.service';

/**
 * UX navigation guard.
 *
 * Never treat this as backend authorization.
 */
export const instructorGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  const resolve = () => {
    const user = auth.currentUser();

    if (!user) {
      return router.createUrlTree(['/login']);
    }

    if (
      user.role === 'INSTRUCTOR' ||
      user.role === 'ADMIN'
    ) {
      return true;
    }

    return router.createUrlTree(['/']);
  };

  // Revalidate against the server so a browser refresh
  // or stale in-memory state does not make the decision
  // solely from an uninitialized signal.
  return auth.loadCurrentUser().pipe(
    map(() => resolve()),
    catchError(() =>
      of(router.createUrlTree(['/login']))
    )
  );
};
</CodeBlock>

Remove unused RxJS imports (`switchMap`) if your TypeScript lint rules enforce that.

### Why call `/auth/me`?

Angular's `currentUser` signal is reset on page refresh.

The server session may still be valid.

Calling `/auth/me` restores the authenticated user state before deciding whether to allow navigation.

This is a straightforward implementation; a production application can cache an initialized auth state and avoid repeated requests during route navigation.

### 18.2 Add a guarded instructor route

**Modify:** `learnhub/src/app/app.routes.ts`

Add the guard to the existing instructor route rather than duplicating it:

<CodeBlock language="typescript" editable>
import { instructorGuard }
  from './core/auth/instructor.guard';

// Inside the existing routes array:
{
  path: 'instructor',
  canActivate: [instructorGuard],
  loadComponent: () =>
    import(
      './features/instructor/instructor-dashboard.component'
    ).then(m => m.InstructorDashboardComponent),
}
</CodeBlock>

The component path is illustrative. Use the actual instructor dashboard component if one already exists.

If it does not exist yet, create a minimal placeholder.

**Create:** `learnhub/src/app/features/instructor/instructor-dashboard.component.ts`

<CodeBlock language="typescript" editable>
import { Component } from '@angular/core';

@Component({
  selector: 'app-instructor-dashboard',
  standalone: true,
  template: `
    <main>
      <h1>Instructor dashboard</h1>
      <p>
        Your owned courses and publishing tools
        will appear here.
      </p>
    </main>
  `,
})
export class InstructorDashboardComponent {}
</CodeBlock>

### 18.3 Hide instructor navigation from students

In your existing navigation component, use the `AuthService.currentUser` signal:

```typescript
readonly auth = inject(AuthService);
```

Template:

```html
@if (
  auth.currentUser()?.role === 'INSTRUCTOR' ||
  auth.currentUser()?.role === 'ADMIN'
) {
  <a routerLink="/instructor">Instructor dashboard</a>
}
```

Hiding a link is only a presentation decision. A student can still type a URL or call the API directly, which is why the Spring Security rules and service ownership checks are mandatory.

---

## 19. Instructor-owned course listing

The instructor dashboard should show only courses owned by the authenticated instructor.

Do not accept an arbitrary `instructorId` query parameter for a normal instructor dashboard.

### Add the service

**Create:** `src/main/java/com/learnhub/service/InstructorCourseQueryService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.security.CurrentActor;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class InstructorCourseQueryService {

    private final JpaCourseRepository courses;
    private final CurrentActor actor;

    public InstructorCourseQueryService(
            JpaCourseRepository courses,
            CurrentActor actor
    ) {
        this.courses = courses;
        this.actor = actor;
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional(readOnly = true)
    public Page<CourseEntity> myCourses(
            int page,
            int size
    ) {
        if (page < 0 || size < 1 || size > 50) {
            throw new IllegalArgumentException(
                "Invalid pagination parameters."
            );
        }

        return courses.findByInstructorAccount_Id(
                actor.id(),
                PageRequest.of(
                    page,
                    size,
                    Sort.by(
                        Sort.Direction.DESC,
                        "id"
                    )
                )
        );
    }
}
</CodeBlock>

For admins, this endpoint still returns only courses they personally own. A separate admin query should provide broader access if required.

### DTO mapping requirement

Do not return `Page<CourseEntity>` directly from a controller.

JPA entities may contain lazy relationships and security-sensitive data.

Map results to explicit response DTOs such as `CourseResponse`, and return the existing Lesson 11 pagination envelope.

The mapping should occur inside a read-only transaction if it accesses lazy relationships.

### Endpoint contract

A suitable route is:

```text
GET /api/v1/instructor/courses?page=0&size=10
```

Add this matcher to `SecurityConfig`:

```java
.requestMatchers(
    HttpMethod.GET,
    "/api/v1/instructor/courses"
).hasAnyRole("INSTRUCTOR", "ADMIN")
```

Implement the controller using your existing course DTO mapper. The endpoint must derive the instructor ID from the session.

---

## 20. PowerShell API verification

Start the backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

In another PowerShell terminal:

```powershell
$base = "http://localhost:8080"
```

### 20.1 Verify public catalog access

```powershell
Invoke-RestMethod `
  "$base/api/v1/courses?page=0&size=5"
```

Expected: `200 OK`.

### 20.2 Verify anonymous writes are blocked

Obtain a CSRF token first:

```powershell
$session = New-Object Microsoft.PowerShell.Commands.WebRequestSession

$csrf = Invoke-RestMethod `
  "$base/api/v1/security/csrf" `
  -WebSession $session

$headers = @{
    "X-XSRF-TOKEN" = $csrf.token
}
```

Try publishing a course anonymously:

```powershell
try {
    Invoke-RestMethod `
      -Method Patch `
      -Uri "$base/api/v1/courses/1/publish" `
      -WebSession $session `
      -Headers $headers
}
catch {
    $_.Exception.Message
}
```

Expected: `401 Unauthorized`.

### 20.3 Verify student self-promotion is blocked

Register and log in using the Lesson 14 workflow.

Then attempt:

```powershell
try {
    Invoke-RestMethod `
      -Method Post `
      -Uri "$base/api/v1/admin/instructors/1/approve" `
      -WebSession $session `
      -Headers $headers
}
catch {
    $_.Exception.Message
}
```

Expected: `403 Forbidden` for a logged-in student with a valid CSRF token.

Remember that the Lesson 14 login invalidates the pre-login CSRF token. Refresh it after logging in.

### 20.4 Verify instructor ownership

Using two separately authenticated instructor sessions:

1. Create a draft as Instructor A.
2. Attempt to publish it as Instructor B.
3. Expect `403 Forbidden`.
4. Verify the course remains `DRAFT`.
5. Publish it as Instructor A.
6. Expect success and `PUBLISHED`.

The automated Testcontainers tests are the preferred reproducible proof of this behavior.

---

## 21. Run tests and inspect the database

Run the new focused tests:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=CourseOwnershipIntegrationTest,InstructorOnboardingIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 15 authorization tests failed."
}
```

Then run all backend tests:

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Full backend verification failed."
}
```

Inspect test reports:

```powershell
Get-ChildItem target\surefire-reports\*.txt |
    Select-String "Tests run:|Failures:|Errors:"
```

Inspect the ownership index in PostgreSQL:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "\d courses"
```

Check for unowned legacy courses:

```powershell
& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT id, title, status FROM courses WHERE instructor_user_id IS NULL ORDER BY id;"
```

Do not backfill these rows based solely on instructor display names.

---

## 22. Engineering review: race conditions and authorization boundaries

The ownership-aware service is a significant improvement, but a production system must consider concurrent changes.

For example, an administrator might reassign a course while an instructor is editing it.

A service that loads a course, checks ownership, and then updates it inside a transaction is not automatically protected from every concurrent reassignment.

Useful hardening strategies include:

- Adding a JPA `@Version` column for optimistic locking.
- Using a pessimistic lock when ownership reassignment and editing must be serialized.
- Using conditional SQL updates that include `instructor_user_id = :actorId`.
- Preventing ordinary instructors from changing the owner association.
- Auditing ownership changes and privileged administrative operations.

For Lesson 15, keep owner reassignment admin-only and do not expose a public owner-changing endpoint.

### Integration issues to resolve before deployment

Earlier lessons evolved the same course API incrementally. Confirm these details in your actual codebase:

| Area | Required alignment |
|---|---|
| Course exceptions | Use the existing `CourseNotFoundException` package and constructor |
| DTO mapping | Reuse the actual Lesson 11 `CourseResponse` mapper |
| Course entity factory | Match the current `CourseEntity.newDraft(...)` signature |
| Draft editing | Preserve owner and instructor display data |
| Publish/archive methods | Use the actual controller HTTP methods |
| Course repository | Preserve earlier repository contracts and tests |
| Security rules | Ensure every mutation reaches `AuthorizedCourseService` |
| Legacy courses | Keep null-owner records admin-only |
| User sessions | Reauthenticate after instructor promotion |

**One particularly important issue:** In earlier lessons, the `CourseRepository.deleteById` contract changed between `boolean` and `void`. Reconcile that interface and its implementations before running the full suite; the authorization changes do not fix a pre-existing compile mismatch.

## 23. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="shield" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise A — Secure draft editing**

      Add an integration test proving that Instructor A cannot update Instructor B's title, price, or description. Verify that the original database values remain unchanged.
    </box>
  </row>  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="database" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise B — Legacy owner backfill**

      Create a trusted admin-only operation that assigns an unowned legacy course to a specific existing instructor account. Validate the target account's role and write an audit record.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="lock-keyhole" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise C — Session-based end-to-end test**

      Authenticate Instructor A and Instructor B through the real Lesson 14 login endpoint. Reuse their sessions to prove that cross-owner modification fails.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="git-compare" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise D — Optimistic locking**

      Add `@Version` to `CourseEntity` with a new Flyway migration. Simulate two concurrent updates and verify that a stale write fails rather than silently overwriting a newer change.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="monitor-smartphone" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise E — Angular authorization UX**

      Add an instructor-only navigation link, a guarded dashboard route, and a friendly forbidden-access page. Verify the backend still rejects direct unauthorized API calls.
    </box>
  </row>
</box>

## 24. Knowledge quiz

{@body const questions=[{q:"Which field is the authoritative LearnHub course ownership key?",options:["courses.instructor display name","courses.instructor_user_id foreign key","Angular's instructorId route parameter"],correct:1,why:"The database foreign key identifies the actual account; names and client values are not authoritative."},{q:"Should the public registration endpoint accept an INSTRUCTOR or ADMIN role from JSON?",options:["Yes, if the client sends it","Only if the email looks trustworthy","No; privileged roles require a trusted workflow"],correct:2,why:"Allowing client-selected roles would permit privilege escalation."},{q:"Why are both role checks and ownership checks necessary?",options:["A role grants general capabilities; ownership restricts particular resources","Ownership replaces authentication","Angular route guards enforce both"],correct:0,why:"An instructor role alone does not authorize editing another instructor's course."},{q:"What should happen when an instructor attempts to publish another instructor's draft under this lesson's policy?",options:["Publish it because both are instructors","Return 403 and leave the draft unchanged","Delete the draft"],correct:1,why:"Unauthorized requests must not mutate the database."},{q:"Does a user's existing session automatically receive new authorities when an admin promotes them?",options:["Yes, always","No; the session principal may retain its old role"],correct:1,why:"A session holds an authentication snapshot; reauthentication or explicit session refresh is needed."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [show,setShow]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Lesson 15 — Resource authorization**
    <text color="secondary" size="xs">5 questions</text>
  </row>
  {#each questions as q,i}
    <box key={i} gap={2}>
      **{i+1}. {q.q}**
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setShow(false);}}>
        {#each q.options as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if show}
        <row align="start" gap={2}>
          <icon name={answers[i]===q.correct?"check-circle":"info"} color={answers[i]===q.correct?"success":"secondary"}/>
          <box flex="1" gap={1}>
            <text size="sm" weight="medium">{answers[i]===q.correct?"Correct":"Correct: "+q.options[q.correct]}</text>
            <text size="sm" color="secondary">{q.why}</text>
          </box>
        </row>
      {/if}
    </box>
    {#if i<questions.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if show}
    <title size="lg">{questions.filter((q,i)=>answers[i]===q.correct).length} / 5 correct</title>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==questions.length} onClick={()=>setShow(true)}>Check answers</button>
    <button variant="outline" color="secondary" onClick={()=>{setAnswers({});setShow(false);}}>Reset</button>
  </row>
</box>

## 25. Completion checklist

{@body const groups=[{name:"Database and roles",items:["Verified Lesson 14 baseline","Applied the next Flyway ownership index migration","Inspected unowned legacy courses","Kept registration restricted to STUDENT","Added trusted instructor promotion"]},{name:"Backend authorization",items:["Enabled method security","Added CurrentActor","Added CourseOwnershipPolicy","Protected draft creation","Protected draft editing","Protected publish and archive","Protected deletion","Removed old unprotected write paths","Added instructor-owned course query"]},{name:"Tests and Angular",items:["Tested owner publishing","Tested cross-owner denial and unchanged database state","Tested admin override","Tested student denial","Tested CSRF enforcement","Tested admin promotion","Added Angular instructor guard","Verified Angular production build","Ran Maven clean verify"]},{name:"Wrap-up",items:["Completed two hands-on exercises","Completed knowledge quiz","Reviewed staged changes","Committed verified changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((n,g)=>n+g.items.length,0)}
{@body const count=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 15 progress**
      <text size="xs" color="secondary">Mark items only after local verification.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${count/total*100}%`} radius="full"/>
  </box>
  {#each groups as group,i}
    <box key={i} gap={2}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!done[`${i}-${j}`]} onChange={v=>setDone(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if count===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 15 checklist completed.**
    </row>
  {/if}
</box>

## 26. Final verification and Git checkpoint

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 15 backend verification failed."
}

git diff --check
git status
git diff --stat

git add src\main src\test
git diff --cached --stat

git commit -m "Enforce instructor course ownership and trusted role assignment"
```

Angular:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 15 Angular build failed."
}

git diff --check
git status

git add src\app
git diff --cached --stat

git commit -m "Guard LearnHub instructor routes"
```

Inspect staged files before committing. Do not commit database credentials, production secrets, or generated build artifacts.

**Verification note:** The supplied code is a reference continuation of Lessons 10–14, not an executed patch to your repository. Some snippets deliberately integrate with existing factories, exception types, and DTO mappers; reconcile those exact signatures before running the tests.

## 27. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="monitor-smartphone" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 16 — Angular + Spring Boot Integration**

    We'll connect the existing LearnHub frontend to the secured backend end to end: environment configuration, typed API services, authentication state, session cookies, CSRF, course catalog data, instructor-owned CRUD, route guards, loading and error states, and full-stack tests.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 16: full Angular standalone and Spring Boot integration, typed course API services, same-origin proxy, session cookies and CSRF, authenticated state restoration, public catalog, instructor-owned CRUD, Angular route guards, HTTP interceptors, loading and error handling, end-to-end contract testing, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 15 with experienced-Java pacing.")}>Start Lesson 16 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 15 deliverable:** LearnHub course management protected by both role-based and resource-level authorization, with trusted instructor promotion, ownership-aware PostgreSQL operations, security integration tests, and Angular instructor route guards.

The key principle is: **being an instructor grants the ability to manage courses, but ownership determines which courses that instructor may manage.**