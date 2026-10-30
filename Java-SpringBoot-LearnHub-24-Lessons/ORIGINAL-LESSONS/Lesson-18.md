# LearnHub Java & Spring Boot — Lesson 18

<text color="secondary" size="sm">Course curriculum · Ordered sections and lessons · Enrollment-gated content · Progress tracking · Angular course player</text>

**Stack:** Java 21 · Spring Boot 3.5.x · PostgreSQL · Flyway · Spring Data JPA · Spring Security · Angular standalone · JUnit 5 · MockMvc · Testcontainers

**Estimated time:** 8–12 hours, including exercises and verification.

## 1. What we're building

In Lesson 17, students gained persistent wishlists and enrollments. Today we'll make those enrollments useful by introducing a structured curriculum and a secure learning experience.

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" radius="lg" padding={3} gap={1}>
    **Course: Advanced Spring Boot**
    <text color="secondary" size="xs">Owned by an instructor · Published to the catalog</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <grid columns={2} gap={2}>
    <grid-item>
      <box border radius="lg" padding={3} gap={2}>
        **Section 1 — Foundations**
        <text size="sm">1. Introduction</text>
        <text size="sm">2. Project setup</text>
        <text size="sm">3. REST controllers</text>
      </box>
    </grid-item>
    <grid-item>
      <box border radius="lg" padding={3} gap={2}>
        **Section 2 — Persistence**
        <text size="sm">1. JPA entities</text>
        <text size="sm">2. Transactions</text>
        <text size="sm">3. Testing</text>
      </box>
    </grid-item>
  </grid>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} gap={1}>
    **Student enrollment**
    <text color="secondary" size="xs">ACTIVE → authorized lesson access</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" radius="lg" padding={3} gap={1}>
    **Lesson progress**
    <text color="secondary" size="xs">Completed lessons / eligible lessons = progress percentage</text>
  </box>
</box>

### Lesson 18 deliverables

| Capability | Who can use it |
|---|---|
| Add course sections | Course owner or admin |
| Add lessons to sections | Course owner or admin |
| Reorder sections and lessons | Course owner or admin |
| View public curriculum outline | Anyone, published courses only |
| Read protected lesson content | Actively enrolled student, course owner, or admin |
| Mark a lesson complete | Actively enrolled student |
| View course progress | Actively enrolled student |
| Access another student's progress | Denied |
| Edit another instructor's curriculum | Denied |

We'll implement **text-based lessons** first. Video hosting, uploads, streaming, and signed media URLs require a separate storage and authorization design.

---

## 2. Security and lifecycle rules

Before coding, define the invariants.

1. A section belongs to exactly one course.
2. A lesson belongs to exactly one section.
3. Section order is unique within a course.
4. Lesson order is unique within a section.
5. Only an instructor who owns the course, or an admin, can modify its curriculum.
6. A public curriculum outline exposes titles and ordering, **not protected lesson bodies**.
7. A student must have an `ACTIVE` enrollment to read lesson content.
8. A student can mark only their own lessons complete.
9. Completing the same lesson twice does not create duplicate progress.
10. Progress is calculated on the server.
11. A lesson must belong to the course whose enrollment is being checked.
12. Unpublished courses are not accessible to ordinary enrolled students.

### Access matrix

| Caller | Public outline | Lesson content | Mark complete | Edit curriculum |
|---|---|---|---|---|
| Anonymous | Published only | No | No | No |
| Student, not enrolled | Published only | No | No | No |
| Student, actively enrolled | Published only | Yes | Yes | No |
| Instructor, course owner | Yes | Yes | No | Yes |
| Other instructor | Published only | No | No | No |
| Admin | Yes | Yes | No | Yes |

**Important:** A wishlist entry does not grant lesson access. Only an eligible enrollment or trusted management role does.

---

## 3. Workspace and baseline verification

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info

git status
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 17 backend baseline is not green."
}
```

Frontend:

```powershell
cd $HOME\angular-learning\learnhub

node --version
npm --version
ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 17 Angular baseline is not green."
}
```

Inspect the Flyway migrations:

```powershell
cd $HOME\angular-learning\learnhub-backend

Get-ChildItem src\main\resources\db\migration |
    Sort-Object Name
```

Lesson 17 used `V8__create_wishlists.sql` as an example. **This lesson uses `V9` only if V9 is unused.** Flyway versions must be unique and applied in order.

Inspect the existing authorization contracts:

```powershell
Get-Content src\main\java\com\learnhub\security\CurrentActor.java

Get-Content src\main\java\com\learnhub\security\CourseOwnershipPolicy.java

Get-Content src\main\java\com\learnhub\persistence\CourseEntity.java

Get-Content src\main\java\com\learnhub\persistence\EnrollmentEntity.java
```

We'll reuse the authenticated database principal ID, existing course owner foreign key, and Lesson 17 enrollment status.

---

## 4. Database design

We need three tables:

- `course_sections`: ordered sections within a course.
- `course_lessons`: ordered lessons within a section.
- `lesson_progress`: student completion records.

We'll store a lesson body as text, not as a publicly accessible URL.

### Flyway migration

**Create:** `src/main/resources/db/migration/V9__create_curriculum_and_progress.sql`

<CodeBlock language="sql" editable>
-- Lesson 18
-- Course curriculum and student progress.
--
-- Rename V9 to the next unused Flyway version
-- if your migration history differs.

CREATE TABLE course_sections (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY
        PRIMARY KEY,

    course_id BIGINT NOT NULL,

    title VARCHAR(180) NOT NULL,

    position INTEGER NOT NULL,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_course_sections_course
        FOREIGN KEY (course_id)
        REFERENCES courses(id)
        ON DELETE RESTRICT,

    CONSTRAINT uq_course_sections_position
        UNIQUE (course_id, position)
        DEFERRABLE INITIALLY IMMEDIATE,

    CONSTRAINT chk_course_sections_position
        CHECK (position >= 0),

    CONSTRAINT chk_course_sections_title
        CHECK (length(btrim(title)) > 0)
);

CREATE TABLE course_lessons (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY
        PRIMARY KEY,

    section_id BIGINT NOT NULL,

    title VARCHAR(180) NOT NULL,

    body TEXT NOT NULL,

    position INTEGER NOT NULL,

    duration_minutes INTEGER NOT NULL DEFAULT 0,

    is_preview BOOLEAN NOT NULL DEFAULT FALSE,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_course_lessons_section
        FOREIGN KEY (section_id)
        REFERENCES course_sections(id)
        ON DELETE RESTRICT,

    CONSTRAINT uq_course_lessons_position
        UNIQUE (section_id, position)
        DEFERRABLE INITIALLY IMMEDIATE,

    CONSTRAINT chk_course_lessons_position
        CHECK (position >= 0),

    CONSTRAINT chk_course_lessons_duration
        CHECK (duration_minutes >= 0),

    CONSTRAINT chk_course_lessons_title
        CHECK (length(btrim(title)) > 0)
);

CREATE TABLE lesson_progress (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY
        PRIMARY KEY,

    enrollment_id BIGINT NOT NULL,

    lesson_id BIGINT NOT NULL,

    completed_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_lesson_progress_enrollment
        FOREIGN KEY (enrollment_id)
        REFERENCES enrollments(id)
        ON DELETE RESTRICT,

    CONSTRAINT fk_lesson_progress_lesson
        FOREIGN KEY (lesson_id)
        REFERENCES course_lessons(id)
        ON DELETE RESTRICT,

    CONSTRAINT uq_lesson_progress_enrollment_lesson
        UNIQUE (enrollment_id, lesson_id)
);

CREATE INDEX idx_course_lessons_section
    ON course_lessons(section_id);

CREATE INDEX idx_lesson_progress_lesson
    ON lesson_progress(lesson_id);

CREATE INDEX idx_course_sections_course
    ON course_sections(course_id);
</CodeBlock>

### Why use `enrollment_id` instead of `student_user_id` in progress?

An enrollment is the durable relationship between a student and a course. Tying progress to it gives us a natural way to enforce student ownership and course membership.

### A database invariant to understand

The schema guarantees that each progress row references a valid enrollment and lesson. It does **not**, by itself, guarantee that the lesson belongs to the enrollment's course.

We must enforce that in the transactional service. A stricter database-level design could use composite keys, additional course IDs, and composite foreign keys.

### Why deferrable unique ordering constraints?

When reordering positions, two rows may temporarily exchange position numbers.

PostgreSQL deferrable constraints allow the application to defer uniqueness checks until the transaction completes, making reorder operations easier to implement safely.

---

## 5. JPA curriculum entities

We'll keep relationships unidirectional where possible. This avoids accidental serialization of entire object graphs and unnecessary collection fetching.

Create the package files under:

`src/main/java/com/learnhub/persistence/`

### 5.1 CourseSectionEntity.java

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import jakarta.persistence.*;
import java.util.Objects;

@Entity
@Table(name = "course_sections")
public class CourseSectionEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "course_id", nullable = false)
    private CourseEntity course;

    @Column(nullable = false, length = 180)
    private String title;

    @Column(nullable = false)
    private int position;

    protected CourseSectionEntity() {}

    public CourseSectionEntity(
            CourseEntity course,
            String title,
            int position
    ) {
        this.course = Objects.requireNonNull(course);
        rename(title);
        this.position = position;
    }

    public void rename(String title) {
        if (title == null || title.isBlank()
                || title.length() > 180) {
            throw new IllegalArgumentException(
                    "Section title must contain 1–180 characters."
            );
        }
        this.title = title.trim();
    }

    public Long getId() { return id; }
    public CourseEntity getCourse() { return course; }
    public String getTitle() { return title; }
    public int getPosition() { return position; }
}
</CodeBlock>

### 5.2 CourseLessonEntity.java

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import jakarta.persistence.*;
import java.util.Objects;

@Entity
@Table(name = "course_lessons")
public class CourseLessonEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "section_id", nullable = false)
    private CourseSectionEntity section;

    @Column(nullable = false, length = 180)
    private String title;

    @Column(nullable = false, columnDefinition = "text")
    private String body;

    @Column(nullable = false)
    private int position;

    @Column(name = "duration_minutes", nullable = false)
    private int durationMinutes;

    @Column(name = "is_preview", nullable = false)
    private boolean preview;

    protected CourseLessonEntity() {}

    public CourseLessonEntity(
            CourseSectionEntity section,
            String title,
            String body,
            int position,
            int durationMinutes
    ) {
        this.section = Objects.requireNonNull(section);
        update(title, body, durationMinutes);
        this.position = position;
        this.preview = false;
    }

    public void update(
            String title,
            String body,
            int durationMinutes
    ) {
        if (title == null || title.isBlank()
                || title.length() > 180) {
            throw new IllegalArgumentException(
                    "Invalid lesson title."
            );
        }
        if (body == null || body.isBlank()) {
            throw new IllegalArgumentException(
                    "Lesson body is required."
            );
        }
        if (durationMinutes < 0) {
            throw new IllegalArgumentException(
                    "Duration cannot be negative."
            );
        }
        this.title = title.trim();
        this.body = body;
        this.durationMinutes = durationMinutes;
    }

    public Long getId() { return id; }
    public CourseSectionEntity getSection() { return section; }
    public String getTitle() { return title; }
    public String getBody() { return body; }
    public int getPosition() { return position; }
    public int getDurationMinutes() { return durationMinutes; }
    public boolean isPreview() { return preview; }
}
</CodeBlock>

We store `is_preview` for future product requirements, but **Lesson 18 does not grant public access to preview lesson bodies**. Preview access requires its own explicitly authorized endpoint and policy.

### 5.3 LessonProgressEntity.java

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import jakarta.persistence.*;
import java.time.OffsetDateTime;
import java.util.Objects;

@Entity
@Table(name = "lesson_progress")
public class LessonProgressEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "enrollment_id", nullable = false)
    private EnrollmentEntity enrollment;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "lesson_id", nullable = false)
    private CourseLessonEntity lesson;

    @Column(
        name = "completed_at",
        insertable = false,
        updatable = false
    )
    private OffsetDateTime completedAt;

    protected LessonProgressEntity() {}

    public LessonProgressEntity(
            EnrollmentEntity enrollment,
            CourseLessonEntity lesson
    ) {
        this.enrollment = Objects.requireNonNull(enrollment);
        this.lesson = Objects.requireNonNull(lesson);
    }

    public Long getId() { return id; }
    public EnrollmentEntity getEnrollment() { return enrollment; }
    public CourseLessonEntity getLesson() { return lesson; }
    public OffsetDateTime getCompletedAt() { return completedAt; }
}
</CodeBlock>

**Important:** We will insert progress with PostgreSQL `ON CONFLICT DO NOTHING`, not by blindly saving this entity, so repeated completion is idempotent.

---

## 6. Repositories

### 6.1 JpaCourseSectionRepository.java

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
import java.util.Optional;

public interface JpaCourseSectionRepository
        extends JpaRepository<CourseSectionEntity, Long> {

    List<CourseSectionEntity>
    findByCourse_IdOrderByPositionAscIdAsc(Long courseId);

    Optional<CourseSectionEntity>
    findByIdAndCourse_Id(Long id, Long courseId);
}
</CodeBlock>

### 6.2 JpaCourseLessonRepository.java

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.Optional;

public interface JpaCourseLessonRepository
        extends JpaRepository<CourseLessonEntity, Long> {

    List<CourseLessonEntity>
    findBySection_IdOrderByPositionAscIdAsc(Long sectionId);

    @Query("""
        SELECT l
        FROM CourseLessonEntity l
        JOIN FETCH l.section s
        WHERE l.id = :lessonId
          AND s.course.id = :courseId
        """)
    Optional<CourseLessonEntity> findInCourse(
            @Param("courseId") Long courseId,
            @Param("lessonId") Long lessonId
    );

    long countBySection_Course_Id(Long courseId);
}
</CodeBlock>

### 6.3 JpaLessonProgressRepository.java

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface JpaLessonProgressRepository
        extends JpaRepository<LessonProgressEntity, Long> {

    @Modifying
    @Query(value = """
        INSERT INTO lesson_progress (
            enrollment_id, lesson_id
        )
        VALUES (:enrollmentId, :lessonId)
        ON CONFLICT (enrollment_id, lesson_id)
        DO NOTHING
        """, nativeQuery = true)
    int completeIfAbsent(
            @Param("enrollmentId") Long enrollmentId,
            @Param("lessonId") Long lessonId
    );

    @Query("""
        SELECT p.lesson.id
        FROM LessonProgressEntity p
        WHERE p.enrollment.id = :enrollmentId
        """)
    List<Long> completedLessonIds(
            @Param("enrollmentId") Long enrollmentId
    );

    @Query("""
        SELECT COUNT(p)
        FROM LessonProgressEntity p
        WHERE p.enrollment.id = :enrollmentId
        """)
    long countCompleted(
            @Param("enrollmentId") Long enrollmentId
    );
}
</CodeBlock>

### 6.4 Extend JpaEnrollmentRepository.java

Add:

```java
Optional<EnrollmentEntity>
findByStudent_IdAndCourse_IdAndStatus(
        Long studentId,
        Long courseId,
        String status
);
```

Ensure `java.util.Optional` is imported.

### Why not return entities from controllers?

`CourseLessonEntity` contains the protected lesson body and relationships. Returning entities directly risks exposing content through the wrong endpoint.

We'll use separate **outline DTOs** and **lesson content DTOs**.

---

## 7. REST DTOs

Create the following files under:

`src/main/java/com/learnhub/web/dto/`

### 7.1 CurriculumWriteRequests.java — use separate files

Java public records require matching filenames. Create these three files.

**SectionWriteRequest.java**

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record SectionWriteRequest(
        @NotBlank @Size(max = 180) String title
) {}
</CodeBlock>

**LessonWriteRequest.java**

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.PositiveOrZero;
import jakarta.validation.constraints.Size;

public record LessonWriteRequest(
        @NotBlank @Size(max = 180) String title,
        @NotBlank String body,
        @PositiveOrZero int durationMinutes
) {}
</CodeBlock>

**ReorderRequest.java**

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import java.util.List;

public record ReorderRequest(
        @NotEmpty List<@NotNull @Positive Long> ids
) {}
</CodeBlock>

### 7.2 SectionOutlineResponse.java

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import java.util.List;

public record SectionOutlineResponse(
        long id,
        String title,
        int position,
        List<LessonOutlineResponse> lessons
) {}
</CodeBlock>

### 7.3 LessonOutlineResponse.java

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

public record LessonOutlineResponse(
        long id,
        String title,
        int position,
        int durationMinutes
) {}
</CodeBlock>

### 7.4 LessonContentResponse.java

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

public record LessonContentResponse(
        long id,
        long courseId,
        long sectionId,
        String title,
        String body,
        int durationMinutes
) {}
</CodeBlock>

### 7.5 CourseProgressResponse.java

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import java.util.List;

public record CourseProgressResponse(
        long courseId,
        long enrollmentId,
        long completedLessons,
        long totalLessons,
        int percentage,
        boolean completed,
        List<Long> completedLessonIds
) {}
</CodeBlock>

The public outline DTO deliberately omits `body`. That omission is a security boundary, not just a bandwidth optimization.

---

## 8. Curriculum authorization

Lesson 15 introduced `CurrentActor` and course ownership checks. We'll reuse the authenticated principal rather than accept an instructor ID from Angular.

To make the lesson self-contained at the SQL boundary, this service checks ownership using `courses.instructor_user_id`. It does not replace the Lesson 15 ownership policy; you can delegate to that policy once the signatures are aligned.

**Create:** `src/main/java/com/learnhub/service/CurriculumAccessService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.persistence.JpaEnrollmentRepository;
import com.learnhub.persistence.EnrollmentEntity;
import com.learnhub.security.CurrentActor;
import com.learnhub.security.SecurityRole;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.stereotype.Service;

import java.util.Objects;

@Service
public class CurriculumAccessService {

    private final CurrentActor actor;
    private final JpaCourseRepository courses;
    private final JpaEnrollmentRepository enrollments;
    private final JdbcTemplate jdbc;

    public CurriculumAccessService(
            CurrentActor actor,
            JpaCourseRepository courses,
            JpaEnrollmentRepository enrollments,
            JdbcTemplate jdbc
    ) {
        this.actor = actor;
        this.courses = courses;
        this.enrollments = enrollments;
        this.jdbc = jdbc;
    }

    public CourseEntity publishedCourse(long courseId) {
        CourseEntity course = courses.findById(courseId)
                .orElseThrow(() ->
                        new CourseNotFoundException(courseId)
                );

        if (course.getStatus() != CourseStatus.PUBLISHED) {
            throw new CourseNotFoundException(courseId);
        }

        return course;
    }

    public CourseEntity manageableCourse(long courseId) {
        CourseEntity course = courses.findById(courseId)
                .orElseThrow(() ->
                        new CourseNotFoundException(courseId)
                );

        var principal = actor.require();

        if (principal.role() == SecurityRole.ADMIN) {
            return course;
        }

        if (principal.role() != SecurityRole.INSTRUCTOR) {
            throw new AccessDeniedException(
                    "Course management is not permitted."
            );
        }

        Long ownerId = jdbc.queryForObject("""
                SELECT instructor_user_id
                FROM courses
                WHERE id = ?
                """,
                Long.class,
                courseId
        );

        if (!Objects.equals(ownerId, principal.id())) {
            throw new AccessDeniedException(
                    "Course management is not permitted."
            );
        }

        return course;
    }

    public EnrollmentEntity activeEnrollment(
            long courseId
    ) {
        // Do not reveal unpublished course content.
        publishedCourse(courseId);

        var principal = actor.require();

        if (principal.role() != SecurityRole.STUDENT) {
            throw new AccessDeniedException(
                    "Active student enrollment required."
            );
        }

        return enrollments
                .findByStudent_IdAndCourse_IdAndStatus(
                        principal.id(),
                        courseId,
                        "ACTIVE"
                )
                .orElseThrow(() ->
                        new AccessDeniedException(
                                "Active enrollment required."
                        )
                );
    }

    public void requireContentAccess(long courseId) {
        var principal = actor.require();

        if (principal.role() == SecurityRole.ADMIN) {
            courses.findById(courseId)
                    .orElseThrow(() ->
                            new CourseNotFoundException(courseId)
                    );
            return;
        }

        if (principal.role() == SecurityRole.INSTRUCTOR) {
            manageableCourse(courseId);
            return;
        }

        activeEnrollment(courseId);
    }
}
</CodeBlock>

Import `CourseNotFoundException` from the package established in Lesson 4.

### Why the role check is not enough

`ROLE_INSTRUCTOR` grants the general capability to manage curriculum. The database owner ID determines **which** curriculum the instructor may manage.

The service must be invoked behind the existing Spring Security method and HTTP authorization rules. If a `CurrentActor` method or principal role accessor differs in your project, adapt it at this boundary.

---

## 9. Instructor curriculum management

We'll use JPA for entities and PostgreSQL row locking for ordered inserts and reorders.

### Why lock the parent course row?

Two instructors' requests—or two tabs belonging to the same instructor—might try to append a section simultaneously.

If both calculate the next position as `3`, one insert fails the unique constraint.

Locking the parent course row serializes position allocation for that course, provided all curriculum mutations follow the same locking convention.

**Create:** `src/main/java/com/learnhub/service/CurriculumManagementService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.persistence.*;
import com.learnhub.web.dto.*;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

@Service
public class CurriculumManagementService {

    private final CurriculumAccessService access;
    private final JpaCourseSectionRepository sections;
    private final JpaCourseLessonRepository lessons;
    private final JdbcTemplate jdbc;

    public CurriculumManagementService(
            CurriculumAccessService access,
            JpaCourseSectionRepository sections,
            JpaCourseLessonRepository lessons,
            JdbcTemplate jdbc
    ) {
        this.access = access;
        this.sections = sections;
        this.lessons = lessons;
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
    @Transactional
    public long addSection(
            long courseId,
            SectionWriteRequest request
    ) {
        var course = access.manageableCourse(courseId);
        lockCourse(courseId);

        Integer nextPosition = jdbc.queryForObject("""
                SELECT COALESCE(MAX(position), -1) + 1
                FROM course_sections
                WHERE course_id = ?
                """,
                Integer.class,
                courseId
        );

        var section = sections.saveAndFlush(
                new CourseSectionEntity(
                        course,
                        request.title(),
                        nextPosition
                )
        );

        return section.getId();
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional
    public long addLesson(
            long courseId,
            long sectionId,
            LessonWriteRequest request
    ) {
        access.manageableCourse(courseId);        lockCourse(courseId);

        var section = sections
                .findByIdAndCourse_Id(sectionId, courseId)
                .orElseThrow(() ->
                        new CurriculumNotFoundException()
                );

        Integer nextPosition = jdbc.queryForObject("""
                SELECT COALESCE(MAX(position), -1) + 1
                FROM course_lessons
                WHERE section_id = ?
                """,
                Integer.class,
                sectionId
        );

        var lesson = lessons.saveAndFlush(
                new CourseLessonEntity(
                        section,
                        request.title(),
                        request.body(),
                        nextPosition,
                        request.durationMinutes()
                )
        );

        return lesson.getId();
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional
    public void updateLesson(
            long courseId,
            long lessonId,
            LessonWriteRequest request
    ) {
        access.manageableCourse(courseId);
        lockCourse(courseId);

        var lesson = lessons
                .findInCourse(courseId, lessonId)
                .orElseThrow(CurriculumNotFoundException::new);

        lesson.update(
                request.title(),
                request.body(),
                request.durationMinutes()
        );
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional
    public void reorderSections(
            long courseId,
            ReorderRequest request
    ) {
        access.manageableCourse(courseId);
        lockCourse(courseId);

        List<Long> actualIds = sections
                .findByCourse_IdOrderByPositionAscIdAsc(
                        courseId
                )
                .stream()
                .map(CourseSectionEntity::getId)
                .toList();

        validateExactPermutation(
                request.ids(), actualIds
        );

        // PostgreSQL defers position uniqueness checks
        // until the transaction commits.
        jdbc.execute(
                "SET CONSTRAINTS uq_course_sections_position DEFERRED"
        );

        for (int position = 0;
             position < request.ids().size();
             position++) {

            jdbc.update("""
                    UPDATE course_sections
                    SET position = ?
                    WHERE id = ? AND course_id = ?
                    """,
                    position,
                    request.ids().get(position),
                    courseId
            );
        }
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional
    public void reorderLessons(
            long courseId,
            long sectionId,
            ReorderRequest request
    ) {
        access.manageableCourse(courseId);
        lockCourse(courseId);

        var section = sections
                .findByIdAndCourse_Id(sectionId, courseId)
                .orElseThrow(CurriculumNotFoundException::new);

        List<Long> actualIds = lessons
                .findBySection_IdOrderByPositionAscIdAsc(
                        section.getId()
                )
                .stream()
                .map(CourseLessonEntity::getId)
                .toList();

        validateExactPermutation(
                request.ids(), actualIds
        );

        jdbc.execute(
                "SET CONSTRAINTS uq_course_lessons_position DEFERRED"
        );

        for (int position = 0;
             position < request.ids().size();
             position++) {

            jdbc.update("""
                    UPDATE course_lessons
                    SET position = ?
                    WHERE id = ? AND section_id = ?
                    """,
                    position,
                    request.ids().get(position),
                    sectionId
            );
        }
    }

    private static void validateExactPermutation(
            List<Long> requested,
            List<Long> actual
    ) {
        if (requested == null
                || requested.size() != actual.size()) {
            throw new IllegalArgumentException(
                    "Reorder must include every item exactly once."
            );
        }

        Set<Long> unique = new HashSet<>(requested);

        if (unique.size() != requested.size()
                || !unique.equals(new HashSet<>(actual))) {
            throw new IllegalArgumentException(
                    "Invalid reorder IDs."
            );
        }
    }
}
</CodeBlock>

### A Hibernate consistency detail

The reorder methods use JDBC updates rather than changing JPA-managed entity positions.

That is acceptable here because the service does not return those managed entities afterward. However, JDBC updates bypass the persistence context. If you later read the same entities in the same transaction, clear or refresh the persistence context first.

### Why require an exact permutation?

Suppose the current lesson IDs are:

```text
[10, 11, 12]
```

Valid reorder:

```text
[12, 10, 11]
```

Invalid reorder:

```text
[12, 12, 11]
```

Also invalid:

```text
[12, 10]
```

The server must not silently drop lessons or move lessons from another section.

### Add the missing exception

**Create:** `src/main/java/com/learnhub/service/CurriculumNotFoundException.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

public final class CurriculumNotFoundException
        extends RuntimeException {

    public CurriculumNotFoundException() {
        super("Curriculum item not found.");
    }
}
</CodeBlock>

Add a `404` handler to the existing `GlobalExceptionHandler`:

```java
@ExceptionHandler(CurriculumNotFoundException.class)
public ResponseEntity<ProblemDetail> curriculumNotFound(
        CurriculumNotFoundException exception
) {
    ProblemDetail problem =
            ProblemDetail.forStatusAndDetail(
                    HttpStatus.NOT_FOUND,
                    exception.getMessage()
            );

    return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(problem);
}
```

### Published-course editing policy

This implementation allows owners to modify curriculum on published courses. That can change students' progress denominators.

For a stricter publishing workflow, limit structural changes to drafts or introduce versioned curriculum releases. Lesson 19 will revisit publishing and lifecycle policy.

---

## 10. Public curriculum outline and protected lesson content

We'll separate these operations in the service layer.

**Create:** `src/main/java/com/learnhub/service/CurriculumReadService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.persistence.*;
import com.learnhub.web.dto.*;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class CurriculumReadService {

    private final CurriculumAccessService access;
    private final JpaCourseSectionRepository sections;
    private final JpaCourseLessonRepository lessons;

    public CurriculumReadService(
            CurriculumAccessService access,
            JpaCourseSectionRepository sections,
            JpaCourseLessonRepository lessons
    ) {
        this.access = access;
        this.sections = sections;
        this.lessons = lessons;
    }

    @Transactional(readOnly = true)
    public List<SectionOutlineResponse> publicOutline(
            long courseId
    ) {
        access.publishedCourse(courseId);
        return outline(courseId);
    }

    @PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
    @Transactional(readOnly = true)
    public List<SectionOutlineResponse> managedOutline(
            long courseId
    ) {
        access.manageableCourse(courseId);
        return outline(courseId);
    }

    private List<SectionOutlineResponse> outline(
            long courseId
    ) {
        return sections
                .findByCourse_IdOrderByPositionAscIdAsc(
                        courseId
                )
                .stream()
                .map(section -> {
                    var items = lessons
                            .findBySection_IdOrderByPositionAscIdAsc(
                                    section.getId()
                            )
                            .stream()
                            .map(lesson ->
                                    new LessonOutlineResponse(
                                            lesson.getId(),
                                            lesson.getTitle(),
                                            lesson.getPosition(),
                                            lesson.getDurationMinutes()
                                    )
                            )
                            .toList();

                    return new SectionOutlineResponse(
                            section.getId(),
                            section.getTitle(),
                            section.getPosition(),
                            items
                    );
                })
                .toList();
    }

    @PreAuthorize("isAuthenticated()")
    @Transactional(readOnly = true)
    public LessonContentResponse lessonContent(
            long courseId,
            long lessonId
    ) {
        // Authorize the course before retrieving its body.
        access.requireContentAccess(courseId);

        var lesson = lessons
                .findInCourse(courseId, lessonId)
                .orElseThrow(CurriculumNotFoundException::new);

        return new LessonContentResponse(
                lesson.getId(),
                courseId,
                lesson.getSection().getId(),
                lesson.getTitle(),
                lesson.getBody(),
                lesson.getDurationMinutes()
        );
    }
}
</CodeBlock>

This is a clear first implementation. It issues one lesson query per section, creating an N+1-style query pattern for large curricula.

For production, load all lessons for the course in one ordered query, group them by section ID, and construct the outline from that map. That refinement is a good exercise using the Lesson 10 query techniques.

---

## 11. Student lesson progress service

### Completion formula

For an actively enrolled student:

\[
\text{Progress \%} =
\begin{cases}
0 & \text{if total lessons}=0\\
\left\lfloor \frac{100 \times \text{completed lessons}}{\text{total lessons}}\right\rfloor & \text{otherwise}
\end{cases}
\]

A course is considered completed only when it has at least one lesson and every current lesson has a completion record.

### Example

<box border radius="lg" padding={3} gap={2}>
  <row align="center" justify="between">
    **4 of 6 lessons completed**
    <title size="lg" tabularNums>66%</title>
  </row>
  <box background="surface-tertiary" radius="full" height="12px" clip>
    <box background="rgba(109,40,217,0.85)" width="66%" height="100%" radius="full"/>
  </box>
  <text size="xs" color="secondary">The displayed percentage is truncated to an integer. Completion is not awarded until all six lessons are complete.</text>
</box>

**Create:** `src/main/java/com/learnhub/service/LessonProgressService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.persistence.*;
import com.learnhub.web.dto.CourseProgressResponse;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class LessonProgressService {

    private final CurriculumAccessService access;
    private final JpaCourseLessonRepository lessons;
    private final JpaLessonProgressRepository progress;

    public LessonProgressService(
            CurriculumAccessService access,
            JpaCourseLessonRepository lessons,
            JpaLessonProgressRepository progress
    ) {
        this.access = access;
        this.lessons = lessons;
        this.progress = progress;
    }

    @PreAuthorize("hasRole('STUDENT')")
    @Transactional
    public CourseProgressResponse complete(
            long courseId,
            long lessonId
    ) {
        var enrollment = access.activeEnrollment(courseId);

        var lesson = lessons
                .findInCourse(courseId, lessonId)
                .orElseThrow(CurriculumNotFoundException::new);

        progress.completeIfAbsent(
                enrollment.getId(),
                lesson.getId()
        );

        return buildProgress(
                courseId,
                enrollment.getId()
        );
    }

    @PreAuthorize("hasRole('STUDENT')")
    @Transactional(readOnly = true)
    public CourseProgressResponse myProgress(
            long courseId
    ) {
        var enrollment = access.activeEnrollment(courseId);

        return buildProgress(
                courseId,
                enrollment.getId()
        );
    }

    private CourseProgressResponse buildProgress(
            long courseId,
            long enrollmentId
    ) {
        long total = lessons.countBySection_Course_Id(
                courseId
        );

        List<Long> completedIds =
                progress.completedLessonIds(enrollmentId);

        long completed = completedIds.size();

        // Defensive guard: legacy progress rows could
        // reference lessons later removed from a course.
        int percentage = total == 0
                ? 0
                : (int) Math.min(
                        100,
                        completed * 100 / total
                );

        return new CourseProgressResponse(
                courseId,
                enrollmentId,
                completed,
                total,
                percentage,
                total > 0 && completed == total,
                completedIds
        );
    }
}
</CodeBlock>

### A critical integrity refinement

The code assumes curriculum lessons are not moved or deleted after progress is recorded. In this lesson, we have no move or delete endpoints, so that assumption holds.

If later lessons introduce moving lessons between courses, count only completion rows whose lessons still belong to the enrollment's course. Otherwise, the progress denominator and numerator could diverge.

### Why use `ON CONFLICT DO NOTHING`?

Two tabs may both mark the same lesson complete. PostgreSQL ensures only one completion row exists.

The second request still returns the current progress state successfully. This is a natural idempotent behavior for a completion operation.

---

## 12. REST controllers

### 12.1 Public curriculum

**Create:** `src/main/java/com/learnhub/web/PublicCurriculumController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.CurriculumReadService;
import com.learnhub.web.dto.SectionOutlineResponse;

import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/courses/{courseId}/curriculum")
public class PublicCurriculumController {

    private final CurriculumReadService service;

    public PublicCurriculumController(
            CurriculumReadService service
    ) {
        this.service = service;
    }

    @GetMapping
    public List<SectionOutlineResponse> outline(
            @PathVariable long courseId
    ) {
        return service.publicOutline(courseId);
    }
}
</CodeBlock>

### 12.2 Instructor curriculum management

**Create:** `src/main/java/com/learnhub/web/InstructorCurriculumController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.CurriculumManagementService;
import com.learnhub.service.CurriculumReadService;
import com.learnhub.web.dto.*;

import jakarta.validation.Valid;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/v1/instructor/courses/{courseId}/curriculum")
public class InstructorCurriculumController {

    private final CurriculumManagementService management;
    private final CurriculumReadService reading;

    public InstructorCurriculumController(
            CurriculumManagementService management,
            CurriculumReadService reading
    ) {
        this.management = management;
        this.reading = reading;
    }

    @GetMapping
    public List<SectionOutlineResponse> outline(
            @PathVariable long courseId
    ) {
        return reading.managedOutline(courseId);
    }

    @PostMapping("/sections")
    public ResponseEntity<Void> addSection(
            @PathVariable long courseId,
            @Valid @RequestBody SectionWriteRequest request
    ) {
        long id = management.addSection(courseId, request);

        return ResponseEntity.created(
                URI.create(
                        "/api/v1/instructor/courses/" +
                        courseId + "/curriculum/sections/" + id
                )
        ).build();
    }

    @PostMapping("/sections/{sectionId}/lessons")
    public ResponseEntity<Void> addLesson(
            @PathVariable long courseId,
            @PathVariable long sectionId,
            @Valid @RequestBody LessonWriteRequest request
    ) {
        long id = management.addLesson(
                courseId, sectionId, request
        );

        return ResponseEntity.created(
                URI.create(
                        "/api/v1/instructor/courses/" +
                        courseId + "/curriculum/lessons/" + id
                )
        ).build();
    }

    @PutMapping("/lessons/{lessonId}")
    public ResponseEntity<Void> updateLesson(
            @PathVariable long courseId,
            @PathVariable long lessonId,
            @Valid @RequestBody LessonWriteRequest request
    ) {
        management.updateLesson(courseId, lessonId, request);
        return ResponseEntity.noContent().build();
    }

    @PutMapping("/sections/order")
    public ResponseEntity<Void> reorderSections(
            @PathVariable long courseId,
            @Valid @RequestBody ReorderRequest request
    ) {
        management.reorderSections(courseId, request);
        return ResponseEntity.noContent().build();
    }

    @PutMapping("/sections/{sectionId}/lessons/order")
    public ResponseEntity<Void> reorderLessons(
            @PathVariable long courseId,
            @PathVariable long sectionId,
            @Valid @RequestBody ReorderRequest request
    ) {
        management.reorderLessons(
                courseId, sectionId, request
        );
        return ResponseEntity.noContent().build();
    }
}
</CodeBlock>

### 12.3 Protected lesson content

**Create:** `src/main/java/com/learnhub/web/LearningContentController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.CurriculumReadService;
import com.learnhub.web.dto.LessonContentResponse;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/learning/courses/{courseId}")
public class LearningContentController {

    private final CurriculumReadService service;

    public LearningContentController(
            CurriculumReadService service
    ) {
        this.service = service;
    }

    @GetMapping("/lessons/{lessonId}")
    public LessonContentResponse lesson(
            @PathVariable long courseId,
            @PathVariable long lessonId
    ) {
        return service.lessonContent(
                courseId, lessonId
        );
    }
}
</CodeBlock>

### 12.4 Student progress

**Create:** `src/main/java/com/learnhub/web/LessonProgressController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.LessonProgressService;
import com.learnhub.web.dto.CourseProgressResponse;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/me/courses/{courseId}/progress")
public class LessonProgressController {

    private final LessonProgressService service;

    public LessonProgressController(
            LessonProgressService service
    ) {
        this.service = service;
    }

    @GetMapping
    public CourseProgressResponse progress(
            @PathVariable long courseId
    ) {
        return service.myProgress(courseId);
    }

    @PutMapping("/lessons/{lessonId}")
    public CourseProgressResponse complete(
            @PathVariable long courseId,
            @PathVariable long lessonId
    ) {
        return service.complete(
                courseId, lessonId
        );
    }
}
</CodeBlock>

### Endpoint summary

| Method | Path | Access |
|---|---|---|
| GET | `/api/v1/courses/{courseId}/curriculum` | Public, published |
| GET | `/api/v1/instructor/courses/{courseId}/curriculum` | Owner/Admin |
| POST | `/api/v1/instructor/courses/{courseId}/curriculum/sections` | Owner/Admin |
| POST | `/api/v1/instructor/courses/{courseId}/curriculum/sections/{sectionId}/lessons` | Owner/Admin |
| PUT | `/api/v1/instructor/courses/{courseId}/curriculum/lessons/{lessonId}` | Owner/Admin |
| PUT | `/api/v1/instructor/courses/{courseId}/curriculum/sections/order` | Owner/Admin |
| PUT | `/api/v1/instructor/courses/{courseId}/curriculum/sections/{sectionId}/lessons/order` | Owner/Admin |
| GET | `/api/v1/learning/courses/{courseId}/lessons/{lessonId}` | Enrolled/Owner/Admin |
| GET | `/api/v1/me/courses/{courseId}/progress` | Enrolled student |
| PUT | `/api/v1/me/courses/{courseId}/progress/lessons/{lessonId}` | Enrolled student |

---

## 13. Spring Security configuration

**Modify:** `src/main/java/com/learnhub/config/SecurityConfig.java`

Add the following authorization rules before broader matchers such as `/api/v1/courses/**`:

<CodeBlock language="java" editable>
// Instructor curriculum endpoints: authenticated
// instructor or admin, with ownership checked in service.
.requestMatchers(
        "/api/v1/instructor/courses/*/curriculum",
        "/api/v1/instructor/courses/*/curriculum/**"
).hasAnyRole("INSTRUCTOR", "ADMIN")

// Protected lesson bodies.
.requestMatchers(
        HttpMethod.GET,
        "/api/v1/learning/courses/*/lessons/*"
).authenticated()

// Student progress endpoints.
.requestMatchers(
        "/api/v1/me/courses/*/progress",
        "/api/v1/me/courses/*/progress/**"
).hasRole("STUDENT")

// Public curriculum outline.
// Only the outline controller uses this route.
.requestMatchers(
        HttpMethod.GET,
        "/api/v1/courses/*/curriculum"
).permitAll()
</CodeBlock>

Ensure `org.springframework.http.HttpMethod` is imported.

### Security boundaries

<box gap={2}>
  <row align="start" gap={2}>
    <icon name="check-circle" color="success"/>
    <box flex="1">
      **Public outline:** Lesson titles, positions, and durations are visible, but lesson bodies are absent.
    </box>
  </row>
  <row align="start" gap={2}>
    <icon name="check-circle" color="success"/>
    <box flex="1">
      **Protected content:** The service verifies an active enrollment or authorized owner/admin before returning a body.
    </box>
  </row>
  <row align="start" gap={2}>
    <icon name="check-circle" color="success"/>
    <box flex="1">
      **Progress:** The enrollment is resolved from the session's principal ID, not from a request parameter.
    </box>
  </row>
  <row align="start" gap={2}>
    <icon name="check-circle" color="success"/>
    <box flex="1">
      **CSRF:** All POST and PUT curriculum/progress operations retain CSRF protection.
    </box>
  </row>
</box>

### Protect against response caching

Protected lesson bodies should not be stored in a shared intermediary cache.

Spring Security normally adds security-related cache headers. Verify the actual response includes `Cache-Control: no-store`, or explicitly set it on the lesson-content response if your configuration overrides those defaults.

Also avoid logging lesson bodies in HTTP request/response logs.

---

## 14. Angular typed API contracts

**Create:** `learnhub/src/app/core/api/curriculum-api.models.ts`

<CodeBlock language="typescript" editable>
export interface LessonOutlineDto {
  id: number;
  title: string;
  position: number;
  durationMinutes: number;
}

export interface SectionOutlineDto {
  id: number;
  title: string;
  position: number;
  lessons: LessonOutlineDto[];
}

export interface LessonContentDto {
  id: number;
  courseId: number;
  sectionId: number;
  title: string;
  body: string;
  durationMinutes: number;
}

export interface CourseProgressDto {
  courseId: number;
  enrollmentId: number;
  completedLessons: number;
  totalLessons: number;
  percentage: number;
  completed: boolean;
  completedLessonIds: number[];
}

export interface SectionWriteDto {
  title: string;
}

export interface LessonWriteDto {
  title: string;
  body: string;
  durationMinutes: number;
}

export interface ReorderDto {
  ids: number[];
}
</CodeBlock>

### Create CurriculumApiService

**Create:** `learnhub/src/app/core/api/curriculum-api.service.ts`

<CodeBlock language="typescript" editable>
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

import {
  SectionOutlineDto,
  LessonContentDto,
  CourseProgressDto,
  SectionWriteDto,
  LessonWriteDto,
  ReorderDto,
} from './curriculum-api.models';

@Injectable({ providedIn: 'root' })
export class CurriculumApiService {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/v1';

  publicOutline(
    courseId: number
  ): Observable<SectionOutlineDto[]> {
    return this.http.get<SectionOutlineDto[]>(
      `${this.base}/courses/${courseId}/curriculum`
    );
  }

  managedOutline(
    courseId: number
  ): Observable<SectionOutlineDto[]> {
    return this.http.get<SectionOutlineDto[]>(
      `${this.base}/instructor/courses/${courseId}/curriculum`
    );
  }

  addSection(
    courseId: number,
    body: SectionWriteDto
  ): Observable<void> {
    return this.http.post<void>(
      `${this.base}/instructor/courses/${courseId}/curriculum/sections`,
      body
    );
  }

  addLesson(
    courseId: number,
    sectionId: number,
    body: LessonWriteDto
  ): Observable<void> {
    return this.http.post<void>(
      `${this.base}/instructor/courses/${courseId}/curriculum/sections/${sectionId}/lessons`,
      body
    );
  }

  updateLesson(
    courseId: number,
    lessonId: number,
    body: LessonWriteDto
  ): Observable<void> {
    return this.http.put<void>(
      `${this.base}/instructor/courses/${courseId}/curriculum/lessons/${lessonId}`,
      body
    );
  }

  reorderSections(
    courseId: number,
    ids: number[]
  ): Observable<void> {
    const body: ReorderDto = { ids };

    return this.http.put<void>(
      `${this.base}/instructor/courses/${courseId}/curriculum/sections/order`,
      body
    );
  }

  reorderLessons(
    courseId: number,
    sectionId: number,
    ids: number[]
  ): Observable<void> {
    return this.http.put<void>(
      `${this.base}/instructor/courses/${courseId}/curriculum/sections/${sectionId}/lessons/order`,
      { ids } satisfies ReorderDto
    );
  }

  lesson(
    courseId: number,
    lessonId: number
  ): Observable<LessonContentDto> {
    return this.http.get<LessonContentDto>(
      `${this.base}/learning/courses/${courseId}/lessons/${lessonId}`
    );
  }

  progress(
    courseId: number
  ): Observable<CourseProgressDto> {
    return this.http.get<CourseProgressDto>(
      `${this.base}/me/courses/${courseId}/progress`
    );
  }

  complete(
    courseId: number,
    lessonId: number
  ): Observable<CourseProgressDto> {
    return this.http.put<CourseProgressDto>(
      `${this.base}/me/courses/${courseId}/progress/lessons/${lessonId}`,
      {}
    );
  }
}
</CodeBlock>

The `POST` methods return `201 Created` with no response body. `HttpClient` can handle that with `Observable<void>`.

For future instructor editors, use the `Location` response header to navigate to the newly created item if necessary.

---

## 15. Angular standalone course player
We'll create a minimal player that loads the public outline, retrieves protected lesson content, and tracks progress.

### Route design

```text
/my-learning/courses/:courseId
```

The route requires an authenticated student, but the backend still checks enrollment before returning lesson bodies or progress.

**Create:** `learnhub/src/app/features/learning/course-player.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  inject,
  signal,
} from '@angular/core';

import { ActivatedRoute } from '@angular/router';

import {
  catchError,
  EMPTY,
  finalize,
  forkJoin,
  of,
  switchMap,
} from 'rxjs';

import { takeUntilDestroyed }
  from '@angular/core/rxjs-interop';

import { CurriculumApiService }
  from '../../core/api/curriculum-api.service';

import {
  CourseProgressDto,
  LessonContentDto,
  SectionOutlineDto,
} from '../../core/api/curriculum-api.models';

@Component({
  selector: 'app-course-player',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main>
      <h1>Course player</h1>

      @if (loading()) {
        <p role="status">Loading course...</p>
      } @else if (error()) {
        <p role="alert">{{ error() }}</p>
      } @else {
        @if (progress(); as currentProgress) {
          <section aria-label="Course progress">
            <p>
              {{ currentProgress.completedLessons }}
              of
              {{ currentProgress.totalLessons }}
              lessons completed
            </p>

            <progress
              [value]="currentProgress.percentage"
              max="100"
            >
              {{ currentProgress.percentage }}%
            </progress>
          </section>
        }

        <nav aria-label="Course lessons">
          @for (section of sections(); track section.id) {
            <h2>{{ section.title }}</h2>

            <ul>
              @for (
                item of section.lessons;
                track item.id
              ) {
                <li>
                  <button
                    type="button"
                    (click)="selectLesson(item.id)"
                  >
                    @if (isCompleted(item.id)) {
                      ✓
                    }
                    {{ item.title }}
                  </button>
                </li>
              }
            </ul>
          }
        </nav>

        @if (lessonLoading()) {
          <p role="status">Loading lesson...</p>
        }

        @if (selectedLesson(); as lesson) {
          <article>
            <h2>{{ lesson.title }}</h2>

            <!-- Render as plain text for safety. -->
            <div class="lesson-body">
              {{ lesson.body }}
            </div>

            <button
              type="button"
              (click)="completeLesson()"
              [disabled]="
                completing() ||
                isCompleted(lesson.id)
              "
            >
              {{
                isCompleted(lesson.id)
                  ? 'Completed'
                  : 'Mark complete'
              }}
            </button>
          </article>
        }
      }
    </main>
  `,
  styles: [`
    .lesson-body {
      white-space: pre-wrap;
      line-height: 1.6;
    }
  `],
})
export class CoursePlayerComponent {
  private readonly api = inject(CurriculumApiService);
  private readonly route = inject(ActivatedRoute);
  private readonly destroyRef = inject(DestroyRef);

  readonly sections = signal<SectionOutlineDto[]>([]);
  readonly progress = signal<CourseProgressDto | null>(null);
  readonly selectedLesson =
    signal<LessonContentDto | null>(null);

  readonly loading = signal(true);
  readonly lessonLoading = signal(false);
  readonly completing = signal(false);
  readonly error = signal<string | null>(null);

  private readonly courseId = Number(
    this.route.snapshot.paramMap.get('courseId')
  );

  constructor() {
    if (!Number.isSafeInteger(this.courseId)
        || this.courseId <= 0) {
      this.error.set('Invalid course ID.');
      this.loading.set(false);
      return;
    }

    forkJoin({
      sections: this.api.publicOutline(this.courseId),
      progress: this.api.progress(this.courseId),
    }).pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe({
      next: result => {
        this.sections.set(result.sections);
        this.progress.set(result.progress);
        this.loading.set(false);
      },
      error: error => {
        this.error.set(
          error.message || 'Unable to load course.'
        );
        this.loading.set(false);
      },
    });
  }

  isCompleted(lessonId: number): boolean {
    return this.progress()
      ?.completedLessonIds.includes(lessonId) ?? false;
  }

  selectLesson(lessonId: number): void {
    this.lessonLoading.set(true);
    this.selectedLesson.set(null);
    this.error.set(null);

    this.api.lesson(
      this.courseId,
      lessonId
    ).pipe(
      finalize(() => this.lessonLoading.set(false)),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe({
      next: lesson => {
        this.selectedLesson.set(lesson);
      },
      error: error => {
        this.error.set(
          error.message || 'Unable to load lesson.'
        );
      },
    });
  }

  completeLesson(): void {
    const lesson = this.selectedLesson();

    if (!lesson || this.completing()) {
      return;
    }

    this.completing.set(true);
    this.error.set(null);

    this.api.complete(
      this.courseId,
      lesson.id
    ).pipe(
      finalize(() => this.completing.set(false)),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe({
      next: progress => {
        this.progress.set(progress);
      },
      error: error => {
        this.error.set(
          error.message || 'Unable to update progress.'
        );
      },
    });
  }
}
</CodeBlock>

### Why render lesson bodies as text?

Angular interpolation escapes HTML, preventing a lesson body from executing arbitrary scripts.

Do not use `[innerHTML]` with untrusted instructor-authored content unless you have a deliberate sanitization and content policy.

### Request-race refinement

The current player starts an independent subscription for every lesson selection. Rapid clicks can allow an older lesson response to arrive after a newer one.

Before production, drive lesson selection through a single `Subject<number>` and `switchMap()` so only the latest selection updates the view.

---

## 16. Instructor curriculum editor

We'll add a basic authoring page. It supports creating sections and lessons, while the API service already supports updating and reordering them.

**Create:** `learnhub/src/app/features/instructor/curriculum-editor.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  inject,
  signal,
} from '@angular/core';

import {
  FormBuilder,
  ReactiveFormsModule,
  Validators,
} from '@angular/forms';

import { ActivatedRoute } from '@angular/router';

import { CurriculumApiService }
  from '../../core/api/curriculum-api.service';

import { SectionOutlineDto }
  from '../../core/api/curriculum-api.models';

@Component({
  selector: 'app-curriculum-editor',
  standalone: true,
  imports: [ReactiveFormsModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main>
      <h1>Curriculum editor</h1>

      @if (error()) {
        <p role="alert">{{ error() }}</p>
      }

      <section>
        <h2>Add section</h2>

        <form
          [formGroup]="sectionForm"
          (ngSubmit)="addSection()"
        >
          <label for="section-title">Section title</label>
          <input
            id="section-title"
            formControlName="title"
          />

          <button
            type="submit"
            [disabled]="sectionForm.invalid || busy()"
          >
            Add section
          </button>
        </form>
      </section>

      @for (section of sections(); track section.id) {
        <section>
          <h2>{{ section.title }}</h2>

          <ol>
            @for (
              lesson of section.lessons;
              track lesson.id
            ) {
              <li>
                {{ lesson.title }}
                ({{ lesson.durationMinutes }} min)
              </li>
            }
          </ol>

          <button
            type="button"
            (click)="selectSection(section.id)"
          >
            Add lesson here
          </button>
        </section>
      }

      @if (selectedSectionId() !== null) {
        <section>
          <h2>Add lesson</h2>

          <form
            [formGroup]="lessonForm"
            (ngSubmit)="addLesson()"
          >
            <label for="lesson-title">Title</label>
            <input
              id="lesson-title"
              formControlName="title"
            />

            <label for="lesson-body">Lesson content</label>
            <textarea
              id="lesson-body"
              formControlName="body"
            ></textarea>

            <label for="lesson-duration">
              Duration in minutes
            </label>
            <input
              id="lesson-duration"
              type="number"
              min="0"
              formControlName="durationMinutes"
            />

            <button
              type="submit"
              [disabled]="lessonForm.invalid || busy()"
            >
              Save lesson
            </button>
          </form>
        </section>
      }
    </main>
  `,
})
export class CurriculumEditorComponent {
  private readonly fb = inject(FormBuilder);
  private readonly api = inject(CurriculumApiService);
  private readonly route = inject(ActivatedRoute);

  private readonly courseId = Number(
    this.route.snapshot.paramMap.get('courseId')
  );

  readonly sections = signal<SectionOutlineDto[]>([]);
  readonly selectedSectionId =
    signal<number | null>(null);
  readonly busy = signal(false);
  readonly error = signal<string | null>(null);

  readonly sectionForm = this.fb.nonNullable.group({
    title: ['', [
      Validators.required,
      Validators.maxLength(180),
    ]],
  });

  readonly lessonForm = this.fb.nonNullable.group({
    title: ['', [
      Validators.required,
      Validators.maxLength(180),
    ]],
    body: ['', Validators.required],
    durationMinutes: [0, Validators.min(0)],
  });

  constructor() {
    if (!Number.isSafeInteger(this.courseId)
        || this.courseId <= 0) {
      this.error.set('Invalid course ID.');
      return;
    }

    this.load();
  }

  load(): void {
    this.api.managedOutline(this.courseId)
      .subscribe({
        next: sections => this.sections.set(sections),
        error: error => this.error.set(
          error.message || 'Unable to load curriculum.'
        ),
      });
  }

  selectSection(sectionId: number): void {
    this.selectedSectionId.set(sectionId);
  }

  addSection(): void {
    if (this.sectionForm.invalid || this.busy()) {
      return;
    }

    this.busy.set(true);
    this.error.set(null);

    this.api.addSection(
      this.courseId,
      this.sectionForm.getRawValue()
    ).subscribe({
      next: () => {
        this.sectionForm.reset();
        this.busy.set(false);
        this.load();
      },
      error: error => {
        this.busy.set(false);
        this.error.set(
          error.message || 'Unable to add section.'
        );
      },
    });
  }

  addLesson(): void {
    const sectionId = this.selectedSectionId();

    if (
      sectionId === null ||
      this.lessonForm.invalid ||
      this.busy()
    ) {
      return;
    }

    this.busy.set(true);
    this.error.set(null);

    this.api.addLesson(
      this.courseId,
      sectionId,
      this.lessonForm.getRawValue()
    ).subscribe({
      next: () => {
        this.lessonForm.reset();
        this.busy.set(false);
        this.load();
      },
      error: error => {
        this.busy.set(false);
        this.error.set(
          error.message || 'Unable to add lesson.'
        );
      },
    });
  }
}
</CodeBlock>

### What remains for a polished editor?

The service layer already supports lesson updates and ordering. Extend this page with an edit form and reorder controls, then test both with ownership checks.

Avoid introducing drag-and-drop until you have verified the reorder API accepts exact permutations and handles concurrent requests.

---

## 17. Angular routes

**Modify:** `learnhub/src/app/app.routes.ts`

Merge these into the existing route tree:

<CodeBlock language="typescript" editable>
{
  path: 'my-learning/courses/:courseId',
  canActivate: [studentGuard],
  loadComponent: () =>
    import(
      './features/learning/course-player.component'
    ).then(m => m.CoursePlayerComponent),
},
{
  path: 'instructor/courses/:courseId/curriculum',
  canActivate: [instructorGuard],
  loadComponent: () =>
    import(
      './features/instructor/curriculum-editor.component'
    ).then(m => m.CurriculumEditorComponent),
},
</CodeBlock>

Import `studentGuard` and `instructorGuard` from the existing Lesson 16–17 auth package.

**Route ordering:** If your application already has a parent `instructor` route with children, add the curriculum editor as a child instead of creating a competing top-level route.

### Connect My Learning

In the Lesson 17 `MyLearningPageComponent`, change the enrollment link to:

```html
<a
  [routerLink]="[
    '/my-learning/courses',
    enrollment.course.id
  ]"
>
  Continue learning
</a>
```

The backend will verify active enrollment before exposing lesson content.

---

## 18. PostgreSQL and MockMvc integration tests

We need to prove five things:

1. Owners can create curriculum.
2. Other instructors cannot change it.
3. Public outlines never expose lesson bodies.
4. Only enrolled students can read lesson bodies.
5. Completion is idempotent and isolated by student.

### Test fixture strategy

Use isolated test records rather than hardcoded seed IDs. Each test should create its own course owner, student, course, section, lesson, and enrollment as required.

We'll use JDBC fixtures and real PostgreSQL constraints.

**Create:** `src/test/java/com/learnhub/web/CurriculumIntegrationTest.java`

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
class CurriculumIntegrationTest
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

    private long course(long instructorId) {
        return jdbc.queryForObject("""
            INSERT INTO courses (
                title, description, instructor,
                category, price, is_free,
                level, rating, status,
                instructor_user_id
            )
            VALUES (
                'Curriculum Test Course',
                'Integration test',
                'Test Instructor',
                'Development',
                0.00,
                true,
                'BEGINNER',
                0.0,
                'PUBLISHED',
                ?
            )
            RETURNING id
            """,
            Long.class,
            instructorId
        );
    }

    private long section(long courseId) {
        return jdbc.queryForObject("""
            INSERT INTO course_sections (
                course_id, title, position
            )
            VALUES (?, 'Getting Started', 0)
            RETURNING id
            """,
            Long.class,
            courseId
        );
    }

    private long lesson(long sectionId) {
        return jdbc.queryForObject("""
            INSERT INTO course_lessons (
                section_id, title, body,
                position, duration_minutes
            )
            VALUES (
                ?, 'Lesson One',
                'PRIVATE_LESSON_BODY',
                0, 10
            )
            RETURNING id
            """,
            Long.class,
            sectionId
        );
    }

    private void enroll(long studentId, long courseId) {
        jdbc.update("""
            INSERT INTO enrollments (
                student_user_id, course_id, status
            )
            VALUES (?, ?, 'ACTIVE')
            """,
            studentId,
            courseId
        );
    }

    @Test
    void publicOutlineNeverContainsLessonBody()
            throws Exception {
        long ownerId = user(
                "outline-owner@example.test",
                "INSTRUCTOR"
        );
        long courseId = course(ownerId);
        long sectionId = section(courseId);
        lesson(sectionId);

        mvc.perform(get(
                    "/api/v1/courses/{id}/curriculum",
                    courseId
                ))
            .andExpect(status().isOk())
            .andExpect(
                jsonPath("$[0].lessons[0].title")
                    .value("Lesson One")
            )
            .andExpect(
                jsonPath("$[0].lessons[0].body")
                    .doesNotExist()
            );
    }

    @Test
    void unenrolledStudentCannotReadLesson()
            throws Exception {
        long ownerId = user(
                "private-owner@example.test",
                "INSTRUCTOR"
        );
        long studentId = user(
                "unenrolled@example.test",
                "STUDENT"
        );
        long courseId = course(ownerId);
        long lessonId = lesson(section(courseId));

        mvc.perform(get(
                    "/api/v1/learning/courses/{c}/lessons/{l}",
                    courseId,
                    lessonId
                )
                .with(user(principal(
                        studentId,
                        "unenrolled@example.test",
                        SecurityRole.STUDENT
                ))))
            .andExpect(status().isForbidden());
    }

    @Test
    void enrolledStudentCanReadLesson()
            throws Exception {
        long ownerId = user(
                "read-owner@example.test",
                "INSTRUCTOR"
        );
        long studentId = user(
                "reader@example.test",
                "STUDENT"
        );
        long courseId = course(ownerId);
        long lessonId = lesson(section(courseId));

        enroll(studentId, courseId);

        mvc.perform(get(
                    "/api/v1/learning/courses/{c}/lessons/{l}",
                    courseId,
                    lessonId
                )
                .with(user(principal(
                        studentId,
                        "reader@example.test",
                        SecurityRole.STUDENT
                ))))
            .andExpect(status().isOk())
            .andExpect(
                jsonPath("$.body")
                    .value("PRIVATE_LESSON_BODY")
            );
    }

    @Test
    void completionIsIdempotent()
            throws Exception {
        long ownerId = user(
                "progress-owner@example.test",
                "INSTRUCTOR"
        );
        long studentId = user(
                "progress-student@example.test",
                "STUDENT"
        );
        long courseId = course(ownerId);
        long lessonId = lesson(section(courseId));

        enroll(studentId, courseId);

        var actor = principal(
                studentId,
                "progress-student@example.test",
                SecurityRole.STUDENT
        );

        for (int attempt = 0; attempt < 2; attempt++) {
            mvc.perform(put(
                        "/api/v1/me/courses/{c}/progress/lessons/{l}",
                        courseId,
                        lessonId
                    )
                    .with(user(actor))
                    .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(
                    jsonPath("$.percentage").value(100)
                );
        }

        Integer count = jdbc.queryForObject("""
            SELECT count(*)
            FROM lesson_progress p
            JOIN enrollments e
              ON e.id = p.enrollment_id
            WHERE e.student_user_id = ?
              AND p.lesson_id = ?
            """,
            Integer.class,
            studentId,
            lessonId
        );

        assertEquals(1, count);
    }

    @Test
    void otherInstructorCannotAddSection()
            throws Exception {
        long ownerId = user(
                "true-owner@example.test",
                "INSTRUCTOR"
        );
        long otherId = user(
                "other-owner@example.test",
                "INSTRUCTOR"
        );
        long courseId = course(ownerId);

        mvc.perform(post(
                    "/api/v1/instructor/courses/{c}/curriculum/sections",
                    courseId
                )
                .with(user(principal(
                        otherId,
                        "other-owner@example.test",
                        SecurityRole.INSTRUCTOR
                )))
                .with(csrf())
                .contentType("application/json")
                .content("{\"title\":\"Unauthorized\"}"))
            .andExpect(status().isForbidden());

        Integer count = jdbc.queryForObject("""
            SELECT count(*)
            FROM course_sections
            WHERE course_id = ?
            """,
            Integer.class,
            courseId
        );

        assertEquals(0, count);
    }
}
</CodeBlock>

### Add three more test cases

The five tests above cover the central security behavior. Add these before treating the feature as complete:

- Owner creates a section and lesson; verify positions `0`, `1`, `2` and correct course association.
- Two students are enrolled in the same course; one completes a lesson and the other remains at `0%`.
- Reordering sections and lessons with a valid permutation succeeds; duplicate or foreign IDs return `400` and leave ordering unchanged.

### A test configuration warning

The tests use synchronous MockMvc requests inside test-managed transactions, as in Lessons 15–17. The fixtures and mutations should roll back after each test.

If you switch to a real HTTP client that executes server requests on separate threads, those server-side transactions will not participate in the test's rollback transaction. Use explicit cleanup or disposable databases for that testing style.

---

## 19. Angular HttpClient contract tests

These tests verify that the frontend calls the correct endpoints and never sends a student ID when recording progress.

**Create:** `learnhub/src/app/core/api/curriculum-api.service.spec.ts`

<CodeBlock language="typescript" editable>
import { TestBed } from '@angular/core/testing';

import { provideHttpClient }
  from '@angular/common/http';

import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';

import { CurriculumApiService }
  from './curriculum-api.service';

describe('CurriculumApiService', () => {
  let api: CurriculumApiService;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });

    api = TestBed.inject(CurriculumApiService);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());

  it('loads public outlines without requesting bodies', () => {
    api.publicOutline(42).subscribe(result => {
      expect(result[0].lessons[0].title)
        .toBe('Introduction');
    });

    const request = http.expectOne(
      '/api/v1/courses/42/curriculum'
    );

    expect(request.request.method).toBe('GET');

    request.flush([{
      id: 1,
      title: 'Foundations',
      position: 0,
      lessons: [{
        id: 10,
        title: 'Introduction',
        position: 0,
        durationMinutes: 5,
      }],
    }]);
  });

  it('loads lesson content from protected route', () => {
    api.lesson(42, 10).subscribe(result => {
      expect(result.body).toBe('Protected content');
    });

    const request = http.expectOne(
      '/api/v1/learning/courses/42/lessons/10'
    );

    expect(request.request.method).toBe('GET');

    request.flush({
      id: 10,
      courseId: 42,
      sectionId: 1,
      title: 'Introduction',
      body: 'Protected content',
      durationMinutes: 5,
    });
  });

  it('marks a lesson complete using session identity', () => {
    api.complete(42, 10).subscribe(result => {
      expect(result.percentage).toBe(50);
    });

    const request = http.expectOne(
      '/api/v1/me/courses/42/progress/lessons/10'
    );

    expect(request.request.method).toBe('PUT');    expect(request.request.body).toEqual({});
    expect(request.request.params.has('studentId'))
      .toBeFalse();

    request.flush({
      courseId: 42,
      enrollmentId: 7,
      completedLessons: 1,
      totalLessons: 2,
      percentage: 50,
      completed: false,
      completedLessonIds: [10],
    });
  });

  it('sends exact section reorder IDs', () => {
    api.reorderSections(42, [3, 1, 2]).subscribe();

    const request = http.expectOne(
      '/api/v1/instructor/courses/42/curriculum/sections/order'
    );

    expect(request.request.method).toBe('PUT');
    expect(request.request.body).toEqual({
      ids: [3, 1, 2],
    });

    request.flush(null, {
      status: 204,
      statusText: 'No Content',
    });
  });
});
</CodeBlock>

## 20. Run the application

### Terminal A — Backend

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

### Terminal C — Database inspection

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "\d course_sections"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "\d course_lessons"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "\d lesson_progress"
```

Verify Flyway applied the new migration:

```powershell
& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT version, description, success FROM flyway_schema_history ORDER BY installed_rank;"
```

### Manual API example

Once an authorized instructor has created a section and lesson, an anonymous visitor can inspect the public outline:

```powershell
$base = "http://localhost:8080"

Invoke-RestMethod `
  "$base/api/v1/courses/1/curriculum"
```

This returns an outline only if course `1` is still published. It should never include a lesson `body`.

For protected operations, use the authenticated session and CSRF flow from Lesson 14. Do not bypass Spring Security for testing convenience.

---

## 21. Browser acceptance scenarios

<box gap={2}>
  <row align="start" gap={2}>
    <icon name="user-round" color="secondary"/>
    <box flex="1">
      **Anonymous visitor:** Can see the curriculum outline for a published course but receives `401` when requesting lesson content.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={2}>
    <icon name="graduation-cap" color="secondary"/>
    <box flex="1">
      **Enrolled student:** Can open lesson content, mark it complete, refresh the browser, and retain progress.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={2}>
    <icon name="lock" color="secondary"/>
    <box flex="1">
      **Unenrolled student:** Can see the public outline but cannot retrieve protected lesson bodies or mark progress.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={2}>
    <icon name="book-open" color="secondary"/>
    <box flex="1">
      **Course owner:** Can create and reorder sections and lessons; another instructor receives `403` and cannot mutate them.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={2}>
    <icon name="shield-check" color="secondary"/>
    <box flex="1">
      **CSRF:** A session-authenticated PUT or POST without the required token receives `403`.
    </box>
  </row>
</box>

---

## 22. Production engineering review

Before shipping, examine the following tradeoffs.

| Area | Current design | Production refinement |
|---|---|---|
| Ordering | Integer positions, deferrable uniqueness | Versioned reorder requests, stale-edit detection |
| Curriculum reads | One query per section | Bulk lesson query or projection |
| Progress | Completion rows | Completion events and audit history |
| Lesson bodies | PostgreSQL text | Content versioning and rich-text sanitization |
| Published edits | Allowed | Curriculum release/version policy |
| Course completion | Derived from lesson count | Snapshot or durable completion award |
| Video | Not implemented | Private object storage and authorized streaming |
| Student access | Active enrollment | Explicit archived-course access policy |

### Three subtle issues

**1. Curriculum changes can change progress.** If an instructor adds a lesson, a student who previously had 100% may fall below 100%. Decide whether completion is a live calculation or a historical achievement.

**2. Concurrent curriculum editing needs conflict detection.** The row lock serializes operations, but it doesn't tell the second editor that the first editor changed the curriculum. An optimistic version or expected-order token is useful.

**3. Content authorization must precede serialization.** Never fetch an entire course entity graph and return it directly, hoping that frontend hiding will protect lesson bodies.

### A note on missing and forbidden resources

This lesson uses `403` for authenticated users without the required enrollment or ownership, and `404` for missing curriculum items or unpublished public courses.

For sensitive draft content, a consistent concealment policy may return `404` to non-owners. The important property is that unauthorized requests never expose the protected body.

---

## 23. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="arrow-up-down" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise A — Full reorder UI**

      Add Move Up and Move Down controls to the Angular curriculum editor. Submit the complete ID permutation, refresh the outline, and test invalid or concurrent reorder requests.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="database" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise B — Remove N+1 queries**

      Replace per-section lesson loading with one bulk query. Use Hibernate SQL logging to compare query counts for a course with 20 sections and 200 lessons.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="shield-check" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise C — Cross-course progress attack**

      Enroll a student in Course A, then attempt to mark a lesson from Course B complete using Course A's ID. Verify the request fails and no progress row is inserted.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="refresh-cw" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise D — Course player request cancellation**

      Refactor lesson selection to a `Subject<number>` and `switchMap()`. Verify rapid clicks never display stale content.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="award" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise E — Completion policy**

      Design a durable course completion record that remains valid when new lessons are added. Explain how it differs from the live progress percentage.
    </box>
  </row>
</box>

## 24. Knowledge quiz

{@body const questions=[{q:"Which record authorizes a student's lesson access?",options:["A wishlist item","An ACTIVE enrollment for the course","A public curriculum outline"],correct:1,why:"An active enrollment is the server-side entitlement for student lesson access."},{q:"Why must the public curriculum DTO omit the lesson body?",options:["To keep protected content out of public responses","Because JPA cannot map text columns","Because Angular cannot display text"],correct:0,why:"A public outline is intentionally distinct from an authenticated lesson-content response."},{q:"What does the unique constraint on lesson_progress enforce?",options:["One progress row per enrollment and lesson","Only one student can complete a lesson","Lessons must be completed in order"],correct:0,why:"The enrollment/lesson pair uniquely identifies an individual student's completion."},{q:"Why validate reorder requests as exact permutations?",options:["To ensure every item remains present exactly once","To make Angular faster","To disable database constraints"],correct:0,why:"A reorder should change positions, not duplicate, omit, or import unrelated items."},{q:"Why might a student's percentage fall after an instructor adds a lesson?",options:["The session cookie expired","The number of required lessons increased","PostgreSQL deleted completion rows"],correct:1,why:"Live progress uses the current curriculum as its denominator."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Lesson 18 — Curriculum and progress**
    <text color="secondary" size="xs">5 questions</text>
  </row>
  {#each questions as question,i}
    <box key={i} gap={2}>
      **{i+1}. {question.q}**
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setGraded(false);}}>
        {#each question.options as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if graded}
        <row align="start" gap={2}>
          <icon name={answers[i]===question.correct?"check-circle":"info"} color={answers[i]===question.correct?"success":"secondary"}/>
          <box flex="1" gap={1}>
            <text size="sm" weight="medium">{answers[i]===question.correct?"Correct":"Correct: "+question.options[question.correct]}</text>
            <text size="sm" color="secondary">{question.why}</text>
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

## 25. Completion checklist

{@body const groups=[{name:"PostgreSQL and JPA",items:["Verified Lesson 17 baseline","Selected next unused Flyway version","Created curriculum and progress tables","Verified uniqueness and foreign keys","Created three JPA entities","Created curriculum repositories","Created progress repository"]},{name:"Backend and security",items:["Added explicit curriculum DTOs","Implemented course ownership authorization","Implemented section and lesson creation","Implemented lesson editing","Implemented section and lesson reorder","Implemented public outline endpoint","Implemented protected lesson content endpoint","Implemented student completion and progress","Configured SecurityFilterChain","Verified CSRF and status codes"]},{name:"Angular",items:["Created typed curriculum API models","Created CurriculumApiService","Created course player","Created instructor curriculum editor","Added guarded routes","Connected My Learning page","Verified loading and error states"]},{name:"Testing and delivery",items:["Added PostgreSQL curriculum tests","Tested enrolled versus unenrolled access","Tested cross-instructor denial","Tested idempotent completion","Added Angular API contract tests","Ran Maven clean verify","Ran Angular tests and build","Completed two exercises","Completed quiz","Reviewed and committed changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((n,g)=>n+g.items.length,0)}
{@body const completed=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 18 progress**
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
        <checkbox key={`${i}-${j}`} checked={!!done[`${i}-${j}`]} onChange={v=>setDone(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if completed===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 18 checklist completed.**
    </row>
  {/if}
</box>

## 26. Final verification and Git checkpoints

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=CurriculumIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Curriculum integration tests failed."
}

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend verification failed."
}

git diff --check
git status
git diff --stat
```

Angular:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd test --watch=false

if ($LASTEXITCODE -ne 0) {
    throw "Angular unit tests failed."
}

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}

git diff --check
git status
git diff --stat
```

After both pass, review and commit:

```powershell
cd $HOME\angular-learning\learnhub-backend

git add src\main src\test
git diff --cached --stat
git commit -m "Add secure course curriculum and student progress"
```

```powershell
cd $HOME\angular-learning\learnhub

git add src\app
git diff --cached --stat
git commit -m "Build LearnHub course player and curriculum editor"
```

### Integration notes before calling the lesson complete

The examples assume the Lesson 14–17 principal and entity signatures are still present. Align the actual `CourseNotFoundException` import, `CurrentActor`, `SecurityRole`, and JPA getters before compiling.

The course player and instructor editor are working reference implementations for the specified endpoints, but still need the identified request-race, query-efficiency, and production UX refinements. The tests have not been executed against your local projects.

## 27. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" radius="lg" padding={3}>
    <icon name="layout-dashboard" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 19 — Instructor Dashboard and Course Publishing Workflow**

    We'll build instructor analytics, course readiness validation, publish/unpublish rules, draft and published lifecycle transitions, curriculum requirements, ownership-safe reporting, optimistic locking, PostgreSQL migrations, and Angular dashboard integration.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 19: instructor dashboard analytics, PostgreSQL aggregation queries, course readiness checks, publishing and archiving workflows, draft versus published lifecycle rules, curriculum validation, optimistic locking, ownership-safe reporting, Spring Security authorization, transactional services, Angular standalone instructor dashboard, MockMvc and Testcontainers tests, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 18 with experienced-Java pacing.")}>Start Lesson 19 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 18 deliverable:** An ordered, PostgreSQL-backed curriculum with secure lesson access, instructor ownership enforcement, idempotent student completion, server-calculated progress, and an Angular course player.

The central principle is: **a published curriculum outline can be public, but lesson content and individual learning progress must be authorized on the server.**