# LearnHub Java & Spring Boot Backend — Lesson 3

<text color="secondary" size="sm">Object-Oriented Programming, Interfaces, Encapsulation, Composition, SOLID Principles, and Clean Domain Design</text>

<row align="center" gap={2} wrap="wrap">
  <badge>Lesson 3 of 24</badge>
  <badge>Experienced Java Track</badge>
  <badge>Steady Pace</badge>
</row>

In Lesson 2, we implemented a validated `Course` record and a `CourseCatalog` supporting search, filtering, sorting, and pagination.

That code works, but our catalog currently has two responsibilities: **storing course data and implementing business queries**. As LearnHub grows, we'll need to retrieve courses from PostgreSQL, manage instructor-owned courses, and expose REST APIs without rewriting our business logic every time the storage mechanism changes.

In this lesson, we'll refactor LearnHub into a cleaner, testable object-oriented design using **interfaces, composition, dependency inversion, and explicit service boundaries**.

## Learning objectives

By the end of Lesson 3, you'll be able to:

- Apply encapsulation, abstraction, polymorphism, and composition to backend design.
- Use interfaces to separate business logic from data access.
- Apply the five SOLID principles to realistic LearnHub examples.
- Implement an in-memory repository without coupling services to storage.
- Introduce dependency injection manually, before using Spring.
- Test business logic using fake repositories.
- Prepare a structure that can evolve into Spring Boot services and Spring Data JPA repositories.

<text color="secondary" size="sm">Estimated time: 2–3 hours including exercises and tests. No Spring Boot dependencies are needed yet.</text>

## 1. Architecture: what changes in Lesson 3?

<box border radius="lg" padding={3} gap={2}>
  <text weight="medium">Lesson 2 — Current design</text>
  <box background="surface-secondary" padding={3} radius="md" align="center" gap={1}>
    **Main.java**
    <text color="secondary" size="xs">Creates courses and invokes catalog methods</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border padding={3} radius="md" align="center" gap={1}>
    **CourseCatalog**
    <text color="secondary" size="xs">Stores List<escape><Course></escape> + implements search and filtering</text>
  </box>
  <divider/>
  <text weight="medium">Lesson 3 — Target design</text>
  <box background="surface-secondary" padding={3} radius="md" align="center" gap={1}>
    **Main.java**
    <text color="secondary" size="xs">Composition root: creates and connects objects</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border padding={3} radius="md" align="center" gap={1}>
    **CourseCatalogService**
    <text color="secondary" size="xs">Business queries and catalog rules</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border padding={3} radius="md" align="center" gap={1}>
    **CourseRepository interface**
    <text color="secondary" size="xs">Storage-independent contract</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" padding={3} radius="md" align="center" gap={1}>
    **InMemoryCourseRepository**
    <text color="secondary" size="xs">Current implementation; PostgreSQL implementation later</text>
  </box>
</box>

The important change is that the service depends on an **abstraction**, not a particular storage implementation.

## 2. Prepare your existing Maven project

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version

mvn clean verify
```

Your Lesson 2 project should have 15 passing tests if you used the provided test files.

Create a Git checkpoint:

```powershell
git status
git add .
git commit -m "Checkpoint LearnHub backend lesson 2"
```

If Git reports nothing to commit, continue.

### Create new directories

```powershell
New-Item -ItemType Directory -Force `
    src\main\java\com\learnhub\repository

New-Item -ItemType Directory -Force `
    src\main\java\com\learnhub\service

New-Item -ItemType Directory -Force `
    src\main\java\com\learnhub\data

New-Item -ItemType Directory -Force `
    src\test\java\com\learnhub\repository

New-Item -ItemType Directory -Force `
    src\test\java\com\learnhub\service
```

Our target structure:

```text
learnhub-backend/
├── pom.xml
├── src/
│   ├── main/java/com/learnhub/
│   │   ├── Main.java
│   │   ├── model/
│   │   │   ├── Course.java
│   │   │   ├── CourseLevel.java
│   │   │   └── CourseStatus.java
│   │   ├── catalog/
│   │   │   ├── CourseCatalog.java
│   │   │   ├── CourseFilter.java
│   │   │   ├── CourseSort.java
│   │   │   └── PageResult.java
│   │   ├── repository/
│   │   │   ├── CourseRepository.java
│   │   │   └── InMemoryCourseRepository.java
│   │   ├── service/
│   │   │   └── CourseCatalogService.java
│   │   └── data/
│   │       └── SampleCourses.java
│   └── test/java/com/learnhub/
│       ├── model/CourseTest.java
│       ├── catalog/CourseCatalogTest.java
│       ├── repository/InMemoryCourseRepositoryTest.java
│       └── service/CourseCatalogServiceTest.java
└── .gitignore
```

We'll preserve the existing Lesson 2 `CourseCatalog` and its tests. That class will remain the reusable **pure query engine**, while a new service coordinates repository access.

This avoids unnecessarily rewriting code that already works.

## 3. OOP principles in backend development

### Encapsulation

Encapsulation means an object controls access to its internal state and preserves its invariants.

For example, `CourseCatalog` already makes a defensive copy:

```java
this.courses = List.copyOf(courses);
```

Callers cannot modify the catalog's internal list through the original collection.

Similarly, our `Course` record validates its fields in the compact constructor.

### Abstraction

Abstraction exposes what an operation does without requiring callers to know how it works.

For example:

```java
repository.findAll();
```

The caller does not need to know whether courses come from a list, PostgreSQL, or a remote service.

### Polymorphism

Multiple implementations can satisfy the same interface.

```java
CourseRepository repository =
        new InMemoryCourseRepository();
```

Later, another implementation can use a database while exposing the same contract.

### Composition over inheritance

Composition connects objects through fields and constructor parameters.

```java
public CourseCatalogService(CourseRepository repository) {
    this.repository = repository;
}
```

This is usually preferable to creating deep inheritance hierarchies such as `BaseCatalog`, `DatabaseCatalog`, `SecureDatabaseCatalog`, and so on.

Inheritance is useful when there is a genuine subtype relationship, but it should not be the default reuse mechanism.

## 4. Create the CourseRepository interface

File: `src/main/java/com/learnhub/repository/CourseRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.repository;

import com.learnhub.model.Course;

import java.util.List;
import java.util.Optional;

/**
 * Storage abstraction for LearnHub courses.
 *
 * This interface defines WHAT operations are available,
 * not HOW data is stored.
 *
 * It deliberately has no Spring or database dependencies.
 */
public interface CourseRepository {

    /**
     * Returns a snapshot of all stored courses,
     * including drafts and archived courses.
     *
     * Authorization is not performed here.
     */
    List<Course> findAll();

    /**
     * Finds a stored course by its unique identifier.
     */
    Optional<Course> findById(long id);

    /**
     * Inserts a new course or replaces an existing one.
     *
     * Returns the saved course.
     */
    Course save(Course course);

    /**
     * Deletes a course by ID.
     *
     * Returns true if a course was removed.
     */
    boolean deleteById(long id);
}
</CodeBlock>

### Why not call this `JpaCourseRepository`?

Because we're not using JPA yet.

The interface belongs to the application's own architecture. The in-memory implementation will satisfy it now; a persistence adapter can satisfy it later.

Also note that `findAll()` includes drafts and archived courses. This is intentional: the repository is a storage abstraction, not a public authorization boundary.

**Never expose unrestricted repository results directly to public API endpoints.** The service or application use case must enforce visibility and authorization.

## 5. Implement the in-memory repository

File: `src/main/java/com/learnhub/repository/InMemoryCourseRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.repository;

import com.learnhub.model.Course;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;

/**
 * In-memory CourseRepository implementation.
 *
 * Suitable for learning and deterministic unit tests.
 *
 * This implementation is not durable: data disappears
 * when the application exits.
 *
 * Methods are synchronized to keep individual repository
 * operations consistent when accessed concurrently.
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
        Objects.requireNonNull(
                initialCourses,
                "Initial courses cannot be null."
        );

        for (Course course : initialCourses) {
            Objects.requireNonNull(
                    course,
                    "Initial courses cannot contain null."
            );

            if (courses.putIfAbsent(course.id(), course) != null) {
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

        if (id <= 0) {
            return Optional.empty();
        }

        return Optional.ofNullable(courses.get(id));
    }

    @Override
    public synchronized Course save(Course course) {

        Objects.requireNonNull(
                course,
                "Course cannot be null."
        );

        courses.put(course.id(), course);

        return course;
    }

    @Override
    public synchronized boolean deleteById(long id) {

        return courses.remove(id) != null;
    }
}
</CodeBlock>

### Engineering notes

- `LinkedHashMap` stores courses by ID and prevents duplicate keys.
- `findAll()` sorts by ID, so results are deterministic.
- `synchronized` protects each individual repository method from concurrent access.
- `save()` has upsert behavior: it inserts or replaces a course with the same ID.
- The repository does not know whether the caller is a student, instructor, or administrator.

The `synchronized` methods do **not** make a sequence of operations across multiple methods transactional. Later, database transactions will handle multi-step consistency.

## 6. Create reusable sample data

In Lesson 2, `Main.java` contained all sample courses. Let's move them to a dedicated class.

File: `src/main/java/com/learnhub/data/SampleCourses.java`

<CodeBlock language="java" editable>
package com.learnhub.data;

import com.learnhub.model.Course;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;

import java.math.BigDecimal;
import java.util.List;

/**
 * Sample data for local development and exercises.
 *
 * This is not production seed data or a database migration.
 */
public final class SampleCourses {

    private SampleCourses() {
    }

    public static List<Course> create() {

        return List.of(
                new Course(
                        1L,
                        "Angular Complete Guide",
                        "Build modern Angular applications.",
                        "Alex Morgan",
                        "Web Development",
                        new BigDecimal("49.99"),
                        false,
                        CourseLevel.BEGINNER,
                        new BigDecimal("4.8"),
                        CourseStatus.PUBLISHED
                ),
                new Course(
                        2L,
                        "Python for Beginners",
                        "Learn programming with Python.",
                        "Sarah Lee",
                        "Programming",
                        BigDecimal.ZERO,
                        true,
                        CourseLevel.BEGINNER,
                        new BigDecimal("4.7"),
                        CourseStatus.PUBLISHED
                ),
                new Course(
                        3L,
                        "UI/UX Design Masterclass",
                        "Learn user experience and interface design.",
                        "Emma Wilson",
                        "Design",
                        new BigDecimal("29.99"),
                        false,
                        CourseLevel.INTERMEDIATE,
                        new BigDecimal("4.9"),
                        CourseStatus.PUBLISHED
                ),
                new Course(
                        4L,
                        "Data Science Essentials",
                        "Explore data analysis fundamentals.",
                        "David Chen",
                        "Data Science",
                        BigDecimal.ZERO,
                        true,
                        CourseLevel.BEGINNER,
                        new BigDecimal("4.6"),
                        CourseStatus.PUBLISHED
                ),
                new Course(
                        5L,
                        "Advanced Spring Boot",
                        "Create enterprise Java applications.",
                        "Alex Morgan",
                        "Web Development",
                        new BigDecimal("79.99"),
                        false,
                        CourseLevel.ADVANCED,
                        new BigDecimal("4.8"),
                        CourseStatus.DRAFT
                )
        );
    }
}
</CodeBlock>

The sample data now has one clear responsibility: constructing a predictable set of course objects.

This is a small application of the **Single Responsibility Principle**.

## 7. Implement CourseCatalogService

Our existing `CourseCatalog` already implements the query logic. Instead of duplicating those filters, the new service will retrieve data through the repository and delegate queries to `CourseCatalog`.

File: `src/main/java/com/learnhub/service/CourseCatalogService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.catalog.CourseCatalog;
import com.learnhub.catalog.CourseFilter;
import com.learnhub.catalog.CourseSort;
import com.learnhub.catalog.PageResult;
import com.learnhub.model.Course;
import com.learnhub.repository.CourseRepository;

import java.util.Objects;
import java.util.Optional;

/**
 * Application service for public course discovery.
 *
 * Depends on the CourseRepository interface,
 * not on a particular storage implementation.
 *
 * A Spring @Service annotation will be introduced
 * when we migrate to Spring Boot.
 */
public final class CourseCatalogService {
    private final CourseRepository repository;

    /**
     * Constructor injection.
     *
     * The caller provides the dependency, making this
     * class independent of the repository implementation.
     */
    public CourseCatalogService(
            CourseRepository repository
    ) {
        this.repository = Objects.requireNonNull(
                repository,
                "Course repository is required."
        );
    }

    /**
     * Returns only published courses.
     *
     * Delegates filtering, sorting, and pagination
     * to the existing Lesson 2 query engine.
     */
    public PageResult<Course> search(
            CourseFilter filter,
            CourseSort sort,
            int pageNumber,
            int pageSize
    ) {

        CourseCatalog catalog = new CourseCatalog(
                repository.findAll()
        );

        return catalog.search(
                filter,
                sort,
                pageNumber,
                pageSize
        );
    }

    /**
     * Returns a published course if it exists.
     *
     * Draft and archived courses are hidden.
     */
    public Optional<Course> findPublishedById(long id) {

        return repository.findById(id)
                .filter(Course::isPubliclyVisible);
    }
}
</CodeBlock>

### What have we achieved?

`CourseCatalogService` knows **what a public catalog operation means**, but not **where the courses are stored**.

The dependency direction is:

```text
CourseCatalogService
        |
        v
CourseRepository (interface)
        ^
        |
InMemoryCourseRepository
```

The service does not construct `InMemoryCourseRepository` internally. That is what makes it easy to test and replace.

### Important limitation

This implementation still calls `repository.findAll()` and filters in memory. That's intentional for Lesson 3.

When we introduce PostgreSQL and Spring Data JPA, we'll move filtering and pagination into database queries. The public service contract can remain similar, but the implementation will need to change.

## 8. Manual dependency injection in Main.java

Before learning Spring's dependency injection container, it's useful to see exactly what dependency injection looks like without a framework.

Replace:

`src/main/java/com/learnhub/Main.java`

<CodeBlock language="java" editable>
package com.learnhub;

import com.learnhub.catalog.CourseFilter;
import com.learnhub.catalog.CourseSort;
import com.learnhub.catalog.PageResult;
import com.learnhub.data.SampleCourses;
import com.learnhub.model.Course;
import com.learnhub.repository.CourseRepository;
import com.learnhub.repository.InMemoryCourseRepository;
import com.learnhub.service.CourseCatalogService;

/**
 * LearnHub console application.
 *
 * Main is the composition root: the place where
 * concrete objects are created and connected.
 */
public final class Main {

    private Main() {
    }

    public static void main(String[] args) {

        // 1. Create the concrete infrastructure adapter.
        CourseRepository repository =
                new InMemoryCourseRepository(
                        SampleCourses.create()
                );

        // 2. Inject the abstraction into the service.
        CourseCatalogService catalogService =
                new CourseCatalogService(repository);

        // 3. Execute an application use case.
        PageResult<Course> publishedCourses =
                catalogService.search(
                        CourseFilter.all(),
                        CourseSort.DEFAULT,
                        0,
                        10
                );

        System.out.println("==============================");
        System.out.println("     LearnHub Backend");
        System.out.println("==============================");

        System.out.println("\nPublished courses:");

        for (Course course : publishedCourses.items()) {
            System.out.printf(
                    "#%d | %s | %s | %s%n",
                    course.id(),
                    course.title(),
                    course.instructor(),
                    course.priceLabel()
            );
        }

        System.out.println(
                "\nTotal published courses: "
                        + publishedCourses.totalItems()
        );

        // 4. Verify draft visibility rules.
        System.out.println(
                "Is draft course #5 publicly visible? "
                        + catalogService
                                .findPublishedById(5L)
                                .isPresent()
        );

        // 5. Demonstrate a repository operation.
        System.out.println(
                "Total stored courses: "
                        + repository.findAll().size()
        );
    }
}
</CodeBlock>

Notice that `Main` constructs both the repository and the service.

Later, Spring Boot will take over this responsibility through its application context and dependency injection container.

### Expected output

```text
==============================
     LearnHub Backend
==============================

Published courses:
#1 | Angular Complete Guide | Alex Morgan | $49.99
#2 | Python for Beginners | Sarah Lee | FREE
#3 | UI/UX Design Masterclass | Emma Wilson | $29.99
#4 | Data Science Essentials | David Chen | FREE

Total published courses: 4
Is draft course #5 publicly visible? false
Total stored courses: 5
```

## 9. Understand the SOLID principles

SOLID is a set of five design principles that help make object-oriented software easier to maintain and extend. They are guidelines, not rules requiring an interface for every class.

<box gap={3}>
  <box border radius="lg" padding={3} gap={2}>
    <row align="center" gap={2}>
      <box background="surface-secondary" radius="md" padding={2}><title size="lg">S</title></box>
      **Single Responsibility Principle**
    </row>
    A module should have one coherent reason to change.

    In LearnHub, `Course` enforces course invariants, `CourseCatalog` performs catalog queries, and `InMemoryCourseRepository` handles storage. These responsibilities are separated instead of being combined in one large class.
  </box>
  <box border radius="lg" padding={3} gap={2}>
    <row align="center" gap={2}>
      <box background="surface-secondary" radius="md" padding={2}><title size="lg">O</title></box>
      **Open/Closed Principle**
    </row>
    Software should be extensible without repeatedly modifying stable consumers.

    We can add a new repository implementation without rewriting `CourseCatalogService`, provided it honors the same repository contract.
  </box>
  <box border radius="lg" padding={3} gap={2}>
    <row align="center" gap={2}>
      <box background="surface-secondary" radius="md" padding={2}><title size="lg">L</title></box>
      **Liskov Substitution Principle**
    </row>
    An implementation must respect the expectations of its abstraction.

    If `CourseRepository.findById()` promises `Optional.empty()` for a missing ID, an implementation should not unexpectedly throw an exception for an ordinary missing course.
  </box>
  <box border radius="lg" padding={3} gap={2}>
    <row align="center" gap={2}>
      <box background="surface-secondary" radius="md" padding={2}><title size="lg">I</title></box>
      **Interface Segregation Principle**
    </row>
    Clients should not depend on operations they don't need.

    Our repository is small, but later we might separate public read operations from administrative writes. A course-search service should not need access to payment operations or user-password management.
  </box>
  <box border radius="lg" padding={3} gap={2}>
    <row align="center" gap={2}>
      <box background="surface-secondary" radius="md" padding={2}><title size="lg">D</title></box>
      **Dependency Inversion Principle**
    </row>
    High-level business logic should depend on abstractions rather than concrete infrastructure.

    `CourseCatalogService` depends on `CourseRepository`, not `InMemoryCourseRepository`. The actual implementation is supplied by the composition root.
  </box>
</box>

### A common misunderstanding

SOLID does not mean every class must have an interface, every method must be generic, or every feature needs multiple abstraction layers.

Our `CourseCatalog` remains a concrete class because it is already a focused, testable query engine. An interface would add little value at this stage.

## 10. Test the in-memory repository

Create:

`src/test/java/com/learnhub/repository/InMemoryCourseRepositoryTest.java`

<CodeBlock language="java" editable>
package com.learnhub.repository;

import com.learnhub.data.SampleCourses;
import com.learnhub.model.Course;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests repository behavior independently
 * of the catalog service.
 */
class InMemoryCourseRepositoryTest {

    private InMemoryCourseRepository repository;

    @BeforeEach
    void setUp() {
        repository = new InMemoryCourseRepository(
                SampleCourses.create()
        );
    }

    @Test
    void findsExistingCourseById() {
        assertEquals(
                "Angular Complete Guide",
                repository.findById(1L)
                        .orElseThrow()
                        .title()
        );
    }

    @Test
    void returnsEmptyForUnknownId() {
        assertTrue(
                repository.findById(999L).isEmpty()
        );
    }

    @Test
    void returnsAllStoredCoursesIncludingDrafts() {
        assertEquals(5, repository.findAll().size());
    }

    @Test
    void deletesExistingCourse() {
        assertTrue(repository.deleteById(2L));
        assertTrue(repository.findById(2L).isEmpty());
        assertEquals(4, repository.findAll().size());
    }

    @Test
    void deletingUnknownCourseReturnsFalse() {
        assertFalse(repository.deleteById(999L));
    }

    @Test
    void savesNewCourse() {
        Course newCourse = SampleCourses.create().getFirst();

        CourseRepository emptyRepository =
                new InMemoryCourseRepository();

        emptyRepository.save(newCourse);

        assertEquals(
                newCourse,
                emptyRepository.findById(newCourse.id())
                        .orElseThrow()
        );
    }

    @Test
    void rejectsDuplicateInitialIds() {
        Course course = SampleCourses.create().getFirst();

        assertThrows(
                IllegalArgumentException.class,
                () -> new InMemoryCourseRepository(
                        List.of(course, course)
                )
        );
    }
}
</CodeBlock>

This verifies storage behavior independently from search and public visibility rules.

## 11. Test the service with a fake repository

The major advantage of dependency inversion is that tests can replace real infrastructure with controlled fakes.

We'll use a small fake implementation of `CourseRepository` inside the test class.

File: `src/test/java/com/learnhub/service/CourseCatalogServiceTest.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.catalog.CourseFilter;
import com.learnhub.catalog.CourseSort;
import com.learnhub.data.SampleCourses;
import com.learnhub.model.Course;
import com.learnhub.repository.CourseRepository;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests service behavior against an abstraction,
 * not a real database or HTTP server.
 */
class CourseCatalogServiceTest {

    /**
     * Minimal fake repository.
     *
     * Writes are intentionally unsupported because
     * these tests only exercise public read operations.
     */
    private static final class FakeCourseRepository
            implements CourseRepository {

        private final List<Course> courses;

        FakeCourseRepository(List<Course> courses) {
            this.courses = List.copyOf(courses);
        }

        @Override
        public List<Course> findAll() {
            return courses;
        }

        @Override
        public Optional<Course> findById(long id) {
            return courses.stream()
                    .filter(course -> course.id() == id)
                    .findFirst();
        }

        @Override
        public Course save(Course course) {
            throw new UnsupportedOperationException(
                    "Not required by these read-only tests."
            );
        }

        @Override
        public boolean deleteById(long id) {
            throw new UnsupportedOperationException(
                    "Not required by these read-only tests."
            );
        }
    }

    private CourseCatalogService createService() {
        return new CourseCatalogService(
                new FakeCourseRepository(
                        SampleCourses.create()
                )
        );
    }

    @Test
    void returnsOnlyPublishedCourses() {
        CourseCatalogService service = createService();

        var result = service.search(
                CourseFilter.all(),
                CourseSort.DEFAULT,
                0,
                10
        );

        assertEquals(4, result.totalItems());
        assertTrue(
                result.items().stream()
                        .allMatch(Course::isPubliclyVisible)
        );
    }

    @Test
    void hidesDraftCourseById() {
        CourseCatalogService service = createService();

        assertTrue(
                service.findPublishedById(5L).isEmpty()
        );
    }

    @Test
    void findsPublishedCourseById() {
        CourseCatalogService service = createService();

        assertEquals(
                "Python for Beginners",
                service.findPublishedById(2L)
                        .orElseThrow()
                        .title()
        );
    }

    @Test
    void filtersFreeCourses() {
        CourseCatalogService service = createService();

        var result = service.search(
                new CourseFilter("", null, true, null),
                CourseSort.DEFAULT,
                0,
                10
        );

        assertEquals(2, result.totalItems());
        assertTrue(
                result.items().stream().allMatch(Course::free)
        );
    }

    @Test
    void rejectsMissingRepositoryDependency() {
        assertThrows(
                NullPointerException.class,
                () -> new CourseCatalogService(null)
        );
    }
}
</CodeBlock>

### Why use a fake instead of Mockito?

Mockito is valuable, and we'll use it later. But writing a fake repository manually makes the interface boundary visible and demonstrates that dependency injection does not require a framework.

The fake deliberately throws for write methods because this service does not need them. That's acceptable for a narrowly scoped test double, but it also reveals a design question: **should public catalog reading have its own smaller interface?**
We'll explore that in the exercises.

## 12. Run all tests and package the application

Return to PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean test
```

Expected test count:

<box border radius="lg" padding={3} gap={2}>
  <row justify="between" align="center">
    <box gap={1}>
      <text color="secondary" size="xs">Expected total tests</text>
      <title size="2xl" tabularNums>27</title>
    </box>
    <icon name="test-tube-diagonal" color="secondary" size="2xl"/>
  </row>
  <grid columns={2} gap={2}>
    <grid-item>
      <text weight="medium">15</text>
      <text color="secondary" size="xs">Previous lesson tests</text>
    </grid-item>
    <grid-item>
      <text weight="medium">12</text>
      <text color="secondary" size="xs">New repository and service tests</text>
    </grid-item>
  </grid>
  <caption>This is the expected count for the exact test files supplied across Lessons 2 and 3, not a claim that your local tests have already run.</caption>
</box>

Package and run:

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 3 Maven verification failed."
}

java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar

if ($LASTEXITCODE -ne 0) {
    throw "LearnHub application execution failed."
}
```

You should see four published courses, while the repository reports five stored courses.

This confirms the important separation between **storage visibility** and **public catalog visibility**.

## 13. Advanced design review: preparing for Spring Boot

Our refactoring gives us a clean foundation, but it is worth examining what we would change for a production system.

### A. Separate read and write interfaces

Our `CourseRepository` currently includes both query and mutation operations. The catalog service only needs read access.

An optional refinement is:

<CodeBlock language="java" editable>
package com.learnhub.repository;

import com.learnhub.model.Course;

import java.util.List;
import java.util.Optional;

/**
 * Read-only contract for course discovery.
 */
public interface CourseReader {

    List<Course> findAll();

    Optional<Course> findById(long id);
}
</CodeBlock>

Then:

```java
public interface CourseRepository extends CourseReader {
    Course save(Course course);
    boolean deleteById(long id);
}
```

And the catalog service could depend on `CourseReader` instead.

This follows the Interface Segregation Principle, although it isn't mandatory for our current small project. If you implement this refinement, update the fake and constructor types consistently.

### B. Avoid leaking database entities into public responses

Right now, our service returns `Course` records. Later, we'll introduce dedicated DTOs such as:

```java
public record CourseResponse(
        long id,
        String title,
        String instructor,
        String category,
        String price,
        boolean free
) {}
```

This helps avoid exposing private instructor information, internal persistence fields, or implementation details in API responses.

### C. Keep authorization out of repositories

The future instructor dashboard will need rules such as:

- Only an authenticated instructor may create courses.
- An instructor may edit only their own courses.
- Students may see only published courses.
- Paid lessons require a verified entitlement.

These rules belong in trusted application services and security policies, supported by appropriately scoped database queries. They should not be enforced solely through Angular route guards or client-supplied IDs.

### D. Know when in-memory design stops scaling

Our current `CourseCatalogService` creates a new catalog from `repository.findAll()` on every search. This is simple and correct for a small exercise.

For production, we'll replace it with repository queries that perform filtering, sorting, and pagination in PostgreSQL, backed by appropriate indexes.

### E. Consider atomicity

Our repository uses `synchronized` for individual methods. That does not guarantee atomicity across multi-step operations.

For example, two operations that separately check whether a course exists and then save a course may still race. Database transactions and unique constraints will matter when we introduce persistence.

## 14. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}><icon name="split" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise A — Interface Segregation**

      Implement the optional `CourseReader` interface above. Refactor `CourseCatalogService` to depend on it and confirm all tests still pass.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}><icon name="shield-check" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise B — Course publishing rules**

      Create a `CoursePublishingService` that can transition a course from `DRAFT` to `PUBLISHED`, but rejects publishing an already archived course. Define clear transition rules and unit tests. For this exercise, don't implement authentication yet.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}><icon name="database" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise C — Repository contract tests**

      Write a reusable set of tests for `CourseRepository` behavior: saving, replacing, finding, deleting, and handling unknown IDs. Think about how the same contract tests could later be applied to a PostgreSQL-backed implementation.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}><icon name="boxes" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise D — Constructor injection**

      Create an alternative `LoggingCourseReader` decorator that wraps a `CourseReader`, logs query operations, and delegates to the underlying implementation without changing the service. Consider how you'd avoid logging sensitive data in production.
    </box>
  </row>
</box>

## 15. Lesson 3 quiz

{@body const questions=[{q:"What is the main purpose of dependency inversion in our LearnHub architecture?",options:["Make every class inherit from a base class","Allow high-level services to depend on abstractions rather than storage details","Eliminate the need for tests","Move business logic into Main.java"],answer:1,why:"CourseCatalogService depends on CourseRepository, allowing different implementations without coupling the service to in-memory storage."},{q:"Which class should enforce public course visibility in the current design?",options:["The console printer","The repository implementation exclusively","The application service or delegated catalog query logic","The Angular component alone"],answer:2,why:"Public visibility is a business rule. The repository stores all course states, while the service ensures public operations expose only published courses."},{q:"What is a composition root?",options:["The place where application dependencies are created and wired together","A superclass for all domain objects","A Java garbage collector","A Maven dependency scope"],answer:0,why:"Main.java creates the concrete repository and injects it into the service. Spring's application context will later handle wiring."},{q:"Which principle suggests that a read-only service should not depend on unrelated write operations?",options:["Liskov Substitution","Interface Segregation","Single Responsibility only","Encapsulation only"],answer:1,why:"Interface Segregation favors focused contracts such as CourseReader when that distinction adds useful clarity."},{q:"Does synchronized on individual repository methods make a multi-step operation transactional?",options:["Yes, always","Only when using records","No, multi-step operations need their own atomicity strategy","Yes, when using LinkedHashMap"],answer:2,why:"Individual method synchronization does not automatically make sequences of method calls atomic. Transactions and database constraints address those concerns later."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
{@body const score=questions.filter((q,i)=>answers[i]===q.answer).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Knowledge check**
    <text color="secondary" size="xs">{Object.keys(answers).length} of {questions.length} answered</text>
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
        <box gap={1}>
          <row align="center" gap={2}>
            <icon name={answers[i]===q.answer?"check-circle":"alert-circle"} color={answers[i]===q.answer?"success":"danger"}/>
            <text size="sm" weight="medium">{answers[i]===q.answer?"Correct":"Correct answer: "+q.options[q.answer]}</text>
          </row>
          <text color="secondary" size="sm">{q.why}</text>
        </box>
      {/if}
    </box>
    {#if i<questions.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if graded}
    <box background="surface-secondary" padding={3} radius="md" gap={1}>
      <title size="xl" tabularNums>{score}/{questions.length}</title>
      <text size="sm">{score===questions.length?"Excellent — all concepts correct.":"Review the explanations and retry any missed concepts."}</text>
    </box>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==questions.length} onClick={()=>setGraded(true)}>Check answers</button>
    <button color="secondary" variant="outline" onClick={()=>{setAnswers({});setGraded(false)}}>Reset</button>
  </row>
</box>

## 16. Lesson completion checklist

{@body const tasks=[{group:"Architecture",items:["Understand encapsulation, abstraction, polymorphism, and composition","Can explain all five SOLID principles","Understand constructor injection and the composition root"]},{group:"Implementation",items:["Created CourseRepository interface","Created InMemoryCourseRepository","Created SampleCourses","Created CourseCatalogService","Updated Main.java to inject dependencies"]},{group:"Testing",items:["Repository tests pass","Service tests pass using a fake repository","All 27 expected tests pass","Maven package and executable JAR work"]},{group:"Practice",items:["Completed at least two exercises","Reviewed public visibility and authorization boundaries","Committed Lesson 3 changes"]}]}
{@body const [completed,setCompleted]=DIL.useState({})}
{@body const total=tasks.reduce((sum,g)=>sum+g.items.length,0)}
{@body const count=Object.values(completed).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 3 progress**
      <text color="secondary" size="xs">Mark tasks after verifying them locally.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${100*count/total}%`} radius="full"/>
  </box>
  {#each tasks as group,i}
    <box gap={2} key={i}>
      **{group.group}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!completed[`${i}-${j}`]} onChange={value=>setCompleted(old=>({...old,[`${i}-${j}`]:value}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if count===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 3 checklist completed.**
    </row>
  {/if}
</box>

## 17. Final verification and Git checkpoint

Run the following commands in Windows PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 3 tests or build failed."
}

java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar

if ($LASTEXITCODE -ne 0) {
    throw "LearnHub application failed."
}

git status

git add .
git commit -m "Refactor LearnHub with repository interfaces and services"
```

A successful verification should leave you with a runnable console application, the original catalog query tests, and new repository/service tests.

## 18. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" radius="lg" padding={3}>
    <icon name="workflow" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 4 — Collections, Streams, Generics, Exceptions, and Advanced Java Practices**

    We'll build on the repository and service architecture by introducing typed domain exceptions, more expressive collection operations, Java 21 stream patterns, immutable data boundaries, and more comprehensive testing.

    We'll also prepare the codebase for the transition to Spring Boot in Lesson 5.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 4: Collections, Streams, Generics, Exceptions, and Advanced Java Practices. Continue from Lesson 3 with experienced-Java pacing, Windows PowerShell commands, exact file paths, complete documented Java code, tests, exercises, quiz, and checklist.")}>Start Lesson 4 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 3 deliverable:** A modular LearnHub Java application with a storage-independent repository interface, an in-memory adapter, a catalog application service, constructor-based dependency injection, and testable business logic. This architecture is ready to evolve into Spring Boot services and persistence adapters.