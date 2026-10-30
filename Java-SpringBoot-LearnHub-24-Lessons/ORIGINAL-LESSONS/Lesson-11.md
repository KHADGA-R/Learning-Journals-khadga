# LearnHub Java & Spring Boot Backend — Lesson 11

<text color="secondary" size="sm">Production-Oriented Course CRUD APIs · Database Search · Filtering · Sorting · Pagination · Transactional Updates · Angular Compatibility</text>

**Stack:** Java 21 · Spring Boot 3.5.x · Spring Data JPA · PostgreSQL · Flyway · Bean Validation · Testcontainers

**Estimated time:** 5–7 hours, including implementation and verification.

In Lesson 10, we mapped instructors and enrollments into JPA entities, investigated lazy loading and N+1 queries, and introduced a database-backed `/api/v1/course-summaries` endpoint.

Today we'll consolidate that work into the main LearnHub course API.

The goal is not simply to add more endpoints. It's to make the catalog **predictable, transactional, compatible with Angular, and ready for authentication and instructor ownership**.

## 1. What you'll build

By the end of this lesson, your backend will support:

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/v1/courses` | Published catalog with filters, sorting, pagination |
| GET | `/api/v1/courses/{id}` | Public published course details |
| POST | `/api/v1/courses` | Create a draft |
| PUT | `/api/v1/courses/{id}` | Update a draft atomically |
| DELETE | `/api/v1/courses/{id}` | Delete an eligible draft |
| POST | `/api/v1/courses/{id}/publish` | Publish a draft |
| POST | `/api/v1/courses/{id}/archive` | Archive a published course |

We'll retain the `/api/v1/course-summaries` endpoint from Lesson 10 temporarily, but **`/api/v1/courses` becomes the canonical catalog API**.

The mutating endpoints are development-only at this stage. **Do not expose them publicly until Spring Security and ownership authorization are implemented.**

## 2. Architecture

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" padding={3} radius="lg" align="center" gap={1}>
    **Angular LearnHub**
    <text size="xs" color="secondary">Course cards · Catalog filters · Course details</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border padding={3} radius="lg" align="center" gap={1}>
    **CourseController**
    <text size="xs" color="secondary">HTTP contracts · Validation · Status codes</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border padding={3} radius="lg" align="center" gap={1}>
    **CourseApplicationService**
    <text size="xs" color="secondary">Transactions · Lifecycle · Ownership preparation</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border padding={3} radius="lg" align="center" gap={1}>
    **Spring Data JPA + Specifications**
    <text size="xs" color="secondary">Database filtering · Sorting · Pagination</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box background="surface-secondary" padding={3} radius="lg" align="center" gap={1}>
    <icon name="database" size="lg"/>
    **PostgreSQL**
    <text size="xs" color="secondary">Courses · Instructors · Enrollments · Constraints</text>
  </box>
</box>

We'll use **JPA Specifications** for dynamic filtering, a stable response DTO for Angular, and service-level transactions for writes.

## 3. Important migration assumptions

Your Lesson 7–10 code already includes a `CourseEntity`, `JpaCourseRepository`, Flyway migrations, Bean Validation, and an existing controller.

Because those earlier files may contain additional fields and methods, **we will preserve them rather than overwrite them blindly**.

Before making changes, confirm the current API contracts and entity mappings:

```powershell
cd $HOME\angular-learning\learnhub-backend

Get-ChildItem src\main\java\com\learnhub -Recurse -Filter *.java |
    Select-String "class CourseController|class CourseEntity|interface JpaCourseRepository"

Get-ChildItem src\main\java\com\learnhub -Recurse -Filter *.java |
    Select-String "record CourseWriteRequest|record CourseResponse|class CourseManagementService"
```

For the complete implementation below, we'll use the entity accessors introduced in the previous lessons. Where your project uses a different accessor or constructor, retain your actual mapping and adapt the corresponding line.

### Baseline verification

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 10 baseline failed."
}

git status
```

Create a checkpoint before refactoring.

---

## 4. Canonical Angular course contract

The existing LearnHub frontend expects recognizable course fields such as title, instructor, category, price, rating, free status, level, thumbnail, duration, and update date.

We'll keep those names consistent.

<CodeBlock language="json">
{
  "items": [
    {
      "id": 1,
      "title": "Angular Complete Guide",
      "description": "Build modern Angular applications.",
      "instructor": "Alex Morgan",
      "instructorId": 1,
      "category": "Web Development",
      "price": 49.99,
      "rating": 4.8,
      "isFree": false,
      "level": "BEGINNER",
      "thumbnail": null,
      "durationHours": null,
      "lastUpdated": null
    }
  ],
  "pageNumber": 0,
  "pageSize": 12,
  "totalItems": 1,
  "totalPages": 1
}
</CodeBlock>

This is an illustrative response shape. The actual values and nullability depend on your database.

The catalog API must never reveal `DRAFT` or `ARCHIVED` courses.

### Filter and sorting contract

| Parameter | Example | Meaning |
|---|---|---|
| `search` | `Angular` | Search title, description, instructor, category |
| `category` | `Web Development` | Case-insensitive exact category |
| `free` | `true` | Free or paid |
| `level` | `BEGINNER` | Course level |
| `sort` | `PRICE_LOW_TO_HIGH` | Supported sort mode |
| `page` | `0` | Zero-based page number |
| `size` | `12` | Page size, maximum 100 |

Supported sort values:

```java
DEFAULT
PRICE_LOW_TO_HIGH
PRICE_HIGH_TO_LOW
RATING_HIGH_TO_LOW
```

These match the `CourseSort` enum introduced earlier.

## 5. Implementation strategy

We'll make the smallest set of architectural changes that gives us a proper production-oriented foundation.

<text weight="medium">Files to create or modify</text>

```text
src/main/java/com/learnhub/
├── persistence/
│   ├── CourseEntity.java                  MODIFY: add safe state-change methods
│   ├── JpaCourseRepository.java           MODIFY: add JpaSpecificationExecutor
│   └── CourseSpecifications.java          NEW
├── service/
│   └── CourseApplicationService.java      NEW
├── web/
│   ├── CourseController.java              REPLACE existing mapping carefully
│   └── dto/
│       ├── CourseCatalogResponse.java     NEW
│       └── CourseDetailsResponse.java     NEW
└── exception/
    └── CourseConflictException.java       NEW

src/test/java/com/learnhub/
└── persistence/
    └── CourseCrudIntegrationTest.java     NEW
```

We'll **reuse** the existing Lesson 8 `CourseWriteRequest`, cross-field pricing validator, `CourseUpsertCommand`, and `ProblemDetail` exception handling. That prevents us from accidentally weakening validation.

The existing Lesson 8 transactional management service can remain temporarily, but only one controller should own the canonical `/api/v1/courses` route.

## 6. Extend the JPA repository

**File:** `src/main/java/com/learnhub/persistence/JpaCourseRepository.java`

Keep its existing declarations and add `JpaSpecificationExecutor<CourseEntity>` to its inheritance list.

For a repository that currently extends `JpaRepository`, the declaration becomes:

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.Optional;

/**
 * Course persistence operations.
 *
 * JpaSpecificationExecutor provides composable,
 * database-executed filters and pagination.
 */
public interface JpaCourseRepository
        extends JpaRepository<CourseEntity, Long>,
                JpaSpecificationExecutor<CourseEntity> {

    /**
     * Fetch the instructor for a single course.
     * Keep this Lesson 10 method if already present.
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
}
</CodeBlock>

If Lesson 7 or 8 added methods such as `findByStatus`, retain them too. Do not duplicate `findWithInstructorById`.

### Why Specifications?

A catalog with four optional filters has many possible combinations.

Without Specifications, it's tempting to create methods such as:

```java
findByStatusAndCategoryAndFreeAndLevel(...)
findByStatusAndFree(...)
findByStatusAndCategory(...)
```

That doesn't scale well.

A Specification lets us compose predicates dynamically while keeping filtering inside PostgreSQL.

## 7. Implement database-backed filters

**File:** `src/main/java/com/learnhub/persistence/CourseSpecifications.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;

import jakarta.persistence.criteria.JoinType;
import jakarta.persistence.criteria.Predicate;

import org.springframework.data.jpa.domain.Specification;

import java.util.ArrayList;
import java.util.List;
import java.util.Locale;

/**
 * Composable course catalog predicates.
 *
 * All filters are executed in the database.
 */
public final class CourseSpecifications {

    private CourseSpecifications() {
    }

    public static Specification<CourseEntity> publishedCatalog(
            String search,
            String category,
            Boolean free,
            CourseLevel level
    ) {
        return (root, query, cb) -> {

            List<Predicate> predicates = new ArrayList<>();

            // Public catalog must never expose drafts.
            predicates.add(
                    cb.equal(
                            root.get("status"),
                            CourseStatus.PUBLISHED
                    )
            );

            if (search != null && !search.isBlank()) {
                String pattern = "%" + escapeLike(
                        search.trim().toLowerCase(Locale.ROOT)
                ) + "%";

                // LEFT JOIN preserves older courses
                // without a linked instructor account.
                var instructor = root.join(
                        "instructorAccount",
                        JoinType.LEFT
                );

                predicates.add(
                        cb.or(
                                cb.like(
                                        cb.lower(root.get("title")),
                                        pattern,
                                        '\\'
                                ),
                                cb.like(
                                        cb.lower(root.get("description")),
                                        pattern,
                                        '\\'
                                ),
                                cb.like(
                                        cb.lower(root.get("instructor")),
                                        pattern,
                                        '\\'
                                ),
                                cb.like(
                                        cb.lower(root.get("category")),
                                        pattern,
                                        '\\'
                                ),
                                cb.like(
                                        cb.lower(instructor.get("displayName")),
                                        pattern,
                                        '\\'
                                )
                        )
                );
            }

            if (category != null && !category.isBlank()) {
                predicates.add(
                        cb.equal(
                                cb.lower(root.get("category")),
                                category.trim()
                                        .toLowerCase(Locale.ROOT)
                        )
                );
            }

            if (free != null) {
                predicates.add(
                        cb.equal(root.get("free"), free)
                );
            }

            if (level != null) {
                predicates.add(
                        cb.equal(root.get("level"), level)
                );
            }

            return cb.and(
                    predicates.toArray(Predicate[]::new)
            );
        };
    }

    /**
     * Treat user-entered % and _ as literal
     * characters, not SQL LIKE wildcards.
     */
    private static String escapeLike(String input) {
        return input
                .replace("\\", "\\\\")
                .replace("%", "\\%")
                .replace("_", "\\_");
    }
}
</CodeBlock>

### Important implementation details

The `root.get("...")` names are **Java entity property names**, not PostgreSQL column names.

For example:

```java
root.get("instructorAccount")
```

corresponds to the Lesson 10 relationship, whereas the database column is `instructor_user_id`.

The `LEFT JOIN` is essential because some older courses have no linked instructor account.

We also escape `%` and `_` so user searches behave as literal substring searches.

**Performance note:** Case-insensitive substring matching across several columns will not scale indefinitely with ordinary B-tree indexes. We'll investigate PostgreSQL trigram indexing and query plans in Lesson 12.

## 8. Sorting and pagination

We'll implement sorting centrally rather than accepting arbitrary property names from the HTTP request.

**File:** `src/main/java/com/learnhub/service/CourseSortMapper.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.catalog.CourseSort;

import org.springframework.data.domain.Sort;

/**
 * Whitelisted API sorting modes.
 *
 * A unique ID tie-breaker ensures deterministic
 * ordering across pages.
 */
public final class CourseSortMapper {

    private CourseSortMapper() {
    }

    public static Sort toJpaSort(CourseSort sort) {
        return switch (sort) {
            case DEFAULT ->
                    Sort.by(Sort.Direction.ASC, "id");

            case PRICE_LOW_TO_HIGH ->
                    Sort.by(Sort.Order.asc("price"),
                            Sort.Order.asc("id"));

            case PRICE_HIGH_TO_LOW ->
                    Sort.by(Sort.Order.desc("price"),
                            Sort.Order.asc("id"));

            case RATING_HIGH_TO_LOW ->
                    Sort.by(Sort.Order.desc("rating"),
                            Sort.Order.asc("id"));
        };
    }
}
</CodeBlock>

This preserves the Lesson 2 sort semantics and avoids allowing clients to sort by sensitive or unexpected entity properties.

### Page size protection

We will enforce:

```text
page >= 0
1 <= size <= 100
```

The page limit is important because a request such as `size=1000000` could otherwise consume excessive database and application resources.

## 9. Create the Angular response DTOs

We'll keep JPA entities out of the HTTP response.

### 9.1 Course details response

**File:** `src/main/java/com/learnhub/web/dto/CourseDetailsResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseEntity;

import java.math.BigDecimal;
import java.time.LocalDate;

/**
 * Explicit public representation of a course.
 *
 * No JPA entities or lazy collections are exposed.
 */
public record CourseDetailsResponse(
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
        CourseStatus status,
        String thumbnail,
        BigDecimal durationHours,
        LocalDate lastUpdated
) {

    public static CourseDetailsResponse from(
            CourseEntity course
    ) {
        var account = course.getInstructorAccount();

        return new CourseDetailsResponse(
                course.getId(),
                course.getTitle(),
                course.getDescription(),
                course.getInstructor(),
                account == null ? null : account.getId(),
                course.getCategory(),
                course.getPrice(),
                course.getRating(),
                course.isFree(),
                course.getLevel(),
                course.getStatus(),
                course.getThumbnail(),
                course.getDurationHours(),
                course.getLastUpdated()
        );
    }
}
</CodeBlock>

This uses conventional entity getters. If your Lesson 7 `CourseEntity` uses `getFree()` rather than `isFree()`, change that one accessor accordingly. Do not add duplicate fields merely to satisfy the mapper.

### 9.2 Paginated catalog response

**File:** `src/main/java/com/learnhub/web/dto/CourseCatalogResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import org.springframework.data.domain.Page;

import java.util.List;

/**
 * Stable page structure consumed by Angular.
 */
public record CourseCatalogResponse(
        List<CourseDetailsResponse> items,
        int pageNumber,
        int pageSize,
        long totalItems,
        int totalPages
) {
    public CourseCatalogResponse {
        items = List.copyOf(items);
    }

    public static CourseCatalogResponse from(
            Page<CourseDetailsResponse> page
    ) {
        return new CourseCatalogResponse(
                page.getContent(),
                page.getNumber(),
                page.getSize(),
                page.getTotalElements(),
                page.getTotalPages()
        );
    }
}
</CodeBlock>

We intentionally expose `items`, `pageNumber`, `pageSize`, `totalItems`, and `totalPages`, consistent with the LearnHub catalog paging model.

### Angular compatibility warning: `isFree`

Java records with a component named `isFree` usually serialize that property as `isFree` through Jackson's record support.

Verify this with an HTTP contract test rather than assuming every JavaBean accessor follows the same naming rule.

## 10. Add explicit state-change methods to CourseEntity

A service should not need to reconstruct a JPA entity or overwrite its ID to change its state.

**File:** `src/main/java/com/learnhub/persistence/CourseEntity.java`

Add these methods inside the existing class:

<CodeBlock language="java" editable>
/**
 * Transition the managed entity to a new status.
 *
 * The application service validates whether
 * the transition is allowed before calling this.
 */
public void changeStatus(CourseStatus newStatus) {
    this.status = java.util.Objects.requireNonNull(
            newStatus,
            "Course status is required."
    );
}

/**
 * Change only editable draft fields.
 *
 * Keep the original Lesson 8 update method if it
 * already performs these assignments atomically.
 */
public void updateDraftFields(
        String title,
        String description,
        String instructor,
        String category,
        java.math.BigDecimal price,
        boolean free,
        CourseLevel level,
        java.math.BigDecimal rating
) {
    this.title = title;
    this.description = description;
    this.instructor = instructor;
    this.category = category;
    this.price = price;
    this.free = free;
    this.level = level;
    this.rating = rating;
}
</CodeBlock>

Ensure `CourseEntity.java` already imports:

```java
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;
```

If Lesson 8 already introduced an atomic `updateDraft(...)` method, **reuse it instead of adding `updateDraftFields(...)`**.

We will not mutate instructor ownership through the course update API. The `instructor` string is still a legacy display field; `instructorAccount` is the future authoritative ownership relationship.

### Why update a managed entity?

Inside a transaction, Hibernate tracks changes to managed entities.

For example:

```java
course.changeStatus(CourseStatus.PUBLISHED);
```

Hibernate can issue the necessary `UPDATE` when the persistence context flushes.

Calling `repository.save(course)` on an already managed entity is usually unnecessary.

## 11. Add a conflict exception

**File:** `src/main/java/com/learnhub/exception/CourseConflictException.java`

<CodeBlock language="java" editable>
package com.learnhub.exception;

/**
 * A requested course operation conflicts with
 * the current persisted resource state.
 */
public final class CourseConflictException
        extends RuntimeException {

    public CourseConflictException(String message) {
        super(message);
    }
}
</CodeBlock>

Map this exception to HTTP `409 Conflict` in your Lesson 8 `@RestControllerAdvice`:

<CodeBlock language="java" editable>
@ExceptionHandler(CourseConflictException.class)
public ResponseEntity<ProblemDetail> handleCourseConflict(
        CourseConflictException exception
) {
    ProblemDetail problem = ProblemDetail.forStatus(
            HttpStatus.CONFLICT
    );

    problem.setTitle("Course state conflict");
    problem.setDetail(exception.getMessage());
    problem.setProperty(
            "errorCode",
            "COURSE_STATE_CONFLICT"
    );

    return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(problem);
}
</CodeBlock>

Required imports if not already present:

```java
import com.learnhub.exception.CourseConflictException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
```

Use your existing `CourseNotFoundException` handler for `404 Not Found`.

**Avoid duplicating exception handlers** if Lesson 8 already maps `InvalidCourseStateException` or `CourseConflictException` to `409`.

## 12. Define a complete write request

To make the CRUD flow self-contained, we'll introduce a clearly named Lesson 11 request type. It uses the same pricing rules as Lesson 8.

**File:** `src/main/java/com/learnhub/web/dto/CourseCommandRequest.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.learnhub.model.CourseLevel;

import jakarta.validation.constraints.*;

import java.math.BigDecimal;

/**
 * Create/update payload for course drafts.
 *
 * Keep this DTO separate from persistence entities.
 */
public record CourseCommandRequest(

        @NotBlank
        @Size(max = 150)
        String title,

        @NotBlank
        String description,

        @NotBlank
        @Size(max = 120)
        String instructor,

        @NotBlank
        @Size(max = 100)
        String category,

        @NotNull
        @DecimalMin("0.00")
        @Digits(integer = 10, fraction = 2)
        BigDecimal price,

        boolean isFree,

        @NotNull
        CourseLevel level,

        @NotNull
        @DecimalMin("0.0")
        @DecimalMax("5.0")
        @Digits(integer = 1, fraction = 1)
        BigDecimal rating
) {
    /**
     * Free courses must have a zero price.
     * Paid courses must have a positive price.
     *
     * @JsonIgnore prevents this derived validator
     * from appearing as a response property.
     */
    @JsonIgnore
    @AssertTrue(message =
            "Free courses require zero price; " +
            "paid courses require positive price.")
    public boolean isPricingConsistent() {
        if (price == null) {
            return true; // @NotNull handles missing price.
        }

        return isFree
                ? price.signum() == 0
                : price.signum() > 0;
    }
}
</CodeBlock>

This is an alternative to the Lesson 8 class-level `@ValidCoursePricing` constraint, not a replacement you must adopt. If your existing `CourseWriteRequest` already covers these fields and rules, you can retain it and substitute that type throughout the following controller and service.

The essential requirement is **one authoritative validation contract**, not two divergent ones.

## 13. Add a course creation factory

A new course starts as `DRAFT`, with a database-generated ID.

Add this static method to your existing `CourseEntity`:

<CodeBlock language="java" editable>
/**
 * Create a new unsaved draft.
 *
 * Hibernate/PostgreSQL assigns the ID on INSERT.
 */
public static CourseEntity newDraft(
        String title,
        String description,
        String instructor,
        String category,
        java.math.BigDecimal price,
        boolean free,
        CourseLevel level,
        java.math.BigDecimal rating
) {
    CourseEntity course = new CourseEntity();

    course.title = title;
    course.description = description;
    course.instructor = instructor;
    course.category = category;
    course.price = price;
    course.free = free;
    course.level = level;
    course.rating = rating;
    course.status = CourseStatus.DRAFT;

    return course;
}
</CodeBlock>

This assumes the existing entity has a protected or package-private no-argument constructor, as required by JPA, and that these fields have the names used in Lessons 7–10.

If your entity has additional required nonnullable columns without database defaults, initialize them here too. **Do not change existing Flyway constraints to make an incomplete factory work.**

## 14. Build the application service

**File:** `src/main/java/com/learnhub/service/CourseApplicationService.java`

This service owns catalog reads and all course lifecycle operations.

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.catalog.CourseSort;
import com.learnhub.exception.CourseConflictException;
import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.CourseSpecifications;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.web.dto.CourseCatalogResponse;
import com.learnhub.web.dto.CourseCommandRequest;
import com.learnhub.web.dto.CourseDetailsResponse;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Objects;

/**
 * Canonical application boundary for course operations.
 *
 * HTTP controllers do not access repositories directly.
 */
@Service
public class CourseApplicationService {

    private final JpaCourseRepository repository;

    public CourseApplicationService(
            JpaCourseRepository repository
    ) {
        this.repository = Objects.requireNonNull(repository);
    }

    @Transactional(readOnly = true)
    public CourseCatalogResponse catalog(
            String search,
            String category,
            Boolean free,
            CourseLevel level,
            CourseSort sort,
            int page,
            int size
    ) {
        validatePage(page, size);

        Page<CourseDetailsResponse> result = repository
                .findAll(
                        CourseSpecifications.publishedCatalog(
                                search, category, free, level
                        ),
                        PageRequest.of(
                                page,
                                size,
                                CourseSortMapper.toJpaSort(sort)
                        )
                )
                .map(CourseDetailsResponse::from);

        return CourseCatalogResponse.from(result);
    }

    @Transactional(readOnly = true)
    public CourseDetailsResponse publicCourse(long id) {
        CourseEntity course = repository.findWithInstructorById(id)
                .filter(c -> c.getStatus() == CourseStatus.PUBLISHED)
                .orElseThrow(() -> new CourseNotFoundException(id));

        return CourseDetailsResponse.from(course);
    }

    @Transactional
    public CourseDetailsResponse createDraft(
            CourseCommandRequest request
    ) {
        Objects.requireNonNull(request);

        CourseEntity course = CourseEntity.newDraft(
                request.title().trim(),
                request.description().trim(),
                request.instructor().trim(),
                request.category().trim(),
                request.price(),
                request.isFree(),
                request.level(),
                request.rating()
        );

        CourseEntity saved = repository.saveAndFlush(course);

        return CourseDetailsResponse.from(saved);
    }

    @Transactional
    public CourseDetailsResponse updateDraft(
            long id,
            CourseCommandRequest request
    ) {
        Objects.requireNonNull(request);

        CourseEntity course = findExisting(id);

        requireStatus(
                course,
                CourseStatus.DRAFT,
                "Only draft courses can be edited."
        );

        course.updateDraftFields(
                request.title().trim(),
                request.description().trim(),
                request.instructor().trim(),
                request.category().trim(),
                request.price(),
                request.isFree(),
                request.level(),
                request.rating()
        );

        repository.flush();

        return CourseDetailsResponse.from(course);
    }

    @Transactional
    public CourseDetailsResponse publish(long id) {
        CourseEntity course = findExisting(id);

        requireStatus(
                course,
                CourseStatus.DRAFT,
                "Only draft courses can be published."
        );

        course.changeStatus(CourseStatus.PUBLISHED);
        repository.flush();

        return CourseDetailsResponse.from(course);
    }

    @Transactional
    public CourseDetailsResponse archive(long id) {
        CourseEntity course = findExisting(id);

        requireStatus(
                course,
                CourseStatus.PUBLISHED,
                "Only published courses can be archived."
        );

        course.changeStatus(CourseStatus.ARCHIVED);
        repository.flush();

        return CourseDetailsResponse.from(course);
    }

    @Transactional
    public void deleteDraft(long id) {
        CourseEntity course = findExisting(id);

        requireStatus(                course,
                CourseStatus.DRAFT,
                "Only draft courses can be deleted."
        );

        // Enrolled courses cannot be deleted because
        // PostgreSQL enforces the enrollment foreign key.
        // The explicit check produces a clearer conflict.
        if (!course.getEnrollments().isEmpty()) {
            throw new CourseConflictException(
                    "A course with enrollments cannot be deleted."
            );
        }

        repository.delete(course);
        repository.flush();
    }

    private CourseEntity findExisting(long id) {
        return repository.findWithInstructorById(id)
                .orElseThrow(() -> new CourseNotFoundException(id));
    }

    private static void requireStatus(
            CourseEntity course,
            CourseStatus expected,
            String message
    ) {
        if (course.getStatus() != expected) {
            throw new CourseConflictException(message);
        }
    }

    private static void validatePage(int page, int size) {
        if (page < 0) {
            throw new IllegalArgumentException(
                    "Page number must be nonnegative."
            );
        }

        if (size < 1 || size > 100) {
            throw new IllegalArgumentException(
                    "Page size must be between 1 and 100."
            );
        }
    }
}
</CodeBlock>

### Important production considerations

**Atomic updates:** `updateDraft()` changes one managed entity inside a transaction. If flush fails, Spring rolls back the transaction.

**Foreign-key safety:** the enrollment foreign key is still the final protection against deletion. The collection check improves the error message, but another transaction could insert an enrollment after that check. The database constraint still decides the outcome.

**Ownership:** the service does not yet know the authenticated instructor. That is intentional for this lesson, but it means the write endpoints are **not safe for public deployment**.

**Validation:** `@Valid` on the controller validates HTTP input. If another application component invokes this service directly, it must pass validated input or the service must independently enforce its invariants. The Lesson 8 command-validation approach can provide that additional boundary.

**Persistence:** the existing `CourseEntity` may contain required metadata fields. The new factory must satisfy all existing constraints before `saveAndFlush()` can succeed.

### A better delete check for a later refactor

Instead of initializing an entire enrollment collection, use the Lesson 10 repository:

```java
boolean existsByCourseId(Long courseId);
```

This is more efficient and avoids loading enrollment rows.

For this lesson, we'll improve the service immediately by using that approach.

Add to `JpaEnrollmentRepository`:

```java
boolean existsByCourseId(Long courseId);
```

Then inject `JpaEnrollmentRepository` into `CourseApplicationService` and replace:

```java
if (!course.getEnrollments().isEmpty()) {
```

with:

```java
if (enrollments.existsByCourseId(id)) {
```

The constructor becomes:

```java
public CourseApplicationService(
        JpaCourseRepository repository,
        JpaEnrollmentRepository enrollments
) {
    this.repository = Objects.requireNonNull(repository);
    this.enrollments = Objects.requireNonNull(enrollments);
}
```

Add the field:

```java
private final JpaEnrollmentRepository enrollments;
```

and import:

```java
import com.learnhub.persistence.JpaEnrollmentRepository;
```

**Use this final repository-existence-check version** for the rest of the lesson.

## 15. Consolidate the canonical controller

**File:** `src/main/java/com/learnhub/web/CourseController.java`

Replace the old controller's route implementations with this version **after confirming that no other controller still maps the same paths**.

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.catalog.CourseSort;
import com.learnhub.model.CourseLevel;
import com.learnhub.service.CourseApplicationService;
import com.learnhub.web.dto.CourseCatalogResponse;
import com.learnhub.web.dto.CourseCommandRequest;
import com.learnhub.web.dto.CourseDetailsResponse;

import jakarta.validation.Valid;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;

/**
 * Canonical LearnHub course API.
 *
 * Write endpoints must be secured before deployment.
 */
@RestController
@RequestMapping("/api/v1/courses")
public class CourseController {

    private final CourseApplicationService service;

    public CourseController(CourseApplicationService service) {
        this.service = service;
    }

    @GetMapping
    public CourseCatalogResponse catalog(
            @RequestParam(required = false) String search,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) Boolean free,
            @RequestParam(required = false) CourseLevel level,
            @RequestParam(defaultValue = "DEFAULT") CourseSort sort,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "12") int size
    ) {
        return service.catalog(
                search,
                category,
                free,
                level,
                sort,
                page,
                size
        );
    }

    @GetMapping("/{id}")
    public CourseDetailsResponse getById(
            @PathVariable long id
    ) {
        return service.publicCourse(id);
    }

    @PostMapping
    public ResponseEntity<CourseDetailsResponse> create(
            @Valid @RequestBody CourseCommandRequest request
    ) {
        CourseDetailsResponse created = service.createDraft(request);

        return ResponseEntity
                .created(URI.create(
                        "/api/v1/courses/" + created.id()
                ))
                .body(created);
    }

    @PutMapping("/{id}")
    public CourseDetailsResponse update(
            @PathVariable long id,
            @Valid @RequestBody CourseCommandRequest request
    ) {
        return service.updateDraft(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable long id) {
        service.deleteDraft(id);
    }

    @PostMapping("/{id}/publish")
    public CourseDetailsResponse publish(
            @PathVariable long id
    ) {
        return service.publish(id);
    }

    @PostMapping("/{id}/archive")
    public CourseDetailsResponse archive(
            @PathVariable long id
    ) {
        return service.archive(id);
    }
}
</CodeBlock>

### HTTP status codes

| Operation | Success | Typical failure |
|---|---|---|
| List published courses | `200 OK` | `400` invalid filter/page |
| Get published course | `200 OK` | `404 Not Found` |
| Create draft | `201 Created` | `400` validation failure |
| Update draft | `200 OK` | `400`, `404`, `409` |
| Delete draft | `204 No Content` | `404`, `409` |
| Publish draft | `200 OK` | `404`, `409` |
| Archive published | `200 OK` | `404`, `409` |

The `Location` header on draft creation identifies the new resource, but our public `GET /api/v1/courses/{id}` deliberately hides drafts. Until we introduce an authenticated instructor management endpoint, the `Location` URI will not be dereferenceable through that public GET. This is a temporary API limitation we will resolve when instructor ownership is implemented.

### Error response consistency

Your Lesson 8 exception handler should produce RFC 9457 Problem Details, for example:

```json
{
  "type": "about:blank",
  "title": "Course state conflict",
  "status": 409,
  "detail": "Only draft courses can be published.",
  "errorCode": "COURSE_STATE_CONFLICT"
}
```

Ensure invalid `page` and `size` values map to `400 Bad Request`, not `500 Internal Server Error`.

A straightforward addition to your global advice is:

```java
@ExceptionHandler(IllegalArgumentException.class)
public ResponseEntity<ProblemDetail> handleInvalidArgument(
        IllegalArgumentException exception
) {
    ProblemDetail problem =
            ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);

    problem.setTitle("Invalid request");
    problem.setDetail(exception.getMessage());
    problem.setProperty("errorCode", "INVALID_REQUEST");

    return ResponseEntity.badRequest().body(problem);
}
```

Only add this if an equivalent handler does not already exist.

**Production caveat:** A broad `IllegalArgumentException` handler can hide programming errors. In a larger application, prefer a dedicated `InvalidCatalogQueryException` for invalid client-supplied filters.

## 16. Database and schema compatibility

This lesson requires **no new Flyway migration** if your Lesson 7–10 schema already includes:

- Database-generated course IDs.
- Title, description, instructor display name, category.
- Price, rating, free flag, level, status.
- Thumbnail, duration, and last-updated fields.
- Optional `instructor_user_id`.
- Enrollment foreign keys.

Check the actual table:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "\d courses"
```

If your schema has different column names or additional nonnullable fields, reconcile the entity factory and DTO mappings **before running tests**.

Do not create a migration merely to match an assumed Java accessor name.

### Instructor ownership preparation

The `instructor_user_id` column from Lesson 9 will eventually become the authoritative ownership key.

For now:

- The public catalog exposes `instructorId` when linked.
- The legacy `instructor` string remains for Angular compatibility.
- The create API does not accept an arbitrary `instructorUserId`.
- Instructor identity will eventually come from the authenticated session, not the request body.

This prevents us from designing a future API where an instructor can claim another user's identity by sending a different ID.

## 17. Test the API with Windows PowerShell

Start PostgreSQL and Spring Boot.

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

In a second PowerShell terminal:

```powershell
$base = "http://localhost:8080/api/v1/courses"
```

### 17.1 List published courses

```powershell
Invoke-RestMethod "$base?page=0&size=12" |
    ConvertTo-Json -Depth 6
```

### 17.2 Search courses

Use a hashtable to build the query safely:

```powershell
$params = @{
    search = "Angular"
    page   = 0
    size   = 10
}

$query = ($params.GetEnumerator() |
    ForEach-Object {
        "$([uri]::EscapeDataString($_.Key))=$([uri]::EscapeDataString([string]$_.Value))"
    }) -join "&"

Invoke-RestMethod "$base`?$query"
```

### 17.3 Filter and sort

```powershell
Invoke-RestMethod `
  "$base`?category=Web%20Development&free=false&sort=PRICE_LOW_TO_HIGH&page=0&size=5"
```

Check that the returned courses are published, paid, and in the requested category.

### 17.4 Create a draft

```powershell
$draft = @{
    title       = "Spring Boot API Masterclass"
    description = "Build production-oriented REST APIs."
    instructor  = "Alex Morgan"
    category    = "Web Development"
    price       = 39.99
    isFree      = $false
    level       = "INTERMEDIATE"
    rating      = 4.7
} | ConvertTo-Json

$created = Invoke-RestMethod `
  -Method Post `
  -Uri $base `
  -ContentType "application/json" `
  -Body $draft

$created | Format-List
$courseId = $created.id
```

Expected: `201 Created` with a generated ID and `DRAFT` status.

### 17.5 Update the draft

```powershell
$updated = @{
    title       = "Spring Boot API Masterclass — Updated"
    description = "Build secure, maintainable REST APIs."
    instructor  = "Alex Morgan"
    category    = "Web Development"
    price       = 44.99
    isFree      = $false
    level       = "INTERMEDIATE"
    rating      = 4.8
} | ConvertTo-Json

Invoke-RestMethod `
  -Method Put `
  -Uri "$base/$courseId" `
  -ContentType "application/json" `
  -Body $updated
```

### 17.6 Publish

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "$base/$courseId/publish"
```

The new course should now appear in the public catalog.

### 17.7 Reject an invalid transition

Try publishing it again:

```powershell
try {
    Invoke-RestMethod `
      -Method Post `
      -Uri "$base/$courseId/publish"
}
catch {
    Write-Host "Expected conflict:" $_.Exception.Message
    $_.ErrorDetails.Message
}
```

Expected: HTTP `409 Conflict`.

### 17.8 Archive

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "$base/$courseId/archive"
```

The course should disappear from public catalog results.

### 17.9 Test validation

```powershell
$invalid = @{
    title       = ""
    description = "Invalid pricing example"
    instructor  = "Alex Morgan"
    category    = "Web Development"
    price       = 50.00
    isFree      = $true
    level       = "BEGINNER"
    rating      = 4.5
} | ConvertTo-Json

try {
    Invoke-RestMethod `
      -Method Post `
      -Uri $base `
      -ContentType "application/json" `
      -Body $invalid
}
catch {
    Write-Host "Expected validation failure:"
    $_.ErrorDetails.Message
}
```

Expected: HTTP `400 Bad Request` with validation details.

---

## 18. PostgreSQL integration tests

We'll test the new application service against a real Testcontainers PostgreSQL database.

For HTTP-level contract verification, we'll also test the controller using MockMvc.

### 18.1 Service integration tests

**File:** `src/test/java/com/learnhub/persistence/CourseCrudIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;
import com.learnhub.catalog.CourseSort;
import com.learnhub.exception.CourseConflictException;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;
import com.learnhub.service.CourseApplicationService;
import com.learnhub.web.dto.CourseCommandRequest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Uses real PostgreSQL through Testcontainers.
 *
 * Spring rolls back each test's database changes.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@Transactional
class CourseCrudIntegrationTest extends PostgresIntegrationTest {

    @Autowired
    private CourseApplicationService service;

    @Autowired
    private JpaCourseRepository repository;

    private CourseCommandRequest request(String title) {
        return new CourseCommandRequest(
                title,
                "Learn REST APIs and Spring Data JPA.",
                "Alex Morgan",
                "Web Development",
                new BigDecimal("39.99"),
                false,
                CourseLevel.INTERMEDIATE,
                new BigDecimal("4.7")
        );
    }

    @Test
    void createsDraftWithDatabaseGeneratedId() {
        var created = service.createDraft(
                request("New Course")
        );

        assertTrue(created.id() > 0);
        assertEquals(CourseStatus.DRAFT, created.status());

        assertTrue(repository.existsById(created.id()));
    }

    @Test
    void updatesDraftAtomically() {
        var created = service.createDraft(
                request("Original Title")
        );

        var updated = service.updateDraft(
                created.id(),
                request("Updated Title")
        );

        assertEquals("Updated Title", updated.title());

        var persisted = repository.findById(created.id())
                .orElseThrow();

        assertEquals(
                "Updated Title",
                persisted.getTitle()
        );
    }

    @Test
    void publishesDraft() {
        var created = service.createDraft(
                request("Publishable Course")
        );

        var published = service.publish(created.id());

        assertEquals(
                CourseStatus.PUBLISHED,
                published.status()
        );
    }

    @Test
    void rejectsSecondPublish() {
        var created = service.createDraft(
                request("One-Time Publish")
        );

        service.publish(created.id());

        assertThrows(
                CourseConflictException.class,
                () -> service.publish(created.id())
        );
    }

    @Test
    void catalogHidesDrafts() {
        var created = service.createDraft(
                request("Hidden Draft")
        );

        var catalog = service.catalog(
                "Hidden Draft",
                null,
                null,
                null,
                CourseSort.DEFAULT,
                0,
                10
        );

        assertTrue(catalog.items().stream()
                .noneMatch(c -> c.id() == created.id()));
    }

    @Test
    void catalogFiltersAndSortsInDatabase() {
        var catalog = service.catalog(
                null,
                "Web Development",
                false,
                null,
                CourseSort.PRICE_LOW_TO_HIGH,
                0,
                10
        );

        var items = catalog.items();

        assertTrue(items.stream().allMatch(c ->
                c.category().equalsIgnoreCase(
                        "Web Development"
                ) && !c.isFree()
        ));

        for (int i = 1; i < items.size(); i++) {
            assertTrue(
                    items.get(i - 1).price()
                            .compareTo(items.get(i).price()) <= 0
            );
        }
    }

    @Test
    void paginationIsBounded() {
        assertThrows(
                IllegalArgumentException.class,
                () -> service.catalog(
                        null, null, null, null,
                        CourseSort.DEFAULT, 0, 101
                )
        );
    }

    @Test
    void deletesOnlyDrafts() {
        var created = service.createDraft(
                request("Temporary Draft")
        );

        service.deleteDraft(created.id());

        assertFalse(repository.existsById(created.id()));
    }
}
</CodeBlock>

These eight tests verify persistence and service behavior, but they don't yet prove that HTTP responses match Angular's JSON expectations.

### Test isolation warning

Because the test class is annotated `@Transactional`, test writes are rolled back after each test.

However, if the service throws a persistence exception that marks the transaction rollback-only, do not continue making database assertions in that same transaction. Use a separate transaction or test boundary to inspect the final database state.

For tests involving real HTTP server threads, a transaction on the test thread will not automatically roll back server-side writes. Use explicit cleanup or isolated databases for those tests.

## 19. Add HTTP contract tests

We'll use MockMvc to verify the actual JSON fields, status codes, and Problem Details.

**File:** `src/test/java/com/learnhub/web/CourseApiContractTest.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * HTTP contract tests backed by real PostgreSQL.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class CourseApiContractTest extends PostgresIntegrationTest {

    @Autowired
    private MockMvc mvc;

    @Test
    void catalogReturnsAngularCompatiblePage() throws Exception {
        mvc.perform(get("/api/v1/courses")
                        .param("page", "0")
                        .param("size", "2"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.items").isArray())
                .andExpect(jsonPath("$.items[0].id").exists())
                .andExpect(jsonPath("$.items[0].title").exists())
                .andExpect(jsonPath("$.items[0].instructor").exists())
                .andExpect(jsonPath("$.items[0].isFree").exists())
                .andExpect(jsonPath("$.pageNumber").value(0))
                .andExpect(jsonPath("$.pageSize").value(2))
                .andExpect(jsonPath("$.totalItems").isNumber());
    }

    @Test
    void draftIsNotPubliclyVisible() throws Exception {
        // Original seed course #5 was a draft.
        // If your test fixtures changed, create a draft
        // through the service instead of relying on ID 5.
        mvc.perform(get("/api/v1/courses/5"))
                .andExpect(status().isNotFound());
    }

    @Test
    void invalidPageReturnsBadRequest() throws Exception {
        mvc.perform(get("/api/v1/courses")
                        .param("page", "-1"))
                .andExpect(status().isBadRequest());
    }

    @Test
    void invalidPricingReturnsProblemDetails() throws Exception {
        String body = """
                {
                  "title": "Invalid Course",
                  "description": "Pricing is inconsistent.",
                  "instructor": "Alex Morgan",
                  "category": "Web Development",
                  "price": 49.99,
                  "isFree": true,
                  "level": "BEGINNER",
                  "rating": 4.5
                }
                """;

        mvc.perform(post("/api/v1/courses")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(body))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.status").value(400));
    }

    @Test
    void unknownCourseReturnsNotFound() throws Exception {
        mvc.perform(get("/api/v1/courses/99999999"))
                .andExpect(status().isNotFound());
    }
}
</CodeBlock>

These five tests bring the Lesson 11 addition to **13 expected tests**: eight service integration tests and five API contract tests.

The expected count assumes you haven't removed or replaced older tests. It is not a claim that the tests have passed locally.

### A test fixture improvement

The `draftIsNotPubliclyVisible()` test relies on the original course #5 still being a draft.

For long-term maintainability, prefer creating the draft inside the test through `CourseApplicationService`, then querying that generated ID through MockMvc.

That removes the dependency on a specific seed ID and makes the test resilient to future catalog changes.

### Run the tests

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=CourseCrudIntegrationTest,CourseApiContractTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 11 integration tests failed."
}
```

Then run the complete suite:

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 11 full verification failed."
}
```

### Resolve older controller tests

If Lessons 6–8 contain tests that mock the previous controller's dependencies, they'll need updating to the new `CourseApplicationService`.

Don't delete those tests merely to make the build green. Preserve their scenarios and migrate them to the new architecture.

The important regressions to retain are:

- Valid and invalid request validation.
- `404` for nonexistent courses.
- `409` for invalid lifecycle transitions.
- Atomic draft updates.
- Published-only public reads.
- Pagination and filtering.
- RFC 9457 error responses.

---

## 20. Align the Angular frontend

The backend now returns a stable paginated catalog response.

We can introduce a focused Angular API client without changing the entire frontend at once.

### 20.1 Define the API model

**File:** `learnhub/src/app/core/models/course-api.model.ts`

Create the parent folders if they don't already exist.

<CodeBlock language="typescript" editable>
export type CourseLevel =
  | 'BEGINNER'
  | 'INTERMEDIATE'
  | 'ADVANCED';

export type CourseStatus =
  | 'DRAFT'
  | 'PUBLISHED'
  | 'ARCHIVED';

export type CourseSort =
  | 'DEFAULT'
  | 'PRICE_LOW_TO_HIGH'
  | 'PRICE_HIGH_TO_LOW'
  | 'RATING_HIGH_TO_LOW';

export interface CourseApiResponse {
  id: number;
  title: string;
  description: string;
  instructor: string;
  instructorId: number | null;
  category: string;
  price: number;
  rating: number;
  isFree: boolean;
  level: CourseLevel;
  status: CourseStatus;
  thumbnail: string | null;
  durationHours: number | null;
  lastUpdated: string | null;
}

export interface CourseCatalogPage {
  items: CourseApiResponse[];
  pageNumber: number;
  pageSize: number;
  totalItems: number;
  totalPages: number;
}

export interface CourseCatalogFilters {
  search?: string;
  category?: string;
  free?: boolean;
  level?: CourseLevel;
  sort?: CourseSort;
  page?: number;
  size?: number;
}
</CodeBlock>

### 20.2 Create the API service

**File:** `learnhub/src/app/core/services/course-api.service.ts`

<CodeBlock language="typescript" editable>
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';

import {
  CourseApiResponse,
  CourseCatalogFilters,
  CourseCatalogPage,
} from '../models/course-api.model';

@Injectable({ providedIn: 'root' })
export class CourseApiService {
  private readonly http = inject(HttpClient);

  // Replace with environment.apiUrl if your app
  // already has centralized environment configuration.
  private readonly baseUrl =
    'http://localhost:8080/api/v1/courses';

  list(
    filters: CourseCatalogFilters = {}
  ): Observable<CourseCatalogPage> {
    let params = new HttpParams();

    if (filters.search?.trim()) {
      params = params.set('search', filters.search.trim());
    }

    if (filters.category?.trim()) {
      params = params.set('category', filters.category.trim());
    }

    if (filters.free !== undefined) {
      params = params.set('free', String(filters.free));
    }

    if (filters.level) {
      params = params.set('level', filters.level);
    }

    if (filters.sort) {
      params = params.set('sort', filters.sort);
    }

    params = params.set('page', filters.page ?? 0);
    params = params.set('size', filters.size ?? 12);

    return this.http.get<CourseCatalogPage>(
      this.baseUrl,
      { params }
    );
  }

  getById(id: number): Observable<CourseApiResponse> {
    return this.http.get<CourseApiResponse>(
      `${this.baseUrl}/${id}`
    );
  }
}
</CodeBlock>

### 20.3 Enable HttpClient

If not already configured, add `provideHttpClient()` to the providers in:

`learnhub/src/app/app.config.ts`

```typescript
import { provideHttpClient } from '@angular/common/http';

export const appConfig = {
  providers: [
    provideHttpClient(),
    // Keep your existing router and other providers.
  ],
};
```

Do not replace the entire `app.config.ts` if it already contains routing, animations, interceptors, or other providers.

### 20.4 CORS

The Angular development server and Spring Boot server usually run on different origins.

Configure CORS for the development frontend only.

**File:** `src/main/java/com/learnhub/config/WebCorsConfig.java`

<CodeBlock language="java" editable>
package com.learnhub.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
/**
 * Development-only CORS configuration.
 *
 * Production origins must be explicitly configured
 * according to the deployment environment.
 */
@Configuration
@Profile("dev")
public class WebCorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:4200")
                .allowedMethods(
                        "GET",
                        "POST",
                        "PUT",
                        "DELETE",
                        "OPTIONS"
                )
                .allowedHeaders("*");
    }
}
</CodeBlock>

Do not enable `allowedOrigins("*")` for credentialed cross-origin requests. When we add session authentication, we'll revisit CORS and CSRF together.

**Angular integration boundary:** Your existing frontend may have its own course interfaces and demo-data services. Adapt the returned DTO into those models if needed; don't assume every component already uses the new interface.

## 21. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="search" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise A — Combined catalog filters**

      Query published beginner courses that are free and belong to the Programming category. Verify that adding a search term narrows the result without changing the other filters. Write an integration test for an empty result.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="database" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise B — Database pagination**

      Insert 25 published test courses in an isolated test transaction. Request pages of size 10. Verify the page sizes are 10, 10, and 5, with no duplicate IDs.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="shield-check" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise C — Ownership preparation**

      Create a read-only repository query that finds courses by `instructorAccount.id`. Do not yet allow an HTTP caller to supply an instructor ID for write authorization. Explain why that would be insecure.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="test-tube" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise D — Transaction rollback**

      Force a database constraint failure during a draft update and verify, in a separate transaction, that the original title and price remain unchanged.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="monitor-smartphone" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise E — Angular catalog integration**

      Replace the demo catalog fetch in one Angular screen with `CourseApiService.list()`. Preserve loading, empty, and error states, and reset `page` to zero whenever the user changes a filter.
    </box>
  </row>
</box>

## 22. Common mistakes and review

| Mistake | Why it matters | Better approach |
|---|---|---|
| Returning `CourseEntity` directly | Exposes persistence details and lazy relationships | Explicit response DTO |
| Filtering after `findAll()` | Loads unnecessary rows into Java | Database Specifications |
| Sorting without an ID tie-breaker | Unstable page ordering | Sort by selected field, then ID |
| Checking ownership from request body | Enables impersonation | Derive identity from authenticated session |
| Updating a published course without a rule | Breaks lifecycle constraints | Enforce transitions in the service |
| Checking for enrollments without a DB constraint | Race conditions | Preserve the foreign key |
| Treating every SQL exception as HTTP 500 | Poor API contract | Translate known constraint conflicts |
| Testing only mocked repositories | Misses real SQL and mapping errors | Testcontainers integration tests |

### One additional production concern: concurrent updates

The current service-level transaction makes each update atomic, but it does not by itself prevent **lost updates** when two instructors edit the same draft concurrently.

A future improvement is optimistic locking:

```java
@Version
private Long version;
```

That requires a matching Flyway migration and a defined API conflict policy. We will not add it casually in this lesson because the frontend also needs a way to submit or honor the expected version.

---

## 23. Knowledge quiz

<LearningQuiz title="Lesson 11 — Course CRUD and JPA" nextQuizQuery="Give me five new intermediate-level LearnHub Spring Boot Lesson 11 questions about JPA Specifications, transaction boundaries, database pagination, HTTP API contracts, and instructor ownership." questions={[{"id":"q1","type":"single_select","question":"A LearnHub catalog request combines optional category, level, free, and search filters. Which approach best keeps filtering in PostgreSQL without creating a repository method for every combination?","options":[{"value":"A","label":"Load all courses and filter with Java streams","feedback":"Filtering after loading all rows increases memory and database transfer costs."},{"value":"B","label":"Use composable JPA Specifications","feedback":"Specifications construct predicates for the supplied filters and execute them through the database query."},{"value":"C","label":"Create a separate controller for each filter combination","feedback":"More controllers do not solve the combinatorial repository-query problem."},{"value":"D","label":"Store the filtered catalog in a static Java list","feedback":"A static list does not reflect database updates reliably and bypasses database filtering."}],"correctValues":["B"],"hint":"Consider which Spring Data abstraction builds dynamic database predicates."},{"id":"q2","type":"true_false","question":"Adding a unique ID as the final sort key makes pagination ordering deterministic for an unchanged dataset.","options":[{"value":"true","label":"True","feedback":"A unique tie-breaker establishes an unambiguous order among rows sharing the primary sort value."},{"value":"false","label":"False","feedback":"Without a unique tie-breaker, rows sharing a sort value can appear in an unspecified relative order; adding the ID resolves that ambiguity."}],"correctValues":["true"],"hint":"Think about two courses with identical prices and how their relative order is determined."},{"id":"q3","type":"multi_select","question":"Which practices help keep the LearnHub course API safe and maintainable? Select every correct answer.","options":[{"value":"A","label":"Return explicit response DTOs","feedback":"DTOs constrain the fields exposed to clients and avoid serializing managed entity graphs."},{"value":"B","label":"Use service-level transactions for multi-step writes","feedback":"Service transactions keep related persistence changes within one unit of work."},{"value":"C","label":"Accept instructor ownership IDs from untrusted request bodies as proof of identity","feedback":"A client-supplied identifier cannot establish who is authenticated or authorized."},{"value":"D","label":"Enforce enrollment uniqueness with a database constraint","feedback":"Database uniqueness remains effective even when concurrent requests race."}],"correctValues":["A","B","D"],"hint":"Evaluate the boundary between client-controlled data, application rules, and database guarantees."},{"id":"q4","type":"single_select","question":"A POST request successfully creates a new draft course with a database-generated ID. Which HTTP status best represents creation?","options":[{"value":"A","label":"200 OK","feedback":"200 can indicate a successful request, but 201 more precisely reports a newly created resource."},{"value":"B","label":"202 Accepted","feedback":"202 indicates accepted processing that may not yet be complete."},{"value":"C","label":"201 Created","feedback":"201 reports successful resource creation and can include a Location header."},{"value":"D","label":"204 No Content","feedback":"204 indicates success without a response body and does not specifically convey creation."}],"correctValues":["C"],"hint":"Distinguish an existing resource being retrieved from a new resource being created."},{"id":"q5","type":"true_false","question":"Annotating a Spring service method with @Transactional automatically verifies that the caller owns the course being edited.","options":[{"value":"true","label":"True","feedback":"A transaction controls persistence work; it does not authenticate a caller or check resource ownership."},{"value":"false","label":"False","feedback":"Ownership authorization requires authenticated identity and explicit permission checks, independently of transaction management."}],"correctValues":["false"],"hint":"Separate database consistency concerns from access-control decisions."}]} />

## 24. Lesson 11 completion checklist

{@body const groups=[{name:"Backend architecture",items:["Verified Lesson 10 baseline","Extended JpaCourseRepository with Specifications","Created CourseSpecifications","Implemented stable sorting and pagination","Created response and request DTOs"]},{name:"Course lifecycle",items:["Added CourseEntity draft factory and update methods","Implemented CourseApplicationService","Preserved transaction boundaries","Mapped 404 and 409 Problem Details","Consolidated canonical course controller","Confirmed no duplicate request mappings"]},{name:"Database and API tests",items:["Verified PostgreSQL schema compatibility","Added service integration tests","Added MockMvc API contract tests","Tested CRUD endpoints from PowerShell","Verified draft visibility and filtering","Ran full Maven verification"]},{name:"Angular and completion",items:["Added Angular API models","Added Angular CourseApiService","Verified HttpClient configuration","Reviewed development CORS configuration","Completed at least two exercises","Committed verified changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((s,g)=>s+g.items.length,0)}
{@body const count=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 11 progress**
      <text color="secondary" size="xs">Check items after local verification.</text>
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
    <row gap={2} align="center">
      <icon name="check-circle" color="success"/>
      **Lesson 11 checklist completed.**
    </row>
  {/if}
</box>

## 25. Final verification and Git checkpoint

Run the backend build:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 11 backend verification failed."
}

git status
git diff --check
git add src\main src\test
git diff --cached --stat
git commit -m "Consolidate course CRUD and database catalog queries"
```

If you integrated the Angular client:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "LearnHub Angular build failed."
}

git status
git add src\app
git diff --cached --stat
git commit -m "Connect LearnHub catalog to canonical course API"
```

Before committing, inspect staged changes for credentials, `.env` files, or accidental generated files.

### Final implementation caveats

The lesson provides a complete reference flow, but because your existing Lesson 7–10 `CourseEntity` and DTO source files weren't attached, you must reconcile their exact getters, required fields, constructors, and existing test fixtures. In particular, do not remove Lesson 8's stronger atomic-update logic or validation to accommodate the example code.

Also verify that the legacy controller has been replaced rather than left alongside the new one. Duplicate `/api/v1/courses` mappings will prevent Spring MVC from starting correctly.

## 26. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" radius="lg" padding={3}>
    <icon name="search-check" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 12 — Advanced Search, Filtering, Indexes, and Flyway**

    We'll improve catalog search with PostgreSQL-specific indexing, examine query plans, add richer sorting and filtering, manage safe Flyway migrations, and make the public course catalog performant as its dataset grows.

    We'll also introduce repeatable database fixtures and test pagination behavior under changing data.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 12: advanced PostgreSQL catalog search, Spring Data JPA Specifications, case-insensitive and substring search, pg_trgm indexes, EXPLAIN ANALYZE, stable sorting, keyset versus offset pagination, Flyway migrations, realistic seed data, Testcontainers performance and contract tests, Angular search integration, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 11 with experienced-Java pacing.")}>Start Lesson 12 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 11 deliverable:** A canonical, database-backed LearnHub course API with dynamic filtering, deterministic sorting, bounded pagination, transactional course lifecycle operations, and explicit Angular response DTOs.

The main architectural milestone is separating **HTTP contracts, business transactions, and database queries** so the application can evolve without coupling Angular directly to Hibernate.