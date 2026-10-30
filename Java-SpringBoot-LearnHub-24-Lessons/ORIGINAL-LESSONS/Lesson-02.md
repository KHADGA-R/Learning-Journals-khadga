# LearnHub Java & Spring Boot Backend — Lesson 2

<text color="secondary" size="sm">Java Fundamentals for Backend Development: Domain Modeling, Validation, Control Flow, BigDecimal, and Collections</text>

<row align="center" gap={2} wrap="wrap">
  <badge>Lesson 2 of 24</badge>
  <badge>Experienced Java Track</badge>
  <badge>Steady Pace</badge>
</row>

In Lesson 1, we built a Java 21 Maven project with a `Course` record, a console catalog, and JUnit tests.

In this lesson, we'll turn that small program into a **typed, validated course catalog with search, filtering, pricing rules, and sorting**. These are the same operations that will eventually power the LearnHub REST API.

Because you already have Java experience, we'll review fundamentals through backend engineering decisions rather than basic syntax drills.

<box border radius="lg" padding={3} gap={2}>
  **Learning objectives**

  By the end, you'll be able to:

  - Model course levels and statuses with enums.
  - Apply Java 21 records and compact constructors to domain validation.
  - Use `BigDecimal` correctly for prices and discounts.
  - Implement safe search, filtering, sorting, and pagination.
  - Use `Optional` for lookups and exceptions for invalid operations.
  - Write parameterized and behavior-focused JUnit tests.
  - Prepare a reusable catalog service for later Spring Boot integration.
</box>

<text color="secondary" size="sm">Estimated time: 90–120 minutes, plus optional exercises.</text>

## 1. How this lesson fits into the backend

<box border radius="lg" padding={3} gap={2}>
  <box background="surface-secondary" radius="md" padding={3} gap={1} align="center">
    <icon name="monitor" color="secondary"/>
    **Angular LearnHub**
    <text color="secondary" size="xs">Search • Categories • Free/Paid • Sort • Course Details</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="md" padding={3} gap={1} align="center">
    <icon name="code-2" color="secondary"/>
    **Java Catalog Logic — Lesson 2**
    <text color="secondary" size="xs">Validation • Filtering • Sorting • Pagination • Pricing</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" radius="md" padding={3} gap={1} align="center">
    <icon name="database" color="secondary"/>
    **In-memory Course List**
    <text color="secondary" size="xs">Later replaced by PostgreSQL and Spring Data JPA</text>
  </box>
</box>

The key architectural principle is to **keep business rules independent of HTTP controllers and databases**. Today, we will implement them in plain Java. Later, Spring Boot will expose them through API endpoints.

## 2. Prepare the Lesson 1 project

Open Windows PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version

mvn clean verify
```

Expected environment:

- Java 21.
- Maven 3.9.x.
- Successful build.
- Five passing tests from Lesson 1.

If the build fails, fix it before proceeding. This lesson assumes the original Lesson 1 project is working.

### Create a lesson checkpoint

If you're using Git:

```powershell
git status

git add .
git commit -m "Complete LearnHub backend lesson 1"
```

If there are no new changes, Git may report nothing to commit. That's fine.

We'll now extend the existing Maven project rather than create a new one.

## 3. Review the Java concepts we'll use

| Java concept | Backend application |
|---|---|
| `enum` | Restrict course levels and publication states |
| `record` | Immutable domain data and future DTOs |
| Compact constructor | Reject invalid data at creation time |
| `BigDecimal` | Exact decimal monetary calculations |
| `Optional<T>` | Represent missing lookup results |
| `Stream<T>` | In-memory search, filtering, and sorting |
| `Comparator<T>` | Define stable sort orders |
| `switch` expression | Select behavior by enum |
| Exceptions | Reject invalid operations |
| JUnit | Verify business rules |

One important distinction: these are **in-memory operations**. When the catalog grows, we'll move filtering, sorting, and pagination into database queries rather than loading every course into Java memory.

## 4. Expand the project structure

We'll keep the existing `Main.java` and replace the Lesson 1 `Course.java` with a richer model.

Create new directories:

```powershell
cd $HOME\angular-learning\learnhub-backend

New-Item -ItemType Directory -Force `
    src\main\java\com\learnhub\catalog

New-Item -ItemType Directory -Force `
    src\main\java\com\learnhub\model

New-Item -ItemType Directory -Force `
    src\test\java\com\learnhub\catalog
```

Target structure:

```text
learnhub-backend/
├── pom.xml
├── src/
│   ├── main/
│   │   └── java/com/learnhub/
│   │       ├── Main.java
│   │       ├── model/
│   │       │   ├── Course.java
│   │       │   ├── CourseLevel.java
│   │       │   └── CourseStatus.java
│   │       └── catalog/
│   │           ├── CourseCatalog.java
│   │           ├── CourseFilter.java
│   │           ├── CourseSort.java
│   │           └── PageResult.java
│   └── test/
│       └── java/com/learnhub/
│           ├── model/
│           │   └── CourseTest.java
│           └── catalog/
│               └── CourseCatalogTest.java
└── .gitignore
```

## 5. Model course levels and publication statuses

### Step 5.1 — CourseLevel enum

File: `src/main/java/com/learnhub/model/CourseLevel.java`

<CodeBlock language="java" editable>
package com.learnhub.model;

/**
 * Defines the supported difficulty levels.
 *
 * Enums prevent arbitrary values such as
 * "beginer", "expert-ish", or "level 7".
 */
public enum CourseLevel {
    BEGINNER,
    INTERMEDIATE,
    ADVANCED
}
</CodeBlock>

### Step 5.2 — CourseStatus enum

File: `src/main/java/com/learnhub/model/CourseStatus.java`

<CodeBlock language="java" editable>
package com.learnhub.model;

/**
 * Represents a course's publication lifecycle.
 *
 * DRAFT courses are not visible in the public catalog.
 * PUBLISHED courses are visible to learners.
 * ARCHIVED courses are excluded from public discovery.
 */
public enum CourseStatus {
    DRAFT,
    PUBLISHED,
    ARCHIVED
}
</CodeBlock>

These enums will become especially useful in the instructor dashboard and course publishing lessons.

### Why enums instead of strings?

Compare:

```java
String level = "begginer"; // Typo compiles successfully.
```

with:

```java
CourseLevel level = CourseLevel.BEGINNER;
```

The enum provides compile-time checking and a finite set of supported values.

When we later accept strings from HTTP requests, Spring will need to convert and validate those values. We'll also decide how to map backend enum names to the existing Angular model.

## 6. Upgrade the Course record

We now need additional fields to support the Angular frontend:

- Course title and description.
- Instructor and category.
- Price and free/paid classification.
- Course level.
- Rating.
- Publication status.

We'll retain the Lesson 1 `free` field for compatibility, but derive its validity from the price.

Replace the contents of:

`src/main/java/com/learnhub/model/Course.java`

<CodeBlock language="java" editable>
package com.learnhub.model;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

/**
 * Immutable representation of a LearnHub course.
 *
 * This is a plain Java domain model for Lessons 1–4.
 * Later, persistence entities and API DTOs may be
 * separate types with different responsibilities.
 */
public record Course(
        long id,
        String title,
        String description,
        String instructor,
        String category,
        BigDecimal price,
        boolean free,
        CourseLevel level,
        BigDecimal rating,
        CourseStatus status
) {

    /**
     * Compact canonical constructor.
     *
     * Validation occurs before a Course instance
     * becomes available to the rest of the program.
     */
    public Course {

        if (id <= 0) {
            throw new IllegalArgumentException(
                    "Course ID must be positive."
            );
        }

        title = requireText(title, "Course title");
        description = requireText(
                description,
                "Course description"
        );
        instructor = requireText(
                instructor,
                "Instructor name"
        );
        category = requireText(
                category,
                "Course category"
        );

        Objects.requireNonNull(level, "Course level is required.");
        Objects.requireNonNull(status, "Course status is required.");

        if (price == null || price.signum() < 0) {
            throw new IllegalArgumentException(
                    "Course price must be zero or greater."
            );
        }

        // Store all prices with exactly two decimal places.
        price = price.setScale(
                2,
                RoundingMode.UNNECESSARY
        );

        if (free != (price.signum() == 0)) {
            throw new IllegalArgumentException(
                    "Free status must match course price."
            );
        }

        if (rating == null
                || rating.compareTo(BigDecimal.ZERO) < 0
                || rating.compareTo(new BigDecimal("5.0")) > 0) {

            throw new IllegalArgumentException(
                    "Rating must be between 0 and 5."
            );
        }

        rating = rating.setScale(
                1,
                RoundingMode.UNNECESSARY
        );
    }

    /**
     * Shared text validation.
     */
    private static String requireText(
            String value,
            String fieldName
    ) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException(
                    fieldName + " is required."
            );
        }

        return value.trim();
    }

    /**
     * Human-readable price for the console application.
     *
     * API responses will later return structured values
     * rather than presentation-specific strings.
     */
    public String priceLabel() {
        return free
                ? "FREE"
                : "$" + price.toPlainString();
    }

    /**
     * Only published courses belong in public discovery.
     */
    public boolean isPubliclyVisible() {
        return status == CourseStatus.PUBLISHED;
    }
}
</CodeBlock>

### Important design decisions

**Validation happens at construction.** Invalid prices, empty titles, and missing enum values cannot create a valid `Course` instance.

**`BigDecimal` is used for price and rating.** This avoids binary floating-point representation issues, although rating values don't necessarily require monetary precision.

**`RoundingMode.UNNECESSARY` rejects unexpected precision.** A price such as `49.999` is rejected rather than silently rounded. This is a deliberate domain rule.

**Publication state is explicit.** We can prevent draft and archived courses from appearing in the public catalog.

**`free` is redundant with price.** We preserve it because it mirrors the Angular frontend's existing model. In a later backend refactor, we may derive `isFree` in the API response instead of storing both.

## 7. Create sample LearnHub course data

We'll move hardcoded sample courses out of `Main.java` and into a catalog class.

Before building the catalog, we need to define how searching and sorting should work.

### Step 7.1 — CourseSort enum

File: `src/main/java/com/learnhub/catalog/CourseSort.java`

<CodeBlock language="java" editable>
package com.learnhub.catalog;

/**
 * Supported sorting options for the public catalog.
 */
public enum CourseSort {
    DEFAULT,
    PRICE_LOW_TO_HIGH,
    PRICE_HIGH_TO_LOW,
    RATING_HIGH_TO_LOW
}
</CodeBlock>

### Step 7.2 — CourseFilter record

File: `src/main/java/com/learnhub/catalog/CourseFilter.java`

<CodeBlock language="java" editable>
package com.learnhub.catalog;

import com.learnhub.model.CourseLevel;

/**
 * A search request expressed as plain Java data.
 *
 * Null category/level/free means no restriction.
 */
public record CourseFilter(
        String search,
        String category,
        Boolean free,
        CourseLevel level
) {

    public CourseFilter {

        // Normalize missing search terms to an empty string.
        search = search == null
                ? ""
                : search.trim();

        // Treat blank categories as no category filter.
        category = category == null || category.isBlank()
                ? null
                : category.trim();
    }

    /**
     * Convenient factory for browsing all courses.
     */
    public static CourseFilter all() {
        return new CourseFilter("", null, null, null);
    }
}
</CodeBlock>

Notice that `Boolean` is used instead of primitive `boolean`.

Why?

- `true` means free courses only.
- `false` means paid courses only.
- `null` means all courses.

This is a useful example of modeling optional filters. In a real HTTP API, we'll validate and map request parameters explicitly.

### Step 7.3 — PageResult record

File: `src/main/java/com/learnhub/catalog/PageResult.java`

<CodeBlock language="java" editable>
package com.learnhub.catalog;

import java.util.List;

/**
 * Generic paginated result.
 *
 * pageNumber uses zero-based indexing.
 */
public record PageResult<T>(
        List<T> items,
        int pageNumber,
        int pageSize,
        long totalItems,
        int totalPages
) {

    public PageResult {
        if (items == null) {
            throw new IllegalArgumentException(
                    "Items cannot be null."
            );
        }

        if (pageNumber < 0 || pageSize <= 0) {
            throw new IllegalArgumentException(
                    "Invalid pagination arguments."
            );
        }

        if (totalItems < 0 || totalPages < 0) {
            throw new IllegalArgumentException(
                    "Pagination totals cannot be negative."
            );
        }

        // Defensive copy prevents callers from modifying
        // the contents of the result through this list.
        items = List.copyOf(items);
    }
}
</CodeBlock>
This structure resembles the pagination metadata that our Spring Boot API will eventually return.

## 8. Implement the CourseCatalog class

File: `src/main/java/com/learnhub/catalog/CourseCatalog.java`

<CodeBlock language="java" editable>
package com.learnhub.catalog;

import com.learnhub.model.Course;

import java.util.Comparator;
import java.util.List;
import java.util.Locale;
import java.util.Objects;
import java.util.Optional;
import java.util.stream.Stream;

/**
 * Plain Java course catalog.
 *
 * Responsibilities:
 * - Find courses by ID
 * - Filter publicly visible courses
 * - Search by text
 * - Filter by category, level, and price type
 * - Sort courses
 * - Return paginated results
 *
 * No Spring Boot or database dependency is required.
 */
public final class CourseCatalog {

    private static final int MAX_PAGE_SIZE = 100;

    private final List<Course> courses;

    public CourseCatalog(List<Course> courses) {
        Objects.requireNonNull(
                courses,
                "Course list is required."
        );

        if (courses.stream().anyMatch(Objects::isNull)) {
            throw new IllegalArgumentException(
                    "Course list cannot contain null values."
            );
        }

        // Keep our own immutable snapshot.
        this.courses = List.copyOf(courses);

        long distinctIds = this.courses.stream()
                .map(Course::id)
                .distinct()
                .count();

        if (distinctIds != this.courses.size()) {
            throw new IllegalArgumentException(
                    "Duplicate course IDs are not allowed."
            );
        }
    }

    /**
     * Find a published course by ID.
     *
     * Draft and archived courses are intentionally hidden
     * from public catalog lookups.
     */
    public Optional<Course> findPublishedById(long id) {

        if (id <= 0) {
            return Optional.empty();
        }

        return courses.stream()
                .filter(Course::isPubliclyVisible)
                .filter(course -> course.id() == id)
                .findFirst();
    }

    /**
     * Search the public catalog with optional filters,
     * sorting, and zero-based pagination.
     */
    public PageResult<Course> search(
            CourseFilter filter,
            CourseSort sort,
            int pageNumber,
            int pageSize
    ) {

        Objects.requireNonNull(
                filter,
                "Course filter is required."
        );

        Objects.requireNonNull(
                sort,
                "Sort option is required."
        );

        if (pageNumber < 0) {
            throw new IllegalArgumentException(
                    "Page number cannot be negative."
            );
        }

        if (pageSize < 1 || pageSize > MAX_PAGE_SIZE) {
            throw new IllegalArgumentException(
                    "Page size must be between 1 and 100."
            );
        }

        String normalizedSearch = normalize(filter.search());

        Stream<Course> stream = courses.stream()
                .filter(Course::isPubliclyVisible)
                .filter(course -> matchesSearch(
                        course,
                        normalizedSearch
                ))
                .filter(course -> matchesCategory(
                        course,
                        filter.category()
                ))
                .filter(course -> filter.free() == null
                        || course.free() == filter.free())
                .filter(course -> filter.level() == null
                        || course.level() == filter.level());

        Comparator<Course> comparator = comparatorFor(sort);

        List<Course> matches = stream
                .sorted(comparator)
                .toList();

        long totalItems = matches.size();

        int totalPages = (int) (
                (totalItems + pageSize - 1L) / pageSize
        );

        // Use long arithmetic to avoid int multiplication overflow.
        long offset = (long) pageNumber * pageSize;

        List<Course> pageItems;

        if (offset >= totalItems) {
            pageItems = List.of();
        } else {
            int from = (int) offset;
            int to = (int) Math.min(
                    offset + pageSize,
                    totalItems
            );

            pageItems = matches.subList(from, to);
        }

        return new PageResult<>(
                pageItems,
                pageNumber,
                pageSize,
                totalItems,
                totalPages
        );
    }

    private boolean matchesSearch(
            Course course,
            String search
    ) {

        if (search.isEmpty()) {
            return true;
        }

        String searchableText = String.join(
                " ",
                course.title(),
                course.description(),
                course.instructor(),
                course.category()
        );

        return normalize(searchableText).contains(search);
    }

    private boolean matchesCategory(
            Course course,
            String category
    ) {

        return category == null
                || course.category().equalsIgnoreCase(category);
    }

    private String normalize(String text) {
        return text.toLowerCase(Locale.ROOT);
    }

    private Comparator<Course> comparatorFor(CourseSort sort) {

        Comparator<Course> byId = Comparator.comparingLong(
                Course::id
        );

        return switch (sort) {

            case DEFAULT -> byId;

            case PRICE_LOW_TO_HIGH ->
                    Comparator.comparing(Course::price)
                            .thenComparing(byId);

            case PRICE_HIGH_TO_LOW ->
                    Comparator.comparing(
                                    Course::price,
                                    Comparator.reverseOrder()
                            )
                            .thenComparing(byId);

            case RATING_HIGH_TO_LOW ->
                    Comparator.comparing(
                                    Course::rating,
                                    Comparator.reverseOrder()
                            )
                            .thenComparing(byId);
        };
    }
}
</CodeBlock>

### What this class demonstrates

1. **Encapsulation:** Catalog data is held in a private immutable list.
2. **Validation:** Invalid pagination arguments are rejected early.
3. **`Optional`:** A missing published course is represented explicitly.
4. **Streams:** Filters compose into a readable processing pipeline.
5. **Comparator composition:** Sorting is deterministic because ties fall back to course ID.
6. **Switch expressions:** Every enum case must be handled.
7. **Safe pagination arithmetic:** The offset uses `long` to avoid integer overflow.

The class is deliberately read-only. Course creation and editing will later be handled by dedicated application services with authorization and database transactions.

## 9. Update Main.java to demonstrate the catalog

The Lesson 1 `Main.java` still uses the old five-field `Course` constructor. Because we've expanded the record, we must update its constructor calls.

Replace the entire file:

`src/main/java/com/learnhub/Main.java`

<CodeBlock language="java" editable>
package com.learnhub;

import com.learnhub.catalog.CourseCatalog;
import com.learnhub.catalog.CourseFilter;
import com.learnhub.catalog.CourseSort;
import com.learnhub.catalog.PageResult;
import com.learnhub.model.Course;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;

import java.math.BigDecimal;
import java.util.List;

/**
 * Entry point for the LearnHub Java console application.
 *
 * Lesson 2 demonstrates reusable catalog business logic.
 */
public final class Main {

    private Main() {
        // Prevent unnecessary instantiation.
    }

    public static void main(String[] args) {

        CourseCatalog catalog = new CourseCatalog(seedCourses());

        System.out.println("==================================");
        System.out.println("     LearnHub Course Catalog");
        System.out.println("==================================");

        // Browse all published courses.
        PageResult<Course> allCourses = catalog.search(
                CourseFilter.all(),
                CourseSort.DEFAULT,
                0,
                10
        );

        printResults("Published courses", allCourses);

        // Search by keyword.
        PageResult<Course> angularCourses = catalog.search(
                new CourseFilter("angular", null, null, null),
                CourseSort.DEFAULT,
                0,
                10
        );

        printResults("Search: angular", angularCourses);

        // Find free beginner courses.
        PageResult<Course> freeBeginners = catalog.search(
                new CourseFilter(
                        "",
                        null,
                        true,
                        CourseLevel.BEGINNER
                ),
                CourseSort.RATING_HIGH_TO_LOW,
                0,
                10
        );

        printResults("Free beginner courses", freeBeginners);

        // Sort published courses by price.
        PageResult<Course> cheapestFirst = catalog.search(
                CourseFilter.all(),
                CourseSort.PRICE_LOW_TO_HIGH,
                0,
                10
        );

        printResults("Price: low to high", cheapestFirst);

        // Demonstrate an Optional lookup.
        catalog.findPublishedById(1L)
                .ifPresent(course ->
                        System.out.println(
                                "\nCourse details: " + course.title()
                        )
                );
    }

    /**
     * Temporary in-memory data.
     *
     * Lesson 10 will introduce persistent database entities.
     */
    private static List<Course> seedCourses() {

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

    /**
     * Console presentation belongs outside the catalog.
     */
    private static void printResults(
            String heading,
            PageResult<Course> page
    ) {

        System.out.println("\n--- " + heading + " ---");

        for (Course course : page.items()) {
            System.out.printf(
                    "#%d | %s | %s | %s | %s%n",
                    course.id(),
                    course.title(),
                    course.category(),
                    course.level(),
                    course.priceLabel()
            );
        }

        System.out.printf(
                "Total: %d | Page: %d of %d%n",
                page.totalItems(),
                page.pageNumber() + 1,
                page.totalPages()
        );
    }
}
</CodeBlock>

The draft Spring Boot course is deliberately excluded from public results. Later, the instructor API will be able to retrieve drafts owned by the authenticated instructor.

### Run the application

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean compile
mvn package

java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar
```

At this point, `mvn package` may fail because the five Lesson 1 tests still use the old `Course` constructor. That is expected until you update the tests in the next section. You can use `mvn compile` to verify production code first, then run the full build after updating the tests.

The catalog should contain:

| Query | Expected result |
|---|---|
| All published courses | 4 |
| Search `angular` | 1 |
| Free beginner courses | 2 |
| Price low to high | Free courses first |
| Public lookup of ID 5 | No result |

## 10. Update the Lesson 1 tests

Replace:

`src/test/java/com/learnhub/model/CourseTest.java`

<CodeBlock language="java" editable>
package com.learnhub.model;

import org.junit.jupiter.api.Test;

import java.math.BigDecimal;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests the Course domain model's invariants.
 */
class CourseTest {

    private Course createCourse(
            long id,
            BigDecimal price,
            boolean free
    ) {
        return new Course(
                id,
                "Angular Complete Guide",
                "Learn Angular.",
                "Alex Morgan",                "Web Development",
                price,
                free,
                CourseLevel.BEGINNER,
                new BigDecimal("4.8"),
                CourseStatus.PUBLISHED
        );
    }

    @Test
    void freeCourseHasFreeLabel() {
        Course course = createCourse(
                2L,
                BigDecimal.ZERO,
                true
        );

        assertEquals("FREE", course.priceLabel());
    }

    @Test
    void paidCourseDisplaysTwoDecimalPlaces() {
        Course course = createCourse(
                1L,
                new BigDecimal("49.99"),
                false
        );

        assertEquals("$49.99", course.priceLabel());
    }

    @Test
    void rejectsNegativePrice() {
        assertThrows(
                IllegalArgumentException.class,
                () -> createCourse(
                        3L,
                        new BigDecimal("-1.00"),
                        false
                )
        );
    }

    @Test
    void rejectsInvalidId() {
        assertThrows(
                IllegalArgumentException.class,
                () -> createCourse(
                        0L,
                        BigDecimal.ZERO,
                        true
                )
        );
    }

    @Test
    void rejectsInconsistentFreeStatus() {
        assertThrows(
                IllegalArgumentException.class,
                () -> createCourse(
                        4L,
                        new BigDecimal("19.99"),
                        true
                )
        );
    }

    @Test
    void rejectsMoreThanTwoPriceDecimalPlaces() {
        assertThrows(
                ArithmeticException.class,
                () -> createCourse(
                        5L,
                        new BigDecimal("19.999"),
                        false
                )
        );
    }

    @Test
    void normalizesPriceScale() {
        Course course = createCourse(
                6L,
                new BigDecimal("20"),
                false
        );

        assertEquals(
                new BigDecimal("20.00"),
                course.price()
        );
    }
}
</CodeBlock>

### Why does the precision test expect `ArithmeticException`?

The record uses:

```java
price.setScale(2, RoundingMode.UNNECESSARY);
```

When a value cannot be represented at two decimal places without rounding, `BigDecimal` throws `ArithmeticException`.

In a production REST API, we might translate that into a more consistent domain validation exception so the client receives a useful HTTP 400 response.

## 11. Add CourseCatalog unit tests

File: `src/test/java/com/learnhub/catalog/CourseCatalogTest.java`

<CodeBlock language="java" editable>
package com.learnhub.catalog;

import com.learnhub.model.Course;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Tests public catalog behavior without a database.
 */
class CourseCatalogTest {

    private CourseCatalog catalog;

    private Course course(
            long id,
            String title,
            String category,
            String price,
            CourseLevel level,
            CourseStatus status
    ) {
        BigDecimal amount = new BigDecimal(price);

        return new Course(
                id,
                title,
                "A course description.",
                "Test Instructor",
                category,
                amount,
                amount.signum() == 0,
                level,
                new BigDecimal("4.5"),
                status
        );
    }

    @BeforeEach
    void setUp() {
        catalog = new CourseCatalog(List.of(
                course(
                        1L,
                        "Angular Complete Guide",
                        "Web Development",
                        "49.99",
                        CourseLevel.BEGINNER,
                        CourseStatus.PUBLISHED
                ),
                course(
                        2L,
                        "Python for Beginners",
                        "Programming",
                        "0",
                        CourseLevel.BEGINNER,
                        CourseStatus.PUBLISHED
                ),
                course(
                        3L,
                        "Advanced Spring Boot",
                        "Web Development",
                        "79.99",
                        CourseLevel.ADVANCED,
                        CourseStatus.DRAFT
                ),
                course(
                        4L,
                        "Data Science Essentials",
                        "Data Science",
                        "0",
                        CourseLevel.BEGINNER,
                        CourseStatus.PUBLISHED
                )
        ));
    }

    @Test
    void publicCatalogExcludesDrafts() {
        PageResult<Course> result = catalog.search(
                CourseFilter.all(),
                CourseSort.DEFAULT,
                0,
                10
        );

        assertEquals(3, result.totalItems());
        assertTrue(
                result.items().stream()
                        .noneMatch(c -> c.id() == 3L)
        );
    }

    @Test
    void searchIsCaseInsensitive() {
        PageResult<Course> result = catalog.search(
                new CourseFilter("ANGULAR", null, null, null),
                CourseSort.DEFAULT,
                0,
                10
        );

        assertEquals(1, result.totalItems());
        assertEquals(1L, result.items().getFirst().id());
    }

    @Test
    void filtersFreeBeginnerCourses() {
        PageResult<Course> result = catalog.search(
                new CourseFilter(
                        "",
                        null,
                        true,
                        CourseLevel.BEGINNER
                ),
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
    void sortsByPriceAscending() {
        PageResult<Course> result = catalog.search(
                CourseFilter.all(),
                CourseSort.PRICE_LOW_TO_HIGH,
                0,
                10
        );

        assertEquals(
                List.of(2L, 4L, 1L),
                result.items().stream().map(Course::id).toList()
        );
    }

    @Test
    void paginatesResults() {
        PageResult<Course> result = catalog.search(
                CourseFilter.all(),
                CourseSort.DEFAULT,
                1,
                2
        );

        assertEquals(3, result.totalItems());
        assertEquals(2, result.totalPages());
        assertEquals(1, result.items().size());
        assertEquals(4L, result.items().getFirst().id());
    }

    @Test
    void rejectsInvalidPageSize() {
        assertThrows(
                IllegalArgumentException.class,
                () -> catalog.search(
                        CourseFilter.all(),
                        CourseSort.DEFAULT,
                        0,
                        0
                )
        );
    }

    @Test
    void cannotFindDraftByPublicLookup() {
        assertTrue(
                catalog.findPublishedById(3L).isEmpty()
        );
    }

    @Test
    void rejectsDuplicateIds() {
        Course duplicate = course(
                1L,
                "Duplicate",
                "Programming",
                "0",
                CourseLevel.BEGINNER,
                CourseStatus.PUBLISHED
        );

        assertThrows(
                IllegalArgumentException.class,
                () -> new CourseCatalog(
                        List.of(duplicate, duplicate)
                )
        );
    }
}
</CodeBlock>

Run:

```powershell
mvn clean test
```

With these files, you should have:

<box border radius="lg" padding={3} gap={2}>
  <row justify="between" align="center">
    <box gap={1}>
      <text color="secondary" size="xs">Expected test count</text>
      <title size="2xl" tabularNums>15</title>
    </box>
    <icon name="test-tube-diagonal" color="secondary" size="2xl"/>
  </row>
  <grid columns={2} gap={2}>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={2} gap={1}>
        <text size="sm" weight="medium">7 model tests</text>
        <text color="secondary" size="xs">Validation and pricing</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={2} gap={1}>
        <text size="sm" weight="medium">8 catalog tests</text>
        <text color="secondary" size="xs">Filtering and pagination</text>
      </box>
    </grid-item>
  </grid>
  <caption>Expected if you use the complete replacement test files above. This is a target, not a report of tests executed on your computer.</caption>
</box>

### Verify the packaged application

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Maven verification failed."
}

java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar

if ($LASTEXITCODE -ne 0) {
    throw "LearnHub application failed."
}
```

## 12. Engineering review: important backend concepts

### Primitive vs reference types

In our model:

```java
long id;
boolean free;
```

are primitives, while:

```java
String title;
BigDecimal price;
CourseLevel level;
```

are reference types.

Primitive fields cannot be null. Reference fields can be null unless validation prevents it.

Later, when handling API requests, we'll need to distinguish a missing field from a field explicitly set to a default value.

### Equality and money

Avoid this for monetary comparison:

```java
price.equals(new BigDecimal("0.00"));
```

`BigDecimal.equals()` considers scale. For example, `0` and `0.00` are numerically equal but not equal according to `equals()`.

Use:

```java
price.compareTo(BigDecimal.ZERO) == 0
```

for numeric equality.

### `Optional` and absence

Our catalog method returns:

```java
Optional<Course>
```

because a course might not exist or might not be publicly visible.

In a future Spring Boot service, a missing course may result in a domain exception that an exception handler maps to HTTP 404.

Avoid using `Optional` indiscriminately for fields or method parameters. Its strongest use case here is a method return value that may be absent.

### Validation at multiple boundaries

We'll eventually validate course data at several levels:

<box gap={2}>
  <row align="center" gap={2}>
    <icon name="shield-check" color="secondary"/>
    **HTTP request validation** — Is the submitted JSON structurally valid?
  </row>
  <row align="center" gap={2}>
    <icon name="shield-check" color="secondary"/>
    **Domain validation** — Is the operation allowed by business rules?
  </row>
  <row align="center" gap={2}>
    <icon name="shield-check" color="secondary"/>
    **Database constraints** — Are uniqueness and relationships enforced?
  </row>
</box>

These layers complement one another. No single layer replaces the others.

### Pagination and performance

Our current implementation sorts and filters the entire in-memory catalog before selecting a page.

That is appropriate for this small Java exercise, but not for a production marketplace with hundreds of thousands of courses.

In later lessons, we'll use Spring Data pagination, database indexes, and query-level filtering.

## 13. Hands-on exercises

These exercises move beyond syntax and focus on realistic backend behavior.

<box gap={3}>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md"><icon name="search" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise A — Search normalization**

      Extend the search logic to collapse repeated whitespace and support a search such as `"  angular   complete  "`. Add tests for leading spaces, repeated spaces, and an empty query.

      <text color="secondary" size="xs">Stretch goal: consider whether accent-insensitive search belongs in Java or in the database.</text>
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md"><icon name="tags" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise B — Category discovery**

      Implement `List<String> publishedCategories()` in `CourseCatalog`. Return unique categories from published courses, sorted alphabetically. Verify that draft-only categories are excluded.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md"><icon name="percent" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise C — Discount calculation**

      Add a separate pricing utility that applies a percentage discount to a paid course.

      Requirements: discount from 0 to 100 inclusive; result never negative; two decimal places using an explicitly chosen rounding policy; original course remains unchanged.

      Example: $49.99 with a 20% discount becomes $39.99 when rounded to two decimal places using `HALF_UP`.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md"><icon name="list-ordered" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise D — Pagination edge cases**

      Test an empty catalog, a page beyond the last page, the maximum allowed page size, a negative page number, and a very large page number.

      Decide whether an empty result should show a human-friendly page count of zero or one at the presentation layer.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>    <box background="surface-secondary" padding={2} radius="md"><icon name="shield" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise E — Draft visibility and ownership**

      Sketch separate service methods for public catalog access and instructor-owned draft access. Do not implement fake security using only a user ID supplied by a caller.

      Write down which future Spring Security principal information each method will need.
    </box>
  </row>
</box>

## 14. Mini challenge: write a reusable pricing function

Try this without looking at the solution first:

Create:

`src/main/java/com/learnhub/catalog/CoursePricing.java`

Implement:

```java
public static BigDecimal discountedPrice(
        BigDecimal originalPrice,
        BigDecimal discountPercent
)
```

The function should validate inputs and return a two-decimal price.

<box border radius="lg" padding={3} gap={2}>
  {@body const [showSolution,setShowSolution]=DIL.useState(false)}
  <row justify="between" align="center">
    **Reference solution**
    <button color="secondary" variant="outline" size="sm" onClick={()=>setShowSolution(!showSolution)}>{showSolution?"Hide solution":"Reveal solution"} <icon name={showSolution?"chevron-up":"chevron-down"} inline/></button>
  </row>
  {#if showSolution}
    <CodeBlock language="java" editable>
package com.learnhub.catalog;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

public final class CoursePricing {

    private static final BigDecimal HUNDRED =
            new BigDecimal("100");

    private CoursePricing() {}

    public static BigDecimal discountedPrice(
            BigDecimal originalPrice,
            BigDecimal discountPercent
    ) {
        Objects.requireNonNull(originalPrice);
        Objects.requireNonNull(discountPercent);

        if (originalPrice.signum() < 0) {
            throw new IllegalArgumentException(
                    "Price cannot be negative."
            );
        }

        if (discountPercent.signum() < 0
                || discountPercent.compareTo(HUNDRED) > 0) {
            throw new IllegalArgumentException(
                    "Discount must be between 0 and 100."
            );
        }

        BigDecimal multiplier = BigDecimal.ONE.subtract(
                discountPercent.divide(HUNDRED)
        );

        return originalPrice.multiply(multiplier)
                .setScale(2, RoundingMode.HALF_UP);
    }
}
</CodeBlock>
    Test with `49.99` and `20`, then with `0`, `100`, and invalid discount percentages.
  {/if}
</box>

## 15. Common mistakes and troubleshooting

| Problem | Likely cause | Solution |
|---|---|---|
| `Course` constructor does not match | Old Lesson 1 constructor calls remain | Update `Main.java` and `CourseTest.java` |
| `getFirst()` is not found | Compiler is using an older Java API | Verify Java 21 and Maven compiler release |
| `ArithmeticException` for price | More than two nonzero decimal places | Validate precision or explicitly round at a defined boundary |
| Draft courses appear in results | Public visibility filter missing | Apply `isPubliclyVisible()` before search and pagination |
| Unexpected search results | Case or whitespace handling | Normalize using `Locale.ROOT` and test |
| Wrong pagination counts | Count calculated after slicing | Count matches before extracting page items |
| Duplicate IDs accepted | Catalog constructor lacks uniqueness check | Validate distinct IDs |
| Tests fail after model change | Tests still construct the old record | Update fixtures alongside the domain model |

### Final PowerShell verification

```powershell
cd $HOME\angular-learning\learnhub-backend

# Confirm Java and Maven use the expected JDK.
java -version
mvn -version

# Clean, compile, test, package, and verify.
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 2 verification failed."
}

# Execute the packaged application.
java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar

if ($LASTEXITCODE -ne 0) {
    throw "LearnHub console application failed."
}

# Review changed files.
git status
```

If everything works, commit your progress:

```powershell
git add .
git commit -m "Implement LearnHub course catalog domain logic"
```

## 16. Lesson 2 knowledge quiz

<LearningQuiz
  title="Java Domain Modeling and Catalog Logic"
  nextQuizQuery="Give me 5 new advanced Java backend questions about BigDecimal, immutable records, comparator stability, validation, pagination, and Optional, without repeating Lesson 2."
  questions={[
    {
      "id":"q1",
      "type":"single_select",
      "question":"Which expression correctly checks whether a BigDecimal price is numerically equal to zero regardless of scale?",
      "options":[
        {"value":"A","label":"price.equals(BigDecimal.ZERO)","feedback":"BigDecimal.equals also compares scale, so 0.00 is not equal to 0 using this method."},
        {"value":"B","label":"price.compareTo(BigDecimal.ZERO) == 0","feedback":"compareTo checks numerical value and treats zero values with different scales as equal."},
        {"value":"C","label":"price.hashCode() == BigDecimal.ZERO.hashCode()","feedback":"Hash codes are not a valid general test of numerical equality, and BigDecimal hash codes are scale-sensitive."},
        {"value":"D","label":"price.toString().equals(\"0\")","feedback":"The textual form can contain a decimal scale, such as 0.00."}
      ],
      "correctValues":["B"],
      "hint":"Think about the difference between representation equality and numerical ordering."
    },
    {
      "id":"q2",
      "type":"single_select",
      "question":"A catalog has 23 matching courses, uses zero-based pages, and a page size of 10. How many items appear on page number 2?",
      "options":[
        {"value":"A","label":"0","feedback":"The third page begins at offset 20, leaving three of the 23 items."},
        {"value":"B","label":"2","feedback":"With 23 matches, subtracting the first two full pages leaves three items."},
        {"value":"C","label":"3","feedback":"Pages 0 and 1 contain ten items each; page 2 contains the remaining three."},
        {"value":"D","label":"10","feedback":"Only three items remain after the first 20 matches."}
      ],
      "correctValues":["C"],
      "hint":"Compute the starting offset for the requested zero-based page."
    },
    {
      "id":"q3",
      "type":"multi_select",
      "question":"Which practices help maintain correct and predictable course catalog behavior? Select all that apply.",
      "options":[
        {"value":"A","label":"Reject duplicate course IDs when creating an in-memory catalog","feedback":"Duplicate IDs would make lookups ambiguous, so rejecting them preserves a useful invariant."},
        {"value":"B","label":"Use a secondary course-ID comparator to resolve sorting ties","feedback":"A deterministic tie-breaker produces predictable result ordering."},
        {"value":"C","label":"Filter published courses only after calculating the public result count","feedback":"Counting drafts before filtering would produce incorrect public pagination metadata."},
        {"value":"D","label":"Validate page size before calculating offsets","feedback":"Rejecting invalid sizes prevents undefined or undesirable pagination behavior."}
      ],
      "correctValues":["A","B","D"],
      "hint":"Evaluate identity, ordering, visibility, and input constraints independently."
    },
    {
      "id":"q4",
      "type":"true_false",
      "question":"A Java record's compact constructor can normalize a component value before that value is assigned to the record's field.",
      "options":[
        {"value":"true","label":"True","feedback":"A compact constructor can reassign its component parameter, and the final normalized value is assigned to the corresponding record field."},
        {"value":"false","label":"False","feedback":"Compact constructors support validation and normalization of component parameters before the implicit field assignments."}
      ],
      "correctValues":["true"],
      "hint":"Consider when record component fields receive their values during construction."
    },
    {
      "id":"q5",
      "type":"true_false",
      "question":"For a large production course catalog, Java Stream filtering of every course in memory is generally preferable to filtering and pagination in PostgreSQL.",
      "options":[
        {"value":"true","label":"True","feedback":"Loading all courses into application memory increases data transfer and processing; indexed database filtering and pagination are generally more scalable."},
        {"value":"false","label":"False","feedback":"Database-side filtering and pagination can reduce the number of rows transferred and the amount of application memory used."}
      ],
      "correctValues":["false"],
      "hint":"Compare the amount of data transferred and held in application memory by each approach."
    }
  ]}
/>

## 17. Lesson completion checklist

{@body const groups=[{title:"Domain modeling",items:["Created CourseLevel and CourseStatus enums","Updated Course record with validated fields","Confirmed BigDecimal price and rating rules","Verified draft visibility rules"]},{title:"Catalog functionality",items:["Created CourseFilter and CourseSort","Created PageResult with defensive copying","Implemented search and category filters","Implemented free/paid and level filters","Implemented sorting and pagination","Updated Main.java with five sample courses"]},{title:"Testing and verification",items:["Updated the Lesson 1 CourseTest","Added eight CourseCatalog tests","Maven verification passes","Packaged JAR runs successfully","Completed at least two exercises","Committed Lesson 2 changes locally"]}]}
{@body const [completed,setCompleted]=DIL.useState({})}
{@body const total=groups.reduce((s,g)=>s+g.items.length,0)}
{@body const done=Object.values(completed).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 2 progress**
      <text color="secondary" size="xs">Track your verified milestones.</text>
    </box>
    <title size="lg" tabularNums>{done}/{total}</title>
  </row>
  <box background="surface-tertiary" radius="full" height="8px" clip>
    <box background="rgba(109,40,217,0.85)" width={`${done/total*100}%`} height="100%" radius="full"/>
  </box>
  {#each groups as group,i}
    <box gap={2} key={i}>
      **{group.title}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!completed[`${i}-${j}`]} onChange={v=>setCompleted(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if done===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 2 checklist completed.**
    </row>
  {/if}
</box>

## 18. What you'll build in Lesson 3

<row align="start" gap={3}>
  <box background="surface-secondary" radius="lg" padding={3}>
    <icon name="boxes" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 3 — Object-Oriented Programming and Clean Domain Design**

    We'll refactor the catalog toward a maintainable backend architecture using interfaces, composition, encapsulation, dependency inversion, polymorphism, and clean service boundaries.

    We'll also examine where records are appropriate, where regular classes are preferable, and how these decisions prepare us for Spring dependency injection and JPA.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 3: Object-Oriented Programming, Interfaces, Encapsulation, Composition, SOLID Principles, and Clean Domain Design. Continue from the Lesson 2 Maven project with experienced-Java pacing, Windows PowerShell commands, complete documented Java code, exercises, quiz, and checklist.")}>Start Lesson 3 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 2 deliverable:** A validated LearnHub course domain model, a reusable in-memory catalog with public visibility rules, search, filters, sorting, pagination, and 15 automated unit tests. This is the business-logic foundation we'll build upon when introducing Spring Boot and PostgreSQL.