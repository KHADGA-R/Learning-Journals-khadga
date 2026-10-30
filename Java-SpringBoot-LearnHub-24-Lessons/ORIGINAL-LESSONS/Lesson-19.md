# LearnHub Java & Spring Boot — Lesson 19

<text color="secondary" size="sm">Instructor dashboard · PostgreSQL analytics · Course readiness · Publishing workflow · Optimistic locking · Angular standalone</text>

**Stack:** Java 21 · Spring Boot 3.5.x · Spring Security · Spring Data JPA · PostgreSQL · Flyway · Angular standalone · JUnit 5 · MockMvc · Testcontainers

**Estimated time:** 8–12 hours, including tests and exercises.

## 1. The goal

In Lesson 18, we built instructor-managed course sections and lessons, enrollment-gated content, and student progress.

Today we'll turn that curriculum into a controlled publishing workflow and give instructors a dashboard to manage their courses.

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" radius="lg" padding={3} gap={1}>
    **Instructor dashboard**
    <text color="secondary" size="xs">My courses · Published count · Active students · Completion metrics</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <grid columns={2} gap={2}>
    <grid-item>
      <box border radius="lg" padding={3} gap={1}>
        **Draft course**
        <text color="secondary" size="xs">Edit curriculum · Validate readiness · Publish</text>
      </box>
    </grid-item>
    <grid-item>
      <box border radius="lg" padding={3} gap={1}>
        **Published course**
        <text color="secondary" size="xs">Accept enrollments · Serve lessons · Archive</text>
      </box>
    </grid-item>
  </grid>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} gap={1}>
    **Transactional publishing service**
    <text color="secondary" size="xs">Owner authorization · Readiness rules · Version checks</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" radius="lg" padding={3} gap={1}>
    **PostgreSQL**
    <text color="secondary" size="xs">Courses · Curriculum · Enrollments · Progress</text>
  </box>
</box>

### Deliverables

By the end of this lesson, LearnHub will support:

- Instructor-only dashboards with ownership-scoped analytics.
- Per-course readiness checks before publication.
- A controlled `DRAFT → PUBLISHED → ARCHIVED` lifecycle.
- Version-aware publish and archive requests.
- Database-backed optimistic locking to prevent stale updates.
- Angular dashboard cards, readiness feedback, and publish/archive actions.
- Integration tests for security, correctness, and concurrency.

## 2. Course lifecycle

<box border radius="xl" padding={3} gap={2}>
  <grid columns={3} gap={2}>
    <grid-item>
      <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
        <icon name="file-pen-line" size="lg"/>
        **DRAFT**
        <text size="xs" color="secondary" textAlign="center">Editable</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
        <icon name="globe" size="lg"/>
        **PUBLISHED**
        <text size="xs" color="secondary" textAlign="center">Public catalog</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
        <icon name="archive" size="lg"/>
        **ARCHIVED**
        <text size="xs" color="secondary" textAlign="center">Not discoverable</text>
      </box>
    </grid-item>
  </grid>
  <row justify="between" align="center">
    <text size="xs" color="secondary">Draft → Publish</text>
    <icon name="arrow-right"/>
    <text size="xs" color="secondary">Published → Archive</text>
  </row>
</box>

### Allowed transitions

| Current state | Action | Result |
|---|---|---|
| DRAFT | Publish, readiness passes | PUBLISHED |
| DRAFT | Publish, readiness fails | `409 Conflict` |
| DRAFT | Archive | `409 Conflict` |
| PUBLISHED | Publish again | `409 Conflict` |
| PUBLISHED | Archive | ARCHIVED |
| ARCHIVED | Publish | `409 Conflict` |
| ARCHIVED | Archive again | `409 Conflict` |

We deliberately **do not implement republishing** archived courses in this lesson. Republish needs a separate policy for curriculum versions, existing enrollments, and student access.

### Important compatibility decision

Lesson 11 already added publish and archive operations to `CourseApplicationService`.

**Do not leave two competing implementations active.** We'll make the new workflow service the authoritative implementation and have the existing endpoints delegate to it, or replace those mappings with the new version-aware endpoints.

For this lesson, the new instructor endpoints will use:

```text
POST /api/v1/instructor/courses/{id}/publish
POST /api/v1/instructor/courses/{id}/archive
```

The old write endpoints under `/api/v1/courses/{id}/publish` and `/archive` must be removed, deprecated safely, or delegated to the same authorization and readiness logic. They must not bypass it.

---

## 3. Baseline verification and workspace

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info

git status
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Fix Lesson 18 backend failures first."
}
```

Angular:

```powershell
cd $HOME\angular-learning\learnhub

node --version
npm --version
ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Fix Lesson 18 Angular failures first."
}
```

Inspect your migration history:

```powershell
cd $HOME\angular-learning\learnhub-backend

Get-ChildItem src\main\resources\db\migration |
    Sort-Object Name
```

Lesson 18 used `V9__create_curriculum_and_progress.sql` as an example. Today we'll use **V10** only if that is the next unused migration version.

Also inspect:

```powershell
Get-Content src\main\java\com\learnhub\persistence\CourseEntity.java

Get-Content src\main\java\com\learnhub\service\CourseApplicationService.java

Get-Content src\main\java\com\learnhub\service\CurriculumManagementService.java

Get-Content src\main\java\com\learnhub\config\SecurityConfig.java
```

Our code assumes the Lesson 15 authenticated principal exposes `id()` and `role()` through `CurrentActor.require()`.

---

## 4. PostgreSQL migration: optimistic locking

Optimistic locking allows two editors to read a course, but prevents a stale editor from overwriting a newer change.

### Example

<box border radius="lg" padding={3} gap={2}>
  <grid columns={2} gap={2}>
    <grid-item>
      **Editor A**
      <text size="sm" color="secondary">Reads version 3</text>
      <row align="center" gap={2}><icon name="check-circle" color="success"/><text size="sm">Saves → version 4</text></row>
    </grid-item>
    <grid-item>
      **Editor B**
      <text size="sm" color="secondary">Reads version 3</text>
      <row align="center" gap={2}><icon name="x-circle" color="danger"/><text size="sm">Saves → conflict</text></row>
    </grid-item>
  </grid>
  <divider color="subtle"/>
  <text size="sm">Editor B must reload before retrying. The server must not silently overwrite Editor A's changes.</text>
</box>

### Flyway migration

**Create:** `src/main/resources/db/migration/V10__course_version_and_publication_metadata.sql`

<CodeBlock language="sql" editable>
-- Lesson 19
-- Use the next unused Flyway version.

ALTER TABLE courses
    ADD COLUMN version BIGINT NOT NULL DEFAULT 0;

ALTER TABLE courses
    ADD COLUMN published_at TIMESTAMPTZ;

ALTER TABLE courses
    ADD COLUMN archived_at TIMESTAMPTZ;

-- Existing published seed courses predate this metadata.
-- Use their last_updated date as an approximate historical
-- publication date only for the migration backfill.
UPDATE courses
SET published_at = last_updated::timestamptz
WHERE status = 'PUBLISHED'
  AND published_at IS NULL;

UPDATE courses
SET archived_at = CURRENT_TIMESTAMP
WHERE status = 'ARCHIVED'
  AND archived_at IS NULL;

ALTER TABLE courses
    ADD CONSTRAINT chk_course_publication_dates
    CHECK (
        (status <> 'PUBLISHED' OR published_at IS NOT NULL)
        AND
        (status <> 'ARCHIVED' OR archived_at IS NOT NULL)
    );

CREATE INDEX idx_courses_instructor_status
    ON courses(instructor_user_id, status, id);

CREATE INDEX idx_enrollments_course_status
    ON enrollments(course_id, status);
</CodeBlock>

### Why the backfill is approximate

The earlier schema did not record a real publication timestamp. `last_updated` is not proof of the original publish event.

For new transitions, we'll set the actual timestamp in the service.

### Why not add `@Version` to curriculum rows yet?

Course-level versioning can protect lifecycle operations, but it does **not automatically version changes to child sections and lessons**.

We'll explicitly increment the parent course version during curriculum edits, ensuring readiness and publishing decisions can detect concurrent changes.

---

## 5. Update the CourseEntity

**Modify:** `src/main/java/com/learnhub/persistence/CourseEntity.java`

Add these fields and methods to the existing entity. Do not replace its earlier mappings or constructors.

<CodeBlock language="java" editable>
import jakarta.persistence.Version;
import java.time.OffsetDateTime;

// Inside CourseEntity:

@Version
@Column(name = "version", nullable = false)
private long version;

@Column(name = "published_at")
private OffsetDateTime publishedAt;

@Column(name = "archived_at")
private OffsetDateTime archivedAt;

public long getVersion() {
    return version;
}

public OffsetDateTime getPublishedAt() {
    return publishedAt;
}

public OffsetDateTime getArchivedAt() {
    return archivedAt;
}

public void publish(OffsetDateTime now) {
    if (status != CourseStatus.DRAFT) {
        throw new CourseConflictException(
                "Only draft courses can be published."
        );
    }

    status = CourseStatus.PUBLISHED;
    publishedAt = now;
    archivedAt = null;
}

public void archive(OffsetDateTime now) {
    if (status != CourseStatus.PUBLISHED) {
        throw new CourseConflictException(
                "Only published courses can be archived."
        );
    }

    status = CourseStatus.ARCHIVED;
    archivedAt = now;
}
</CodeBlock>

Import `CourseConflictException` from its existing Lesson 11 package. These methods assume the existing entity's status field is named `status`.

**Important:** Your earlier `changeStatus()` method must no longer be used to publish or archive a course, because it bypasses timestamp updates and readiness validation. Remove public access to it where possible, or ensure all callers route through the new workflow.

### Add version-aware repository methods

**Modify:** `src/main/java/com/learnhub/persistence/JpaCourseRepository.java`

Retain existing methods and add:

<CodeBlock language="java" editable>
@Query("""
    SELECT c
    FROM CourseEntity c
    WHERE c.instructorAccount.id = :instructorId
    ORDER BY c.id DESC
    """)
List<CourseEntity> findOwnedBy(
        @Param("instructorId") Long instructorId
);

@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("""
    UPDATE CourseEntity c
    SET c.version = c.version + 1
    WHERE c.id = :courseId
    """)
int incrementVersion(
        @Param("courseId") Long courseId
);
</CodeBlock>

Imports:

```java
import java.util.List;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.repository.query.Param;
```

**Why use `incrementVersion`?** Child curriculum edits do not dirty the parent `CourseEntity` automatically. A version increment gives us a shared change token.

**Caution:** JPQL bulk updates bypass the persistence context and normal `@Version` checks. Use this method only for a controlled curriculum mutation after acquiring the parent course lock, and do not reuse a stale managed `CourseEntity` afterward.

---

## 6. Readiness rules

A course is publishable only when:

- It has a nonblank title, description, category, and instructor display name.
- It has an assigned instructor account.
- It has at least one section.
- Every section contains at least one lesson.
- Every lesson has a nonblank title and body.
- All lesson durations are nonnegative.
- Its price/free combination is valid.

For now, we require a minimum of **one complete lesson**. You can increase that threshold later.

### Readiness DTO

**Create:** `src/main/java/com/learnhub/web/dto/CourseReadinessResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import java.util.List;

public record CourseReadinessResponse(
        long courseId,
        long version,
        boolean ready,
        int sectionCount,
        int lessonCount,
        List<String> issues
) {}
</CodeBlock>

### Readiness service

**Create:** `src/main/java/com/learnhub/service/CourseReadinessService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.persistence.*;
import com.learnhub.web.dto.CourseReadinessResponse;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

@Service
public class CourseReadinessService {

    private final JpaCourseRepository courses;
    private final JpaCourseSectionRepository sections;
    private final JpaCourseLessonRepository lessons;

    public CourseReadinessService(
            JpaCourseRepository courses,
            JpaCourseSectionRepository sections,
            JpaCourseLessonRepository lessons
    ) {
        this.courses = courses;
        this.sections = sections;
        this.lessons = lessons;
    }

    @Transactional(readOnly = true)
    public CourseReadinessResponse check(long courseId) {
        CourseEntity course = courses.findById(courseId)
                .orElseThrow(() ->
                        new CourseNotFoundException(courseId)
                );

        List<String> issues = new ArrayList<>();

        if (blank(course.getTitle())) {
            issues.add("Course title is required.");
        }
        if (blank(course.getDescription())) {
            issues.add("Course description is required.");
        }
        if (blank(course.getCategory())) {
            issues.add("Course category is required.");
        }
        if (blank(course.getInstructor())) {
            issues.add("Instructor display name is required.");
        }
        if (course.getInstructorAccount() == null) {
            issues.add("A course owner must be assigned.");
        }

        if (course.getPrice() == null ||
                course.getPrice().signum() < 0 ||
                (course.isFree() &&
                 course.getPrice().compareTo(BigDecimal.ZERO) != 0) ||
                (!course.isFree() &&
                 course.getPrice().signum() == 0)) {
            issues.add("Course pricing is inconsistent.");
        }

        var courseSections =
                sections.findByCourse_IdOrderByPositionAscIdAsc(courseId);

        int lessonCount = 0;

        if (courseSections.isEmpty()) {
            issues.add("Add at least one section.");
        }

        for (var section : courseSections) {
            var sectionLessons =
                    lessons.findBySection_IdOrderByPositionAscIdAsc(
                            section.getId()
                    );

            if (sectionLessons.isEmpty()) {
                issues.add(
                        "Section '" + section.getTitle() +
                        "' needs at least one lesson."
                );
            }

            for (var lesson : sectionLessons) {
                lessonCount++;

                if (blank(lesson.getTitle()) ||
                        blank(lesson.getBody()) ||
                        lesson.getDurationMinutes() < 0) {
                    issues.add(
                            "Lesson " + lesson.getId() +
                            " contains invalid content."
                    );
                }
            }
        }

        return new CourseReadinessResponse(
                courseId,
                course.getVersion(),
                issues.isEmpty(),
                courseSections.size(),
                lessonCount,
                List.copyOf(issues)
        );
    }

    private static boolean blank(String value) {
        return value == null || value.isBlank();
    }
}
</CodeBlock>

Import `CourseNotFoundException` from its actual Lesson 4 package.

**Security boundary:** `check()` is an internal service method. The controller exposing its result must check ownership. A readiness response can reveal unpublished course structure, so it is not public.

### An important transaction refinement

The `check()` method uses its own read-only transaction when called directly. When invoked from a publishing transaction through another Spring bean, it participates in the existing transaction.

This lets the publishing service validate the current database state under the parent course lock.

---

## 7. Version-aware publishing

We'll require the frontend to send the version it last observed.

### Request DTO

**Create:** `src/main/java/com/learnhub/web/dto/CourseVersionRequest.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.PositiveOrZero;

public record CourseVersionRequest(
        @NotNull @PositiveOrZero Long expectedVersion
) {}
</CodeBlock>

### Publishing service

**Create:** `src/main/java/com/learnhub/service/CoursePublishingService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.web.dto.CourseReadinessResponse;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.OffsetDateTime;

@Service
public class CoursePublishingService {

    private final CurriculumAccessService access;
    private final CourseReadinessService readiness;
    private final JpaCourseRepository courses;
    private final JdbcTemplate jdbc;

    public CoursePublishingService(
            CurriculumAccessService access,
            CourseReadinessService readiness,
            JpaCourseRepository courses,
            JdbcTemplate jdbc
    ) {
        this.access = access;
        this.readiness = readiness;
        this.courses = courses;
        this.jdbc = jdbc;
    }

    private void lockCourse(long courseId) {
        jdbc.queryForObject("""
                SELECT id
                FROM courses
                WHERE id = ?
                FOR UPDATE
                """,
                Long.class,
                courseId
        );
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional(readOnly = true)
    public CourseReadinessResponse readiness(long courseId) {
        access.manageableCourse(courseId);
        return readiness.check(courseId);
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional
    public void publish(long courseId, long expectedVersion) {
        lockCourse(courseId);

        var course = access.manageableCourse(courseId);

        requireVersion(course.getVersion(), expectedVersion);

        if (course.getStatus() != CourseStatus.DRAFT) {
            throw new CourseConflictException(
                    "Only draft courses can be published."
            );
        }

        CourseReadinessResponse report = readiness.check(courseId);

        if (!report.ready()) {
            throw new CourseConflictException(
                    "Course is not ready: " +
                    String.join("; ", report.issues())
            );
        }

        course.publish(OffsetDateTime.now());
        courses.saveAndFlush(course);
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional
    public void archive(long courseId, long expectedVersion) {
        lockCourse(courseId);

        var course = access.manageableCourse(courseId);

        requireVersion(course.getVersion(), expectedVersion);

        if (course.getStatus() != CourseStatus.PUBLISHED) {
            throw new CourseConflictException(
                    "Only published courses can be archived."
            );
        }

        course.archive(OffsetDateTime.now());
        courses.saveAndFlush(course);
    }

    private static void requireVersion(
            long actual,
            long expected
    ) {
        if (actual != expected) {
            throw new StaleCourseVersionException();
        }
    }
}
</CodeBlock>

### Stale version exception

**Create:** `src/main/java/com/learnhub/service/StaleCourseVersionException.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

public final class StaleCourseVersionException
        extends RuntimeException {

    public StaleCourseVersionException() {
        super(
            "The course changed. Refresh before retrying."
        );
    }
}
</CodeBlock>

Map stale versions to `409 Conflict`:

<CodeBlock language="java" editable>
@ExceptionHandler({
    StaleCourseVersionException.class,
    org.springframework.orm.ObjectOptimisticLockingFailureException.class
})
public ResponseEntity<ProblemDetail> staleCourse(
        Exception exception
) {
    ProblemDetail problem =
            ProblemDetail.forStatusAndDetail(
                    HttpStatus.CONFLICT,
                    "The course changed. Refresh before retrying."
            );

    problem.setTitle("Stale course version");

    return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(problem);
}
</CodeBlock>

### Why lock and version-check?

The row lock serializes publishing against other operations that lock the same course. The expected-version check rejects stale user intent even when the transaction itself can proceed safely.

**Crucial limitation:** The earlier Lesson 18 curriculum mutations must also increment the course version. Otherwise, an instructor could change lessons while another tab still holds an apparently current version.

We'll fix that next.

## 8. Make curriculum edits participate in versioning

**Modify:** `CurriculumManagementService.java` from Lesson 18.

After every successful section or lesson mutation, increment the parent course version.

Use the existing course-row lock consistently, and update the version through JDBC:

<CodeBlock language="java" editable>
private void bumpCourseVersion(long courseId) {
    int changed = jdbc.update("""
            UPDATE courses
            SET version = version + 1
            WHERE id = ?
            """,
            courseId
    );

    if (changed != 1) {
        throw new CurriculumNotFoundException();
    }
}
</CodeBlock>

Call `bumpCourseVersion(courseId)` exactly once after each successful operation:

- `addSection`
- `addLesson`
- `updateLesson`
- `reorderSections`
- `reorderLessons`

For example, in `addLesson`:

```java
var lesson = lessons.saveAndFlush(
        new CourseLessonEntity(
                section,
                request.title(),
                request.body(),
                nextPosition,
                request.durationMinutes()
        )
);

bumpCourseVersion(courseId);
return lesson.getId();
```

In `updateLesson`, explicitly flush the updated lesson before incrementing:

```java
lesson.update(
        request.title(),
        request.body(),
        request.durationMinutes()
);

lessons.flush();
bumpCourseVersion(courseId);
```

For the JDBC-based reorder methods, call the bump after the final position update.

### Ordering and consistency caveat

Lesson 18's `addSection` and `addLesson` load a managed course entity before acquiring the course lock. That can leave stale course state in the persistence context.

Refactor those methods so the course lock is acquired first, then the course and ownership are loaded. The same lock-first convention must apply to every operation that changes publication-relevant course state.

Also, once `@Version` is enabled, **all course-editing code from Lesson 11 must respect optimistic locking**. An old course update endpoint that accepts no expected version can still overwrite a user's stale form values even though Hibernate increments the database version.

Before production, make course detail editing accept an expected version as well.

---

## 9. PostgreSQL instructor analytics

The dashboard should answer:

- How many courses do I own?
- How many are drafts, published, or archived?
- How many active enrollments do those courses have?
- How many students have completed the current curriculum?
- What are the metrics for each individual course?

The instructor ID must come from the authenticated principal.

### 9.1 Aggregation strategy

We must avoid joining enrollments directly to lesson progress without careful grouping.

Consider a course with 10 students and 8 lesson completion rows per student. A naïve join can multiply the enrollment rows and produce an inflated student count.

We'll aggregate enrollments separately and use `COUNT(DISTINCT ...)` where necessary.

### 9.2 Dashboard DTOs

**Create:** `src/main/java/com/learnhub/web/dto/InstructorCourseRow.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

public record InstructorCourseRow(
        long id,
        String title,
        String status,
        long version,
        long sectionCount,
        long lessonCount,
        long activeEnrollments,
        long completedStudents
) {}
</CodeBlock>

**Create:** `src/main/java/com/learnhub/web/dto/InstructorDashboardResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import java.util.List;

public record InstructorDashboardResponse(
        long totalCourses,
        long draftCourses,
        long publishedCourses,
        long archivedCourses,
        long activeEnrollments,
        long completedStudents,
        List<InstructorCourseRow> courses
) {}
</CodeBlock>

`completedStudents` here means students with an active enrollment who have completed every lesson in the course's **current** curriculum, and only when the course has at least one lesson. It is not a permanent certificate count.

### 9.3 Repository using JdbcTemplate

**Create:** `src/main/java/com/learnhub/persistence/InstructorAnalyticsRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.web.dto.InstructorCourseRow;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public class InstructorAnalyticsRepository {

    private final JdbcTemplate jdbc;

    public InstructorAnalyticsRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public List<InstructorCourseRow> findCourses(
            long instructorId
    ) {
        return jdbc.query("""
            WITH section_counts AS (
                SELECT
                    s.course_id,
                    COUNT(*) AS section_count
                FROM course_sections s
                GROUP BY s.course_id
            ),
            lesson_counts AS (
                SELECT
                    s.course_id,
                    COUNT(l.id) AS lesson_count
                FROM course_sections s
                LEFT JOIN course_lessons l
                  ON l.section_id = s.id
                GROUP BY s.course_id
            ),
            enrollment_counts AS (
                SELECT
                    e.course_id,
                    COUNT(*) AS active_enrollments
                FROM enrollments e
                WHERE e.status = 'ACTIVE'
                GROUP BY e.course_id
            ),
            completed_enrollments AS (
                SELECT
                    e.course_id,
                    COUNT(*) AS completed_students
                FROM enrollments e
                JOIN lesson_counts lc
                  ON lc.course_id = e.course_id
                LEFT JOIN lesson_progress p
                  ON p.enrollment_id = e.id
                LEFT JOIN course_lessons l
                  ON l.id = p.lesson_id
                LEFT JOIN course_sections s
                  ON s.id = l.section_id
                 AND s.course_id = e.course_id
                WHERE e.status = 'ACTIVE'
                  AND lc.lesson_count > 0
                GROUP BY e.id, e.course_id, lc.lesson_count
                HAVING COUNT(s.id) = lc.lesson_count
            ),
            completed_counts AS (
                SELECT
                    course_id,
                    COUNT(*) AS completed_students
                FROM completed_enrollments
                GROUP BY course_id
            )
            SELECT
                c.id,
                c.title,
                c.status,
                c.version,
                COALESCE(sc.section_count, 0),
                COALESCE(lc.lesson_count, 0),
                COALESCE(ec.active_enrollments, 0),
                COALESCE(cc.completed_students, 0)
            FROM courses c
            LEFT JOIN section_counts sc
              ON sc.course_id = c.id
            LEFT JOIN lesson_counts lc
              ON lc.course_id = c.id
            LEFT JOIN enrollment_counts ec
              ON ec.course_id = c.id
            LEFT JOIN completed_counts cc
              ON cc.course_id = c.id
            WHERE c.instructor_user_id = ?
            ORDER BY c.id DESC
            """,
            (rs, row) -> new InstructorCourseRow(
                    rs.getLong(1),
                    rs.getString(2),
                    rs.getString(3),
                    rs.getLong(4),
                    rs.getLong(5),
                    rs.getLong(6),
                    rs.getLong(7),
                    rs.getLong(8)
            ),
            instructorId
        );
    }
}
</CodeBlock>

### SQL review

The `completed_enrollments` CTE groups by enrollment ID and course ID. An enrollment counts as completed only if its number of course-matching completion records equals the course's lesson count.

Because `lesson_progress` has a unique `(enrollment_id, lesson_id)` constraint, completion rows cannot duplicate the same lesson for one enrollment.

The query returns **course-level aggregates**, not individual student identities. That's appropriate for an instructor dashboard.

### Performance note

This query aggregates the entire database before filtering the final result to one instructor. It is clear and correct for the course, but can become expensive at scale.

An advanced version should scope the CTEs to the authenticated instructor's course IDs first, then aggregate only those rows.

---

## 10. Dashboard service

**Create:** `src/main/java/com/learnhub/service/InstructorDashboardService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.persistence.InstructorAnalyticsRepository;
import com.learnhub.security.CurrentActor;
import com.learnhub.web.dto.*;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Servicepublic class InstructorDashboardService {

    private final InstructorAnalyticsRepository analytics;
    private final CurrentActor actor;

    public InstructorDashboardService(
            InstructorAnalyticsRepository analytics,
            CurrentActor actor
    ) {
        this.analytics = analytics;
        this.actor = actor;
    }

    @PreAuthorize("hasRole('INSTRUCTOR')")
    @Transactional(readOnly = true)
    public InstructorDashboardResponse myDashboard() {
        List<InstructorCourseRow> rows =
                analytics.findCourses(actor.id());

        return new InstructorDashboardResponse(
                rows.size(),
                count(rows, "DRAFT"),
                count(rows, "PUBLISHED"),
                count(rows, "ARCHIVED"),
                rows.stream()
                        .mapToLong(
                                InstructorCourseRow::activeEnrollments
                        )
                        .sum(),
                rows.stream()
                        .mapToLong(
                                InstructorCourseRow::completedStudents
                        )
                        .sum(),
                rows
        );
    }

    private static long count(
            List<InstructorCourseRow> rows,
            String status
    ) {
        return rows.stream()
                .filter(row -> status.equals(row.status()))
                .count();
    }
}
</CodeBlock>

This endpoint is intentionally instructor-specific. An admin-wide analytics endpoint should be separate and explicitly authorized, not implemented by accepting arbitrary `instructorId` query parameters.

---

## 11. REST controllers and security

### 11.1 InstructorDashboardController

**Create:** `src/main/java/com/learnhub/web/InstructorDashboardController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.InstructorDashboardService;
import com.learnhub.web.dto.InstructorDashboardResponse;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/instructor/dashboard")
public class InstructorDashboardController {

    private final InstructorDashboardService service;

    public InstructorDashboardController(
            InstructorDashboardService service
    ) {
        this.service = service;
    }

    @GetMapping
    public InstructorDashboardResponse dashboard() {
        return service.myDashboard();
    }
}
</CodeBlock>

### 11.2 InstructorPublishingController

**Create:** `src/main/java/com/learnhub/web/InstructorPublishingController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.CoursePublishingService;
import com.learnhub.web.dto.*;

import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/instructor/courses")
public class InstructorPublishingController {

    private final CoursePublishingService service;

    public InstructorPublishingController(
            CoursePublishingService service
    ) {
        this.service = service;
    }

    @GetMapping("/{courseId}/readiness")
    public CourseReadinessResponse readiness(
            @PathVariable long courseId
    ) {
        return service.readiness(courseId);
    }

    @PostMapping("/{courseId}/publish")
    public ResponseEntity<Void> publish(
            @PathVariable long courseId,
            @Valid @RequestBody CourseVersionRequest request
    ) {
        service.publish(
                courseId,
                request.expectedVersion()
        );
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/{courseId}/archive")
    public ResponseEntity<Void> archive(
            @PathVariable long courseId,
            @Valid @RequestBody CourseVersionRequest request
    ) {
        service.archive(
                courseId,
                request.expectedVersion()
        );
        return ResponseEntity.noContent().build();
    }
}
</CodeBlock>

### 11.3 SecurityFilterChain

**Modify:** `src/main/java/com/learnhub/config/SecurityConfig.java`

Add these rules before any broader matcher:

<CodeBlock language="java" editable>
.requestMatchers(
        "/api/v1/instructor/dashboard",
        "/api/v1/instructor/dashboard/**"
).hasRole("INSTRUCTOR")

.requestMatchers(
        "/api/v1/instructor/courses/*/readiness",
        "/api/v1/instructor/courses/*/publish",
        "/api/v1/instructor/courses/*/archive"
).hasAnyRole("INSTRUCTOR", "ADMIN")
</CodeBlock>

Retain the Lesson 18 curriculum rules and all Lesson 14 session/CSRF configuration.

### Do not forget method security

Confirm that your configuration has:

```java
@EnableMethodSecurity
```

on a configuration class.

The HTTP rules are the first boundary. `@PreAuthorize` and ownership checks in services protect the business operations even when called from another controller or service.

### Retire the old publishing endpoints

Search for the Lesson 11 mappings:

```powershell
cd $HOME\angular-learning\learnhub-backend

Get-ChildItem src\main\java -Recurse -Filter *.java |
    Select-String -Pattern 'publish|archive'
```

Remove or redirect the old `CourseController` publishing methods. Do not leave them calling the earlier `changeStatus()` logic.

**Also audit the old course update and delete endpoints.** If they still use only role checks, they must be brought under the Lesson 15 ownership policy. Publishing security is incomplete if another instructor can mutate or delete the same course through a legacy route.

---

## 12. Angular typed dashboard API

**Create:** `learnhub/src/app/core/api/instructor-dashboard.models.ts`

<CodeBlock language="typescript" editable>
export type CourseStatus =
  | 'DRAFT'
  | 'PUBLISHED'
  | 'ARCHIVED';

export interface InstructorCourseRowDto {
  id: number;
  title: string;
  status: CourseStatus;
  version: number;
  sectionCount: number;
  lessonCount: number;
  activeEnrollments: number;
  completedStudents: number;
}

export interface InstructorDashboardDto {
  totalCourses: number;
  draftCourses: number;
  publishedCourses: number;
  archivedCourses: number;
  activeEnrollments: number;
  completedStudents: number;
  courses: InstructorCourseRowDto[];
}

export interface CourseReadinessDto {
  courseId: number;
  version: number;
  ready: boolean;
  sectionCount: number;
  lessonCount: number;
  issues: string[];
}

export interface CourseVersionRequest {
  expectedVersion: number;
}
</CodeBlock>

**Create:** `learnhub/src/app/core/api/instructor-dashboard-api.service.ts`

<CodeBlock language="typescript" editable>
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

import {
  InstructorDashboardDto,
  CourseReadinessDto,
  CourseVersionRequest,
} from './instructor-dashboard.models';

@Injectable({ providedIn: 'root' })
export class InstructorDashboardApiService {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/v1/instructor';

  dashboard(): Observable<InstructorDashboardDto> {
    return this.http.get<InstructorDashboardDto>(
      `${this.base}/dashboard`
    );
  }

  readiness(courseId: number):
      Observable<CourseReadinessDto> {
    return this.http.get<CourseReadinessDto>(
      `${this.base}/courses/${courseId}/readiness`
    );
  }

  publish(
    courseId: number,
    expectedVersion: number
  ): Observable<void> {
    const body: CourseVersionRequest = {
      expectedVersion,
    };

    return this.http.post<void>(
      `${this.base}/courses/${courseId}/publish`,
      body
    );
  }

  archive(
    courseId: number,
    expectedVersion: number
  ): Observable<void> {
    return this.http.post<void>(
      `${this.base}/courses/${courseId}/archive`,
      { expectedVersion } satisfies CourseVersionRequest
    );
  }
}
</CodeBlock>

The URLs are relative to the Angular origin. The Lesson 16 development proxy forwards `/api` to Spring Boot, and the Lesson 14 CSRF interceptor protects the POST requests.

---

## 13. Angular standalone instructor dashboard

We'll use signals for UI state, with a loading state, readiness report, and explicit stale-version handling.

**Create:** `learnhub/src/app/features/instructor/instructor-dashboard.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  inject,
  signal,
} from '@angular/core';

import { RouterLink } from '@angular/router';
import { HttpErrorResponse } from '@angular/common/http';
import { finalize } from 'rxjs';
import { takeUntilDestroyed }
  from '@angular/core/rxjs-interop';

import { InstructorDashboardApiService }
  from '../../core/api/instructor-dashboard-api.service';

import {
  InstructorDashboardDto,
  CourseReadinessDto,
  InstructorCourseRowDto,
} from '../../core/api/instructor-dashboard.models';

@Component({
  selector: 'app-instructor-dashboard',
  standalone: true,
  imports: [RouterLink],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main>
      <h1>Instructor dashboard</h1>

      @if (loading()) {
        <p role="status">Loading dashboard...</p>
      }

      @if (error()) {
        <p role="alert">{{ error() }}</p>
        <button type="button" (click)="load()">
          Refresh
        </button>
      }

      @if (dashboard(); as data) {
        <section aria-label="Course metrics">
          <p>Total courses: {{ data.totalCourses }}</p>
          <p>Drafts: {{ data.draftCourses }}</p>
          <p>Published: {{ data.publishedCourses }}</p>
          <p>Archived: {{ data.archivedCourses }}</p>
          <p>
            Active enrollments:
            {{ data.activeEnrollments }}
          </p>
          <p>
            Completed students:
            {{ data.completedStudents }}
          </p>
        </section>

        <h2>My courses</h2>

        @for (course of data.courses; track course.id) {
          <article>
            <h3>{{ course.title }}</h3>

            <p>Status: {{ course.status }}</p>
            <p>Version: {{ course.version }}</p>
            <p>
              {{ course.sectionCount }} sections,
              {{ course.lessonCount }} lessons
            </p>
            <p>
              {{ course.activeEnrollments }}
              active enrollments
            </p>

            <a [routerLink]="[
              '/instructor/courses',
              course.id,
              'curriculum'
            ]">
              Edit curriculum
            </a>

            @if (course.status === 'DRAFT') {
              <button
                type="button"
                [disabled]="busyId() !== null"
                (click)="checkReadiness(course.id)"
              >
                Check readiness
              </button>

              <button
                type="button"
                [disabled]="busyId() !== null"
                (click)="publish(course)"
              >
                Publish
              </button>
            }

            @if (course.status === 'PUBLISHED') {
              <button
                type="button"
                [disabled]="busyId() !== null"
                (click)="archive(course)"
              >
                Archive
              </button>
            }

            @if (
              readiness()?.courseId === course.id;
              as matchingReport
            ) {
              @if (matchingReport) {
                <section aria-label="Readiness">
                  <p>
                    {{
                      readiness()!.ready
                        ? 'Ready to publish'
                        : 'Not ready'
                    }}
                  </p>

                  <ul>
                    @for (
                      issue of readiness()!.issues;
                      track issue
                    ) {
                      <li>{{ issue }}</li>
                    }
                  </ul>
                </section>
              }
            }
          </article>
        }
      }
    </main>
  `,
})
export class InstructorDashboardComponent {
  private readonly api =
    inject(InstructorDashboardApiService);

  private readonly destroyRef = inject(DestroyRef);

  readonly dashboard =
    signal<InstructorDashboardDto | null>(null);

  readonly readiness =
    signal<CourseReadinessDto | null>(null);

  readonly loading = signal(true);
  readonly busyId = signal<number | null>(null);
  readonly error = signal<string | null>(null);

  constructor() {
    this.load();
  }

  load(): void {
    this.loading.set(true);
    this.error.set(null);

    this.api.dashboard()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: data => {
          this.dashboard.set(data);
          this.loading.set(false);
        },
        error: error => {
          this.error.set(
            error.message || 'Unable to load dashboard.'
          );
          this.loading.set(false);
        },
      });
  }

  checkReadiness(courseId: number): void {
    this.busyId.set(courseId);
    this.error.set(null);

    this.api.readiness(courseId)
      .pipe(
        finalize(() => this.busyId.set(null)),
        takeUntilDestroyed(this.destroyRef)
      )
      .subscribe({
        next: report => this.readiness.set(report),
        error: error => this.showError(error),
      });
  }

  publish(course: InstructorCourseRowDto): void {
    this.changeStatus(
      course,
      this.api.publish(course.id, course.version)
    );
  }

  archive(course: InstructorCourseRowDto): void {
    this.changeStatus(
      course,
      this.api.archive(course.id, course.version)
    );
  }

  private changeStatus(
    course: InstructorCourseRowDto,
    request: import('rxjs').Observable<void>
  ): void {
    this.busyId.set(course.id);
    this.error.set(null);

    request.pipe(
      finalize(() => this.busyId.set(null)),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe({
      next: () => {
        this.readiness.set(null);
        this.load();
      },
      error: error => this.showError(error),
    });
  }

  private showError(error: unknown): void {
    if (
      error instanceof HttpErrorResponse &&
      error.status === 409
    ) {
      this.error.set(
        'The course is not ready or has changed. ' +
        'Refresh the dashboard and check readiness.'
      );
      return;
    }

    this.error.set(
      'The operation could not be completed.'
    );
  }
}
</CodeBlock>

### Template correction

Angular's new control-flow `@if` syntax does not support using `as` to alias an arbitrary boolean expression and then treating the alias as a readiness DTO.

Replace this section:

```html
@if (
  readiness()?.courseId === course.id;
  as matchingReport
) {
  ...
}
```

with the simpler valid template:

```html
@if (readiness(); as report) {
  @if (report.courseId === course.id) {
    <section aria-label="Readiness">
      <p>
        {{ report.ready ? 'Ready to publish' : 'Not ready' }}
      </p>

      <ul>
        @for (issue of report.issues; track issue) {
          <li>{{ issue }}</li>
        }
      </ul>
    </section>
  }
}
```

Use this corrected block in the final component.

### Route integration

**Modify:** `learnhub/src/app/app.routes.ts`

```typescript
{
  path: 'instructor/dashboard',
  canActivate: [instructorGuard],
  loadComponent: () =>
    import(
      './features/instructor/instructor-dashboard.component'
    ).then(m => m.InstructorDashboardComponent),
},
```

Keep this alongside the Lesson 18 curriculum editor route.

The guard improves navigation UX. Spring Security and service-level ownership checks remain authoritative.

---

## 14. PostgreSQL + MockMvc integration tests

These tests must verify more than HTTP status codes. We want to prove that unauthorized or stale operations leave the database unchanged.

We'll extend the existing `PostgresIntegrationTest` and reuse the `LearnHubPrincipal` introduced in Lesson 14.

### Test fixture

**Create:** `src/test/java/com/learnhub/web/InstructorPublishingIntegrationTest.java`

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

import static org.junit.jupiter.api.Assertions.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class InstructorPublishingIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired MockMvc mvc;
    @Autowired JdbcTemplate jdbc;
    @Autowired PasswordEncoder encoder;

    private long user(String email, String role) {
        return jdbc.queryForObject("""
            INSERT INTO app_users (
                email, display_name, password_hash,
                role, enabled
            )
            VALUES (?, 'Test Instructor', ?, ?, true)
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

    private long draft(long ownerId) {
        return jdbc.queryForObject("""
            INSERT INTO courses (
                title, description, instructor,
                category, price, is_free,
                level, rating, status,
                instructor_user_id
            )
            VALUES (
                'Publishing Test Course',
                'A complete test description',
                'Test Instructor',
                'Development',
                0.00,
                true,
                'BEGINNER',
                0.0,
                'DRAFT',
                ?
            )
            RETURNING id
            """,
            Long.class,
            ownerId
        );
    }

    private void curriculum(long courseId) {
        Long sectionId = jdbc.queryForObject("""
            INSERT INTO course_sections (
                course_id, title, position
            )
            VALUES (?, 'Foundations', 0)
            RETURNING id
            """,
            Long.class,
            courseId
        );

        jdbc.update("""
            INSERT INTO course_lessons (
                section_id, title, body,
                position, duration_minutes
            )
            VALUES (?, 'Introduction',
                    'Lesson body', 0, 5)
            """,
            sectionId
        );
    }

    private String status(long courseId) {
        return jdbc.queryForObject("""
            SELECT status FROM courses WHERE id = ?
            """,
            String.class,
            courseId
        );
    }

    @Test
    void ownerCanPublishReadyDraft() throws Exception {
        String email = "publish-owner@example.test";
        long ownerId = user(email, "INSTRUCTOR");
        long courseId = draft(ownerId);
        curriculum(courseId);

        mvc.perform(post(
                    "/api/v1/instructor/courses/{id}/publish",
                    courseId
                )
                .with(user(principal(
                        ownerId,
                        email,
                        SecurityRole.INSTRUCTOR
                )))
                .with(csrf())
                .contentType("application/json")
                .content("{\"expectedVersion\":0}"))
            .andExpect(status().isNoContent());

        assertEquals("PUBLISHED", status(courseId));

        Long version = jdbc.queryForObject("""
            SELECT version FROM courses WHERE id = ?
            """, Long.class, courseId);

        assertEquals(1L, version);
    }

    @Test
    void incompleteDraftCannotPublish() throws Exception {
        String email = "incomplete-owner@example.test";
        long ownerId = user(email, "INSTRUCTOR");
        long courseId = draft(ownerId);

        mvc.perform(post(
                    "/api/v1/instructor/courses/{id}/publish",
                    courseId
                )
                .with(user(principal(
                        ownerId,
                        email,
                        SecurityRole.INSTRUCTOR
                )))
                .with(csrf())
                .contentType("application/json")
                .content("{\"expectedVersion\":0}"))
            .andExpect(status().isConflict());

        assertEquals("DRAFT", status(courseId));
    }

    @Test
    void anotherInstructorCannotPublish()
            throws Exception {
        long ownerId = user(
                "real-owner@example.test",
                "INSTRUCTOR"
        );
        String email = "other-instructor@example.test";
        long otherId = user(email, "INSTRUCTOR");
        long courseId = draft(ownerId);
        curriculum(courseId);

        mvc.perform(post(
                    "/api/v1/instructor/courses/{id}/publish",
                    courseId
                )
                .with(user(principal(
                        otherId,
                        email,
                        SecurityRole.INSTRUCTOR
                )))
                .with(csrf())
                .contentType("application/json")
                .content("{\"expectedVersion\":0}"))
            .andExpect(status().isForbidden());

        assertEquals("DRAFT", status(courseId));
    }

    @Test
    void staleVersionCannotPublish() throws Exception {
        String email = "stale-owner@example.test";
        long ownerId = user(email, "INSTRUCTOR");
        long courseId = draft(ownerId);
        curriculum(courseId);

        jdbc.update("""
            UPDATE courses
            SET version = 1
            WHERE id = ?
            """, courseId);

        mvc.perform(post(
                    "/api/v1/instructor/courses/{id}/publish",
                    courseId
                )
                .with(user(principal(
                        ownerId,
                        email,
                        SecurityRole.INSTRUCTOR
                )))
                .with(csrf())
                .contentType("application/json")
                .content("{\"expectedVersion\":0}"))
            .andExpect(status().isConflict());

        assertEquals("DRAFT", status(courseId));
    }

    @Test
    void publishingRequiresCsrf() throws Exception {
        String email = "csrf-owner@example.test";
        long ownerId = user(email, "INSTRUCTOR");
        long courseId = draft(ownerId);
        curriculum(courseId);

        mvc.perform(post(
                    "/api/v1/instructor/courses/{id}/publish",
                    courseId
                )
                .with(user(principal(
                        ownerId,
                        email,
                        SecurityRole.INSTRUCTOR
                )))
                .contentType("application/json")
                .content("{\"expectedVersion\":0}"))
            .andExpect(status().isForbidden());

        assertEquals("DRAFT", status(courseId));
    }

    @Test
    void anonymousDashboardIsUnauthorized()
            throws Exception {
        mvc.perform(get("/api/v1/instructor/dashboard"))
            .andExpect(status().isUnauthorized());
    }
}
</CodeBlock>

The fixture assumes the earlier `courses` schema allows the omitted nullable/default columns. It also assumes the actual `LearnHubPrincipal` constructor matches the previous lessons.

### Add these essential tests

The class above is the foundation. Extend it with:

| Test | Assertion |
|---|---|
| Owner archives published course | Status `ARCHIVED`, timestamp populated, version increments |
| Archive draft | `409`, status unchanged |
| Publish archived course | `409`, status unchanged |
| Readiness response | Missing curriculum issues are returned |
| Instructor A dashboard | No courses or metrics belonging to Instructor B |
| Student requests dashboard | `403` |
| Stale archive request | `409`, published state preserved |
| Curriculum edit increments version | Old publish version rejected |
| Two competing publishers | Exactly one transition succeeds |

### A test for dashboard isolation

Create two instructors, each with one course, and request the dashboard as Instructor A.

Verify:

```text
totalCourses = 1
courses[0].id = Instructor A's course ID
Instructor B's course ID is absent
```

Also create enrollments and completion rows for both courses and verify the aggregate counts do not cross ownership boundaries.

### True concurrency test

A synchronous MockMvc test with `@Transactional` does not simulate two independent database transactions.

For the concurrency exercise, use two threads and separate transactions, or issue real HTTP requests against a disposable Testcontainers database. Coordinate their start with a barrier, then assert:

- Only one publish transition succeeds.
- The other request receives a conflict.
- The course version increments once.
- The database status is `PUBLISHED`.

---

## 15. Angular API contract tests

**Create:** `learnhub/src/app/core/api/instructor-dashboard-api.service.spec.ts`

<CodeBlock language="typescript" editable>
import { TestBed } from '@angular/core/testing';
import { provideHttpClient }
  from '@angular/common/http';

import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';

import { InstructorDashboardApiService }
  from './instructor-dashboard-api.service';

describe('InstructorDashboardApiService', () => {
  let api: InstructorDashboardApiService;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });

    api = TestBed.inject(
      InstructorDashboardApiService
    );
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());

  it('loads the authenticated instructor dashboard', () => {
    api.dashboard().subscribe();

    const request = http.expectOne(
      '/api/v1/instructor/dashboard'
    );

    expect(request.request.method).toBe('GET');
    expect(request.request.params.has('instructorId'))
      .toBeFalse();

    request.flush({
      totalCourses: 0,
      draftCourses: 0,
      publishedCourses: 0,
      archivedCourses: 0,
      activeEnrollments: 0,
      completedStudents: 0,
      courses: [],
    });
  });

  it('publishes using the observed course version', () => {
    api.publish(42, 7).subscribe();

    const request = http.expectOne(      '/api/v1/instructor/courses/42/publish'
    );

    expect(request.request.method).toBe('POST');
    expect(request.request.body).toEqual({
      expectedVersion: 7,
    });

    request.flush(null, {
      status: 204,
      statusText: 'No Content',
    });
  });

  it('archives using the observed course version', () => {
    api.archive(42, 8).subscribe();

    const request = http.expectOne(
      '/api/v1/instructor/courses/42/archive'
    );

    expect(request.request.method).toBe('POST');
    expect(request.request.body).toEqual({
      expectedVersion: 8,
    });

    request.flush(null, {
      status: 204,
      statusText: 'No Content',
    });
  });

  it('loads readiness for a specific course', () => {
    api.readiness(42).subscribe(report => {
      expect(report.ready).toBeFalse();
      expect(report.issues.length).toBe(1);
    });

    const request = http.expectOne(
      '/api/v1/instructor/courses/42/readiness'
    );

    expect(request.request.method).toBe('GET');

    request.flush({
      courseId: 42,
      version: 3,
      ready: false,
      sectionCount: 0,
      lessonCount: 0,
      issues: ['Add at least one section.'],
    });
  });
});
</CodeBlock>

These tests verify the Angular request contract. They do not replace backend authorization or browser-level CSRF tests.

---

## 16. Run LearnHub end to end

### Terminal A — Spring Boot

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

### Terminal B — Angular

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd serve --proxy-config proxy.conf.json
```

### Terminal C — PostgreSQL

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT version, description, success FROM flyway_schema_history ORDER BY installed_rank;"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT id, title, status, version, published_at, archived_at FROM courses ORDER BY id DESC LIMIT 20;"
```

Verify the following browser flow:

1. Sign in as an instructor with a course you own.
2. Open `/instructor/dashboard`.
3. Inspect course counts and the course list.
4. Open a draft course's curriculum editor.
5. Add a section and a lesson.
6. Return to the dashboard and refresh.
7. Check readiness.
8. Publish the draft.
9. Verify that it appears in the public catalog.
10. Archive it and verify it disappears from public discovery.

### Important archive policy

Lesson 18's student content service requires a published course before granting enrollment-based access. Therefore, **archiving a course currently removes lesson access for enrolled students**, even though their enrollment and progress rows remain.

That is an explicit consequence of the current design, not an accidental frontend behavior. Before production, decide whether archived courses remain accessible to existing students. If they should, change the access policy deliberately and test it.

---

## 17. Engineering review: important integration corrections

There are several areas where the earlier lessons must be reconciled before this implementation is production-ready.

**Optimistic locking and curriculum edits.** All curriculum mutations must acquire the same parent course lock and increment its version. Any code path that bypasses that protocol can invalidate the stale-edit guarantee.

**Course ownership and JPA versioning.** The Lesson 15 owner checks must remain in force for course editing, publishing, archiving, and deletion. Adding `@Version` does not replace authorization.

**Legacy publish and archive methods.** Remove competing routes that still call the older `changeStatus()` method. A single publishing service must enforce readiness, ownership, timestamps, and expected versions.

**Historical timestamps.** The Flyway backfill uses an approximate date for older published courses. Do not present those values as verified original publication events.

**Dashboard performance.** The SQL aggregation is suitable for learning and modest datasets. For large catalogs, filter course IDs earlier, paginate course rows, and measure query plans with `EXPLAIN (ANALYZE, BUFFERS)`.

**Published curriculum edits.** Lesson 18 permits edits to published curricula. That can alter readiness, completion percentages, and the meaning of historical progress. Consider locking published curricula or introducing versioned releases.

**JPA persistence-context synchronization.** A direct JDBC version bump must not be followed by flushing a stale managed `CourseEntity` in the same persistence context. In particular, inspect the old `CourseApplicationService` update methods and refactor their version handling rather than mixing unmanaged SQL changes with stale JPA state.

**Angular dashboard state.** The example refreshes after a successful transition. For production, prevent stale overlapping loads and consider a dedicated operation error per course rather than one page-level error.

**Error response consistency.** Readiness failures, stale versions, and invalid transitions currently return `409`. Consider adding stable machine-readable error codes so Angular can distinguish them without parsing message text.

---

## 18. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="shield-check" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise A — Ownership-safe analytics**

      Create two instructors, each with multiple courses and enrollments. Prove that neither dashboard exposes the other's course IDs, counts, or completion metrics.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="git-compare" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise B — Stale editor simulation**

      Open the same draft in two browser tabs. Modify the curriculum in one tab, then attempt to publish using the old version in the other. Verify a `409` and a safe refresh workflow.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="database" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise C — Optimize PostgreSQL analytics**

      Rewrite the aggregation query to filter the instructor's course IDs before aggregating. Compare query plans and execution times against the original.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="list-checks" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise D — Rich readiness rules**

      Add requirements for a minimum total lesson duration, nonempty section titles, and a valid thumbnail. Return stable issue codes as well as human-readable messages.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="archive" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise E — Archived-course access policy**

      Decide whether existing students retain access to archived courses. Implement the rule consistently in enrollment eligibility, protected lesson access, and Angular navigation.
    </box>
  </row>
</box>

## 19. Knowledge quiz

{@body const questions=[{q:"What does @Version protect against?",options:["Unauthorized instructor access","Lost updates from conflicting entity changes","Invalid CSRF tokens"],correct:1,why:"Optimistic locking detects conflicting updates to a versioned entity; authorization remains a separate concern."},{q:"Why must curriculum edits increment the parent course version?",options:["Child changes do not automatically change the parent's version","PostgreSQL requires every table to have a version","It makes Angular routes faster"],correct:0,why:"Readiness and publication depend on child content, so their changes must invalidate stale course versions."},{q:"Why is a direct join between enrollments and progress risky in dashboard analytics?",options:["It may multiply enrollment rows and inflate counts","It disables foreign keys","It exposes session cookies"],correct:0,why:"One-to-many joins can duplicate parent rows; aggregate carefully before combining metrics."},{q:"Where does the dashboard obtain the instructor ID?",options:["A query parameter supplied by Angular","The authenticated principal on the server","A browser localStorage value"],correct:1,why:"Server-derived identity prevents users from selecting another instructor's private analytics."},{q:"What should happen when a draft has no lessons and the instructor tries to publish?",options:["Publish and hide it later","Return a readiness conflict and leave it in DRAFT","Automatically create a sample lesson"],correct:1,why:"Publishing is a controlled transition that must not occur when readiness requirements fail."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Lesson 19 — Publishing and analytics**
    <text color="secondary" size="xs">5 questions</text>
  </row>
  {#each questions as q,i}
    <box key={i} gap={2}>
      **{i+1}. {q.q}**
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setGraded(false);}}>
        {#each q.options as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if graded}
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
  {#if graded}
    <title size="lg">{questions.filter((q,i)=>answers[i]===q.correct).length} / 5 correct</title>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==questions.length} onClick={()=>setGraded(true)}>Check answers</button>
    <button variant="outline" color="secondary" onClick={()=>{setAnswers({});setGraded(false);}}>Reset</button>
  </row>
</box>

## 20. Completion checklist

{@body const groups=[{name:"Database and backend",items:["Verified Lesson 18 baseline","Selected next unused Flyway version","Added course version and publication timestamps","Updated CourseEntity with @Version","Implemented readiness rules","Implemented version-aware publish/archive","Made curriculum mutations increment course version","Removed legacy publishing bypasses"]},{name:"Security and analytics",items:["Implemented instructor-scoped SQL analytics","Created dashboard DTOs and service","Created instructor REST controllers","Protected routes in SecurityFilterChain","Verified @PreAuthorize and owner checks","Verified 403/404/409 behavior","Reviewed archived-course access policy"]},{name:"Angular",items:["Created typed dashboard models","Created dashboard API service","Built standalone dashboard component","Added readiness display","Connected publish and archive actions","Handled stale version conflicts","Added guarded instructor route","Verified same-origin CSRF"]},{name:"Tests and delivery",items:["Added PostgreSQL publishing tests","Tested readiness failures","Tested cross-instructor denial","Tested stale version rejection","Tested dashboard isolation","Added Angular HTTP tests","Ran Maven clean verify","Ran Angular tests and build","Completed two exercises","Completed quiz","Reviewed and committed changes"]}]}
{@body const [checked,setChecked]=DIL.useState({})}
{@body const total=groups.reduce((n,g)=>n+g.items.length,0)}
{@body const completed=Object.values(checked).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 19 progress**
      <text color="secondary" size="xs">Check items after local verification.</text>
    </box>
    <title size="lg" tabularNums>{completed}/{total}</title>
  </row>
  <box background="surface-tertiary" radius="full" height="8px" clip>
    <box background="rgba(109,40,217,0.85)" radius="full" height="100%" width={`${completed/total*100}%`}/>
  </box>
  {#each groups as group,i}
    <box key={i} gap={2}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!checked[`${i}-${j}`]} onChange={v=>setChecked(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if completed===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 19 checklist completed.**
    </row>
  {/if}
</box>

## 21. Final verification and Git checkpoints

### Backend

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=InstructorPublishingIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Publishing integration tests failed."
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

### Commit after verification

```powershell
cd $HOME\angular-learning\learnhub-backend

git add src\main src\test
git diff --cached --stat
git commit -m "Add instructor analytics and versioned publishing workflow"
```

```powershell
cd $HOME\angular-learning\learnhub

git add src\app
git diff --cached --stat
git commit -m "Build instructor dashboard and publishing controls"
```

These are implementation instructions, not executed changes. The code must be reconciled with the exact Lesson 14–18 entity and security signatures in your workspace, and all tests must pass locally before committing.

## 22. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="credit-card" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 20 — Secure Payments and Paid Enrollments**

    We'll add a payment-provider integration, server-created checkout sessions, verified webhooks, idempotent fulfillment, transaction-safe payment records, refunds, and paid-course enrollment entitlements.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 20: secure paid-course checkout, payment-provider integration, server-created checkout sessions, webhook signature verification, idempotent event processing, PostgreSQL Flyway migrations, payment and entitlement entities, transactional paid enrollments, refunds and failure states, Spring Security, Angular standalone checkout integration, MockMvc and Testcontainers tests, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 19 with experienced-Java pacing.")}>Start Lesson 20 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 19 deliverable:** An instructor-owned dashboard, PostgreSQL analytics, curriculum readiness validation, controlled publishing and archiving, optimistic locking, and Angular management UI.

The central principle is: **publishing is a validated, authorized state transition—not simply changing a status field.**