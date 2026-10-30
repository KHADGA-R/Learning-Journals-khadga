# LearnHub Java & Spring Boot Backend — Lesson 4

<text color="secondary" size="sm">Collections, Streams, Generics, Exceptions, and Advanced Java Practices</text>

<row align="center" gap={2} wrap="wrap">
  <badge>Lesson 4 of 24</badge>
  <badge>Experienced Java Track</badge>
  <badge>Steady Pace</badge>
</row>

In Lesson 3, we separated LearnHub into a repository interface, an in-memory repository, and a catalog service. We also introduced constructor-based dependency injection and tested the architecture without Spring.

Now we'll strengthen that foundation with **production-oriented Java techniques**: immutable collections, generics, stream processing, typed exceptions, result handling, and reliable service-level tests.

This is our final plain-Java lesson before introducing Spring Boot in Lesson 5.

<box border radius="lg" padding={3} gap={2}>
  **Learning objectives**

  By the end, you'll be able to:

  - Choose appropriate Java collection types for backend operations.
  - Use streams and collectors for grouping, aggregation, and summaries.
  - Design reusable generic types without unnecessary abstraction.
  - Create a typed exception hierarchy for business operations.
  - Implement safe course creation and publishing rules.
  - Distinguish domain failures from infrastructure failures.
  - Use Java 21 features such as records, sealed types, and pattern-matching `switch`.
  - Test error paths, immutable boundaries, and business invariants.
</box>

<text color="secondary" size="sm">Estimated time: 2–3 hours, including implementation and exercises.</text>

## 1. Where Lesson 4 fits

<box border radius="lg" padding={3} gap={2}>
  <box background="surface-secondary" padding={3} radius="md" align="center" gap={1}>
    **Main.java**
    <text color="secondary" size="xs">Application composition root</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <grid columns={2} gap={2}>
    <grid-item>
      <box border padding={3} radius="md" align="center" gap={1}>
        **CourseCatalogService**
        <text color="secondary" size="xs">Read and search</text>
      </box>
    </grid-item>
    <grid-item>
      <box border padding={3} radius="md" align="center" gap={1}>
        **CourseManagementService**
        <text color="secondary" size="xs">Create and publish</text>
      </box>
    </grid-item>
  </grid>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border padding={3} radius="md" align="center" gap={1}>
    **CourseRepository interface**
    <text color="secondary" size="xs">Storage contract</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" padding={3} radius="md" align="center" gap={1}>
    **InMemoryCourseRepository**
    <text color="secondary" size="xs">Current storage implementation</text>
  </box>
</box>

We'll add course management operations while preserving the Lesson 3 architecture. We'll also introduce reusable catalog summaries and domain exceptions.

## 2. Verify your existing project

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
mvn clean verify
```

Expected baseline: 27 passing tests if you followed the complete Lesson 3 implementation.

Create a checkpoint:

```powershell
git status
git add .
git commit -m "Checkpoint LearnHub backend lesson 3"
```

If Git reports nothing to commit, continue.

### Create the new directories

```powershell
New-Item -ItemType Directory -Force `
    src\main\java\com\learnhub\exception

New-Item -ItemType Directory -Force `
    src\main\java\com\learnhub\common

New-Item -ItemType Directory -Force `
    src\test\java\com\learnhub\common

New-Item -ItemType Directory -Force `
    src\test\java\com\learnhub\service
```

We'll create these new files:

```text
src/
├── main/java/com/learnhub/
│   ├── common/
│   │   └── Result.java
│   ├── exception/
│   │   ├── CourseNotFoundException.java
│   │   ├── DuplicateCourseException.java
│   │   └── InvalidCourseStateException.java
│   ├── catalog/
│   │   └── CatalogSummary.java
│   └── service/
│       └── CourseManagementService.java
└── test/java/com/learnhub/
    ├── common/
    │   └── ResultTest.java
    └── service/
        ├── CourseManagementServiceTest.java
        └── CourseCatalogSummaryTest.java
```

We'll also update `CourseCatalogService.java`, `InMemoryCourseRepository.java`, and `Main.java`.

## 3. Java collections for backend development

Java's collection framework provides different data structures with different guarantees.

| Collection | Typical use | Important property |
|---|---|---|
| `List<T>` | Ordered course results | Allows duplicates |
| `Set<T>` | Unique categories or completed lesson IDs | No duplicate elements |
| `Map<K,V>` | Course lookup by ID | Unique keys |
| `Queue<T>` | Work items | Queue-oriented operations |
| `Deque<T>` | Stack/queue operations | Supports both ends |
| `EnumSet<E>` | Allowed enum states | Compact enum-specific set |
| `EnumMap<K,V>` | Counts by enum | Efficient enum-keyed map |

### List vs Set vs Map

<box border radius="lg" padding={3} gap={2}>
  <grid columns={3} gap={2}>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={2} gap={2}>
        <text weight="medium" textAlign="center">List</text>
        <box gap={1}>
          {#each ["Angular","Python","Angular"] as item,i}
            <box background="surface" border radius="sm" padding={1} key={i}>
              <text size="xs" textAlign="center">{item}</text>
            </box>
          {/each}
        </box>
        <text color="secondary" size="2xs" textAlign="center">Ordered; duplicates allowed</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={2} gap={2}>
        <text weight="medium" textAlign="center">Set</text>
        <box gap={1}>
          {#each ["Angular","Python"] as item,i}
            <box background="surface" border radius="sm" padding={1} key={i}>
              <text size="xs" textAlign="center">{item}</text>
            </box>
          {/each}
        </box>
        <text color="secondary" size="2xs" textAlign="center">Unique elements</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={2} gap={2}>
        <text weight="medium" textAlign="center">Map</text>
        <box gap={1}>
          {#each ["1 → Angular","2 → Python"] as item,i}
            <box background="surface" border radius="sm" padding={1} key={i}>
              <text size="xs" textAlign="center">{item}</text>
            </box>
          {/each}
        </box>
        <text color="secondary" size="2xs" textAlign="center">Unique keys, mapped values</text>
      </box>
    </grid-item>
  </grid>
</box>

Our `InMemoryCourseRepository` already uses:

```java
private final Map<Long, Course> courses =
        new LinkedHashMap<>();
```

This gives us key-based lookup while preserving insertion order internally. Our `findAll()` method then sorts results by ID, so its external ordering is explicit.

### Immutability boundaries

Our Lesson 3 repository returns:

```java
return courses.values()
        .stream()
        .sorted(Comparator.comparingLong(Course::id))
        .toList();
```

`Stream.toList()` returns an unmodifiable list.

Similarly, `List.copyOf()` creates an unmodifiable snapshot of a collection.

This matters because a caller should not be able to corrupt repository state by modifying a returned list.

**Note:** Unmodifiable collections are not necessarily deeply immutable. If their elements reference mutable objects, those objects may still change. Our `Course` record contains immutable value types, which simplifies this concern.

## 4. Improve repository mutation semantics

In Lesson 3, `save()` replaced any existing course with the same ID. That's useful for a simple repository, but application services should distinguish between creating a new course and updating an existing one.

We'll keep the repository's upsert contract, but enforce create-only behavior in `CourseManagementService`.

First, let's strengthen the repository constructor and keep a deterministic snapshot.

Replace:

`src/main/java/com/learnhub/repository/InMemoryCourseRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.repository;

import com.learnhub.model.Course;

import java.util.Collection;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;

/**
 * Thread-safe, in-memory implementation of CourseRepository.
 *
 * Individual operations are synchronized.
 * Multi-step service operations are not automatically
 * atomic across multiple repository calls.
 */
public final class InMemoryCourseRepository
        implements CourseRepository {

    private final Map<Long, Course> courses =
            new LinkedHashMap<>();

    public InMemoryCourseRepository() {
    }

    public InMemoryCourseRepository(
            Collection<Course> initialCourses
    ) {
        Objects.requireNonNull(initialCourses);

        for (Course course : initialCourses) {
            Objects.requireNonNull(course);

            if (courses.putIfAbsent(
                    course.id(),
                    course
            ) != null) {
                throw new IllegalArgumentException(
                        "Duplicate course ID: " + course.id()
                );
            }
        }
    }

    @Override
    public synchronized List<Course> findAll() {
        return courses.values()
                .stream()
                .sorted(Comparator.comparingLong(Course::id))
                .toList();
    }

    @Override
    public synchronized Optional<Course> findById(long id) {
        return id <= 0
                ? Optional.empty()
                : Optional.ofNullable(courses.get(id));
    }

    @Override
    public synchronized Course save(Course course) {
        Objects.requireNonNull(course);
        courses.put(course.id(), course);
        return course;
    }

    @Override
    public synchronized boolean deleteById(long id) {
        return courses.remove(id) != null;
    }
}
</CodeBlock>

This remains compatible with Lesson 3 tests.

Later, the database will enforce uniqueness using primary keys and constraints, and Spring will provide transactional boundaries.

## 5. Typed domain exceptions

A common backend mistake is to throw `IllegalArgumentException` for every possible failure.

That makes it difficult to distinguish:

- A missing course.
- A duplicate course ID.
- An invalid course state transition.
- A malformed HTTP request.
- A database outage.

We'll introduce explicit exception types.

### Step 5.1 — CourseNotFoundException

File: `src/main/java/com/learnhub/exception/CourseNotFoundException.java`

<CodeBlock language="java" editable>
package com.learnhub.exception;

/**
 * Raised when a required course does not exist.
 */
public final class CourseNotFoundException
        extends RuntimeException {

    public CourseNotFoundException(long courseId) {
        super("Course not found: " + courseId);
    }
}
</CodeBlock>

### Step 5.2 — DuplicateCourseException

File: `src/main/java/com/learnhub/exception/DuplicateCourseException.java`

<CodeBlock language="java" editable>
package com.learnhub.exception;

/**
 * Raised when a create operation conflicts
 * with an existing course identifier.
 */
public final class DuplicateCourseException
        extends RuntimeException {

    public DuplicateCourseException(long courseId) {
        super("Course already exists: " + courseId);
    }
}
</CodeBlock>

### Step 5.3 — InvalidCourseStateException

File: `src/main/java/com/learnhub/exception/InvalidCourseStateException.java`

<CodeBlock language="java" editable>
package com.learnhub.exception;

/**
 * Raised when a requested lifecycle transition
 * violates course publishing rules.
 */
public final class InvalidCourseStateException
        extends RuntimeException {

    public InvalidCourseStateException(String message) {
        super(message);
    }
}
</CodeBlock>

### Why unchecked exceptions?

These are application/domain failures that callers generally handle at an application boundary.

When we introduce Spring Web, a global exception handler will translate them into appropriate HTTP responses, for example:

| Exception | Future HTTP status |
|---|---|
| `CourseNotFoundException` | 404 Not Found |
| `DuplicateCourseException` | 409 Conflict |
| `InvalidCourseStateException` | 409 Conflict |
| Request validation failure | 400 Bad Request |
| Unexpected server error | 500 Internal Server Error |

The mapping is a design decision we'll implement later. Not every domain exception automatically corresponds to a particular HTTP status.

## 6. Build CourseManagementService

Now we'll implement course creation and publishing rules.

### Publishing rules

We'll use this simple lifecycle:

<box border radius="lg" padding={3} gap={2}>
  <row justify="center" align="center" gap={2}>
    <box background="surface-secondary" radius="md" padding={3} align="center">
      **DRAFT**
    </box>
    <icon name="arrow-right" color="secondary"/>
    <box border radius="md" padding={3} align="center">
      **PUBLISHED**
    </box>
    <icon name="arrow-right" color="secondary"/>
    <box background="surface-secondary" radius="md" padding={3} align="center">
      **ARCHIVED**
    </box>
  </row>
  <text color="secondary" size="sm" textAlign="center">For this lesson, publishing is allowed only from DRAFT. Archived courses cannot be republished.</text>
</box>

File: `src/main/java/com/learnhub/service/CourseManagementService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.exception.DuplicateCourseException;
import com.learnhub.exception.InvalidCourseStateException;
import com.learnhub.model.Course;
import com.learnhub.model.CourseStatus;
import com.learnhub.repository.CourseRepository;

import java.util.Objects;

/**
 * Application service for course management.
 *
 * Separates business operations from storage details.
 *
 * Authentication and instructor ownership will be
 * introduced in later Spring Security lessons.
 */
public final class CourseManagementService {

    private final CourseRepository repository;

    public CourseManagementService(
            CourseRepository repository
    ) {
        this.repository = Objects.requireNonNull(repository);
    }

    /**
     * Creates a new draft course.
     *
     * Rejects duplicate IDs and courses that are
     * already published or archived.
     */
    public Course createDraft(Course course) {

        Objects.requireNonNull(course, "Course is required.");

        if (course.status() != CourseStatus.DRAFT) {
            throw new InvalidCourseStateException(
                    "New courses must start as DRAFT."
            );
        }

        if (repository.findById(course.id()).isPresent()) {
            throw new DuplicateCourseException(course.id());
        }

        return repository.save(course);
    }

    /**
     * Publishes an existing draft course.
     *
     * Because Course is immutable, a new record is
     * created with the updated publication status.
     */
    public Course publish(long courseId) {

        Course existing = repository.findById(courseId)
                .orElseThrow(
                        () -> new CourseNotFoundException(courseId)
                );

        if (existing.status() != CourseStatus.DRAFT) {
            throw new InvalidCourseStateException(
                    "Only DRAFT courses can be published."            );
        }

        Course published = withStatus(
                existing,
                CourseStatus.PUBLISHED
        );

        return repository.save(published);
    }

    /**
     * Archives a published course.
     */
    public Course archive(long courseId) {

        Course existing = repository.findById(courseId)
                .orElseThrow(
                        () -> new CourseNotFoundException(courseId)
                );

        if (existing.status() != CourseStatus.PUBLISHED) {
            throw new InvalidCourseStateException(
                    "Only PUBLISHED courses can be archived."
            );
        }

        return repository.save(
                withStatus(existing, CourseStatus.ARCHIVED)
        );
    }

    /**
     * Copies an immutable record with a different status.
     *
     * This explicit copying will later motivate
     * dedicated domain entities or update methods.
     */
    private Course withStatus(
            Course course,
            CourseStatus status
    ) {
        return new Course(
                course.id(),
                course.title(),
                course.description(),
                course.instructor(),
                course.category(),
                course.price(),
                course.free(),
                course.level(),
                course.rating(),
                status
        );
    }
}
</CodeBlock>

### Important concurrency limitation

The code checks for an existing course and then calls `save()` in two separate repository operations.

Although each in-memory repository method is synchronized, the combined check-and-save operation is **not atomic**. Two concurrent calls could both observe that an ID is absent and then overwrite one another.

For this learning project, we will test the sequential business rules. In the database phase, we'll enforce uniqueness using constraints and transactions rather than relying on this check alone.

## 7. Streams and collectors: catalog summaries

We'll now use `Stream`, `Collectors.groupingBy`, `EnumMap`, and `BigDecimal.reduce` to build analytics for the future instructor dashboard.

### Step 7.1 — CatalogSummary record

File: `src/main/java/com/learnhub/catalog/CatalogSummary.java`

<CodeBlock language="java" editable>
package com.learnhub.catalog;

import com.learnhub.model.CourseLevel;

import java.math.BigDecimal;
import java.util.Map;

/**
 * Summary statistics for published courses.
 */
public record CatalogSummary(
        long totalCourses,
        long freeCourses,
        long paidCourses,
        BigDecimal totalListedPrice,
        Map<CourseLevel, Long> coursesByLevel
) {

    public CatalogSummary {
        if (totalCourses < 0
                || freeCourses < 0
                || paidCourses < 0) {
            throw new IllegalArgumentException(
                    "Course counts cannot be negative."
            );
        }

        if (freeCourses + paidCourses != totalCourses) {
            throw new IllegalArgumentException(
                    "Free and paid counts must match total."
            );
        }

        if (totalListedPrice == null
                || totalListedPrice.signum() < 0) {
            throw new IllegalArgumentException(
                    "Total listed price cannot be negative."
            );
        }

        // Defensive copy of the summary map.
        coursesByLevel = Map.copyOf(coursesByLevel);
    }
}
</CodeBlock>

### Step 7.2 — Update CourseCatalogService

Replace:

`src/main/java/com/learnhub/service/CourseCatalogService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.catalog.CatalogSummary;
import com.learnhub.catalog.CourseCatalog;
import com.learnhub.catalog.CourseFilter;
import com.learnhub.catalog.CourseSort;
import com.learnhub.catalog.PageResult;
import com.learnhub.model.Course;
import com.learnhub.model.CourseLevel;
import com.learnhub.repository.CourseRepository;

import java.math.BigDecimal;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.stream.Collectors;

/**
 * Public course catalog application service.
 *
 * Business queries remain independent of
 * the concrete repository implementation.
 */
public final class CourseCatalogService {

    private final CourseRepository repository;

    public CourseCatalogService(
            CourseRepository repository
    ) {
        this.repository = Objects.requireNonNull(repository);
    }

    public PageResult<Course> search(
            CourseFilter filter,
            CourseSort sort,
            int pageNumber,
            int pageSize
    ) {
        return new CourseCatalog(repository.findAll())
                .search(filter, sort, pageNumber, pageSize);
    }

    public Optional<Course> findPublishedById(long id) {
        return repository.findById(id)
                .filter(Course::isPubliclyVisible);
    }

    /**
     * Returns all distinct public categories
     * in alphabetical order.
     */
    public List<String> publishedCategories() {

        return repository.findAll()
                .stream()
                .filter(Course::isPubliclyVisible)
                .map(Course::category)
                .distinct()
                .sorted(String.CASE_INSENSITIVE_ORDER)
                .toList();
    }

    /**
     * Builds a summary of the public catalog.
     *
     * All statistics are calculated from the same
     * immutable list snapshot.
     */
    public CatalogSummary summarize() {

        List<Course> published = repository.findAll()
                .stream()
                .filter(Course::isPubliclyVisible)
                .toList();

        long total = published.size();

        long free = published.stream()
                .filter(Course::free)
                .count();

        long paid = total - free;

        BigDecimal totalListedPrice = published.stream()
                .map(Course::price)
                .reduce(
                        BigDecimal.ZERO,
                        BigDecimal::add
                );

        Map<CourseLevel, Long> byLevel = published.stream()
                .collect(Collectors.groupingBy(
                        Course::level,
                        () -> new EnumMap<>(CourseLevel.class),
                        Collectors.counting()
                ));

        return new CatalogSummary(
                total,
                free,
                paid,
                totalListedPrice,
                byLevel
        );
    }
}
</CodeBlock>

### Stream operations explained

```java
.filter(Course::isPubliclyVisible)
```

Selects only published courses.

```java
.map(Course::category)
```

Transforms a stream of courses into a stream of category names.

```java
.distinct()
```

Removes duplicate categories.

```java
.reduce(BigDecimal.ZERO, BigDecimal::add)
```

Aggregates monetary values using exact decimal arithmetic.

```java
Collectors.groupingBy(...)
```

Groups courses by their difficulty level and counts them.

### Why use EnumMap?

`EnumMap` is designed specifically for enum keys. It is compact and efficient and provides predictable enum-order iteration.

However, `Map.copyOf()` does not guarantee preserving the source map's iteration order. If ordered serialization becomes a requirement, we'll define that explicitly in our API DTO mapping.

### Business interpretation

`totalListedPrice` is the sum of the published course prices. It is **not revenue**, because no purchases or payments have occurred.

This distinction matters when we build instructor analytics later.

## 8. Advanced Java 21: generic result types

Exceptions are useful for unexpected or rejected operations. But some operations have expected alternative outcomes, such as a successful search or a business validation failure.

A generic result type can make those alternatives explicit.

For this lesson, we'll build a small example using Java 21 sealed interfaces and records. **We won't replace every exception with a result type.** The goal is to understand the tradeoff and use each approach intentionally.

File: `src/main/java/com/learnhub/common/Result.java`

<CodeBlock language="java" editable>
package com.learnhub.common;

import java.util.Objects;
import java.util.function.Function;

/**
 * Represents either a successful value or a failure.
 *
 * T is the type of the successful result.
 *
 * The sealed interface restricts implementations to
 * the two nested records defined below.
 */
public sealed interface Result<T>
        permits Result.Success, Result.Failure {

    /**
     * Successful result containing a non-null value.
     */
    record Success<T>(T value) implements Result<T> {

        public Success {
            Objects.requireNonNull(
                    value,
                    "Success value cannot be null."
            );
        }
    }

    /**
     * Failed result containing an error message.
     */
    record Failure<T>(String message) implements Result<T> {

        public Failure {
            if (message == null || message.isBlank()) {
                throw new IllegalArgumentException(
                        "Failure message is required."
                );
            }
        }
    }

    /**
     * Factory method for successful results.
     */
    static <T> Result<T> success(T value) {
        return new Success<>(value);
    }

    /**
     * Factory method for failed results.
     */
    static <T> Result<T> failure(String message) {
        return new Failure<>(message);
    }

    /**
     * Transforms the successful value.
     *
     * Failure results remain failures.
     */
    default <R> Result<R> map(
            Function<? super T, ? extends R> mapper
    ) {
        Objects.requireNonNull(mapper);

        return switch (this) {
            case Success<T> success ->
                    Result.success(
                            mapper.apply(success.value())
                    );

            case Failure<T> failure ->
                    Result.failure(failure.message());
        };
    }
}
</CodeBlock>

### What does `T` represent?

`Result<T>` is generic. It can represent different successful value types:

```java
Result<Course>
Result<String>
Result<Integer>
Result<List<Course>>
```

The compiler checks the successful value's type.

### What does `sealed` provide?

A sealed interface restricts which types may implement it. In this example, only `Success` and `Failure` are permitted.

That makes the pattern-matching `switch` exhaustive without a `default` branch.

### Example usage

```java
Result<String> result = Result.success("Angular");

Result<Integer> length = result.map(String::length);

String description = switch (length) {
    case Result.Success<Integer> success ->
            "Length: " + success.value();

    case Result.Failure<Integer> failure ->
            "Error: " + failure.message();
};

System.out.println(description);
```

Expected output:

```text
Length: 7
```

### When should we use this pattern?

A result type can be useful when failure is an ordinary, expected outcome that the caller should handle explicitly.

Exceptions are often more appropriate for invariant violations, unexpected infrastructure failures, or application operations that should stop immediately.

One limitation of our example is that `Failure` stores only a string. A larger application might use typed error codes, structured validation details, or a richer error hierarchy.

We'll introduce standardized HTTP error responses when we start Spring Web.

## 9. Update Main.java to demonstrate the new features

Replace:

`src/main/java/com/learnhub/Main.java`

<CodeBlock language="java" editable>
package com.learnhub;

import com.learnhub.catalog.CatalogSummary;
import com.learnhub.catalog.CourseFilter;
import com.learnhub.catalog.CourseSort;
import com.learnhub.catalog.PageResult;
import com.learnhub.common.Result;
import com.learnhub.data.SampleCourses;
import com.learnhub.exception.InvalidCourseStateException;
import com.learnhub.model.Course;
import com.learnhub.repository.CourseRepository;
import com.learnhub.repository.InMemoryCourseRepository;
import com.learnhub.service.CourseCatalogService;
import com.learnhub.service.CourseManagementService;

/**
 * LearnHub console application.
 *
 * Demonstrates:
 * - Constructor-based dependency injection
 * - Repository abstraction
 * - Course lifecycle management
 * - Stream-based catalog summaries
 * - Typed domain exceptions
 * - Generic result types
 */
public final class Main {

    private Main() {
    }

    public static void main(String[] args) {

        // Composition root: connect concrete infrastructure
        // to application services.
        CourseRepository repository =
                new InMemoryCourseRepository(
                        SampleCourses.create()
                );

        CourseCatalogService catalogService =
                new CourseCatalogService(repository);

        CourseManagementService managementService =
                new CourseManagementService(repository);

        printCatalog(catalogService);

        printSummary(catalogService);

        // Course #5 starts as a draft.
        System.out.println("\nPublishing course #5...");

        Course published = managementService.publish(5L);

        System.out.println(
                "Published: " + published.title()
        );

        // The public catalog now contains five courses.
        printSummary(catalogService);

        // Demonstrate an invalid lifecycle transition.
        try {
            managementService.publish(5L);
        } catch (InvalidCourseStateException exception) {            System.out.println(
                    "\nExpected business error: "
                            + exception.getMessage()
            );
        }

        // Demonstrate the generic Result<T> type.
        Result<String> result =
                Result.success("LearnHub");

        Result<Integer> length =
                result.map(String::length);

        String message = switch (length) {
            case Result.Success<Integer> success ->
                    "Application name length: "
                            + success.value();

            case Result.Failure<Integer> failure ->
                    "Operation failed: "
                            + failure.message();
        };

        System.out.println("\n" + message);
    }

    private static void printCatalog(
            CourseCatalogService catalogService
    ) {

        System.out.println("==============================");
        System.out.println("     LearnHub Backend");
        System.out.println("==============================");

        PageResult<Course> page = catalogService.search(
                CourseFilter.all(),
                CourseSort.DEFAULT,
                0,
                10
        );

        System.out.println("\nPublished courses:");

        for (Course course : page.items()) {
            System.out.printf(
                    "#%d | %s | %s%n",
                    course.id(),
                    course.title(),
                    course.priceLabel()
            );
        }
    }

    private static void printSummary(
            CourseCatalogService catalogService
    ) {

        CatalogSummary summary = catalogService.summarize();

        System.out.println("\nCatalog summary:");
        System.out.println(
                "Total: " + summary.totalCourses()
        );
        System.out.println(
                "Free: " + summary.freeCourses()
        );
        System.out.println(
                "Paid: " + summary.paidCourses()
        );
        System.out.println(
                "Total listed price: $"
                        + summary.totalListedPrice()
        );
        System.out.println(
                "By level: " + summary.coursesByLevel()
        );
    }
}
</CodeBlock>

### Expected behavior

Before publishing course #5:

| Metric | Expected |
|---|---:|
| Published courses | 4 |
| Free courses | 2 |
| Paid courses | 2 |
| Total listed price | $79.98 |

After publishing course #5:

| Metric | Expected |
|---|---:|
| Published courses | 5 |
| Free courses | 2 |
| Paid courses | 3 |
| Total listed price | $159.97 |

The second attempt to publish course #5 should produce a handled business exception, not terminate the application unexpectedly.

## 10. Test course management operations

File: `src/test/java/com/learnhub/service/CourseManagementServiceTest.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.data.SampleCourses;
import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.exception.DuplicateCourseException;
import com.learnhub.exception.InvalidCourseStateException;
import com.learnhub.model.Course;
import com.learnhub.model.CourseStatus;
import com.learnhub.repository.InMemoryCourseRepository;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests course lifecycle business rules.
 */
class CourseManagementServiceTest {

    private InMemoryCourseRepository repository;
    private CourseManagementService service;

    @BeforeEach
    void setUp() {
        repository = new InMemoryCourseRepository(
                SampleCourses.create()
        );

        service = new CourseManagementService(repository);
    }

    @Test
    void publishesDraftCourse() {

        Course published = service.publish(5L);

        assertEquals(
                CourseStatus.PUBLISHED,
                published.status()
        );

        assertEquals(
                CourseStatus.PUBLISHED,
                repository.findById(5L)
                        .orElseThrow()
                        .status()
        );
    }

    @Test
    void rejectsPublishingAlreadyPublishedCourse() {

        assertThrows(
                InvalidCourseStateException.class,
                () -> service.publish(1L)
        );
    }

    @Test
    void rejectsPublishingUnknownCourse() {

        assertThrows(
                CourseNotFoundException.class,
                () -> service.publish(999L)
        );
    }

    @Test
    void archivesPublishedCourse() {

        Course archived = service.archive(1L);

        assertEquals(
                CourseStatus.ARCHIVED,
                archived.status()
        );
    }

    @Test
    void rejectsArchivingDraftCourse() {

        assertThrows(
                InvalidCourseStateException.class,
                () -> service.archive(5L)
        );
    }

    @Test
    void createsNewDraftCourse() {

        Course originalDraft = SampleCourses.create()
                .stream()
                .filter(course -> course.id() == 5L)
                .findFirst()
                .orElseThrow();

        // Use an empty repository to verify creation.
        InMemoryCourseRepository emptyRepository =
                new InMemoryCourseRepository();

        CourseManagementService emptyService =
                new CourseManagementService(emptyRepository);

        Course saved = emptyService.createDraft(
                originalDraft
        );

        assertEquals(
                CourseStatus.DRAFT,
                saved.status()
        );

        assertEquals(
                saved,
                emptyRepository.findById(5L)
                        .orElseThrow()
        );
    }

    @Test
    void rejectsDuplicateCourseId() {

        Course existingDraft = SampleCourses.create()
                .stream()
                .filter(course -> course.id() == 5L)
                .findFirst()
                .orElseThrow();

        assertThrows(
                DuplicateCourseException.class,
                () -> service.createDraft(existingDraft)
        );
    }

    @Test
    void rejectsCreatingPublishedCourse() {

        Course publishedCourse = SampleCourses.create()
                .getFirst();

        InMemoryCourseRepository emptyRepository =
                new InMemoryCourseRepository(List.of());

        CourseManagementService emptyService =
                new CourseManagementService(emptyRepository);

        assertThrows(
                InvalidCourseStateException.class,
                () -> emptyService.createDraft(
                        publishedCourse
                )
        );
    }

    @Test
    void archivedCourseCannotBePublishedAgain() {

        service.archive(1L);

        assertThrows(
                InvalidCourseStateException.class,
                () -> service.publish(1L)
        );
    }
}
</CodeBlock>

This gives us nine new tests for lifecycle operations.

## 11. Test catalog summaries

File: `src/test/java/com/learnhub/service/CourseCatalogSummaryTest.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.data.SampleCourses;
import com.learnhub.model.CourseLevel;
import com.learnhub.repository.InMemoryCourseRepository;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests stream-based catalog aggregations.
 */
class CourseCatalogSummaryTest {

    private CourseCatalogService service;

    @BeforeEach
    void setUp() {
        service = new CourseCatalogService(
                new InMemoryCourseRepository(
                        SampleCourses.create()
                )
        );
    }

    @Test
    void countsOnlyPublishedCourses() {

        var summary = service.summarize();

        assertEquals(4, summary.totalCourses());
        assertEquals(2, summary.freeCourses());
        assertEquals(2, summary.paidCourses());
    }

    @Test
    void sumsPublishedCoursePrices() {

        var summary = service.summarize();

        assertEquals(
                0,
                new BigDecimal("79.98")
                        .compareTo(summary.totalListedPrice())
        );
    }

    @Test
    void groupsPublishedCoursesByLevel() {

        var summary = service.summarize();

        assertEquals(
                3L,
                summary.coursesByLevel()
                        .get(CourseLevel.BEGINNER)
        );

        assertEquals(
                1L,
                summary.coursesByLevel()
                        .get(CourseLevel.INTERMEDIATE)
        );

        assertFalse(
                summary.coursesByLevel()
                        .containsKey(CourseLevel.ADVANCED)
        );
    }

    @Test
    void returnsUniquePublishedCategories() {

        List<String> categories =
                service.publishedCategories();

        assertEquals(
                List.of(
                        "Data Science",
                        "Design",
                        "Programming",
                        "Web Development"
                ),
                categories
        );
    }

    @Test
    void handlesEmptyRepository() {

        CourseCatalogService emptyService =
                new CourseCatalogService(
                        new InMemoryCourseRepository()
                );

        var summary = emptyService.summarize();

        assertEquals(0, summary.totalCourses());
        assertEquals(0, summary.freeCourses());
        assertEquals(0, summary.paidCourses());

        assertTrue(summary.coursesByLevel().isEmpty());
        assertTrue(emptyService.publishedCategories().isEmpty());
    }

    @Test
    void summaryMapCannotBeModified() {

        var summary = service.summarize();

        assertThrows(
                UnsupportedOperationException.class,
                () -> summary.coursesByLevel()
                        .put(CourseLevel.ADVANCED, 100L)
        );
    }
}
</CodeBlock>

This adds six tests for stream aggregation and immutable summaries.

## 12. Test the generic Result type

File: `src/test/java/com/learnhub/common/ResultTest.java`

<CodeBlock language="java" editable>
package com.learnhub.common;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests the generic Result<T> abstraction.
 */
class ResultTest {

    @Test
    void mapsSuccessfulValue() {

        Result<String> result =
                Result.success("LearnHub");

        Result<Integer> mapped =
                result.map(String::length);

        assertInstanceOf(
                Result.Success.class,
                mapped
        );

        Result.Success<Integer> success =
                (Result.Success<Integer>) mapped;

        assertEquals(8, success.value());
    }

    @Test
    void preservesFailureDuringMapping() {

        Result<String> result =
                Result.failure("Course not found");

        Result<Integer> mapped =
                result.map(String::length);

        assertInstanceOf(
                Result.Failure.class,
                mapped
        );

        Result.Failure<Integer> failure =
                (Result.Failure<Integer>) mapped;

        assertEquals(
                "Course not found",
                failure.message()
        );
    }

    @Test
    void rejectsBlankFailureMessage() {

        assertThrows(
                IllegalArgumentException.class,
                () -> Result.failure(" ")
        );
    }

    @Test
    void rejectsNullSuccessValue() {

        assertThrows(
                NullPointerException.class,
                () -> Result.success(null)
        );
    }
}
</CodeBlock>

This adds four tests.

## 13. Run the full test suite

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean test
```
<box border radius="lg" padding={3} gap={2}>
  <row justify="between" align="center">
    <box gap={1}>
      <text color="secondary" size="xs">Expected test total</text>
      <title size="2xl" tabularNums>46</title>
    </box>
    <icon name="test-tube-diagonal" color="secondary" size="2xl"/>
  </row>
  <grid columns={2} gap={2}>
    <grid-item>
      <text weight="medium">27</text>
      <text color="secondary" size="xs">Lessons 1–3 retained tests</text>
    </grid-item>
    <grid-item>
      <text weight="medium">19</text>
      <text color="secondary" size="xs">New Lesson 4 tests</text>
    </grid-item>
  </grid>
  <caption>Expected if you retained the 27 Lesson 3 tests and added all 19 new tests. This is a target, not a report of tests executed on your computer.</caption>
</box>

Build and run:

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 4 verification failed."
}

java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar

if ($LASTEXITCODE -ne 0) {
    throw "LearnHub application failed."
}
```

If Maven reports a compilation error, inspect the file path, package declaration, imports, and Java version before changing dependencies.

## 14. Advanced engineering review

Before moving to Spring Boot, there are several important distinctions to understand.

### Exceptions versus result types

| Situation | Recommended approach |
|---|---|
| Required course does not exist during a management operation | Typed domain exception |
| User submits invalid JSON to an API | Request validation error |
| Optional course lookup returns nothing | `Optional<Course>` |
| Operation has multiple expected outcomes | Consider `Result<T>` |
| Database connection fails | Infrastructure exception, logged and handled at the boundary |
| Programmer passes a null required dependency | Fail fast with `Objects.requireNonNull()` |

The goal is not to eliminate exceptions. It's to make failure behavior understandable and consistent.

### Stream processing versus database processing

Our current code uses Java streams to calculate catalog statistics:

```java
published.stream()
        .filter(Course::free)
        .count();
```

For a small in-memory catalog, this is appropriate.

For a large PostgreSQL-backed catalog, an equivalent SQL query may be preferable:

```sql
SELECT COUNT(*)
FROM courses
WHERE status = 'PUBLISHED'
  AND price = 0;
```

We'll revisit these choices when implementing Spring Data JPA.

### Immutable data versus mutable entities

Our `Course` is a Java record. Publishing it creates a new record with the changed status.

Later, we'll distinguish between:

- Immutable API DTOs.
- Domain objects that enforce business rules.
- JPA persistence entities that Hibernate manages.

Not every object should be a record, and not every object should be mutable.

### Concurrency and transactional correctness

Our in-memory repository is synchronized at the method level, but course creation still has a check-then-save race.

When we introduce PostgreSQL, we'll use database constraints, transactional services, and appropriate conflict handling. We should never assume that a preliminary existence check guarantees uniqueness.

### A useful design improvement for later

The current `CourseManagementService` copies every `Course` field to change its status. As the model grows, this becomes tedious.

In a later refactor, we could move the lifecycle transition into the domain model itself, for example:

```java
Course published = course.publish();
```

That would centralize transition rules in the domain object rather than spreading them across services. We'll evaluate that tradeoff when designing persistent entities.

## 15. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="workflow" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise A — Typed publication transitions**

      Add a method that checks whether a transition from one `CourseStatus` to another is allowed. Use a Java 21 `switch` expression and write tests for all status combinations.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="chart-no-axes-column-increasing" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise B — Category analytics**

      Add a method that groups published courses by category and returns a map of category names to course counts. Ensure the returned map cannot be modified.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="layers" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise C — Extend the generic Result type**

      Add `flatMap()` to `Result<T>` so successful operations can be chained without producing nested result objects. Write tests proving that failures bypass subsequent mapping functions.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="shield-check" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise D — Course creation constraints**

      Implement a repository operation that atomically inserts a course only when its ID is absent. Update `CourseManagementService` to use it, and test that duplicate creation is rejected.

      <text color="secondary" size="xs">Advanced extension: write a concurrent test and explain why the database still needs a unique constraint.</text>
    </box>
  </row>
</box>

## 16. Lesson 4 quiz

{@body const questions=[{q:"Which collection is most appropriate for mapping unique course IDs to Course objects?",options:["List<Course>","Map<Long, Course>","Queue<Course>","Deque<Course>"],correct:1,why:"A Map provides key-based lookup and enforces unique keys."},{q:"Why do we use BigDecimal.reduce instead of summing prices as doubles?",options:["It guarantees thread safety","It supports exact decimal arithmetic for monetary amounts","It automatically converts currencies","It eliminates the need for validation"],correct:1,why:"BigDecimal avoids binary floating-point representation errors in decimal calculations."},{q:"What does a sealed interface provide in Java 21?",options:["Automatic database persistence","A restricted set of permitted implementations","Automatic Spring dependency injection","Mutable record fields"],correct:1,why:"Sealed types restrict the implementation hierarchy and support exhaustive pattern matching."},{q:"Why is the current createDraft existence check insufficient for concurrent uniqueness?",options:["Records cannot be used in repositories","The check and save are separate operations","Java maps allow duplicate keys","JUnit cannot test concurrency"],correct:1,why:"Two concurrent operations may both pass the existence check before either saves. Atomic inserts or database constraints are needed."},{q:"Which statement about totalListedPrice is correct?",options:["It represents actual marketplace revenue","It is the sum of published course prices, not purchases","It includes archived course prices","It is a payment-provider balance"],correct:1,why:"Listed prices describe catalog offerings. Revenue requires verified purchases and financial records."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
{@body const score=questions.filter((q,i)=>answers[i]===q.correct).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Java 21 and backend design quiz**
    <text color="secondary" size="xs">{Object.keys(answers).length} of 5 answered</text>
  </row>
  {#each questions as q,i}
    <box gap={2} key={i}>
      <text weight="medium">{i+1}. {q.q}</text>
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setGraded(false)}}>
        {#each q.options as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if graded}
        <row align="center" gap={2}>
          <icon name={answers[i]===q.correct?"check-circle":"alert-circle"} color={answers[i]===q.correct?"success":"danger"}/>
          <text size="sm" weight="medium">{answers[i]===q.correct?"Correct":"Correct answer: "+q.options[q.correct]}</text>
        </row>
        <text color="secondary" size="sm">{q.why}</text>
      {/if}
    </box>
    {#if i<questions.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if graded}
    <box background="surface-secondary" radius="md" padding={3} gap={1}>
      <title size="xl" tabularNums>{score}/5</title>
      <text size="sm">{score===5?"All answers correct.":"Review the explanations, then try again."}</text>
    </box>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==5} onClick={()=>setGraded(true)}>Check answers</button>
    <button color="secondary" variant="outline" onClick={()=>{setAnswers({});setGraded(false)}}>Reset</button>
  </row>
</box>

## 17. Lesson completion checklist

{@body const groups=[{name:"Core Java",items:["Reviewed List, Set, Map, and EnumMap","Understand immutable collection boundaries","Used Stream, map, filter, reduce, and groupingBy","Understand generics, sealed interfaces, and pattern matching"]},{name:"Implementation",items:["Created three typed domain exceptions","Implemented CourseManagementService","Implemented CatalogSummary","Updated CourseCatalogService","Implemented generic Result<T>","Updated Main.java"]},{name:"Verification",items:["Added CourseManagementService tests","Added catalog summary tests","Added Result<T> tests","All 46 expected tests pass","Maven clean verify succeeds","Packaged application runs"]},{name:"Practice",items:["Completed at least two exercises","Reviewed concurrency and transaction limitations","Committed Lesson 4 changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((s,g)=>s+g.items.length,0)}
{@body const count=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    <box gap={1}>
      **Lesson 4 progress**
      <text color="secondary" size="xs">Check each milestone after local verification.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" width={`${100*count/total}%`} height="100%" radius="full"/>
  </box>
  {#each groups as group,i}
    <box gap={2} key={i}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!done[`${i}-${j}`]} onChange={v=>setDone(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if count===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 4 checklist completed.**
    </row>
  {/if}
</box>

## 18. Final verification and Git checkpoint

Run:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 4 verification failed."
}

java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar

if ($LASTEXITCODE -ne 0) {
    throw "LearnHub application failed."
}

git status
git add .
git commit -m "Add LearnHub course lifecycle, summaries and typed results"
```

## 19. Next lesson: Spring Boot begins

<row align="start" gap={3}>
  <box background="surface-secondary" radius="lg" padding={3}>
    <icon name="leaf" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 5 — Spring Boot Project Setup and Dependency Injection**

    We'll convert the existing Maven application into a Spring Boot project, introduce `@SpringBootApplication`, `@Service`, `@Repository`, configuration, bean registration, and Spring's dependency injection container.

    We'll preserve our domain model and existing business tests wherever possible, and start a local Spring Boot server that will eventually serve the Angular frontend.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 5: Spring Boot Project Setup, Spring Initializr, Maven configuration, @SpringBootApplication, @Service, @Repository, Dependency Injection, Application Context, application.yml, and first server startup. Continue from Lesson 4 with experienced-Java pacing, Windows PowerShell commands, exact file paths, complete code, tests, exercises, quiz, and checklist.")}>Start Lesson 5 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 4 deliverable:** A plain-Java LearnHub backend foundation with reusable collection operations, catalog analytics, typed domain exceptions, course lifecycle rules, generic result handling, and a comprehensive automated test suite.

You're now ready to transition from Java application design into Spring Boot development.