# LearnHub Java & Spring Boot — Lesson 17

<text color="secondary" size="sm">PostgreSQL wishlist · Student enrollments · JPA relationships · Transactional services · Angular integration · Security testing</text>

**Stack:** Java 21 · Spring Boot 3.5.x · Spring Security 6.x · PostgreSQL · Flyway · Spring Data JPA · Angular standalone · JUnit 5 · MockMvc · Testcontainers

**Estimated time:** 7–10 hours, including exercises and verification.

## 1. Today's goal

In Lesson 16, we connected the Angular frontend to the secured Spring Boot backend. We now have a public catalog, authenticated sessions, and instructor-owned course management.

Today we'll build two student-facing features that persist across devices and browser sessions:

- **Wishlist:** A signed-in user can save or remove courses they are interested in.
- **Enrollments:** An eligible student can enroll in a published course, with duplicate prevention and a durable enrollment record.

We'll use the `app_users`, `courses`, and `enrollments` tables introduced in Lesson 9, and add a new `wishlists` table.

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    **Angular LearnHub**
    <text size="xs" color="secondary">Wishlist buttons · My Learning page · Enrollment actions</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Session-authenticated REST APIs**
    <text size="xs" color="secondary">/api/v1/me/wishlist · /api/v1/me/enrollments</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Student learning services**
    <text size="xs" color="secondary">Role checks · Course eligibility · Transaction boundaries</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring Data JPA**
    <text size="xs" color="secondary">Database uniqueness · No client-supplied user IDs</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    **PostgreSQL**
    <text size="xs" color="secondary">wishlists · enrollments · courses · app_users</text>
  </box>
</box>

### API behavior

| Action | Endpoint | Expected result |
|---|---|---|
| List wishlist | `GET /api/v1/me/wishlist` | `200` |
| Save course | `PUT /api/v1/me/wishlist/{courseId}` | `204` |
| Remove course | `DELETE /api/v1/me/wishlist/{courseId}` | `204` |
| List enrollments | `GET /api/v1/me/enrollments` | `200` |
| Enroll in course | `POST /api/v1/me/enrollments` | `201` |
| Enroll again | Same POST | `409` |
| Enroll in draft | Same POST | `404` |
| Enroll in paid course | Same POST | `409` until payment is implemented |

**Design decision:** Lesson 17 enables enrollment in **free, published courses only**. Paid courses require a verified payment entitlement, which we'll implement later. We must never treat a client-supplied `paid: true` flag as proof of payment.

---

## 2. Verify the Lesson 16 baseline

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info

git status
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Fix the backend baseline before Lesson 17."
}
```

Then verify Angular:

```powershell
cd $HOME\angular-learning\learnhub

node --version
npm --version
ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Fix the Angular baseline before Lesson 17."
}
```

Check the migration sequence:

```powershell
cd $HOME\angular-learning\learnhub-backend

Get-ChildItem src\main\resources\db\migration |
    Sort-Object Name
```

The original Lesson 9 migration already created `enrollments`. **Do not create that table again.**

For the new wishlist migration, use the next unused Flyway version. The examples below assume `V7` was the last migration from Lesson 15, so we will use `V8`.

Also inspect the existing entities and security classes:

```powershell
Get-Content src\main\java\com\learnhub\persistence\EnrollmentEntity.java

Get-Content src\main\java\com\learnhub\persistence\AppUserEntity.java

Get-Content src\main\java\com\learnhub\security\CurrentActor.java

Get-Content src\main\java\com\learnhub\config\SecurityConfig.java
```

### Compatibility note

The Lesson 10 `EnrollmentEntity` already mapped `student`, `course`, `enrolledAt`, and `status`. We'll extend that entity rather than duplicate it.

The Lesson 14–15 security layer should expose the authenticated database user ID through `CurrentActor.id()` and the current role through `CurrentActor.require().role()`.

---

## 3. Data model and invariants

<box border radius="xl" padding={3} gap={2}>
  <grid columns={2} gap={2}>
    <grid-item>
      <box border radius="lg" padding={3} gap={1}>
        **app_users**
        <text color="secondary" size="xs">id PK · role · enabled</text>
      </box>
    </grid-item>
    <grid-item>
      <box border radius="lg" padding={3} gap={1}>
        **courses**
        <text color="secondary" size="xs">id PK · status · is_free</text>
      </box>
    </grid-item>
  </grid>
  <row justify="center"><icon name="git-merge" color="secondary"/></row>
  <grid columns={2} gap={2}>
    <grid-item>
      <box background="surface-secondary" radius="lg" padding={3} gap={1}>
        **wishlists**
        <text size="xs" color="secondary">user_id FK · course_id FK · unique pair</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="lg" padding={3} gap={1}>
        **enrollments**
        <text size="xs" color="secondary">student_user_id FK · course_id FK · unique pair</text>
      </box>
    </grid-item>
  </grid>
</box>

### Invariants

1. A wishlist item belongs to the authenticated user.
2. A user can save the same course only once.
3. Only published courses can be newly wishlisted.
4. Removing a wishlist item is idempotent.
5. Only users with the `STUDENT` role may enroll.
6. A course must be `PUBLISHED` and free to allow enrollment.
7. A student can have at most one enrollment row per course.
8. Enrollment status is server-controlled.
9. Instructors cannot enroll in their own courses through this student API.
10. An enrollment record is not deleted simply because the student removes the course from their wishlist.

### Why two separate tables?

Wishlist represents **interest**.

Enrollment represents **participation and access**.

They have different lifecycle rules. Removing a bookmark must not revoke a legitimate enrollment.

---

## 4. Flyway migration: wishlist table

**Create:**

`src/main/resources/db/migration/V8__create_wishlists.sql`

<CodeBlock language="sql" editable>
-- Lesson 17
-- Create a persistent wishlist relationship.
--
-- Existing tables:
--   app_users(id)
--   courses(id)
--
-- Use the next unused Flyway version if V8 exists.

CREATE TABLE wishlists (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY
        PRIMARY KEY,

    user_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_wishlists_user
        FOREIGN KEY (user_id)
        REFERENCES app_users(id)
        ON DELETE CASCADE,

    CONSTRAINT fk_wishlists_course
        FOREIGN KEY (course_id)
        REFERENCES courses(id)
        ON DELETE RESTRICT,

    CONSTRAINT uq_wishlists_user_course
        UNIQUE (user_id, course_id)
);

-- The unique constraint already provides an index
-- beginning with user_id.
--
-- This additional index supports course-centric
-- lookups and foreign-key maintenance.
CREATE INDEX idx_wishlists_course_id
    ON wishlists(course_id);
</CodeBlock>

### Why `ON DELETE CASCADE` for users?

If an account is deleted through a trusted deletion workflow, its wishlist bookmarks can be removed automatically.

### Why `ON DELETE RESTRICT` for courses?

A course with active wishlist references cannot be silently deleted without a deliberate cleanup policy.

Archiving a course does not delete it.

### Why not add a second index on `(user_id, course_id)`?

The unique constraint already creates an index supporting that lookup. A duplicate index would waste storage and increase write overhead.

---

## 5. Inspect the existing enrollment constraints

The Lesson 9 migration already established:

```sql
CONSTRAINT uq_enrollments_student_course
    UNIQUE (student_user_id, course_id)
```

It also restricted enrollment status:

```sql
CHECK (
    status IN (
        'ACTIVE',
        'COMPLETED',
        'CANCELLED'
    )
)
```

These constraints are valuable because application-level checks alone cannot prevent concurrent duplicate inserts.

Verify the schema:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "\d enrollments"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "\d wishlists"
```

The second command will succeed only after Flyway has applied the new migration.

### Duplicate handling policy

For wishlist `PUT`, repeated saves should succeed without creating duplicates.

For enrollment `POST`, repeated enrollment should return a conflict rather than creating another row.

We'll enforce both policies in services **and** PostgreSQL.

## 6. JPA entities

We will map the wishlist as its own entity, with lazy `ManyToOne` associations. We will reuse the existing enrollment entity.

### 6.1 WishlistEntity

**Create:** `src/main/java/com/learnhub/persistence/WishlistEntity.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import jakarta.persistence.*;
import java.time.OffsetDateTime;
import java.util.Objects;

@Entity
@Table(
    name = "wishlists",
    uniqueConstraints = @UniqueConstraint(
        name = "uq_wishlists_user_course",
        columnNames = {"user_id", "course_id"}
    )
)
public class WishlistEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id", nullable = false)
    private AppUserEntity user;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "course_id", nullable = false)
    private CourseEntity course;

    @Column(
        name = "created_at",
        nullable = false,
        insertable = false,
        updatable = false
    )
    private OffsetDateTime createdAt;

    protected WishlistEntity() {
        // Required by Hibernate.
    }

    public WishlistEntity(
            AppUserEntity user,
            CourseEntity course
    ) {
        this.user = Objects.requireNonNull(user);
        this.course = Objects.requireNonNull(course);
    }

    public Long getId() {
        return id;
    }

    public AppUserEntity getUser() {
        return user;
    }

    public CourseEntity getCourse() {
        return course;
    }

    public OffsetDateTime getCreatedAt() {
        return createdAt;
    }
}
</CodeBlock>

We intentionally do not add a bidirectional `List<WishlistEntity>` to `AppUserEntity` or `CourseEntity`. The application can query bookmarks directly, avoiding unnecessary collection loading and cascade behavior.

### 6.2 EnrollmentEntity alignment

**Modify:** `src/main/java/com/learnhub/persistence/EnrollmentEntity.java`

Retain the existing Lesson 10 entity. Ensure its constructor and status accessors support this contract:

```java
public EnrollmentEntity(
        AppUserEntity student,
        CourseEntity course
) {
    this.student = Objects.requireNonNull(student);
    this.course = Objects.requireNonNull(course);
    this.status = "ACTIVE";
}

public Long getId() {
    return id;
}

public AppUserEntity getStudent() {
    return student;
}

public CourseEntity getCourse() {
    return course;
}

public String getStatus() {
    return status;
}

public OffsetDateTime getEnrolledAt() {
    return enrolledAt;
}
```

Keep the existing JPA annotations:

```java
@ManyToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "student_user_id")
private AppUserEntity student;

@ManyToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "course_id")
private CourseEntity course;

@Column(
    name = "enrolled_at",
    insertable = false,
    updatable = false
)
private OffsetDateTime enrolledAt;

@Column(name = "status", nullable = false)
private String status = "ACTIVE";
```

Don't add duplicate fields. Ensure the existing entity's imports include `java.util.Objects` and `java.time.OffsetDateTime`.

---

## 7. Spring Data repositories

We need efficient user-scoped queries and safe insert operations.

### 7.1 Wishlist repository

**Create:** `src/main/java/com/learnhub/persistence/JpaWishlistRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface JpaWishlistRepository
        extends JpaRepository<WishlistEntity, Long> {

    /**
     * Load the course in the same query to avoid N+1
     * queries while mapping wishlist response DTOs.
     */
    @Query("""
        SELECT w
        FROM WishlistEntity w
        JOIN FETCH w.course c
        WHERE w.user.id = :userId
        ORDER BY w.id DESC
        """)
    List<WishlistEntity> findAllWithCourseByUserId(
            @Param("userId") Long userId
    );

    /**
     * Native ON CONFLICT provides atomic, idempotent
     * insertion even when requests race.
     *
     * Requires PostgreSQL and the unique constraint.
     */
    @Modifying
    @Query(value = """
        INSERT INTO wishlists (user_id, course_id)
        VALUES (:userId, :courseId)
        ON CONFLICT (user_id, course_id)
        DO NOTHING
        """, nativeQuery = true)
    int insertIfAbsent(
            @Param("userId") Long userId,
            @Param("courseId") Long courseId
    );

    @Modifying
    @Query("""
        DELETE FROM WishlistEntity w
        WHERE w.user.id = :userId
          AND w.course.id = :courseId
        """)
    int deleteForUserAndCourse(
            @Param("userId") Long userId,
            @Param("courseId") Long courseId
    );
}
</CodeBlock>

The `INSERT ... ON CONFLICT DO NOTHING` approach is stronger than:

```java
if (!repository.existsBy...) {
    repository.save(...);
}
```

Two requests can both observe that a bookmark does not exist and then attempt to insert it. PostgreSQL resolves that race atomically.

### 7.2 Enrollment repository

**Modify:** `src/main/java/com/learnhub/persistence/JpaEnrollmentRepository.java`

Keep its existing methods and add:

<CodeBlock language="java" editable>
@Query("""
    SELECT e
    FROM EnrollmentEntity e
    JOIN FETCH e.course c
    WHERE e.student.id = :studentId
    ORDER BY e.id DESC
    """)
List<EnrollmentEntity> findAllWithCourseByStudentId(
        @Param("studentId") Long studentId
);

@Modifying
@Query(value = """
    INSERT INTO enrollments (
        student_user_id,
        course_id,
        status
    )
    VALUES (:studentId, :courseId, 'ACTIVE')
    ON CONFLICT (student_user_id, course_id)
    DO NOTHING
    """, nativeQuery = true)
int insertIfAbsent(
        @Param("studentId") Long studentId,
        @Param("courseId") Long courseId
);
</CodeBlock>

Required imports:

```java
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.List;
```

A returned update count of `1` means an enrollment was inserted. A count of `0` means that pair already exists.

**Important:** We are not deleting a cancelled enrollment and inserting another. The unique constraint applies across all statuses. A future re-enrollment workflow must explicitly define how `CANCELLED` transitions back to `ACTIVE`.

---

## 8. API response DTOs

Avoid returning JPA entities directly. Responses should expose course data and enrollment metadata without serializing users, passwords, or lazy relationship graphs.

### 8.1 Shared course summary

**Create:** `src/main/java/com/learnhub/web/dto/LearningCourseResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.model.CourseLevel;
import com.learnhub.persistence.CourseEntity;

import java.math.BigDecimal;

/**
 * Safe subset of course fields for wishlist
 * and enrollment responses.
 */
public record LearningCourseResponse(
        long id,
        String title,
        String instructor,
        String category,
        BigDecimal price,
        boolean isFree,
        CourseLevel level,
        String thumbnail
) {
    public static LearningCourseResponse from(
            CourseEntity course
    ) {
        return new LearningCourseResponse(
                course.getId(),
                course.getTitle(),
                course.getInstructor(),
                course.getCategory(),
                course.getPrice(),
                course.isFree(),
                course.getLevel(),
                course.getThumbnail()
        );
    }
}
</CodeBlock>

The getter names must match your existing `CourseEntity`. If Lesson 11 changed `isFree()` or `getThumbnail()`, use those current accessors.

### 8.2 Wishlist response

**Create:** `src/main/java/com/learnhub/web/dto/WishlistItemResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import java.time.OffsetDateTime;

public record WishlistItemResponse(
        long courseId,
        LearningCourseResponse course,
        OffsetDateTime savedAt
) {}
</CodeBlock>

### 8.3 Enrollment response

**Create:** `src/main/java/com/learnhub/web/dto/EnrollmentResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import java.time.OffsetDateTime;

public record EnrollmentResponse(
        long id,
        LearningCourseResponse course,
        String status,
        OffsetDateTime enrolledAt
) {}
</CodeBlock>

### 8.4 Enrollment request

**Create:** `src/main/java/com/learnhub/web/dto/EnrollCourseRequest.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;

public record EnrollCourseRequest(
        @NotNull
        @Positive
        Long courseId
) {}
</CodeBlock>

Notice what's absent: `studentId`, `role`, `paymentStatus`, and `enrollmentStatus`.

Those are never accepted from an untrusted enrollment request.

---

## 9. Consistent business exceptions

We want to distinguish missing courses from courses that exist but cannot currently be enrolled in.

**Create:** `src/main/java/com/learnhub/service/EnrollmentConflictException.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

public final class EnrollmentConflictException
        extends RuntimeException {

    public EnrollmentConflictException(String message) {
        super(message);
    }
}
</CodeBlock>

**Create:** `src/main/java/com/learnhub/service/StudentAccessDeniedException.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import org.springframework.security.access.AccessDeniedException;

public final class StudentAccessDeniedException
        extends AccessDeniedException {

    public StudentAccessDeniedException() {
        super("Student access is required.");
    }
}
</CodeBlock>

Add this to your existing `GlobalExceptionHandler`:

<CodeBlock language="java" editable>
@ExceptionHandler(EnrollmentConflictException.class)
public ResponseEntity<ProblemDetail> enrollmentConflict(
        EnrollmentConflictException exception
) {
    ProblemDetail problem =
            ProblemDetail.forStatusAndDetail(
                    HttpStatus.CONFLICT,
                    exception.getMessage()
            );

    problem.setTitle("Enrollment conflict");

    return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(problem);
}
</CodeBlock>

Add the imports for `EnrollmentConflictException`, `ProblemDetail`, `HttpStatus`, `ResponseEntity`, and `ExceptionHandler` if needed.

The existing Lesson 8 handler should continue mapping missing courses to `404` and access denials to `403`.

---

## 10. Transactional wishlist service

**Create:** `src/main/java/com/learnhub/service/WishlistService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.persistence.JpaWishlistRepository;
import com.learnhub.security.CurrentActor;
import com.learnhub.web.dto.LearningCourseResponse;
import com.learnhub.web.dto.WishlistItemResponse;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class WishlistService {

    private final JpaWishlistRepository wishlists;
    private final JpaCourseRepository courses;
    private final CurrentActor actor;

    public WishlistService(
            JpaWishlistRepository wishlists,
            JpaCourseRepository courses,
            CurrentActor actor
    ) {
        this.wishlists = wishlists;
        this.courses = courses;
        this.actor = actor;
    }

    @PreAuthorize("isAuthenticated()")
    @Transactional(readOnly = true)
    public List<WishlistItemResponse> list() {
        return wishlists
                .findAllWithCourseByUserId(actor.id())
                .stream()
                .map(item -> new WishlistItemResponse(
                        item.getCourse().getId(),
                        LearningCourseResponse.from(
                                item.getCourse()
                        ),
                        item.getCreatedAt()
                ))
                .toList();
    }

    /**
     * Idempotent PUT:
     * - first save inserts one row;
     * - repeated saves do nothing.
     */
    @PreAuthorize("isAuthenticated()")
    @Transactional
    public void save(long courseId) {
        if (courseId <= 0) {
            throw new IllegalArgumentException(
                    "Course ID must be positive."
            );
        }

        CourseEntity course = courses.findById(courseId)
                .orElseThrow(() ->
                        new IllegalArgumentException(
                                "Course not found."
                        )
                );

        // Hide unpublished courses from normal users.
        if (course.getStatus() != CourseStatus.PUBLISHED) {
            throw new IllegalArgumentException(
                    "Course not found."
            );
        }

        wishlists.insertIfAbsent(actor.id(), courseId);
    }

    /**
     * Idempotent DELETE:
     * removing a missing bookmark is still successful.
     */
    @PreAuthorize("isAuthenticated()")
    @Transactional
    public void remove(long courseId) {
        if (courseId <= 0) {
            throw new IllegalArgumentException(
                    "Course ID must be positive."
            );
        }

        wishlists.deleteForUserAndCourse(
                actor.id(),
                courseId
        );
    }
}
</CodeBlock>

### Fix the missing-course status

The initial implementation above throws `IllegalArgumentException`, which the Lesson 8 handler maps to `400`.

That is not the desired API contract for a missing or unpublished course.

Use the existing `CourseNotFoundException` instead, both when the course does not exist and when its status is not `PUBLISHED`:

```java
CourseEntity course = courses.findById(courseId)
        .orElseThrow(() ->
                new CourseNotFoundException(courseId)
        );

if (course.getStatus() != CourseStatus.PUBLISHED) {
    throw new CourseNotFoundException(courseId);
}
```

Import your existing exception from its actual package. This is the **final intended implementation** of that section.

### Wishlist role policy

For this lesson, any authenticated account can use a wishlist. Only students can enroll.

That means instructors may bookmark courses but cannot enroll through the student enrollment API.

If your product requires wishlist access to be student-only, replace `isAuthenticated()` with `hasRole('STUDENT')` consistently.

### Archived courses already in wishlists

A course that was published when bookmarked may later be archived.

Our list API still returns the saved item. The Angular UI should mark it unavailable rather than offer enrollment.

We will expose course availability in a later refinement if the existing DTO does not already include status.

---

## 11. Transactional enrollment service

This service is more security-sensitive than the wishlist service because enrollment grants access to learning content.

**Create:** `src/main/java/com/learnhub/service/EnrollmentService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.persistence.JpaEnrollmentRepository;
import com.learnhub.security.CurrentActor;
import com.learnhub.security.SecurityRole;
import com.learnhub.web.dto.EnrollmentResponse;
import com.learnhub.web.dto.LearningCourseResponse;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class EnrollmentService {

    private final JpaEnrollmentRepository enrollments;
    private final JpaCourseRepository courses;
    private final CurrentActor actor;

    public EnrollmentService(
            JpaEnrollmentRepository enrollments,
            JpaCourseRepository courses,
            CurrentActor actor
    ) {
        this.enrollments = enrollments;
        this.courses = courses;
        this.actor = actor;
    }

    @PreAuthorize("hasRole('STUDENT')")
    @Transactional(readOnly = true)
    public List<EnrollmentResponse> listMine() {
        return enrollments
                .findAllWithCourseByStudentId(actor.id())
                .stream()
                .map(enrollment -> new EnrollmentResponse(
                        enrollment.getId(),
                        LearningCourseResponse.from(
                                enrollment.getCourse()
                        ),
                        enrollment.getStatus(),
                        enrollment.getEnrolledAt()
                ))
                .toList();
    }

    /**
     * Free-course enrollment only.
     *
     * Authorization and eligibility are evaluated
     * on the server, never from client-supplied flags.
     */
    @PreAuthorize("hasRole('STUDENT')")
    @Transactional
    public void enroll(long courseId) {
        if (courseId <= 0) {
            throw new IllegalArgumentException(
                    "Course ID must be positive."
            );
        }

        if (actor.require().role() != SecurityRole.STUDENT) {
            throw new StudentAccessDeniedException();
        }

        CourseEntity course = courses.findById(courseId)
                .orElseThrow(() ->
                        new CourseNotFoundException(courseId)
                );

        if (course.getStatus() != CourseStatus.PUBLISHED) {
            throw new CourseNotFoundException(courseId);
        }

        if (!course.isFree()) {
            throw new EnrollmentConflictException(
                    "Paid enrollment requires a verified purchase."
            );
        }

        int inserted = enrollments.insertIfAbsent(
                actor.id(),
                courseId
        );

        if (inserted == 0) {
            throw new EnrollmentConflictException(
                    "You are already enrolled in this course."
            );
        }
    }
}
</CodeBlock>

Import `CourseNotFoundException` from its existing package, and `StudentAccessDeniedException` from `com.learnhub.service` if needed.

### Why this is transactional

The service checks course eligibility and inserts an enrollment as one application operation.

The unique database constraint prevents duplicate enrollment rows, even under concurrent requests.

There is one further concurrency issue: a course might be archived after eligibility is checked but before enrollment is inserted.

For stricter consistency, lock the course row during the eligibility check, and ensure publish/archive operations use compatible locking or a shared concurrency policy.

### Add a locking query

**Modify:** `JpaCourseRepository.java`

<CodeBlock language="java" editable>
@Lock(LockModeType.PESSIMISTIC_READ)
@Query("""
    SELECT c    FROM CourseEntity c
    WHERE c.id = :id
    """)
Optional<CourseEntity> findForEnrollment(
        @Param("id") Long id
);
</CodeBlock>

Imports:

```java
import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.Optional;
```

Then replace `courses.findById(courseId)` in the enrollment service with:

```java
courses.findForEnrollment(courseId)
```

This is useful only when other lifecycle operations coordinate with the same locking strategy. A standalone read lock cannot guarantee the entire business invariant if other operations bypass the intended concurrency protocol.

### Why `ON CONFLICT` instead of exception-driven duplication?

PostgreSQL unique violations can abort the current transaction. Catching the exception and continuing inside the same transaction is not a reliable duplicate-handling strategy.

`ON CONFLICT DO NOTHING` allows the repository to return a deterministic insertion count without poisoning the transaction.

---

## 12. REST controllers

### 12.1 WishlistController

**Create:** `src/main/java/com/learnhub/web/WishlistController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.WishlistService;
import com.learnhub.web.dto.WishlistItemResponse;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/me/wishlist")
public class WishlistController {

    private final WishlistService service;

    public WishlistController(WishlistService service) {
        this.service = service;
    }

    @GetMapping
    public List<WishlistItemResponse> list() {
        return service.list();
    }

    @PutMapping("/{courseId}")
    public ResponseEntity<Void> save(
            @PathVariable long courseId
    ) {
        service.save(courseId);
        return ResponseEntity.noContent().build();
    }

    @DeleteMapping("/{courseId}")
    public ResponseEntity<Void> remove(
            @PathVariable long courseId
    ) {
        service.remove(courseId);
        return ResponseEntity.noContent().build();
    }
}
</CodeBlock>

### 12.2 EnrollmentController

**Create:** `src/main/java/com/learnhub/web/EnrollmentController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.EnrollmentService;
import com.learnhub.web.dto.EnrollCourseRequest;
import com.learnhub.web.dto.EnrollmentResponse;

import jakarta.validation.Valid;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/v1/me/enrollments")
public class EnrollmentController {

    private final EnrollmentService service;

    public EnrollmentController(
            EnrollmentService service
    ) {
        this.service = service;
    }

    @GetMapping
    public List<EnrollmentResponse> list() {
        return service.listMine();
    }

    @PostMapping
    public ResponseEntity<Void> enroll(
            @Valid @RequestBody
            EnrollCourseRequest request
    ) {
        service.enroll(request.courseId());

        return ResponseEntity
                .created(
                    URI.create(
                        "/api/v1/me/enrollments"
                    )
                )
                .build();
    }
}
</CodeBlock>

The `201 Created` response points to the enrollment collection because we have not exposed a per-enrollment detail endpoint. A more precise API can return the created enrollment ID and a resource-specific Location URI.

---

## 13. Spring Security route rules

**Modify:** `src/main/java/com/learnhub/config/SecurityConfig.java`

Add these matchers **before** the final fallback rule, preserving your existing session, CSRF, login, and public catalog configuration:

<CodeBlock language="java" editable>
.requestMatchers(
    "/api/v1/me/wishlist",
    "/api/v1/me/wishlist/**"
).authenticated()

.requestMatchers(
    "/api/v1/me/enrollments",
    "/api/v1/me/enrollments/**"
).hasRole("STUDENT")
</CodeBlock>

### Ordering matters

If you have a broad rule such as:

```java
.requestMatchers("/api/v1/**").permitAll()
```

before these matchers, the broad rule may match first.

Keep specific protected routes ahead of broad public matchers.

### Keep CSRF enabled

Wishlist `PUT` and `DELETE`, and enrollment `POST`, are unsafe HTTP methods and must require the existing session's CSRF token.

Do not disable CSRF merely to make Angular requests succeed.

### A note about instructor self-enrollment

The `STUDENT` role restriction already prevents an account with only `ROLE_INSTRUCTOR` from enrolling.

For future multi-role accounts, add an explicit check that the authenticated user's ID is not equal to `course.instructor_user_id`.

---

## 14. Angular API models

We will reuse the `LearningCourseResponse` shape in both features.

**Create:** `learnhub/src/app/core/api/learning-api.models.ts`

<CodeBlock language="typescript" editable>
import { CourseLevel } from './course-api.models';

export interface LearningCourseDto {
  id: number;
  title: string;
  instructor: string;
  category: string;
  price: number;
  isFree: boolean;
  level: CourseLevel;
  thumbnail: string | null;
}

export interface WishlistItemDto {
  courseId: number;
  course: LearningCourseDto;
  savedAt: string | null;
}

export type EnrollmentStatus =
  | 'ACTIVE'
  | 'COMPLETED'
  | 'CANCELLED';

export interface EnrollmentDto {
  id: number;
  course: LearningCourseDto;
  status: EnrollmentStatus;
  enrolledAt: string | null;
}

export interface EnrollCourseRequest {
  courseId: number;
}
</CodeBlock>

We keep `savedAt` and `enrolledAt` nullable because database-generated timestamps can be null in an entity immediately after a native insert or before a refresh. Our read queries should ordinarily return their persisted values.

## 15. Angular wishlist API service

**Create:** `learnhub/src/app/core/api/wishlist-api.service.ts`

<CodeBlock language="typescript" editable>
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

import {
  WishlistItemDto,
} from './learning-api.models';

@Injectable({ providedIn: 'root' })
export class WishlistApiService {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/v1/me/wishlist';

  list(): Observable<WishlistItemDto[]> {
    return this.http.get<WishlistItemDto[]>(
      this.base
    );
  }

  /**
   * Idempotent save.
   * Repeating this request does not create duplicates.
   */
  save(courseId: number): Observable<void> {
    return this.http.put<void>(
      `${this.base}/${courseId}`,
      {}
    );
  }

  remove(courseId: number): Observable<void> {
    return this.http.delete<void>(
      `${this.base}/${courseId}`
    );
  }
}
</CodeBlock>

## 16. Angular enrollment API service

**Create:** `learnhub/src/app/core/api/enrollment-api.service.ts`

<CodeBlock language="typescript" editable>
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

import {
  EnrollmentDto,
  EnrollCourseRequest,
} from './learning-api.models';

@Injectable({ providedIn: 'root' })
export class EnrollmentApiService {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/v1/me/enrollments';

  list(): Observable<EnrollmentDto[]> {
    return this.http.get<EnrollmentDto[]>(
      this.base
    );
  }

  enroll(courseId: number): Observable<void> {
    const request: EnrollCourseRequest = {
      courseId,
    };

    return this.http.post<void>(
      this.base,
      request
    );
  }
}
</CodeBlock>

Both services use relative URLs, so the Lesson 16 same-origin proxy and Angular XSRF configuration continue to apply.

---

## 17. Wishlist page with server-backed state

The existing Angular LearnHub frontend may have a localStorage-based wishlist service. Do not run both sources of truth independently.

For this lesson, authenticated wishlist state comes from PostgreSQL.

**Create:** `learnhub/src/app/features/wishlist/wishlist-page.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  inject,
  signal,
} from '@angular/core';

import { RouterLink } from '@angular/router';

import { WishlistApiService }
  from '../../core/api/wishlist-api.service';

import { WishlistItemDto }
  from '../../core/api/learning-api.models';

@Component({
  selector: 'app-wishlist-page',
  standalone: true,
  imports: [RouterLink],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main>
      <h1>My wishlist</h1>

      @if (loading()) {
        <p role="status">Loading wishlist...</p>
      } @else if (error()) {
        <p role="alert">{{ error() }}</p>
        <button type="button" (click)="load()">
          Retry
        </button>
      } @else if (items().length === 0) {
        <p>You haven't saved any courses yet.</p>
        <a routerLink="/courses">Explore courses</a>
      } @else {
        <ul>
          @for (item of items(); track item.courseId) {
            <li>
              <h2>
                <a [routerLink]="[
                  '/courses',
                  item.courseId
                ]">
                  {{ item.course.title }}
                </a>
              </h2>

              <p>{{ item.course.instructor }}</p>

              <button
                type="button"
                (click)="remove(item.courseId)"
                [disabled]="busyId() === item.courseId"
              >
                Remove from wishlist
              </button>
            </li>
          }
        </ul>
      }
    </main>
  `,
})
export class WishlistPageComponent {
  private readonly api = inject(WishlistApiService);

  readonly items = signal<WishlistItemDto[]>([]);
  readonly loading = signal(true);
  readonly busyId = signal<number | null>(null);
  readonly error = signal<string | null>(null);

  constructor() {
    this.load();
  }

  load(): void {
    this.loading.set(true);
    this.error.set(null);

    this.api.list().subscribe({
      next: items => {
        this.items.set(items);
        this.loading.set(false);
      },
      error: error => {
        this.error.set(
          error.message || 'Unable to load wishlist.'
        );
        this.loading.set(false);
      },
    });
  }

  remove(courseId: number): void {
    if (this.busyId() !== null) {
      return;
    }

    this.busyId.set(courseId);
    this.error.set(null);

    this.api.remove(courseId).subscribe({
      next: () => {
        this.items.update(items =>
          items.filter(
            item => item.courseId !== courseId
          )
        );
        this.busyId.set(null);
      },
      error: error => {
        this.error.set(
          error.message || 'Unable to remove course.'
        );
        this.busyId.set(null);
      },
    });
  }
}
</CodeBlock>

### Adding a wishlist button to course cards

The existing LearnHub course card component can call:

```typescript
wishlistApi.save(course.id).subscribe({
  next: () => {
    // Update the card's saved state.
  },
  error: error => {
    // Display a useful error message.
  },
});
```

For a complete UI, centralize saved-course IDs in a shared store or signal service. That allows the catalog and wishlist page to reflect the same state.

Do not mark a course as saved until the backend confirms success, unless you implement an explicit optimistic update with rollback.

---

## 18. My Learning page

**Create:** `learnhub/src/app/features/learning/my-learning-page.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  inject,
  signal,
} from '@angular/core';

import { RouterLink } from '@angular/router';

import { EnrollmentApiService }
  from '../../core/api/enrollment-api.service';

import { EnrollmentDto }
  from '../../core/api/learning-api.models';

@Component({
  selector: 'app-my-learning-page',
  standalone: true,
  imports: [RouterLink],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main>
      <h1>My learning</h1>

      @if (loading()) {
        <p role="status">Loading enrollments...</p>
      } @else if (error()) {
        <p role="alert">{{ error() }}</p>
        <button type="button" (click)="load()">
          Retry
        </button>
      } @else if (enrollments().length === 0) {
        <p>You haven't enrolled in a course yet.</p>
        <a routerLink="/courses">
          Explore free courses
        </a>
      } @else {
        <ul>
          @for (
            enrollment of enrollments();
            track enrollment.id
          ) {
            <li>
              <h2>{{ enrollment.course.title }}</h2>
              <p>Status: {{ enrollment.status }}</p>
              <p>
                Instructor:
                {{ enrollment.course.instructor }}
              </p>

              <a [routerLink]="[
                '/courses',
                enrollment.course.id
              ]">
                View course
              </a>
            </li>
          }
        </ul>
      }
    </main>
  `,
})
export class MyLearningPageComponent {
  private readonly api = inject(EnrollmentApiService);

  readonly enrollments = signal<EnrollmentDto[]>([]);
  readonly loading = signal(true);
  readonly error = signal<string | null>(null);

  constructor() {
    this.load();
  }

  load(): void {
    this.loading.set(true);
    this.error.set(null);

    this.api.list().subscribe({
      next: enrollments => {
        this.enrollments.set(enrollments);
        this.loading.set(false);
      },
      error: error => {
        this.error.set(
          error.message ||
          'Unable to load your enrollments.'
        );
        this.loading.set(false);
      },
    });
  }
}
</CodeBlock>

For now, "View course" opens the existing public course detail page. Lesson 18 will add actual lesson content and progress tracking.

---

## 19. Course enrollment action

Add a reusable button to the existing course detail page.

**Create:** `learnhub/src/app/features/learning/enroll-button.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  inject,
  input,
  signal,
} from '@angular/core';

import { Router } from '@angular/router';

import { EnrollmentApiService }
  from '../../core/api/enrollment-api.service';

import { AuthService }
  from '../../core/auth/auth.service';

@Component({
  selector: 'app-enroll-button',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <button
      type="button"
      (click)="enroll()"
      [disabled]="busy() || enrolled() || !isFree()"
    >
      @if (busy()) {
        Enrolling...
      } @else if (enrolled()) {
        Enrolled
      } @else if (!isFree()) {
        Purchase required
      } @else {
        Enroll for free
      }
    </button>

    @if (message()) {
      <p role="status">{{ message() }}</p>
    }
  `,
})
export class EnrollButtonComponent {
  private readonly api = inject(EnrollmentApiService);
  private readonly auth = inject(AuthService);
  private readonly router = inject(Router);

  readonly courseId = input.required<number>();
  readonly isFree = input.required<boolean>();

  readonly busy = signal(false);
  readonly enrolled = signal(false);
  readonly message = signal<string | null>(null);

  enroll(): void {
    if (this.busy() || this.enrolled()) {
      return;
    }

    const user = this.auth.currentUser();

    if (!user) {
      void this.router.navigateByUrl('/login');
      return;
    }

    if (user.role !== 'STUDENT') {
      this.message.set(
        'Student access is required for enrollment.'
      );
      return;
    }

    this.busy.set(true);
    this.message.set(null);

    this.api.enroll(this.courseId()).subscribe({
      next: () => {
        this.enrolled.set(true);
        this.busy.set(false);
        this.message.set('Enrollment successful.');
      },
      error: error => {
        this.busy.set(false);

        if (error.status === 409) {
          this.message.set(
            error.message || 'Already enrolled or purchase required.'
          );
          return;
        }

        this.message.set(
          error.message || 'Unable to enroll.'
        );
      },
    });
  }
}
</CodeBlock>

In your course detail template:

```html
<app-enroll-button
  [courseId]="course.id"
  [isFree]="course.isFree"
/>
```

Import `EnrollButtonComponent` in the standalone parent component's `imports` array.

### Existing enrollment state

The button currently starts with `enrolled = false` on every page load. It must eventually initialize that state from the authenticated user's enrollment list or a dedicated `GET /api/v1/me/enrollments/{courseId}` endpoint.

Otherwise, a returning student might see "Enroll for free" even though they are already enrolled. The backend still correctly prevents duplicates, but the UI would be misleading.

---

## 20. Route integration

**Modify:** `learnhub/src/app/app.routes.ts`

Merge these route entries with your existing routes:

<CodeBlock language="typescript" editable>
{
  path: 'wishlist',
  canActivate: [authGuard],
  loadComponent: () =>
    import(
      './features/wishlist/wishlist-page.component'
    ).then(m => m.WishlistPageComponent),
},
{
  path: 'my-learning',
  canActivate: [studentGuard],
  loadComponent: () =>
    import(
      './features/learning/my-learning-page.component'
    ).then(m => m.MyLearningPageComponent),
},
</CodeBlock>

The `authGuard` and `studentGuard` names are not guaranteed to exist in the Lesson 16 project. Create them or adapt your existing auth guard.

**Create:** `learnhub/src/app/core/auth/student.guard.ts`

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
} from 'rxjs';

import { AuthService } from './auth.service';

export const studentGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  return auth.initialize().pipe(
    map(user => {
      if (!user) {
        return router.createUrlTree(['/login']);
      }

      return user.role === 'STUDENT'
        ? true
        : router.createUrlTree(['/']);
    }),
    catchError(() =>
      of(router.createUrlTree(['/login']))
    )
  );
};
</CodeBlock>

For `authGuard`, use the same structure but permit any authenticated user.

Remember: these guards are for navigation. The Spring Security filter chain and service methods remain authoritative.

## 21. PostgreSQL integration tests

We need to test the real constraints and security boundary, not just mock repositories.

The tests below extend the existing `PostgresIntegrationTest` from Lesson 7.

### 21.1 Wishlist integration tests

**Create:** `src/test/java/com/learnhub/web/WishlistIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;
import com.learnhub.security.LearnHubPrincipal;
import com.learnhub.security.SecurityRole;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class WishlistIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired MockMvc mvc;
    @Autowired JdbcTemplate jdbc;
    @Autowired PasswordEncoder encoder;

    private long student() {
        return jdbc.queryForObject("""
            INSERT INTO app_users (
                email, display_name, password_hash,
                role, enabled
            )
            VALUES (
                'wishlist-test@example.test',
                'Wishlist Student',
                ?, 'STUDENT', true
            )
            RETURNING id
            """,
            Long.class,
            encoder.encode("TestPassword123!")
        );
    }

    private LearnHubPrincipal principal(long id) {
        return new LearnHubPrincipal(
                id,
                "wishlist-test@example.test",
                encoder.encode("TestPassword123!"),
                SecurityRole.STUDENT,
                true
        );
    }

    @Test
    void repeatedPutCreatesOnlyOneBookmark()
            throws Exception {
        long id = student();

        for (int attempt = 0; attempt < 2; attempt++) {
            mvc.perform(
                    put("/api/v1/me/wishlist/2")
                        .with(user(principal(id)))
                        .with(csrf())
                )
                .andExpect(status().isNoContent());
        }

        Integer count = jdbc.queryForObject("""
            SELECT count(*)
            FROM wishlists
            WHERE user_id = ? AND course_id = 2
            """,
            Integer.class,
            id
        );

        assertEquals(1, count);
    }

    @Test
    void deletingMissingBookmarkIsIdempotent()
            throws Exception {
        long id = student();

        mvc.perform(
                delete("/api/v1/me/wishlist/2")
                    .with(user(principal(id)))
                    .with(csrf())
            )
            .andExpect(status().isNoContent());
    }

    @Test
    void unpublishedCourseCannotBeWishlisted()
            throws Exception {
        long id = student();

        // Lesson 7 seed: course 5 is DRAFT.
        mvc.perform(
                put("/api/v1/me/wishlist/5")
                    .with(user(principal(id)))
                    .with(csrf())
            )
            .andExpect(status().isNotFound());
    }

    @Test
    void anonymousWishlistRequestIsUnauthorized()
            throws Exception {
        mvc.perform(get("/api/v1/me/wishlist"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void mutationRequiresCsrf()
            throws Exception {
        long id = student();

        mvc.perform(
                put("/api/v1/me/wishlist/2")
                    .with(user(principal(id)))
            )
            .andExpect(status().isForbidden());
    }
}
</CodeBlock>

The tests assume course `2` is published and course `5` remains a draft. For a more isolated suite, create courses in test fixtures rather than depending on sample seed IDs.

### 21.2 Enrollment integration tests

**Create:** `src/test/java/com/learnhub/web/EnrollmentIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;
import com.learnhub.security.LearnHubPrincipal;
import com.learnhub.security.SecurityRole;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class EnrollmentIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired MockMvc mvc;
    @Autowired JdbcTemplate jdbc;
    @Autowired PasswordEncoder encoder;

    private long insertUser(String role, String email) {
        return jdbc.queryForObject("""
            INSERT INTO app_users (
                email, display_name, password_hash,
                role, enabled
            )
            VALUES (?, 'Test User', ?, ?, true)
            RETURNING id
            """,
            Long.class,
            email,
            encoder.encode("TestPassword123!"),
            role
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
    void freePublishedCourseCanBeEnrolledOnce()
            throws Exception {
        String email = "enroll-once@example.test";
        long studentId = insertUser("STUDENT", email);

        var request = post("/api/v1/me/enrollments")
                .contentType("application/json")
                .content("{\"courseId\":2}")
                .with(user(principal(
                        studentId,
                        email,
                        SecurityRole.STUDENT
                )))
                .with(csrf());

        mvc.perform(request)
                .andExpect(status().isCreated());

        mvc.perform(                post("/api/v1/me/enrollments")
                    .contentType("application/json")
                    .content("{\"courseId\":2}")
                    .with(user(principal(
                            studentId,
                            email,
                            SecurityRole.STUDENT
                    )))
                    .with(csrf())
            )
            .andExpect(status().isConflict());

        Integer count = jdbc.queryForObject("""
            SELECT count(*)
            FROM enrollments
            WHERE student_user_id = ?
              AND course_id = 2
            """,
            Integer.class,
            studentId
        );

        assertEquals(1, count);
    }

    @Test
    void paidCourseRequiresPurchase()
            throws Exception {
        String email = "paid-course@example.test";
        long studentId = insertUser("STUDENT", email);

        // Lesson 7 seed: course 1 is paid.
        mvc.perform(
                post("/api/v1/me/enrollments")
                    .contentType("application/json")
                    .content("{\"courseId\":1}")
                    .with(user(principal(
                            studentId,
                            email,
                            SecurityRole.STUDENT
                    )))
                    .with(csrf())
            )
            .andExpect(status().isConflict());
    }

    @Test
    void draftCourseIsNotEnrollable()
            throws Exception {
        String email = "draft-course@example.test";
        long studentId = insertUser("STUDENT", email);

        mvc.perform(
                post("/api/v1/me/enrollments")
                    .contentType("application/json")
                    .content("{\"courseId\":5}")
                    .with(user(principal(
                            studentId,
                            email,
                            SecurityRole.STUDENT
                    )))
                    .with(csrf())
            )
            .andExpect(status().isNotFound());
    }

    @Test
    void instructorCannotUseStudentEnrollmentApi()
            throws Exception {
        String email = "instructor-enroll@example.test";
        long instructorId = insertUser(
                "INSTRUCTOR",
                email
        );

        mvc.perform(
                post("/api/v1/me/enrollments")
                    .contentType("application/json")
                    .content("{\"courseId\":2}")
                    .with(user(principal(
                            instructorId,
                            email,
                            SecurityRole.INSTRUCTOR
                    )))
                    .with(csrf())
            )
            .andExpect(status().isForbidden());
    }

    @Test
    void missingCsrfBlocksEnrollment()
            throws Exception {
        String email = "csrf-enroll@example.test";
        long studentId = insertUser("STUDENT", email);

        mvc.perform(
                post("/api/v1/me/enrollments")
                    .contentType("application/json")
                    .content("{\"courseId\":2}")
                    .with(user(principal(
                            studentId,
                            email,
                            SecurityRole.STUDENT
                    )))
            )
            .andExpect(status().isForbidden());
    }
}
</CodeBlock>

### Important test details

- `@Transactional` on these synchronous MockMvc tests rolls back fixture rows after each test.
- PostgreSQL identity sequences do not roll back, which is why user IDs should be read from `RETURNING id`.
- The test principal constructor must match the actual Lesson 14 implementation.
- These tests do not simulate two truly concurrent transactions. Add a separate concurrency test for duplicate inserts.
- `@WithMockUser` alone is insufficient for these services because `CurrentActor` expects `LearnHubPrincipal`.

### Isolation test to add

Create two students, save the same course for both, and verify:

```text
Student A sees A's wishlist.
Student B sees B's wishlist.
Neither sees the other's private bookmarks.
```

This proves that all wishlist queries are scoped by the authenticated database user ID.

---

## 22. Angular HttpClient tests

**Create:** `learnhub/src/app/core/api/learning-api.spec.ts`

<CodeBlock language="typescript" editable>
import { TestBed } from '@angular/core/testing';

import {
  provideHttpClient,
} from '@angular/common/http';

import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';

import { WishlistApiService }
  from './wishlist-api.service';

import { EnrollmentApiService }
  from './enrollment-api.service';

describe('Learning API services', () => {
  let wishlist: WishlistApiService;
  let enrollments: EnrollmentApiService;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });

    wishlist = TestBed.inject(WishlistApiService);
    enrollments = TestBed.inject(EnrollmentApiService);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    http.verify();
  });

  it('uses PUT for idempotent wishlist saving', () => {
    wishlist.save(42).subscribe();

    const request = http.expectOne(
      '/api/v1/me/wishlist/42'
    );

    expect(request.request.method).toBe('PUT');

    request.flush(null, {
      status: 204,
      statusText: 'No Content',
    });
  });

  it('uses DELETE for wishlist removal', () => {
    wishlist.remove(42).subscribe();

    const request = http.expectOne(
      '/api/v1/me/wishlist/42'
    );

    expect(request.request.method).toBe('DELETE');

    request.flush(null, {
      status: 204,
      statusText: 'No Content',
    });
  });

  it('posts only the course ID for enrollment', () => {
    enrollments.enroll(42).subscribe();

    const request = http.expectOne(
      '/api/v1/me/enrollments'
    );

    expect(request.request.method).toBe('POST');

    expect(request.request.body).toEqual({
      courseId: 42,
    });

    expect(request.request.body.studentId)
      .toBeUndefined();

    request.flush(null, {
      status: 201,
      statusText: 'Created',
    });
  });

  it('lists enrollments from the authenticated route', () => {
    enrollments.list().subscribe();

    const request = http.expectOne(
      '/api/v1/me/enrollments'
    );

    expect(request.request.method).toBe('GET');
    expect(request.request.params.has('studentId'))
      .toBeFalse();

    request.flush([]);
  });
});
</CodeBlock>

These tests verify the Angular request contract. They do not replace backend security tests or browser-level CSRF checks.

---

## 23. Run the complete application

### Terminal A — PostgreSQL

Verify PostgreSQL is running, then inspect the database:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT version, description, success FROM flyway_schema_history ORDER BY installed_rank;"
```

### Terminal B — Spring Boot

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

Check health:

```powershell
Invoke-RestMethod `
  "http://localhost:8080/actuator/health"
```

### Terminal C — Angular

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd serve --proxy-config proxy.conf.json
```

Open `http://localhost:4200`.

### Verify the browser session and CSRF flow

The Lesson 16 implementation should already:

1. Restore the authenticated user through `/api/v1/auth/me`.
2. Request `/api/v1/security/csrf` before unsafe authentication operations.
3. Use `XSRF-TOKEN` and `X-XSRF-TOKEN`.
4. Send the session cookie automatically through same-origin requests.
5. Refresh CSRF after login.

In DevTools, verify wishlist `PUT`, wishlist `DELETE`, and enrollment `POST` carry the expected CSRF header.

Do not disable CSRF to work around a `403`.

---

## 24. Manual acceptance scenarios

| Scenario | Expected |
|---|---|
| Anonymous user opens wishlist | Login required |
| Signed-in student saves published course | Wishlist contains one item |
| Student saves same course twice | Still one item |
| Student removes course twice | Both requests succeed |
| Student refreshes browser | Wishlist remains |
| Student enrolls in free published course | `201 Created` |
| Student enrolls again | `409 Conflict` |
| Student tries paid course | `409 Conflict` |
| Student tries draft course | `404 Not Found` |
| Instructor tries student enrollment | `403 Forbidden` |
| Student A requests own enrollments | No Student B records |
| Unsafe request omits CSRF | `403 Forbidden` |

### Inspect actual database records

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT user_id, course_id, created_at FROM wishlists ORDER BY id DESC LIMIT 20;"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT student_user_id, course_id, status, enrolled_at FROM enrollments ORDER BY id DESC LIMIT 20;"
```

Avoid exposing student emails or other personal data in logs or screenshots.

---

## 25. Engineering review: production concerns

### Database uniqueness is mandatory

Application-level duplicate checks are not sufficient when requests run concurrently. The unique constraints and PostgreSQL `ON CONFLICT` statements provide the final protection.

### Enrollment does not prove payment

This lesson intentionally blocks paid enrollments. When payments arrive, enrollment must be granted only after the backend verifies a trusted payment event or entitlement.

### User deletion needs a deliberate policy

The wishlist migration uses `ON DELETE CASCADE` for the user FK. Existing enrollments use `ON DELETE RESTRICT`. Account deletion therefore requires a broader retention and cleanup design.

### Archive behavior needs product rules

Existing enrollments should generally remain recorded when a course is archived. Whether enrolled students retain access to archived content is a separate policy decision.

### Avoid exposing enrollment existence across users

All student APIs use `/me/` and derive the user ID from `CurrentActor`. Never add an unrestricted `studentId` query parameter to these endpoints.

### Concurrency and eligibility

A row lock during enrollment is only effective when competing lifecycle operations participate in a compatible transaction strategy. For strict enrollment eligibility, document and test that strategy.

### Additional hardening before production

- Add paginated wishlist and enrollment list endpoints when collections grow.
- Avoid relying on fixed sample course IDs in integration tests.
- Use a typed enrollment status enum once lifecycle transitions are introduced.
- Add an audit trail for enrollment creation and cancellation.
- Use optimistic locking or an appropriate database locking policy for conflicting course lifecycle changes.
- Make wishlist visibility and course availability explicit in response DTOs.
- Treat unexpected database constraint failures as controlled API errors rather than exposing SQL details.

---

## 26. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="users" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise A — Multi-user isolation**

      Create two student accounts, add the same published course to both wishlists, and prove that each student sees only their own records. Repeat for enrollments.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="git-merge" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise B — Concurrent duplicate enrollment**

      Launch two enrollment requests for the same student and course in separate transactions. Verify that one inserts a row, the other reports a conflict, and PostgreSQL contains exactly one enrollment.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="bookmark-check" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise C — Shared wishlist state**

      Implement a signal-based wishlist store. Show saved state on catalog cards and course detail pages, and keep it synchronized after saves and removals.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="shield-check" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise D — Authenticated browser flow**

      Create a browser-level test that logs in as a student, saves a course, enrolls in it, refreshes the page, and verifies both records persist. Confirm CSRF protection is active.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="database" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise E — Enrollment lifecycle design**

      Propose a `CANCELLED → ACTIVE` re-enrollment workflow. Explain how it interacts with the existing unique constraint, audit history, and future payment entitlements.
    </box>
  </row>
</box>

## 27. Knowledge quiz

{@body const questions=[{q:"Why is a database unique constraint needed for enrollments?",a:["It replaces authentication","It prevents duplicate rows even when requests race","It automatically verifies payment"],correct:1,why:"A unique constraint enforces the student/course invariant at the database boundary."},{q:"Why is wishlist saving implemented as PUT?",a:["Repeated saves should have the same resulting state","PUT automatically disables CSRF","PUT can only be used by administrators"],correct:0,why:"An idempotent PUT represents the desired state: the course is saved."},{q:"Where should the student ID come from during enrollment?",a:["The request body","An Angular route parameter","The authenticated server-side principal"],correct:2,why:"The server must derive identity from the authenticated session to prevent cross-user operations."},{q:"Why does Lesson 17 reject enrollment in paid courses?",a:["Spring Data JPA cannot store paid enrollments","A verified payment entitlement is not implemented yet","PostgreSQL cannot store payment status"],correct:1,why:"Paid access must be based on trusted backend payment verification, not client assertions."},{q:"What happens when a student removes a course from their wishlist?",a:["The enrollment is also deleted","The course is archived","Only the wishlist relationship is removed"],correct:2,why:"Wishlist interest and enrollment access have separate persistence and lifecycle rules."}]}
{@body const [selected,setSelected]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    **Lesson 17 — Persistence and security**
    <text color="secondary" size="xs">5 questions</text>
  </row>
  {#each questions as q,i}
    <box key={i} gap={2}>
      **{i+1}. {q.q}**
      <radio-group direction="col" value={selected[i]===undefined?"":String(selected[i])} onChange={v=>{setSelected(old=>({...old,[i]:Number(v)}));setGraded(false);}}>
        {#each q.a as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if graded}
        <row align="start" gap={2}>
          <icon name={selected[i]===q.correct?"check-circle":"info"} color={selected[i]===q.correct?"success":"secondary"}/>
          <box flex="1" gap={1}>
            <text size="sm" weight="medium">{selected[i]===q.correct?"Correct":"Correct answer: "+q.a[q.correct]}</text>
            <text size="sm" color="secondary">{q.why}</text>
          </box>
        </row>
      {/if}
    </box>
    {#if i<questions.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if graded}
    <title size="lg">{questions.filter((q,i)=>selected[i]===q.correct).length} / 5 correct</title>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(selected).length!==questions.length} onClick={()=>setGraded(true)}>Check answers</button>
    <button variant="outline" color="secondary" onClick={()=>{setSelected({});setGraded(false);}}>Reset</button>
  </row>
</box>

## 28. Completion checklist

{@body const sections=[{name:"Database and JPA",items:["Verified Lesson 16 baseline","Selected next unused Flyway version","Created wishlists migration","Verified existing enrollment uniqueness","Created WishlistEntity","Extended EnrollmentEntity without duplicating mappings","Added wishlist repository","Extended enrollment repository"]},{name:"Backend APIs and security",items:["Created response and request DTOs","Implemented transactional wishlist service","Implemented transactional enrollment service","Handled missing courses and enrollment conflicts","Created wishlist controller","Created enrollment controller","Protected /me routes in SecurityFilterChain","Confirmed CSRF remains enabled"]},{name:"Angular integration",items:["Created typed learning API models","Implemented WishlistApiService","Implemented EnrollmentApiService","Connected wishlist page","Connected My Learning page","Connected enrollment button","Added guarded routes","Verified session restoration and CSRF"]},{name:"Verification",items:["Added PostgreSQL wishlist tests","Added PostgreSQL enrollment tests","Added Angular HTTP tests","Verified duplicate prevention","Verified student isolation","Verified paid and draft rejection","Ran Maven clean verify","Ran Angular tests and build","Completed two exercises","Completed quiz","Reviewed and committed changes"]}]}
{@body const [checks,setChecks]=DIL.useState({})}
{@body const total=sections.reduce((sum,s)=>sum+s.items.length,0)}
{@body const finished=Object.values(checks).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 17 progress**
      <text color="secondary" size="xs">Check each item after local verification.</text>
    </box>
    <title size="lg" tabularNums>{finished}/{total}</title>
  </row>
  <box background="surface-tertiary" radius="full" height="8px" clip>
    <box background="rgba(109,40,217,0.85)" radius="full" height="100%" width={`${finished/total*100}%`}/>
  </box>
  {#each sections as section,i}
    <box key={i} gap={2}>
      **{section.name}**
      {#each section.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!checks[`${i}-${j}`]} onChange={v=>setChecks(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if finished===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 17 checklist completed.**
    </row>
  {/if}
</box>

## 29. Final verification and Git checkpoints

### Backend

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=WishlistIntegrationTest,EnrollmentIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 17 integration tests failed."
}

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend verification failed."
}

git diff --check
git status
git diff --stat
```

### Angular

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd test --watch=false

if ($LASTEXITCODE -ne 0) {
    throw "Angular tests failed."
}

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}

git diff --check
git status
git diff --stat
```

### Git checkpoints

After the builds and tests pass:

```powershell
cd $HOME\angular-learning\learnhub-backend

git add src\main src\test
git diff --cached --stat

git commit -m "Persist student wishlists and free course enrollments"
```

```powershell
cd $HOME\angular-learning\learnhub

git add src\app
git diff --cached --stat

git commit -m "Connect LearnHub wishlist and enrollment UI to backend"
```

Inspect staged changes before committing. Never commit real passwords, session cookies, database credentials, or private configuration.

### Final integration cautions

The code is a reference continuation of the previous lessons, not an executed patch. Before treating it as production-ready, verify these specific items:

- The existing `CourseNotFoundException` package and constructor.
- The actual `CourseEntity` getter names and `EnrollmentEntity` mappings.
- The `LearnHubPrincipal` constructor and role representation.
- The `GlobalExceptionHandler`'s access-denied behavior.
- The Lesson 16 Angular auth guard and XSRF configuration.
- The next unused Flyway migration number.
- The course lifecycle locking strategy and any future paid-entitlement workflow.

The sample tests rely on the seeded course IDs from earlier lessons; replace those assumptions with isolated fixtures for a more durable suite.

## 30. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="book-open-check" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 18 — Course Curriculum, Lessons, and Learning Progress**

    We'll build instructor-managed sections and lessons, authenticated lesson access, enrollment-gated content, progress tracking, completion rules, PostgreSQL relationships, and an Angular course player.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 18: instructor-managed course sections and lessons, PostgreSQL Flyway migrations, JPA relationships, ordered curriculum, secure enrollment-gated lesson content, ownership authorization, student progress tracking, completion rules, transactional services, REST DTOs, Angular standalone course player, MockMvc and Testcontainers integration tests, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 17 with experienced-Java pacing.")}>Start Lesson 18 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 17 deliverable:** PostgreSQL-backed wishlists and free-course enrollments with server-derived identity, database-enforced uniqueness, protected REST APIs, Angular integration, and cross-layer tests.

The central distinction is: **a wishlist records interest; an enrollment records learning access.**