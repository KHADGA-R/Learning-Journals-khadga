# LearnHub Java & Spring Boot Backend — Lesson 10

<text color="secondary" size="sm">Spring Data JPA Relationships, Hibernate Fetching, N+1 Queries, JPQL, DTO Projections, and Database Pagination</text>

**Stack:** Java 21 · Spring Boot 3.5.x · PostgreSQL · Hibernate · Flyway · JUnit 5 · Testcontainers  
**Estimated time:** 5–7 hours, including refactoring and tests.

In Lesson 9, we created the relational foundation for LearnHub: `app_users`, `courses`, and `enrollments`. We also practiced SQL joins, constraints, indexes, transactions, and query plans.

Today we'll map those relationships into Java and learn to query them efficiently.

The central principle is:

**JPA entities model persistence. Services enforce business rules. DTOs define what Angular receives.**

We'll preserve the existing course API while adding instructor relationships and enrollment-aware database queries.

## 1. Learning objectives

By the end of this lesson, you'll understand:

- `@ManyToOne` and `@OneToMany` relationships.
- Owning versus inverse sides of a JPA relationship.
- Lazy loading and the persistence context.
- The N+1 query problem and how to detect it.
- `@EntityGraph` and JPQL fetch joins.
- DTO projections for efficient read APIs.
- Database-level pagination with `Pageable`.
- Transaction boundaries and `LazyInitializationException`.
- Why JPA entities should not be serialized directly to Angular.
- Integration testing against real PostgreSQL.

## 2. Where Lesson 10 fits

<box border radius="xl" padding={3} gap={2}>
  <text weight="medium">LearnHub persistence architecture</text>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    **Angular LearnHub**
    <text color="secondary" size="xs">Course catalog and instructor information</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring MVC Controllers**
    <text color="secondary" size="xs">Stable JSON response DTOs</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Application Services**
    <text color="secondary" size="xs">Transactions, ownership rules, orchestration</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring Data JPA**
    <text color="secondary" size="xs">JPQL, entity graphs, projections, pagination</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    <icon name="database" size="lg"/>
    **PostgreSQL**
    <text color="secondary" size="xs">Foreign keys, joins, indexes, constraints</text>
  </box>
</box>

### Relationship map

| Parent | Child | Foreign key |
|---|---|---|
| `app_users` | `courses` | `courses.instructor_user_id` |
| `app_users` | `enrollments` | `enrollments.student_user_id` |
| `courses` | `enrollments` | `enrollments.course_id` |

The foreign keys were introduced in Lesson 9. We will now represent them using JPA associations.

**Scope:** The new user and enrollment entities are persistence models only. We are not implementing authentication, registration, enrollment creation APIs, or authorization yet.

---

## 3. Verify the Lesson 9 baseline

Open Windows PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info

git status
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 9 verification failed."
}
```

Create a checkpoint:

```powershell
git add .
git commit -m "Checkpoint before JPA relationship mapping"
```

If Git reports nothing to commit, continue.

### Check your Flyway version

In PostgreSQL:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub
```

Inside `psql`:

```sql
SELECT version, description, success
FROM flyway_schema_history
ORDER BY installed_rank;
```

You should see the successful migration that created `app_users`, `enrollments`, and `courses.instructor_user_id`.

If your Lesson 9 migration used a version other than V3, retain your actual migration numbering. **Never edit a migration that has already been applied.**

## 4. Project structure

We'll introduce a dedicated relationship model under the existing persistence package.

```text
learnhub-backend/
└── src/
    ├── main/
    │   ├── java/com/learnhub/
    │   │   ├── persistence/
    │   │   │   ├── CourseEntity.java           # modify
    │   │   │   ├── JpaCourseRepository.java     # modify
    │   │   │   ├── AppUserEntity.java           # new
    │   │   │   ├── EnrollmentEntity.java        # new
    │   │   │   ├── JpaAppUserRepository.java    # new
    │   │   │   ├── JpaEnrollmentRepository.java # new
    │   │   │   ├── CourseCatalogQueryRepository.java # new
    │   │   │   └── CourseCatalogRow.java        # new
    │   │   ├── service/
    │   │   │   └── CourseReadService.java       # new
    │   │   └── web/
    │   │       ├── dto/
    │   │       │   └── CourseSummaryResponse.java # new
    │   │       └── CourseQueryController.java   # new
    │   └── resources/db/migration/
    │       └── V4__link_sample_instructors.sql  # new
    └── test/java/com/learnhub/persistence/
        └── JpaRelationshipTest.java             # new
```

We'll expose a new **read-only query endpoint** at:

```text
GET /api/v1/course-summaries
```

That keeps Lesson 6–9's existing `/api/v1/courses` contract unchanged while we explore efficient JPA queries.

Later, we can migrate the main course catalog to the new query implementation.

## 5. Model users with `@Entity`

Create:

`src/main/java/com/learnhub/persistence/AppUserEntity.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import jakarta.persistence.*;

import java.time.OffsetDateTime;
import java.util.ArrayList;
import java.util.List;

/**
 * Persistence representation of app_users.
 *
 * This is NOT an authentication implementation.
 */
@Entity
@Table(name = "app_users")
public class AppUserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 254)
    private String email;

    @Column(name = "display_name", nullable = false, length = 120)
    private String displayName;

    @Column(nullable = false, length = 20)
    private String role;

    @Column(name = "created_at", nullable = false,
            insertable = false, updatable = false)
    private OffsetDateTime createdAt;

    /**
     * Inverse side: CourseEntity owns the foreign key.
     *
     * Avoid EAGER fetching and cascading deletion.
     */
    @OneToMany(mappedBy = "instructor", fetch = FetchType.LAZY)
    private List<CourseEntity> taughtCourses = new ArrayList<>();

    @OneToMany(mappedBy = "student", fetch = FetchType.LAZY)
    private List<EnrollmentEntity> enrollments = new ArrayList<>();

    protected AppUserEntity() {
    }

    public Long getId() {
        return id;
    }

    public String getEmail() {
        return email;
    }

    public String getDisplayName() {
        return displayName;
    }

    public String getRole() {
        return role;
    }

    public OffsetDateTime getCreatedAt() {
        return createdAt;
    }

    public List<CourseEntity> getTaughtCourses() {
        return taughtCourses;
    }

    public List<EnrollmentEntity> getEnrollments() {
        return enrollments;
    }
}
</CodeBlock>

### Why not return `AppUserEntity` directly from an API?

The entity contains an email address, role, and relationships. A controller that serializes it directly risks exposing data and triggering lazy-loading behavior.

Instead, an API response should explicitly select fields such as:

```json
{
  "instructorId": 1,
  "instructorName": "Alex Morgan"
}
```

This is an important distinction between **database shape** and **public API shape**.

## 6. Model the enrollment relationship

Create:

`src/main/java/com/learnhub/persistence/EnrollmentEntity.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import jakarta.persistence.*;

import java.time.OffsetDateTime;

/**
 * Enrollment is a first-class entity because it
 * has its own identity, status, and timestamp.
 */
@Entity
@Table(name = "enrollments")
public class EnrollmentEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(
            name = "student_user_id",
            nullable = false
    )
    private AppUserEntity student;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(
            name = "course_id",
            nullable = false
    )
    private CourseEntity course;

    @Column(name = "enrolled_at", nullable = false,
            insertable = false, updatable = false)
    private OffsetDateTime enrolledAt;

    @Column(nullable = false, length = 20)
    private String status = "ACTIVE";

    protected EnrollmentEntity() {
    }

    public EnrollmentEntity(
            AppUserEntity student,
            CourseEntity course
    ) {
        this.student = student;
        this.course = course;
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

    public OffsetDateTime getEnrolledAt() {
        return enrolledAt;
    }

    public String getStatus() {
        return status;
    }
}
</CodeBlock>

### Why `@ManyToOne(fetch = LAZY)`?

JPA's default for `@ManyToOne` is eager fetching. Explicit `LAZY` is generally a better starting point when we want control over SQL queries.

But lazy fetching isn't a universal performance fix. It moves the decision about loading related data to the point of access.

If you access a lazy relationship for every row in a list, you may create an N+1 query problem.

## 7. Add relationships to CourseEntity

Open:

`src/main/java/com/learnhub/persistence/CourseEntity.java`

Keep every existing field, factory, method, and mapping from Lesson 7.

Add the following imports:

```java
import java.util.ArrayList;
import java.util.List;
```

Add these fields:

<CodeBlock language="java" editable>
/**
 * Owning side of the instructor relationship.
 *
 * The FK lives in courses.instructor_user_id.
 * Nullable for older courses without linked accounts.
 */
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "instructor_user_id")
private AppUserEntity instructorAccount;

/**
 * Inverse side of the enrollment relationship.
 *
 * Do not use CascadeType.REMOVE here:
 * enrollment history must not disappear accidentally.
 */
@OneToMany(mappedBy = "course", fetch = FetchType.LAZY)
private List<EnrollmentEntity> enrollments = new ArrayList<>();
</CodeBlock>

Add getters and a linking method:

<CodeBlock language="java" editable>
public AppUserEntity getInstructorAccount() {
    return instructorAccount;
}

public List<EnrollmentEntity> getEnrollments() {
    return enrollments;
}

/**
 * Links a course to an existing instructor account.
 * The caller must enforce instructor permissions.
 */
public void assignInstructor(AppUserEntity instructorAccount) {
    this.instructorAccount = instructorAccount;
}
</CodeBlock>

### Owning versus inverse side

`CourseEntity.instructorAccount` is the owning side because it maps the actual foreign-key column.

`AppUserEntity.taughtCourses` is the inverse side:

```java
@OneToMany(mappedBy = "instructorAccount")
```

**Important correction:** In `AppUserEntity.java` from Section 5, use `mappedBy = "instructorAccount"` rather than `mappedBy = "instructor"` for `taughtCourses`. `mappedBy` refers to the **Java field name**, not the database column or the older instructor display-name field.

The final annotation is:

```java
@OneToMany(mappedBy = "instructorAccount", fetch = FetchType.LAZY)
private List<CourseEntity> taughtCourses = new ArrayList<>();
```

For enrollment relationships, the owning sides are `EnrollmentEntity.student` and `EnrollmentEntity.course`.

### Avoid dangerous cascades

Do not add `CascadeType.ALL` to these relationships simply to make persistence easier.

Deleting an instructor should not automatically delete all courses, and deleting a course should not automatically delete enrollment records.

Our Lesson 9 foreign keys already restrict those operations.

## 8. Create Spring Data repositories

### 8.1 User repository

File:

`src/main/java/com/learnhub/persistence/JpaAppUserRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface JpaAppUserRepository
        extends JpaRepository<AppUserEntity, Long> {

    Optional<AppUserEntity> findByEmailIgnoreCase(String email);
}
</CodeBlock>

### 8.2 Enrollment repository

File:

`src/main/java/com/learnhub/persistence/JpaEnrollmentRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface JpaEnrollmentRepository
        extends JpaRepository<EnrollmentEntity, Long> {

    boolean existsByStudentIdAndCourseId(
            Long studentId,
            Long courseId
    );

    List<EnrollmentEntity> findByStudentId(Long studentId);

    long countByCourseId(Long courseId);
}
</CodeBlock>

Spring Data interprets `StudentId` and `CourseId` as nested paths through the relationships to their `id` properties.

For example:

```java
existsByStudentIdAndCourseId(3L, 1L)
```

means approximately:

```sql
SELECT EXISTS (
    SELECT 1
    FROM enrollments
    WHERE student_user_id = 3
      AND course_id = 1
);
```

The exact generated SQL can differ.

Remember: `existsBy...` is a convenience check, not a replacement for the database's unique constraint under concurrency.

## 9. Flyway V4 — Link the sample instructors

Lesson 9 introduced `app_users`, but the seeded courses still have null `instructor_user_id` values.

We'll create two synthetic instructor records and associate the existing sample courses with them.

**File:** `src/main/resources/db/migration/V4__link_sample_instructors.sql`

<CodeBlock language="sql" editable>
-- Lesson 10: Link existing demo courses to demo instructors.
--
-- These are synthetic development accounts.
-- They cannot log in and have no credentials.

INSERT INTO app_users (email, display_name, role)
VALUES
    ('alex.morgan@learnhub.example', 'Alex Morgan', 'INSTRUCTOR'),
    ('jordan.lee@learnhub.example', 'Jordan Lee', 'INSTRUCTOR')
ON CONFLICT DO NOTHING;

-- Preserve the original instructor text field.
-- Link only the known seed records, not every course
-- that happens to share a display name.

UPDATE courses
SET instructor_user_id = (
    SELECT id
    FROM app_users
    WHERE lower(email) = 'alex.morgan@learnhub.example'
)
WHERE id IN (1, 5);

UPDATE courses
SET instructor_user_id = (
    SELECT id
    FROM app_users
    WHERE lower(email) = 'jordan.lee@learnhub.example'
)
WHERE id = 2;
</CodeBlock>

We intentionally leave courses 3 and 4 unlinked because the seed records do not have corresponding instructor accounts yet.

This demonstrates a realistic migration concern: **existing data may be incomplete during a phased transition**.

If V4 is already used in your project, choose the next unused migration version.

### Apply and verify

Set your Lesson 7 database environment variables, then start the backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

In another PowerShell window:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub
```

Run:

```sql
SELECT
    c.id,
    c.title,
    c.instructor AS legacy_instructor,
    u.display_name AS linked_instructor
FROM courses c
LEFT JOIN app_users u
    ON u.id = c.instructor_user_id
ORDER BY c.id;
```

Expected: courses 1, 2, and 5 have linked instructor records. Courses 3 and 4 still use their legacy display-name fields.

**Note:** This migration deliberately does not create enrollments. We'll use transactional test fixtures for relationship tests, rather than inserting fake student activity into the development database.

---

## 10. Understand the N+1 query problem

Imagine loading 100 courses:

```java
List<CourseEntity> courses = repository.findAll();

for (CourseEntity course : courses) {
    AppUserEntity instructor = course.getInstructorAccount();

    if (instructor != null) {
        System.out.println(instructor.getDisplayName());
    }
}
```

The first query retrieves the courses.

Accessing each lazy instructor relationship may trigger additional SQL queries.

<box border radius="lg" padding={3} gap={2}>
  <text weight="medium">Illustrative N+1 query pattern</text>
  <grid columns={2} gap={2}>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={3} gap={1}>
        <text color="secondary" size="xs">Without controlled fetching</text>
        <title size="2xl" tabularNums>101</title>
        <text size="xs" color="secondary">Potential queries for 100 courses</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={3} gap={1}>
        <text color="secondary" size="xs">With an appropriate fetch plan</text>
        <title size="2xl" tabularNums>1</title>
        <text size="xs" color="secondary">Possible query count for a simple list</text>
      </box>
    </grid-item>
  </grid>
  <text size="xs" color="secondary">Illustration, not a benchmark. Actual query counts depend on shared instructors, persistence-context state, caching, and the fetch strategy.</text>
</box>

If 100 courses reference only two instructors, Hibernate may load only those two distinct instructor records. So N+1 describes a **query pattern**, not a guaranteed exact count.

### Three common solutions

1. **Entity graph:** load specified relationships alongside the entity.
2. **Fetch join:** explicitly fetch relationships using JPQL.
3. **DTO projection:** retrieve only the columns the API needs.

For read-heavy catalog endpoints, DTO projections are often the simplest and most efficient choice.

## 11. Use `@EntityGraph`

Open:

`src/main/java/com/learnhub/persistence/JpaCourseRepository.java`

Keep the existing methods and add:

<CodeBlock language="java" editable>
/**
 * Fetch the optional instructor account with the course.
 *
 * The entity graph controls relationship fetching.
 */
@EntityGraph(attributePaths = "instructorAccount")
@Query("""
       SELECT c
       FROM CourseEntity c
       WHERE c.id = :id
       """)
Optional<CourseEntity> findWithInstructorById(
        @Param("id") Long id
);
</CodeBlock>

Add imports:

```java
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.Optional;
```

Now:

```java
CourseEntity course = repository
        .findWithInstructorById(1L)
        .orElseThrow();
```

retrieves the course with its instructor relationship initialized.

The query may use a join or another fetching strategy chosen by Hibernate.

### Entity graph versus fetch join

An entity graph describes **what relationships should be fetched**.

A JPQL fetch join describes **how the query joins and fetches the association**.

For example:

```java
@Query("""
       SELECT c
       FROM CourseEntity c
       LEFT JOIN FETCH c.instructorAccount
       WHERE c.id = :id
       """)
Optional<CourseEntity> findByIdFetchingInstructor(
        @Param("id") Long id
);
```

Both approaches are useful.

### Avoid collection fetch joins with ordinary pagination

This is risky:

```java
SELECT c
FROM CourseEntity c
LEFT JOIN FETCH c.enrollments
```

combined with `Pageable`.

A collection fetch join multiplies SQL rows for courses with multiple enrollments. Hibernate may need to paginate in memory or reject the query, depending on configuration.

For paginated lists, prefer projections, two-step fetching, or a carefully designed query strategy.

---

## 12. JPQL DTO projections

Let's create a query result containing exactly the data our Angular course cards need.

### 12.1 Create CourseCatalogRow

**File:** `src/main/java/com/learnhub/persistence/CourseCatalogRow.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.model.CourseLevel;

import java.math.BigDecimal;
import java.time.LocalDate;

/**
 * Read-model projection returned by JPQL.
 *
 * No Hibernate proxies or entity collections.
 */
public record CourseCatalogRow(
        Long id,
        String title,
        String description,
        String instructorName,
        Long instructorId,
        String category,
        BigDecimal price,
        BigDecimal rating,
        boolean free,
        CourseLevel level,
        String thumbnail,
        BigDecimal durationHours,
        LocalDate lastUpdated
) {
}
</CodeBlock>

### 12.2 Create CourseCatalogQueryRepository

**File:** `src/main/java/com/learnhub/persistence/CourseCatalogQueryRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.model.CourseStatus;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.Repository;
import org.springframework.data.repository.query.Param;

/**
 * Read-only repository with a DTO projection.
 *
 * Filters and pagination execute in PostgreSQL,
 * rather than after loading every course into Java.
 */
public interface CourseCatalogQueryRepository
        extends Repository<CourseEntity, Long> {

    @Query(
        value = """
            SELECT new com.learnhub.persistence.CourseCatalogRow(
                c.id,
                c.title,
                c.description,
                c.instructor,
                u.id,
                c.category,
                c.price,
                c.rating,
                c.free,
                c.level,
                c.thumbnail,
                c.durationHours,
                c.lastUpdated
            )
            FROM CourseEntity c
            LEFT JOIN c.instructorAccount u
            WHERE c.status = :status
              AND (
                  :search IS NULL
                  OR lower(c.title) LIKE lower(
                      concat('%', :search, '%')
                  )
                  OR lower(c.description) LIKE lower(
                      concat('%', :search, '%')
                  )
              )
              AND (
                  :category IS NULL
                  OR lower(c.category) = lower(:category)
              )
            ORDER BY c.id ASC
            """,
        countQuery = """
            SELECT COUNT(c.id)
            FROM CourseEntity c
            WHERE c.status = :status
              AND (
                  :search IS NULL
                  OR lower(c.title) LIKE lower(
                      concat('%', :search, '%')
                  )
                  OR lower(c.description) LIKE lower(
                      concat('%', :search, '%')
                  )
              )
              AND (
                  :category IS NULL
                  OR lower(c.category) = lower(:category)
              )
            """
    )
    Page<CourseCatalogRow> findCatalog(
            @Param("status") CourseStatus status,
            @Param("search") String search,
            @Param("category") String category,
            Pageable pageable
    );
}
</CodeBlock>

This query uses a JPQL constructor expression:

```java
new com.learnhub.persistence.CourseCatalogRow(...)
```

Hibernate constructs the record directly from selected columns.

The join does not require loading `AppUserEntity` objects into the returned result.

### A subtle projection detail

We deliberately select:

```java
c.instructor
```

for the public instructor display name, rather than:

```java
u.displayName
```

because some older courses do not yet have linked instructor accounts.

The optional `instructorId` gives us a path toward real instructor profiles without breaking the existing Angular display.

### Search behavior

This first database-backed query searches title and description. The earlier in-memory catalog also searched instructor and category text.

We'll keep this new endpoint separate until the database query matches the entire existing search contract.

Also, `LIKE` treats `%` and `_` as wildcard characters. A production literal-substring search should escape those characters or use a suitable PostgreSQL search strategy.

## 13. Database-level pagination

Spring Data JPA uses `Pageable` to express pagination.

```java
PageRequest.of(0, 10)
```

means:

- Page number: `0` (zero-based).
- Page size: `10`.
- First 10 matching rows.

The database performs the paging instead of the application retrieving every row.

### Why use `Page`?

A `Page<T>` contains:

- The current page's records.
- Total matching record count.
- Total number of pages.
- Page number and size.

This typically requires a data query and a count query.

For large datasets where the frontend only needs to know whether more results exist, `Slice<T>` may avoid the cost of a total-count query.

### Deterministic ordering matters

Our query includes:

```sql
ORDER BY c.id ASC
```

Without a stable order, successive pages may return overlapping or missing records even when the data hasn't changed.

Ordering by a nonunique field, such as rating, should include a unique tie-breaker:

```sql
ORDER BY rating DESC, id ASC
```

Offset pagination can still shift when records are inserted or deleted between requests. Keyset pagination is another option for large, frequently changing datasets.

## 14. Create an Angular-safe response DTO

**File:** `src/main/java/com/learnhub/web/dto/CourseSummaryResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.model.CourseLevel;
import com.learnhub.persistence.CourseCatalogRow;

import java.math.BigDecimal;
import java.time.LocalDate;

/**
 * Public read-only course summary.
 *
 * Contains no JPA entities, lazy collections,
 * email addresses, or internal persistence state.
 */
public record CourseSummaryResponse(
        long id,
        String title,
        String description,
        String instructor,
        Long instructorId,
        String category,
        BigDecimal price,
        BigDecimal rating,
        boolean isFree,
        CourseLevel level,
        String thumbnail,
        BigDecimal durationHours,
        LocalDate lastUpdated
) {

    public static CourseSummaryResponse from(
            CourseCatalogRow row
    ) {
        return new CourseSummaryResponse(
                row.id(),
                row.title(),
                row.description(),
                row.instructorName(),
                row.instructorId(),
                row.category(),
                row.price(),
                row.rating(),
                row.free(),
                row.level(),
                row.thumbnail(),
                row.durationHours(),
                row.lastUpdated()
        );
    }
}</CodeBlock>

### Page response DTO

Create:

`src/main/java/com/learnhub/web/dto/CourseSummaryPageResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import org.springframework.data.domain.Page;

import java.util.List;

/**
 * Stable pagination contract for Angular.
 */
public record CourseSummaryPageResponse(
        List<CourseSummaryResponse> items,
        long totalItems,
        int pageNumber,
        int pageSize,
        int totalPages
) {
    public CourseSummaryPageResponse {
        items = List.copyOf(items);
    }

    public static CourseSummaryPageResponse from(
            Page<CourseSummaryResponse> page
    ) {
        return new CourseSummaryPageResponse(
                page.getContent(),
                page.getTotalElements(),
                page.getNumber(),
                page.getSize(),
                page.getTotalPages()
        );
    }
}
</CodeBlock>

## 15. Create the read service

**File:** `src/main/java/com/learnhub/service/CourseReadService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseCatalogQueryRepository;
import com.learnhub.web.dto.CourseSummaryPageResponse;
import com.learnhub.web.dto.CourseSummaryResponse;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Objects;

/**
 * Read-only application service for the new
 * database-backed course catalog.
 */
@Service
public class CourseReadService {

    private final CourseCatalogQueryRepository repository;

    public CourseReadService(
            CourseCatalogQueryRepository repository
    ) {
        this.repository = Objects.requireNonNull(repository);
    }

    @Transactional(readOnly = true)
    public CourseSummaryPageResponse searchPublished(
            String search,
            String category,
            int page,
            int size
    ) {
        if (page < 0) {
            throw new IllegalArgumentException(
                    "Page must be zero or greater."
            );
        }

        if (size < 1 || size > 100) {
            throw new IllegalArgumentException(
                    "Page size must be between 1 and 100."
            );
        }

        String normalizedSearch = normalize(search);
        String normalizedCategory = normalize(category);

        Page<CourseSummaryResponse> result =
                repository.findCatalog(
                        CourseStatus.PUBLISHED,
                        normalizedSearch,
                        normalizedCategory,
                        PageRequest.of(page, size)
                ).map(CourseSummaryResponse::from);

        return CourseSummaryPageResponse.from(result);
    }

    private String normalize(String value) {
        if (value == null || value.isBlank()) {
            return null;
        }

        return value.trim();
    }
}
</CodeBlock>

### Why `@Transactional(readOnly = true)`?

The transaction provides a defined persistence context for the read operation.

The read-only hint can also help Hibernate avoid unnecessary dirty-checking work. It does **not** guarantee that all database writes are impossible.

Our service converts the projection into DTOs before returning, so no lazy entity relationships escape the transaction.

## 16. Create the query controller

**File:** `src/main/java/com/learnhub/web/CourseQueryController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.CourseReadService;
import com.learnhub.web.dto.CourseSummaryPageResponse;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;

import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

/**
 * Read-only API used to explore efficient JPA
 * projections and database pagination.
 */
@RestController
@RequestMapping("/api/v1/course-summaries")
@Validated
public class CourseQueryController {

    private final CourseReadService readService;

    public CourseQueryController(
            CourseReadService readService
    ) {
        this.readService = readService;
    }

    @GetMapping
    public CourseSummaryPageResponse list(
            @RequestParam(required = false) String search,
            @RequestParam(required = false) String category,
            @RequestParam(defaultValue = "0")
            @Min(0) int page,
            @RequestParam(defaultValue = "10")
            @Min(1) @Max(100) int size
    ) {
        return readService.searchPublished(
                search,
                category,
                page,
                size
        );
    }
}
</CodeBlock>

We validate pagination both at the controller boundary and in the service, because the service may be called by code other than HTTP.

For consistent Problem Details responses, add a handler for `jakarta.validation.ConstraintViolationException` to your Lesson 8 global exception handler, or rely on the service's `IllegalArgumentException` checks until that handler is in place. Spring MVC method-validation exception types can vary by validation path.

### Test with PowerShell

Start Spring Boot, then open a second terminal:

```powershell
$base = "http://localhost:8080/api/v1/course-summaries"

Invoke-RestMethod -Uri "$base?page=0&size=10" |
    ConvertTo-Json -Depth 5
```

Search by title:

```powershell
Invoke-RestMethod `
  -Uri "$base?search=Angular&page=0&size=5"
```

Filter by category:

```powershell
Invoke-RestMethod `
  -Uri "$base?category=Web%20Development&page=0&size=5"
```

Test pagination:

```powershell
$page0 = Invoke-RestMethod -Uri "$base?page=0&size=2"
$page1 = Invoke-RestMethod -Uri "$base?page=1&size=2"

$page0.items | Select-Object id, title, instructorId
$page1.items | Select-Object id, title, instructorId
```

Expected for the original four published seed courses:

- Page 0 contains courses 1 and 2.
- Page 1 contains courses 3 and 4.
- Both pages report `totalItems = 4`.

If you've created additional published courses in earlier lessons, the results will differ.

## 17. Integration testing with PostgreSQL

We need to verify three things:

1. JPA relationships map correctly to the real Flyway schema.
2. Fetching an instructor through an entity graph works.
3. The projection returns correct data and paginates in the database.

We'll use the shared PostgreSQL Testcontainers base from Lesson 8.

### 17.1 Create the test file

**File:** `src/test/java/com/learnhub/persistence/JpaRelationshipTest.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;
import com.learnhub.model.CourseStatus;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.domain.PageRequest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Real PostgreSQL integration tests.
 *
 * Flyway migrations run in the Testcontainers database.
 * Each test's writes are rolled back afterward.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@Transactional
class JpaRelationshipTest extends PostgresIntegrationTest {

    @Autowired
    private JpaCourseRepository courses;

    @Autowired
    private JpaAppUserRepository users;

    @Autowired
    private JpaEnrollmentRepository enrollments;

    @Autowired
    private CourseCatalogQueryRepository catalog;

    @Autowired
    private JdbcTemplate jdbc;

    @Test
    void seededCourseHasLinkedInstructor() {
        CourseEntity course = courses.findById(1L)
                .orElseThrow();

        assertNotNull(course.getInstructorAccount());

        assertEquals(
                "Alex Morgan",
                course.getInstructorAccount().getDisplayName()
        );
    }

    @Test
    void entityGraphLoadsInstructor() {
        CourseEntity course = courses
                .findWithInstructorById(1L)
                .orElseThrow();

        assertNotNull(course.getInstructorAccount());

        assertEquals(
                "Alex Morgan",
                course.getInstructorAccount().getDisplayName()
        );
    }

    @Test
    void optionalInstructorCanBeMissing() {
        CourseEntity course = courses.findById(3L)
                .orElseThrow();

        assertNull(course.getInstructorAccount());

        // Legacy display name is still available.
        assertNotNull(course.getInstructor());
    }

    @Test
    void enrollmentMapsBothRelationships() {
        Long studentId = jdbc.queryForObject(
                """
                INSERT INTO app_users
                    (email, display_name, role)
                VALUES (
                    'lesson10.student@example.test',
                    'Lesson 10 Student',
                    'STUDENT'
                )
                RETURNING id
                """,
                Long.class
        );

        assertNotNull(studentId);

        Long enrollmentId = jdbc.queryForObject(
                """
                INSERT INTO enrollments
                    (student_user_id, course_id)
                VALUES (?, ?)
                RETURNING id
                """,
                Long.class,
                studentId,
                1L
        );

        EnrollmentEntity enrollment = enrollments
                .findById(enrollmentId)
                .orElseThrow();

        assertEquals(
                studentId,
                enrollment.getStudent().getId()
        );

        assertEquals(
                1L,
                enrollment.getCourse().getId()
        );

        assertEquals(
                "ACTIVE",
                enrollment.getStatus()
        );
    }

    @Test
    void catalogProjectionReturnsInstructorId() {
        var result = catalog.findCatalog(
                CourseStatus.PUBLISHED,
                "Angular",
                null,
                PageRequest.of(0, 10)
        );

        assertFalse(result.isEmpty());

        CourseCatalogRow first = result.getContent().getFirst();

        assertEquals("Angular Complete Guide", first.title());
        assertNotNull(first.instructorId());
    }

    @Test
    void catalogProjectionPaginates() {
        var firstPage = catalog.findCatalog(
                CourseStatus.PUBLISHED,
                null,
                null,
                PageRequest.of(0, 2)
        );

        var secondPage = catalog.findCatalog(
                CourseStatus.PUBLISHED,
                null,
                null,
                PageRequest.of(1, 2)
        );

        assertEquals(2, firstPage.getNumberOfElements());
        assertEquals(2, secondPage.getNumberOfElements());

        assertEquals(4, firstPage.getTotalElements());

        assertNotEquals(
                firstPage.getContent().getFirst().id(),
                secondPage.getContent().getFirst().id()
        );
    }

    @Test
    void categoryFilterRunsInDatabase() {
        var result = catalog.findCatalog(
                CourseStatus.PUBLISHED,
                null,
                "Web Development",
                PageRequest.of(0, 10)
        );

        assertTrue(result.getContent().stream()
                .allMatch(row ->
                        row.category()
                                .equalsIgnoreCase("Web Development")
                ));
    }
}
</CodeBlock>

These seven tests assume the original seed catalog remains in place and the V4 migration links the known instructors.

They also assume the test database is not being modified by other concurrently executing test classes. If your Maven build enables parallel test execution, use isolated database fixtures or disable parallel execution for these shared-seed integration tests.

### 17.2 Run only the new tests

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=JpaRelationshipTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 10 JPA integration tests failed."
}
```

### 17.3 Run the complete suite

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 10 verification failed."
}
```

The new test file contributes **seven expected tests**. The full suite count depends on the earlier tests you've retained.

## 18. Diagnose N+1 queries with Hibernate statistics

For learning purposes, Hibernate can report query and entity-fetch statistics.

Add this to a **development or test profile**, not necessarily your production configuration.

**File:** `src/main/resources/application-dev.yml`

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.stat: DEBUG
```

Start the backend with the `dev` profile:

```powershell
mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

Inspect logs for repeated instructor or enrollment queries.

### Example: inefficient access

```java
List<CourseEntity> allCourses = jpa.findAll();

for (CourseEntity course : allCourses) {
    if (course.getInstructorAccount() != null) {
        course.getInstructorAccount().getDisplayName();
    }
}
```

Depending on persistence-context state, this may issue additional queries.

### Better for a course details screen

```java
CourseEntity course = jpa
        .findWithInstructorById(courseId)
        .orElseThrow();
```

### Better for a course catalog

```java
Page<CourseCatalogRow> page = catalog.findCatalog(
        CourseStatus.PUBLISHED,
        null,
        null,
        PageRequest.of(0, 20)
);
```

The DTO projection retrieves selected columns directly.

### Why not always use `@EntityGraph`?

Fetching unnecessary associations can be expensive too.

For a page that displays only title, price, and thumbnail, loading full user and enrollment entities wastes resources.

The right fetch strategy depends on the use case.

---

## 19. Understand `LazyInitializationException`

Consider:

```java
public CourseEntity loadCourse(long id) {
    return repository.findById(id).orElseThrow();
}
```

If this method returns a detached entity and the controller later accesses:

```java
course.getEnrollments().size();
```

Hibernate may throw `LazyInitializationException` because the collection was not initialized and the persistence context is closed.

### Do not fix this by enabling Open Session in View

We already configured:

```yaml
spring:
  jpa:
    open-in-view: false
```

Keep it disabled.

Instead:

- Fetch the required relationships inside the service transaction.
- Use entity graphs or explicit queries.
- Prefer DTO projections for API responses.
- Return complete DTOs from the service.

This makes database access predictable.

## 20. Pagination and performance review

Our new query performs pagination in PostgreSQL, but there are still important considerations.

### Count queries

A `Page<T>` may execute a count query in addition to the data query.

On a large marketplace, counting all matching courses can become expensive.

For infinite scrolling, a `Slice<T>` or keyset pagination may be more appropriate.

### Index support

Our Lesson 7 indexes include status and category combinations.

The new query filters by:

```sql
status
lower(category)
lower(title)
lower(description)
```

A normal index on `category` does not necessarily optimize `lower(category)`.

For large datasets, consider expression indexes or PostgreSQL `pg_trgm` indexes for substring search.

Don't add these indexes yet without measuring representative workloads.

### Query plans

Run:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    c.id,
    c.title,
    c.price,
    u.display_name
FROM courses c
LEFT JOIN app_users u
    ON u.id = c.instructor_user_id
WHERE c.status = 'PUBLISHED'
ORDER BY c.id
LIMIT 20;
```

Compare this plan with the SQL emitted by Hibernate.

Look for:

- Sequential scans versus index scans.
- Join strategy.
- Estimated versus actual row counts.
- Sort operations.
- Buffer hits and reads.

With a five-course seed catalog, the planner may reasonably prefer sequential scans.

## 21. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="network" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise A — Fetch students for a course**

      Add a repository method that retrieves enrollments for one course, including the student relationship. Compare a JPQL fetch join with an entity graph. Verify the generated SQL and ensure no student email addresses appear in the API response.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="gauge" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise B — Measure N+1**

      Create several courses with distinct instructors in an isolated test. Measure SQL statements for a naive lazy-loading loop and for an entity-graph query. Explain the difference using Hibernate statistics.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="search" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise C — Extend the catalog query**

      Add optional `free` and `level` filters. Preserve the existing category and search behavior. Write tests for combined filters and empty result pages.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="layers" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise D — Instructor course summaries**

      Add a read-only endpoint:

      `GET /api/v1/instructors/{id}/course-summaries`

      Use a DTO projection and database pagination. Return only published courses and avoid serializing `AppUserEntity`.
    </box>
  </row>
</box>

## 22. Knowledge quiz

{@body const questions=[{q:"Which entity owns the instructor foreign-key relationship?",opts:["AppUserEntity.taughtCourses","CourseEntity.instructorAccount","CourseSummaryResponse","CourseReadService"],a:1,why:"The owning side is the entity mapping the foreign-key column, courses.instructor_user_id."},{q:"What is the N+1 query problem?",opts:["One query returns too many columns","A query loads N+1 pages","Loading a list triggers additional queries for related entities","A database has more than one index"],a:2,why:"Lazy relationship access across a collection can cause many extra database queries."},{q:"Which approach is especially suitable for paginated course-card responses?",opts:["Return JPA entities directly","Fetch every enrollment collection","JPQL DTO projection","Enable Open Session in View"],a:2,why:"DTO projections select the fields the API needs without exposing managed entities or lazy collections."},{q:"Why are collection fetch joins problematic with pagination?",opts:["They always delete duplicate records","They can multiply result rows and force in-memory pagination","They disable transactions","They require JDBC rather than Hibernate"],a:1,why:"A one-to-many fetch join can produce multiple SQL rows per parent, complicating correct pagination."},{q:"What is the role of @Transactional(readOnly = true)?",opts:["It guarantees all writes are impossible","It defines a read transaction and provides optimization hints","It automatically creates indexes","It converts entities to DTOs"],a:1,why:"It provides transaction context and can optimize read operations, but is not a substitute for permissions or database constraints."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
{@body const score=questions.filter((q,i)=>answers[i]===q.a).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Lesson 10 knowledge check**
    <text color="secondary" size="xs">{Object.keys(answers).length}/5 answered</text>
  </row>
  {#each questions as q,i}
    <box key={i} gap={2}>
      <text weight="medium">{i+1}. {q.q}</text>
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setGraded(false)}}>
        {#each q.opts as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if graded}
        <row align="center" gap={2}>
          <icon name={answers[i]===q.a?"check-circle":"alert-circle"} color={answers[i]===q.a?"success":"danger"}/>
          <text size="sm" weight="medium">{answers[i]===q.a?"Correct":"Answer: "+q.opts[q.a]}</text>
        </row>
        <text color="secondary" size="sm">{q.why}</text>
      {/if}
    </box>
    {#if i<questions.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if graded}
    <box background="surface-secondary" radius="md" padding={3} gap={1}>
      <title size="xl" tabularNums>{score}/5</title>
      <text size="sm">{score===5?"All answers correct.":"Review the explanations and try again."}</text>
    </box>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==5} onClick={()=>setGraded(true)}>Check answers</button>
    <button color="secondary" variant="outline" onClick={()=>{setAnswers({});setGraded(false)}}>Reset</button>
  </row>
</box>

## 23. Completion checklist

{@body const groups=[{name:"JPA relationships",items:["Verified Lesson 9 baseline","Created AppUserEntity","Created EnrollmentEntity","Mapped CourseEntity.instructorAccount","Mapped inverse collections correctly","Created user and enrollment repositories"]},{name:"Flyway and fetching",items:["Applied the instructor-linking migration","Verified linked instructor records","Implemented @EntityGraph lookup","Explained N+1 queries","Inspected Hibernate SQL"]},{name:"Queries and API",items:["Created CourseCatalogRow projection","Implemented JPQL query and count query","Implemented database pagination","Created CourseReadService","Created Angular-safe response DTOs","Tested course-summaries endpoint"]},{name:"Tests and completion",items:["Added seven PostgreSQL integration tests","Ran Maven clean verify","Reviewed EXPLAIN ANALYZE output","Completed at least two exercises","Committed Lesson 10 changes"]}]}
{@body const [completed,setCompleted]=DIL.useState({})}
{@body const total=groups.reduce((s,g)=>s+g.items.length,0)}
{@body const count=Object.values(completed).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    <box gap={1}>
      **Lesson 10 progress**
      <text color="secondary" size="xs">Check each item after verifying it locally.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" width={`${100*count/total}%`} height="100%" radius="full"/>
  </box>
  {#each groups as group,i}
    <box key={i} gap={2}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!completed[`${i}-${j}`]} onChange={v=>setCompleted(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if count===total}
    <row gap={2} align="center">
      <icon name="check-circle" color="success"/>
      **Lesson 10 checklist completed.**
    </row>
  {/if}
</box>

## 24. Final verification and Git checkpoint

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 10 verification failed."
}

git status
git add src\main src\test
git commit -m "Map LearnHub JPA relationships and add paginated DTO queries"
```

Before committing, verify that no credentials or local environment files are staged.

### Final architecture review

| Concern | Lesson 10 approach |
|---|---|
| Database relationships | Foreign keys plus JPA associations |
| Fetching | Explicit lazy relationships and entity graphs |
| Catalog reads | JPQL constructor projection |
| Pagination | PostgreSQL-backed `Pageable` |
| Transactions | Service-level read-only transactions |
| Angular compatibility | Explicit DTOs, not serialized entities |
| Database testing | PostgreSQL Testcontainers |

The original `/api/v1/courses` API remains available. The new `/api/v1/course-summaries` endpoint is a focused learning implementation; it does not yet replace the original catalog's complete filtering and sorting behavior.

## 25. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="book-open" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 11 — Production-Oriented Course CRUD APIs**

    We'll consolidate the two catalog read paths, implement database-backed filtering and sorting, introduce consistent pagination, strengthen course creation and update workflows, and prepare instructor ownership for the security lessons.

    We'll also ensure Angular's existing course cards, detail pages, and catalog filters work against one stable API.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 11: production-oriented Course CRUD APIs, database-backed catalog search and filtering, sorting, pagination, course DTO mapping, transactional updates, instructor ownership preparation, PostgreSQL integration tests, and Angular frontend compatibility. Continue from Lesson 10 with experienced-Java pacing, exact file paths, complete documented code, Windows PowerShell commands, exercises, quiz, and checklist.")}>Start Lesson 11 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 10 deliverable:** LearnHub now has mapped JPA relationships, explicit fetch strategies, database-backed DTO projections, and paginated catalog queries.

The most important takeaway is that **a correct entity relationship does not automatically produce an efficient API**. Query design, transaction scope, and DTO boundaries are what turn those relationships into predictable backend behavior.