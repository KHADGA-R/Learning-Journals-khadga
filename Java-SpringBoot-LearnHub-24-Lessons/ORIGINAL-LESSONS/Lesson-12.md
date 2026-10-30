# LearnHub Java & Spring Boot Backend — Lesson 12

<text color="secondary" size="sm">Advanced PostgreSQL Search · JPA Specifications · `pg_trgm` · Query Plans · Pagination · Flyway · Angular Integration</text>

**Stack:** Java 21 · Spring Boot 3.5.x · PostgreSQL · Spring Data JPA · Hibernate · Flyway · JUnit 5 · Testcontainers · Angular standalone components

**Estimated time:** 6–8 hours, including database benchmarking and tests.

In Lesson 11, we consolidated the LearnHub catalog into `/api/v1/courses` with database-backed filters, sorting, pagination, transactional CRUD operations, and Angular-safe response DTOs.

Lesson 12 takes that API from *functionally correct* toward *measurably efficient*.

The central lesson is:

**An index is not automatically a performance improvement. A query plan tells you whether PostgreSQL actually benefits from it.**

We'll build a realistic catalog dataset, improve search, introduce PostgreSQL trigram indexes, test query behavior, and prepare a safer strategy for large or changing catalogs.

## 1. Learning objectives

By the end, you should be able to:

1. Implement case-insensitive, literal-substring search with JPA Specifications.
2. Understand how PostgreSQL handles `LIKE`, `ILIKE`, `LOWER`, and trigram indexes.
3. Create safe, versioned Flyway migrations.
4. Inspect `EXPLAIN (ANALYZE, BUFFERS)` output.
5. Explain when PostgreSQL chooses sequential scans or bitmap index scans.
6. Design deterministic offset pagination and understand keyset pagination.
7. Seed realistic development data without contaminating production.
8. Write PostgreSQL-backed integration and HTTP contract tests.
9. Connect Angular catalog search, filters, sorting, and pagination.
10. Measure performance without writing brittle timing-based tests.

## 2. Lesson 12 architecture

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    **Angular LearnHub catalog**
    <text size="xs" color="secondary">Search · Category · Free/Paid · Level · Sort · Page</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **GET /api/v1/courses**
    <text size="xs" color="secondary">Validated query parameters · Stable page DTO</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **CourseApplicationService**
    <text size="xs" color="secondary">Read transaction · Whitelisted sorting · Bounded pages</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **JPA Specifications**
    <text size="xs" color="secondary">Dynamic predicates · Literal search · Published-only</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    <icon name="database" size="lg"/>
    **PostgreSQL**
    <text size="xs" color="secondary">GIN trigram indexes · B-tree indexes · EXPLAIN ANALYZE</text>
  </box>
</box>

### Today's scope

We'll keep the Lesson 11 API response format:

```json
{
  "items": [],
  "pageNumber": 0,
  "pageSize": 12,
  "totalItems": 0,
  "totalPages": 0
}
```

No authentication or instructor-management APIs are added today. The write endpoints remain development-only until Spring Security and ownership checks are implemented.

---

## 3. Verify your Lesson 11 baseline

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info

git status
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 11 baseline verification failed."
}
```

Create a Git checkpoint if there are changes to preserve:

```powershell
git add .
git commit -m "Checkpoint before LearnHub search optimization"
```

Check existing migrations:

```powershell
Get-ChildItem src\main\resources\db\migration |
    Sort-Object Name
```

**Migration rule:** Never modify a Flyway migration that has already been applied. If V4 is your latest version, use V5 below. Otherwise choose the next available version.

### Inspect the existing search implementation

```powershell
Get-Content src\main\java\com\learnhub\persistence\CourseSpecifications.java

Get-Content src\main\java\com\learnhub\service\CourseSortMapper.java
```

The lesson assumes the Lesson 11 entity properties:

`title`, `description`, `instructor`, `instructorAccount`, `category`, `price`, `rating`, `free`, `level`, and `status`.

We will preserve those names and the existing Angular DTO contract.

---

## 4. PostgreSQL search fundamentals

Consider a course titled:

`Advanced Spring Boot and PostgreSQL`

Different SQL expressions behave differently:

| Expression | Behavior |
|---|---|
| `title LIKE '%spring%'` | Case-sensitive substring match |
| `title ILIKE '%spring%'` | Case-insensitive substring match |
| `lower(title) LIKE '%spring%'` | Case-normalized substring match |
| `title LIKE 'Spring%'` | Case-sensitive prefix match |
| `title = 'Spring Boot'` | Exact match |

PostgreSQL's `ILIKE` is convenient, but JPQL is designed to be portable across databases. Our existing JPA Specification uses `LOWER(...) LIKE ...`, so we'll optimize that expression rather than introducing database-specific SQL into the Java code.

### Why ordinary indexes may not help

A conventional B-tree index on `title` is generally not useful for a pattern with a leading wildcard:

```sql
WHERE lower(title) LIKE '%spring%'
```

PostgreSQL cannot use the beginning of the string to narrow the search.

A trigram index is designed for searches involving character sequences within strings.

### What is a trigram?

A trigram is a sequence of three consecutive characters.

For example, the word `spring` contains these internal three-character sequences:

<row gap={2} wrap="wrap">
  {#each ["spr","pri","rin","ing"] as tri}
    <box border radius="md" padding={{x:3,y:2}} key={tri}>
      <text weight="medium">{tri}</text>
    </box>
  {/each}
</row>

PostgreSQL's `pg_trgm` extension supports trigram-based matching and indexing.

Short searches of one or two characters may not yield useful trigram index conditions. PostgreSQL can choose a sequential scan instead.

That is expected behavior, not necessarily a failure.

## 5. Flyway V5 — Enable `pg_trgm` and create search indexes

Create:

`src/main/resources/db/migration/V5__course_search_indexes.sql`

<CodeBlock language="sql" editable>
-- Lesson 12: PostgreSQL catalog search indexes.
--
-- Run through Flyway using the normal application
-- migration identity. The pg_trgm extension must be
-- installed or creatable by that identity.

CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- These expressions match the LOWER(...) LIKE
-- predicates generated by our JPA Specifications.
CREATE INDEX IF NOT EXISTS idx_courses_title_trgm
    ON courses USING gin (lower(title) gin_trgm_ops);

CREATE INDEX IF NOT EXISTS idx_courses_description_trgm
    ON courses USING gin (lower(description) gin_trgm_ops);

CREATE INDEX IF NOT EXISTS idx_courses_instructor_trgm
    ON courses USING gin (lower(instructor) gin_trgm_ops);

CREATE INDEX IF NOT EXISTS idx_courses_category_trgm
    ON courses USING gin (lower(category) gin_trgm_ops);

CREATE INDEX IF NOT EXISTS idx_users_display_name_trgm
    ON app_users USING gin (lower(display_name) gin_trgm_ops);

-- Helps published catalog ordering and filtering.
CREATE INDEX IF NOT EXISTS idx_courses_status_id
    ON courses (status, id);

CREATE INDEX IF NOT EXISTS idx_courses_status_price_id
    ON courses (status, price, id);

CREATE INDEX IF NOT EXISTS idx_courses_status_rating_id
    ON courses (status, rating DESC, id);
</CodeBlock>

### Why these indexes?

The GIN indexes help PostgreSQL evaluate substring predicates. The B-tree indexes are candidates for queries that filter by publication status and order by ID, price, or rating.

However, adding all these indexes increases storage and slows inserts and updates. We're creating them for learning and benchmarking; production index selection should be driven by real workload measurements.

### PostgreSQL extension permissions

`CREATE EXTENSION pg_trgm` requires appropriate database privileges. On supported PostgreSQL installations, a user with `CREATE` privilege on the database can typically install this trusted extension.

If your application account cannot create it, use a database administrator to install the extension first:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U postgres -h localhost -d learnhub `
  -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"
```

The Flyway migration can then execute `CREATE EXTENSION IF NOT EXISTS` without needing to create the extension again.

For Testcontainers, ensure the migration identity can install the extension in the fresh test database.

### Verify the migration

Start the backend using your existing database configuration:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

In another PowerShell terminal:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT extname FROM pg_extension WHERE extname = 'pg_trgm';"
```

List the indexes:

```powershell
& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT tablename, indexname FROM pg_indexes WHERE schemaname = 'public' AND indexname LIKE 'idx_%trgm' ORDER BY indexname;"
```

If the migration fails, inspect the actual PostgreSQL error before modifying anything. Don't edit an already successfully applied migration.

---

## 6. Improve the JPA search Specification

Lesson 11 already introduced dynamic filters. Today we'll make the search implementation more explicit and testable.

**Replace:** `src/main/java/com/learnhub/persistence/CourseSpecifications.java`

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
 * Public catalog predicates.
 *
 * - All predicates execute in PostgreSQL.
 * - Only PUBLISHED courses are returned.
 * - Substring searches are case-insensitive.
 * - SQL LIKE wildcards supplied by users are escaped.
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

            predicates.add(cb.equal(
                    root.get("status"),
                    CourseStatus.PUBLISHED
            ));

            if (search != null && !search.isBlank()) {
                String pattern = "%"
                        + escapeLike(
                            search.trim().toLowerCase(Locale.ROOT)
                        )
                        + "%";

                // Optional relationship: legacy courses
                // may have no linked instructor account.
                var instructor = root.join(
                        "instructorAccount",
                        JoinType.LEFT
                );

                predicates.add(cb.or(
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
                                cb.lower(
                                        instructor.get("displayName")
                                ),
                                pattern,
                                '\\'
                        )
                ));
            }

            if (category != null && !category.isBlank()) {
                predicates.add(cb.equal(
                        cb.lower(root.get("category")),
                        category.trim().toLowerCase(Locale.ROOT)
                ));
            }

            if (free != null) {
                predicates.add(cb.equal(
                        root.get("free"),
                        free
                ));
            }

            if (level != null) {
                predicates.add(cb.equal(
                        root.get("level"),
                        level
                ));
            }

            return cb.and(predicates.toArray(Predicate[]::new));
        };
    }

    /**
     * Escapes PostgreSQL LIKE metacharacters.
     *
     * % and _ must behave as literal search input,
     * not as user-controlled wildcards.
     */
    static String escapeLike(String input) {
        return input
                .replace("\\", "\\\\")
                .replace("%", "\\%")
                .replace("_", "\\_");
    }
}
</CodeBlock>

### Why preserve the `LEFT JOIN`?

The earlier migration linked some courses to instructor accounts, but not all of them.

An inner join would silently remove courses that have no linked instructor.

The `LEFT JOIN` allows both old and new records to appear.

### Why not `query.distinct(true)`?

This Specification joins a many-to-one association, which does not multiply a course into multiple rows.

A `DISTINCT` is therefore unnecessary for this join.

If we later join a one-to-many association such as enrollments, we must revisit row multiplication and pagination carefully.

### Search limits

A user can submit an extremely long search string. The database may have to evaluate expensive predicates for every candidate row.

We'll enforce a 100-character search limit at the API boundary and add tests.

---

## 7. Validate catalog query parameters

Create:

`src/main/java/com/learnhub/web/dto/CourseSearchRequest.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.catalog.CourseSort;
import com.learnhub.model.CourseLevel;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.Size;

/**
 * Validated GET query parameters.
 *
 * Spring MVC binds query parameters to this record
 * through its constructor.
 */
public record CourseSearchRequest(
        @Size(max = 100)
        String search,

        @Size(max = 100)
        String category,

        Boolean free,

        CourseLevel level,

        CourseSort sort,

        @Min(0)
        Integer page,

        @Min(1)
        @Max(100)
        Integer size
) {
    public CourseSearchRequest {
        sort = sort == null ? CourseSort.DEFAULT : sort;
        page = page == null ? 0 : page;
        size = size == null ? 12 : size;
    }
}
</CodeBlock>

This creates a single object representing a catalog request.

### Update the GET method in CourseController

**File:** `src/main/java/com/learnhub/web/CourseController.java`

Replace only the existing `@GetMapping` catalog method with:

<CodeBlock language="java" editable>
@GetMapping
public CourseCatalogResponse catalog(
        @Valid @ModelAttribute CourseSearchRequest request
) {
    return service.catalog(
            request.search(),
            request.category(),
            request.free(),
            request.level(),
            request.sort(),
            request.page(),
            request.size()
    );
}
</CodeBlock>

Add:

```java
import com.learnhub.web.dto.CourseSearchRequest;
```

Keep all other CRUD methods unchanged.

### Binding and validation note

Spring MVC supports constructor binding for model attributes, but the exact exception raised for malformed query parameters depends on whether binding or Bean Validation failed.

Your Lesson 8 `@RestControllerAdvice` should already handle validation failures. Confirm it also handles invalid enum values and type conversion as `400 Bad Request`.

For a simple implementation, keep the existing Lesson 11 `@RequestParam` method if the model-attribute binding differs in your project. The important behavior is consistent validation and HTTP status codes, not adopting a record for its own sake.

---

## 8. Deterministic sorting

The Lesson 11 `CourseSortMapper` remains valid.

**File:** `src/main/java/com/learnhub/service/CourseSortMapper.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.catalog.CourseSort;
import org.springframework.data.domain.Sort;

/**
 * Whitelist of supported catalog sort modes.
 *
 * Every mode includes ID as a stable tie-breaker.
 */
public final class CourseSortMapper {

    private CourseSortMapper() {
    }

    public static Sort toJpaSort(CourseSort sort) {
        return switch (sort) {
            case DEFAULT ->
                    Sort.by(
                            Sort.Order.asc("id")
                    );

            case PRICE_LOW_TO_HIGH ->
                    Sort.by(
                            Sort.Order.asc("price"),
                            Sort.Order.asc("id")
                    );

            case PRICE_HIGH_TO_LOW ->
                    Sort.by(
                            Sort.Order.desc("price"),
                            Sort.Order.asc("id")
                    );

            case RATING_HIGH_TO_LOW ->
                    Sort.by(
                            Sort.Order.desc("rating"),
                            Sort.Order.asc("id")
                    );
        };
    }
}
</CodeBlock>

Why the ID tie-breaker?

Suppose three courses all cost `$49.99`. Sorting by price alone doesn't specify their relative order. Sorting by `(price, id)` does.

**Important distinction:** deterministic sorting does not prevent page shifting when courses are inserted, deleted, or repriced between requests. That is one reason to consider keyset pagination.

## 9. Build realistic development data

Five sample courses are useful for verifying functionality, but they're too few to demonstrate meaningful query-planner decisions.

We'll generate **2,000 synthetic courses** for a local development database.

This data is for learning and benchmarking, not production.

### Create the seed script

**File:** `src/test/resources/sql/lesson12_benchmark_seed.sql`

<CodeBlock language="sql" editable>
-- Lesson 12 synthetic catalog benchmark data.
--
-- DEVELOPMENT DATABASE ONLY.
-- Uses a distinctive title prefix so the rows can
-- be identified and removed safely.
--
-- Requires the Lesson 9 courses schema and the
-- optional metadata defaults from Lessons 7–10.

INSERT INTO courses (
    title,
    description,
    instructor,
    category,
    price,
    free,
    level,
    rating,
    status
)
SELECT
    'L12 Benchmark Course ' || gs,
    CASE gs % 4
        WHEN 0 THEN 'Advanced PostgreSQL indexing and search'
        WHEN 1 THEN 'Modern Angular component development'
        WHEN 2 THEN 'Spring Boot backend architecture'
        ELSE 'Python data analysis and visualization'
    END,
    CASE gs % 3
        WHEN 0 THEN 'Alex Morgan'
        WHEN 1 THEN 'Jordan Lee'
        ELSE 'Taylor Reed'
    END,
    CASE gs % 4
        WHEN 0 THEN 'Data Science'
        WHEN 1 THEN 'Web Development'
        WHEN 2 THEN 'Programming'
        ELSE 'Design'
    END,
    CASE
        WHEN gs % 5 = 0 THEN 0.00
        ELSE (9 + gs % 90)::numeric(12,2)
    END,
    gs % 5 = 0,
    CASE gs % 3
        WHEN 0 THEN 'BEGINNER'
        WHEN 1 THEN 'INTERMEDIATE'
        ELSE 'ADVANCED'
    END,
    (3.0 + (gs % 21) / 10.0)::numeric(2,1),
    CASE
        WHEN gs % 10 = 0 THEN 'DRAFT'
        ELSE 'PUBLISHED'
    END
FROM generate_series(1, 2000) AS gs
WHERE NOT EXISTS (
    SELECT 1
    FROM courses
    WHERE title = 'L12 Benchmark Course ' || gs
);
</CodeBlock>

### Schema compatibility checkpoint

This script assumes:

- `courses.id` is database-generated.
- The listed columns exist with compatible types.
- Additional course columns are nullable or have defaults.
- The `level` and `status` columns accept the listed string values.

If your schema uses PostgreSQL native enum types, add explicit casts to those types. If additional required fields exist, supply them.

The `NOT EXISTS` clause avoids inserting a second row with the same benchmark title during ordinary sequential runs. It is **not** a concurrency-safe uniqueness guarantee.

### Load into the development database

First verify that you're connected to the intended local database:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT current_database(), current_user;"
```

Then:

```powershell
& $psql -X -v ON_ERROR_STOP=1 `
  -U learnhub_app -h localhost -d learnhub `
  -f "src/test/resources/sql/lesson12_benchmark_seed.sql"

if ($LASTEXITCODE -ne 0) {
    throw "Benchmark seed failed."
}
```

Verify:

```powershell
& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT status, COUNT(*) FROM courses WHERE title LIKE 'L12 Benchmark Course %' GROUP BY status;"
```

Expected for a fresh seed:

| Status | Count |
|---|---:|
| PUBLISHED | 1,800 |
| DRAFT | 200 |

The exact counts may differ if you've previously modified benchmark rows.

### Cleanup

When you're finished benchmarking, use this only against your local disposable development database:

```sql
DELETE FROM courses
WHERE title LIKE 'L12 Benchmark Course %';
```

If any benchmark course has enrollments, the foreign-key constraint may prevent deletion. Do not disable constraints to force cleanup.

---

## 10. Measure with `EXPLAIN ANALYZE`

A query plan describes how PostgreSQL executes a query.

`EXPLAIN ANALYZE` goes further: it actually runs the query and measures execution.

For a `SELECT`, this is safe from a data-modification perspective. For an `UPDATE` or `DELETE`, wrap it in a transaction and roll it back if you only intend to benchmark.

### 10.1 Refresh planner statistics

After inserting benchmark data:

```sql
ANALYZE courses;
ANALYZE app_users;
```

### 10.2 Benchmark substring search

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, price, rating
FROM courses
WHERE status = 'PUBLISHED'
  AND lower(title) LIKE '%benchmark%'
ORDER BY id
LIMIT 20;
```

Look for:

- `Seq Scan`
- `Bitmap Index Scan`
- `Bitmap Heap Scan`
- `Index Scan`
- `Sort`
- `Execution Time`
- `Buffers`

### 10.3 Benchmark selective search

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title
FROM courses
WHERE lower(description) LIKE '%postgresql%'
  AND status = 'PUBLISHED';
```

The planner may use the trigram index, combine indexes, or choose a sequential scan.

At 2,000 rows, a sequential scan may still be cheaper.

### 10.4 Compare with index scans discouraged

For learning only:

```sql
BEGIN;

SET LOCAL enable_seqscan = off;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title
FROM courses
WHERE lower(description) LIKE '%postgresql%'
  AND status = 'PUBLISHED';

ROLLBACK;
```

This helps you inspect an alternative plan. It does **not** prove the alternative is faster, and it should not be used as a production tuning default.

### Query plan interpretation

<box border radius="lg" padding={3} gap={2}>
  <text weight="medium">Illustrative plan choices</text>
  <grid columns={2} gap={2}>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={3} gap={1}>
        **Sequential scan**
        <text size="sm">May be efficient when the table is small or many rows match.</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={3} gap={1}>
        **Bitmap index scan**
        <text size="sm">Can help locate selective substring matches using a trigram index.</text>
      </box>
    </grid-item>
  </grid>
  <text color="secondary" size="xs">These are possible execution strategies, not guaranteed outcomes for the sample dataset.</text>
</box>

### Record your measurements

Use a simple table in your learning journal:

| Query | Plan | Rows returned | Execution time | Buffers |
|---|---|---:|---:|---|
| Title contains `benchmark` | Record result | Record | Record | Record |
| Description contains `postgresql` | Record result | Record | Record | Record |
| Category + price sort | Record result | Record | Record | Record |

Run each query multiple times. Cache warming, concurrent activity, table statistics, and hardware can all affect the result.

## 11. Offset pagination versus keyset pagination

Our existing API uses offset pagination through Spring Data `Pageable`.

For example:

```java
PageRequest.of(10, 20)
```

requests page 10, size 20. Conceptually, PostgreSQL skips the first 200 matching records and returns the next 20.

### Offset pagination

```sql
SELECT id, title, price
FROM courses
WHERE status = 'PUBLISHED'
ORDER BY price ASC, id ASC
LIMIT 20 OFFSET 200;
```

Advantages: straightforward page numbers, total counts, and direct navigation to page N.

Tradeoffs: deep offsets can become expensive, and records inserted or removed between requests can shift page boundaries.

### Keyset pagination

For the same ascending `(price, id)` ordering, suppose the last course on the current page has:

```text
price = 49.99
id = 125
```

The next page can use:

```sql
SELECT id, title, price
FROM courses
WHERE status = 'PUBLISHED'
  AND (price, id) > (49.99, 125)
ORDER BY price ASC, id ASC
LIMIT 20;
```

This asks PostgreSQL to continue after a known position rather than skip a growing number of records.

<box border radius="lg" padding={3} gap={2}>
  <text weight="medium">Pagination comparison</text>
  <table>
    <table-row><table-cell width="35%">**Property**</table-cell><table-cell>**Offset**</table-cell><table-cell>**Keyset**</table-cell></table-row>
    <table-row><table-cell>Jump to page 25</table-cell><table-cell>Natural</table-cell><table-cell>Not natural</table-cell></table-row>
    <table-row><table-cell>Deep pages</table-cell><table-cell>May slow down</table-cell><table-cell>Often more efficient</table-cell></table-row>
    <table-row><table-cell>Total count</table-cell><table-cell>Common</table-cell><table-cell>Optional</table-cell></table-row>
    <table-row><table-cell>Infinite scroll</table-cell><table-cell>Possible</table-cell><table-cell>Good fit</table-cell></table-row>
    <table-row><table-cell>Concurrent changes</table-cell><table-cell>Can shift rows</table-cell><table-cell>Reduces shifting, but not a snapshot</table-cell></table-row>
  </table>
</box>

### A complete keyset repository example

For this exercise, we'll use the default ID sort, which avoids cursor complexity involving decimal prices and mixed sort directions.

Create:

`src/main/java/com/learnhub/persistence/CourseCursorRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.model.CourseStatus;

import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.Repository;
import org.springframework.data.repository.query.Param;

import java.util.List;

/**
 * Cursor-based published catalog read.
 *
 * The caller requests size + 1 rows to determine
 * whether another page exists.
 */
public interface CourseCursorRepository
        extends Repository<CourseEntity, Long> {

    @Query("""
           SELECT c
           FROM CourseEntity c
           WHERE c.status = :status
             AND c.id > :afterId
           ORDER BY c.id ASC
           """)
    List<CourseEntity> findNextPublished(
            @Param("status") CourseStatus status,
            @Param("afterId") Long afterId,
            Pageable pageable
    );
}
</CodeBlock>

Create:

`src/main/java/com/learnhub/web/dto/CourseCursorPageResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import java.util.List;

/**
 * Cursor page without a total-count query.
 */
public record CourseCursorPageResponse(
        List<CourseDetailsResponse> items,
        Long nextCursor,
        boolean hasMore
) {
    public CourseCursorPageResponse {
        items = List.copyOf(items);
    }
}
</CodeBlock>

Create:

`src/main/java/com/learnhub/service/CourseCursorService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseCursorRepository;
import com.learnhub.web.dto.CourseCursorPageResponse;
import com.learnhub.web.dto.CourseDetailsResponse;

import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class CourseCursorService {

    private final CourseCursorRepository repository;

    public CourseCursorService(CourseCursorRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public CourseCursorPageResponse next(
            Long afterId,
            int size
    ) {
        if (size < 1 || size > 100) {
            throw new IllegalArgumentException(
                    "Size must be between 1 and 100."
            );
        }

        if (afterId != null && afterId < 0) {
            throw new IllegalArgumentException(
                    "Cursor cannot be negative."            );
        }

        List<CourseDetailsResponse> fetched =
                repository.findNextPublished(
                        CourseStatus.PUBLISHED,
                        afterId == null ? 0L : afterId,
                        PageRequest.of(0, size + 1)
                ).stream()
                 .map(CourseDetailsResponse::from)
                 .toList();

        boolean hasMore = fetched.size() > size;

        List<CourseDetailsResponse> items = hasMore
                ? fetched.subList(0, size)
                : fetched;

        Long nextCursor = hasMore && !items.isEmpty()
                ? items.getLast().id()
                : null;

        return new CourseCursorPageResponse(
                items,
                nextCursor,
                hasMore
        );
    }
}
</CodeBlock>

**Important fetching detail:** `CourseDetailsResponse.from()` accesses the optional instructor relationship. This example runs inside a transaction, so it won't cause a detached-entity error, but it **may still cause N+1 queries**.

For a high-volume cursor endpoint, replace entity fetching with a DTO projection or a carefully designed to-one fetch plan. We'll keep the cursor endpoint as a learning exercise rather than presenting it as a fully optimized public API.

### Add the cursor endpoint

Create:

`src/main/java/com/learnhub/web/CourseCursorController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.service.CourseCursorService;
import com.learnhub.web.dto.CourseCursorPageResponse;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/course-feed")
public class CourseCursorController {

    private final CourseCursorService service;

    public CourseCursorController(CourseCursorService service) {
        this.service = service;
    }

    @GetMapping
    public CourseCursorPageResponse next(
            @RequestParam(required = false) Long afterId,
            @RequestParam(defaultValue = "12") int size
    ) {
        return service.next(afterId, size);
    }
}
</CodeBlock>

We use a separate `/api/v1/course-feed` route so that the existing paginated `/api/v1/courses` contract remains unchanged.

### PowerShell cursor testing

```powershell
$feed = "http://localhost:8080/api/v1/course-feed"

$first = Invoke-RestMethod "$feed`?size=5"
$first.items | Select-Object id, title

if ($first.hasMore) {
    $second = Invoke-RestMethod `
      "$feed`?afterId=$($first.nextCursor)&size=5"

    $second.items | Select-Object id, title
}
```

Expected: IDs in the second response are greater than the final ID in the first response.

**Cursor limitations:** The cursor is a raw database ID for this lesson. Production cursors should be opaque, bound to the query's filters and sort order, and validated. Cursor pagination also doesn't guarantee a consistent historical snapshot while data changes.

## 12. Prevent N+1 queries in the main catalog

The Lesson 11 catalog currently uses:

```java
repository.findAll(specification, pageable)
          .map(CourseDetailsResponse::from);
```

The DTO mapper accesses `course.getInstructorAccount()`. Without an explicit fetch plan, that can trigger extra SQL queries.

For this many-to-one relationship, an entity graph is appropriate.

Update:

`src/main/java/com/learnhub/persistence/JpaCourseRepository.java`

Add this method declaration to the existing interface:

<CodeBlock language="java" editable>
/**
 * Fetch the optional instructor account when reading
 * a page of courses.
 *
 * This graph contains only a to-one association;
 * do not add enrollment collections to it.
 */
@Override
@EntityGraph(attributePaths = "instructorAccount")
Page<CourseEntity> findAll(
        Specification<CourseEntity> specification,
        Pageable pageable
);
</CodeBlock>

Add imports:

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.domain.Specification;
```

Spring Data JPA applies the fetch graph to the entity retrieval query. The pagination count query is handled separately.

This is an appropriate approach for the current small entity graph. For an even leaner catalog, the Lesson 10 DTO-projection approach can avoid loading complete entities.

**Checkpoint:** Verify the SQL logs. Do not assume that adding an annotation guarantees an optimal query plan in every Hibernate version.

---

## 13. Add automated PostgreSQL integration tests

We will reuse the Lesson 8–11 `PostgresIntegrationTest` base, which provides a fresh PostgreSQL Testcontainers database with Flyway migrations.

The tests below deliberately create their own data and use distinctive titles.

### 13.1 Specification and pagination tests

Create:

`src/test/java/com/learnhub/persistence/CourseSearchIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;
import com.learnhub.catalog.CourseSort;
import com.learnhub.model.CourseLevel;
import com.learnhub.service.CourseApplicationService;
import com.learnhub.web.dto.CourseCommandRequest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.HashSet;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Real PostgreSQL tests.
 *
 * Writes are rolled back after each test.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@Transactional
class CourseSearchIntegrationTest extends PostgresIntegrationTest {

    @Autowired
    private CourseApplicationService courses;

    private long createPublished(
            String title,
            String category,
            BigDecimal price,
            boolean free
    ) {
        var request = new CourseCommandRequest(
                title,
                "Lesson 12 search fixture",
                "Lesson 12 Instructor",
                category,
                price,
                free,
                CourseLevel.BEGINNER,
                new BigDecimal("4.5")
        );

        var draft = courses.createDraft(request);
        courses.publish(draft.id());

        return draft.id();
    }

    @Test
    void searchIsCaseInsensitive() {
        long id = createPublished(
                "L12 Distinctive QuArTz Course",
                "Programming",
                new BigDecimal("25.00"),
                false
        );

        var result = courses.catalog(
                "quartz",
                null,
                null,
                null,
                CourseSort.DEFAULT,
                0,
                20
        );

        assertTrue(result.items().stream()
                .anyMatch(course -> course.id() == id));
    }

    @Test
    void percentAndUnderscoreAreLiteral() {
        long id = createPublished(
                "L12 Search 50%_Complete",
                "Programming",
                new BigDecimal("25.00"),
                false
        );

        var literalMatch = courses.catalog(
                "50%_Complete",
                null,
                null,
                null,
                CourseSort.DEFAULT,
                0,
                20
        );

        assertTrue(literalMatch.items().stream()
                .anyMatch(course -> course.id() == id));

        var unrelated = courses.catalog(
                "50XComplete",
                null,
                null,
                null,
                CourseSort.DEFAULT,
                0,
                20
        );

        assertTrue(unrelated.items().stream()
                .noneMatch(course -> course.id() == id));
    }

    @Test
    void combinedFiltersApplyTogether() {
        long matchingId = createPublished(
                "L12 Unique Filter Target",
                "Programming",
                BigDecimal.ZERO,
                true
        );

        createPublished(
                "L12 Unique Filter Paid",
                "Programming",
                new BigDecimal("19.00"),
                false
        );

        var result = courses.catalog(
                "L12 Unique Filter",
                "programming",
                true,
                CourseLevel.BEGINNER,
                CourseSort.DEFAULT,
                0,
                20
        );

        assertEquals(1, result.totalItems());
        assertEquals(matchingId, result.items().getFirst().id());
    }

    @Test
    void paginationUsesStableUniqueIds() {
        String prefix = "L12 Paging Fixture";

        for (int i = 0; i < 25; i++) {
            createPublished(
                    prefix + " " + i,
                    "Programming",
                    new BigDecimal("20.00"),
                    false
            );
        }

        var first = courses.catalog(
                prefix, null, null, null,
                CourseSort.PRICE_LOW_TO_HIGH, 0, 10
        );

        var second = courses.catalog(
                prefix, null, null, null,
                CourseSort.PRICE_LOW_TO_HIGH, 1, 10
        );

        var third = courses.catalog(
                prefix, null, null, null,
                CourseSort.PRICE_LOW_TO_HIGH, 2, 10
        );

        assertEquals(25, first.totalItems());
        assertEquals(10, first.items().size());
        assertEquals(10, second.items().size());
        assertEquals(5, third.items().size());

        Set<Long> ids = new HashSet<>();

        first.items().forEach(c ->
                assertTrue(ids.add(c.id())));
        second.items().forEach(c ->
                assertTrue(ids.add(c.id())));
        third.items().forEach(c ->
                assertTrue(ids.add(c.id())));

        assertEquals(25, ids.size());
    }

    @Test
    void draftIsExcludedFromSearch() {
        var draft = courses.createDraft(
                new CourseCommandRequest(
                        "L12 Unpublished Secret",
                        "Should never appear in public search",
                        "Lesson 12 Instructor",
                        "Programming",
                        BigDecimal.ZERO,
                        true,
                        CourseLevel.BEGINNER,
                        new BigDecimal("4.5")
                )
        );

        var result = courses.catalog(
                "L12 Unpublished Secret",
                null, null, null,
                CourseSort.DEFAULT, 0, 10
        );

        assertTrue(result.items().stream()
                .noneMatch(c -> c.id() == draft.id()));
    }
}
</CodeBlock>

These five tests cover:

- Case-insensitive search.
- Literal `%` and `_` handling.
- Combined filters.
- Stable pagination.
- Published-only visibility.

They verify behavior against PostgreSQL rather than relying on mocked repository results.

### A fixture caveat

The tests assume your Testcontainers database is isolated from the development database and that the test class does not execute in parallel with another class modifying the same fixtures.

They also assume the Lesson 11 `CourseCommandRequest` and course factory satisfy all existing database constraints.

### 13.2 HTTP contract tests

Create:

`src/test/java/com/learnhub/web/CourseSearchContractTest.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * HTTP-level catalog contract tests.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class CourseSearchContractTest extends PostgresIntegrationTest {

    @Autowired
    private MockMvc mvc;

    @Test
    void catalogKeepsAngularPageContract() throws Exception {
        mvc.perform(get("/api/v1/courses")
                        .param("page", "0")
                        .param("size", "2"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.items").isArray())
                .andExpect(jsonPath("$.pageNumber").value(0))
                .andExpect(jsonPath("$.pageSize").value(2))
                .andExpect(jsonPath("$.totalItems").isNumber())
                .andExpect(jsonPath("$.totalPages").isNumber());
    }

    @Test
    void invalidPageSizeReturns400() throws Exception {
        mvc.perform(get("/api/v1/courses")
                        .param("size", "101"))
                .andExpect(status().isBadRequest());
    }

    @Test
    void negativePageReturns400() throws Exception {
        mvc.perform(get("/api/v1/courses")
                        .param("page", "-1"))
                .andExpect(status().isBadRequest());
    }

    @Test
    void invalidSortReturns400() throws Exception {
        mvc.perform(get("/api/v1/courses")
                        .param("sort", "UNRECOGNIZED"))
                .andExpect(status().isBadRequest());
    }

    @Test
    void longSearchReturns400() throws Exception {
        mvc.perform(get("/api/v1/courses")
                        .param("search", "x".repeat(101)))
                .andExpect(status().isBadRequest());
    }
}
</CodeBlock>

**Expected new tests:** 10 total (five persistence tests and five HTTP contract tests).

The full Maven test count depends on your earlier lessons.

### 13.3 Test the cursor endpoint

Create:

`src/test/java/com/learnhub/persistence/CourseCursorIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;
import com.learnhub.service.CourseCursorService;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(classes = LearnHubApplication.class)
@Transactional
class CourseCursorIntegrationTest extends PostgresIntegrationTest {

    @Autowired
    private CourseCursorService cursorService;

    @Test
    void nextPageStartsAfterPreviousCursor() {
        var first = cursorService.next(null, 2);

        assertEquals(2, first.items().size());
        assertTrue(first.hasMore());
        assertNotNull(first.nextCursor());

        var second = cursorService.next(
                first.nextCursor(),
                2
        );

        assertFalse(second.items().isEmpty());

        long lastFirstId = first.items().getLast().id();
        long firstSecondId = second.items().getFirst().id();

        assertTrue(firstSecondId > lastFirstId);
    }

    @Test
    void rejectsInvalidCursorPageSize() {
        assertThrows(
                IllegalArgumentException.class,
                () -> cursorService.next(null, 101)
        );
    }
}
</CodeBlock>

This adds two more tests, bringing the Lesson 12 reference suite to **12 new tests**.

### Run the new tests

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=CourseSearchIntegrationTest,CourseSearchContractTest,CourseCursorIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 12 tests failed."
}
```

Then:

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 12 full verification failed."
}
```

### What not to test

Avoid tests such as:

```java
assertTrue(queryTimeMillis < 10);
```

Execution time varies with CPU, disk, cache state, container startup, and system load.

Instead, test correctness automatically and record performance measurements separately.

For advanced performance testing, you can inspect PostgreSQL query plans in a controlled benchmark environment, but avoid asserting that the optimizer must choose one specific plan.

## 14. Angular catalog search integration

The Lesson 11 Angular `CourseApiService` already supports the catalog filters. We'll now connect it to a reusable standalone search component.

We'll implement:

- Debounced text search.
- Category and free/paid filters.
- Sort selection.
- Pagination.
- Loading, error, and empty states.
- Cancellation of obsolete search requests.

### 14.1 Confirm HttpClient is available

Your Angular `app.config.ts` should include:

```typescript
provideHttpClient()
```

Keep any existing interceptors and router providers.

### 14.2 Create a standalone catalog component

**File:** `learnhub/src/app/features/catalog/course-catalog-page.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  inject,
  signal,
} from '@angular/core';

import { CommonModule } from '@angular/common';
import { FormControl, ReactiveFormsModule } from '@angular/forms';

import {
  catchError,
  combineLatest,
  debounceTime,
  distinctUntilChanged,
  map,
  of,
  startWith,
  switchMap,
  tap,
} from 'rxjs';

import { takeUntilDestroyed } from
  '@angular/core/rxjs-interop';

import { CourseApiService } from
  '../../core/services/course-api.service';

import {
  CourseApiResponse,
  CourseCatalogFilters,
  CourseSort,
} from '../../core/models/course-api.model';

@Component({
  selector: 'app-course-catalog-page',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <section class="catalog">
      <h1>Explore courses</h1>

      <label>
        Search courses
        <input
          type="search"
          [formControl]="searchControl"
          placeholder="Search title, topic, instructor..."
        />
      </label>

      <label>
        Category
        <select [formControl]="categoryControl">
          <option value="">All categories</option>
          <option value="Web Development">Web Development</option>
          <option value="Programming">Programming</option>
          <option value="Design">Design</option>
          <option value="Data Science">Data Science</option>
        </select>
      </label>

      <label>
        Price
        <select [formControl]="freeControl">
          <option value="all">All courses</option>
          <option value="free">Free</option>
          <option value="paid">Paid</option>
        </select>
      </label>

      <label>
        Sort
        <select [formControl]="sortControl">
          <option value="DEFAULT">Default</option>
          <option value="PRICE_LOW_TO_HIGH">
            Price: low to high
          </option>
          <option value="PRICE_HIGH_TO_LOW">
            Price: high to low
          </option>
          <option value="RATING_HIGH_TO_LOW">
            Highest rated
          </option>
        </select>
      </label>

      @if (loading()) {
        <p role="status">Loading courses...</p>
      }

      @if (error()) {
        <p role="alert">{{ error() }}</p>
      }

      @if (!loading() && !error() && courses().length === 0) {
        <p>No courses match your filters.</p>
      }

      <div class="course-grid">
        @for (course of courses(); track course.id) {
          <article class="course-card">
            <h2>{{ course.title }}</h2>
            <p>{{ course.description }}</p>
            <p>By {{ course.instructor }}</p>
            <p>{{ course.category }}</p>

            @if (course.isFree) {
              <strong>Free</strong>
            } @else {
              <strong>{{ course.price | currency:'USD' }}</strong>
            }

            <p>Rating: {{ course.rating }}</p>
          </article>
        }
      </div>

      <nav aria-label="Catalog pagination">
        <button
          type="button"
          [disabled]="page() === 0 || loading()"
          (click)="goToPage(page() - 1)"
        >
          Previous
        </button>

        <span>
          Page {{ page() + 1 }} of {{ totalPages() }}
        </span>

        <button
          type="button"
          [disabled]="
            loading() ||
            page() + 1 >= totalPages()
          "
          (click)="goToPage(page() + 1)"
        >
          Next
        </button>
      </nav>
    </section>
  `,
})
export class CourseCatalogPageComponent {
  private readonly api = inject(CourseApiService);
  private readonly destroyRef = inject(DestroyRef);

  readonly searchControl = new FormControl('', {
    nonNullable: true,
  });

  readonly categoryControl = new FormControl('', {
    nonNullable: true,
  });

  readonly freeControl = new FormControl('all', {
    nonNullable: true,
  });

  readonly sortControl = new FormControl<CourseSort>(
    'DEFAULT',
    { nonNullable: true }
  );

  readonly courses = signal<CourseApiResponse[]>([]);
  readonly page = signal(0);
  readonly totalPages = signal(0);
  readonly loading = signal(false);
  readonly error = signal<string | null>(null);

  // Triggers explicit page changes.
  private readonly pageControl = new FormControl(0, {
    nonNullable: true,
  });

  constructor() {
    const search$ = this.searchControl.valueChanges.pipe(
      startWith(this.searchControl.value),
      debounceTime(300),
      distinctUntilChanged()
    );

    const category$ = this.categoryControl.valueChanges.pipe(
      startWith(this.categoryControl.value)
    );

    const free$ = this.freeControl.valueChanges.pipe(
      startWith(this.freeControl.value)
    );

    const sort$ = this.sortControl.valueChanges.pipe(
      startWith(this.sortControl.value)
    );

    const filters$ = combineLatest([
      search$,
      category$,
      free$,
      sort$,
    ]).pipe(
      map(([search, category, free, sort]) => ({
        search,
        category,
        free,
        sort,
      })),
      // A changed filter always restarts pagination.
      tap(() => {
        this.page.set(0);
        this.pageControl.setValue(0, { emitEvent: false });
      })
    );

    combineLatest([
      filters$,
      this.pageControl.valueChanges.pipe(
        startWith(0)
      ),
    ]).pipe(
      map(([filters, page]) => {
        const request: CourseCatalogFilters = {
          search: filters.search,
          category: filters.category,
          free: filters.free === 'all'
            ? undefined
            : filters.free === 'free',
          sort: filters.sort,
          page,
          size: 12,
        };

        return request;
      }),
      tap(() => {
        this.loading.set(true);
        this.error.set(null);
      }),
      switchMap(filters =>
        this.api.list(filters).pipe(
          catchError(() => {
            this.error.set(
              'Could not load courses. Please try again.'
            );

            return of({
              items: [],
              pageNumber: 0,
              pageSize: 12,
              totalItems: 0,
              totalPages: 0,
            });
          })
        )
      ),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(result => {
      this.courses.set(result.items);
      this.page.set(result.pageNumber);
      this.totalPages.set(result.totalPages);
      this.loading.set(false);
    });
  }

  goToPage(nextPage: number): void {
    if (
      nextPage < 0 ||
      nextPage >= this.totalPages()
    ) {
      return;
    }

    this.pageControl.setValue(nextPage);
  }
}
</CodeBlock>

### Important reactive behavior

`switchMap` cancels the previous HTTP subscription when a new search request starts.

This helps prevent a slow response to an older search from overwriting newer results.

The component resets pagination when filters change. It also preserves loading and error states.

**Integration note:** This is a standalone learning component, not an instruction to replace your existing styled LearnHub catalog wholesale. Reuse its observable pipeline in your existing catalog component if you already have card layouts, routing, favorites, or other UI behavior.

### 14.3 Register the route

In:

`learnhub/src/app/app.routes.ts`

Add the catalog route to your existing route array, adjusting the path to avoid duplicates:

```typescript
{
  path: 'courses',
  loadComponent: () =>
    import(
      './features/catalog/course-catalog-page.component'
    ).then(m => m.CourseCatalogPageComponent),
}
```

Keep your existing routes.

### 14.4 Start both applications

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

Frontend, in a second terminal:

```powershell
cd $HOME\angular-learning\learnhub

npm install
ng.cmd serve
```

Open the Angular application at `http://localhost:4200` and navigate to the catalog route.

### 14.5 Verify the Angular build

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}
```

### One reactive improvement to consider

The example combines a filter stream and a page stream. In a rapidly changing UI, resetting the page control without emitting an event can briefly combine a new filter with an old page emission before the next explicit page change.

For a more sophisticated production catalog, model filter and page events in one reducer-like state stream, so every filter change produces a single atomic state update with `page = 0`.

That's a useful follow-up exercise once the basic integration works.

## 15. API verification with PowerShell

With Spring Boot running, open another terminal:

```powershell
$base = "http://localhost:8080/api/v1/courses"
```

### Case-insensitive search

```powershell
Invoke-RestMethod "$base`?search=SPRING&page=0&size=5" |
    ConvertTo-Json -Depth 5
```

### Search with spaces

```powershell
$search = [uri]::EscapeDataString("Spring Boot")

Invoke-RestMethod "$base`?search=$search&page=0&size=5"
```

### Combined filters

```powershell
Invoke-RestMethod `
  "$base`?category=Programming&free=true&level=BEGINNER&sort=DEFAULT&page=0&size=10"
```

### Price sorting

```powershell
$result = Invoke-RestMethod `
  "$base`?sort=PRICE_LOW_TO_HIGH&page=0&size=10"

$result.items |
    Select-Object id, title, price |
    Format-Table
```

### Empty search

```powershell
Invoke-RestMethod `
  "$base`?search=unlikely-course-name-xyz&page=0&size=10"
```

Expected: an empty `items` array and zero matching items.

### Invalid page size

```powershell
try {
    Invoke-RestMethod "$base`?size=101"
}
catch {    $_.Exception.Message
    $_.ErrorDetails.Message
}
```

Expected: HTTP `400 Bad Request`.

### Cursor pagination

```powershell
$feed = "http://localhost:8080/api/v1/course-feed"

$first = Invoke-RestMethod "$feed`?size=3"
$first | ConvertTo-Json -Depth 5

if ($first.hasMore) {
    $second = Invoke-RestMethod `
      "$feed`?afterId=$($first.nextCursor)&size=3"

    $second | ConvertTo-Json -Depth 5
}
```

---

## 16. Performance and correctness review

Before declaring the catalog optimized, distinguish these concerns:

| Concern | Verification |
|---|---|
| Search correctness | Integration tests for case and literal wildcards |
| Filtering correctness | Combined-filter tests |
| Publication safety | Drafts excluded from public results |
| Pagination correctness | Stable order and unique IDs across pages |
| Query count | Hibernate SQL logs or statistics |
| Query execution strategy | PostgreSQL `EXPLAIN ANALYZE` |
| Index usefulness | Compare plans and measured work |
| Angular compatibility | HTTP JSON contract tests and Angular build |
| Concurrency behavior | Separate tests for changing data |

### Production decisions we are deliberately postponing

**Full-text search:** PostgreSQL `tsvector` and `tsquery` are often more suitable than substring search for relevance-ranked natural-language queries. They solve a different search problem.

**Advanced keyset pagination:** Supporting price ascending, price descending, and rating descending requires sort-specific cursor comparisons and appropriate indexes.

**Large catalogs:** A `Page<T>` count query can become expensive. Infinite scrolling often works better with `Slice<T>` or keyset pagination.

**Search ranking:** Our current sort modes prioritize deterministic ordering. They do not rank results by textual relevance.

**Caching:** Do not introduce caching until query correctness, invalidation requirements, and real performance bottlenecks are understood.

---

## 17. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="search" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise A — Search escaping**

      Add integration tests for backslashes, `%`, and `_` appearing in course titles. Verify they are treated as literal input and cannot broaden the search unexpectedly.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="gauge" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise B — Index comparison**

      Run the same substring query before and after adding a trigram index in a disposable database. Record execution plans, execution times, and buffer usage. Explain why a sequential scan may still win.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="database" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise C — Large catalog pagination**

      Generate 10,000 synthetic courses in a disposable database. Compare offset pagination at pages 0, 100, and 400 with keyset pagination. Record plans and timings rather than asserting a fixed performance threshold.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="arrow-down-up" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise D — Price-based keyset pagination**

      Implement a cursor for `PRICE_LOW_TO_HIGH` using `(price, id)`. Ensure equal-price courses are not skipped or duplicated. Define how the cursor encodes both values.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="monitor-smartphone" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise E — Angular catalog state**

      Refactor the filter and pagination observable streams into one state machine. Make changing any filter atomically reset the page to zero. Add a component test verifying the outgoing HTTP parameters.
    </box>
  </row>
</box>

## 18. Lesson 12 quiz

<LearningQuiz title="Lesson 12 — PostgreSQL Search and Pagination" nextQuizQuery="Give me five new intermediate-to-advanced LearnHub Lesson 12 quiz questions on PostgreSQL trigram indexes, query plans, JPA Specifications, literal LIKE escaping, and cursor pagination, without repeating the previous questions." questions={[{"id":"q1","type":"single_select","question":"A PostgreSQL catalog query uses lower(title) LIKE '%spring%'. Which index is designed to support this kind of substring search?","options":[{"value":"A","label":"A B-tree index on the raw title column","feedback":"A raw-column B-tree index generally cannot efficiently support a leading-wildcard search on lower(title)."},{"value":"B","label":"A GIN trigram expression index on lower(title)","feedback":"A GIN index with gin_trgm_ops on lower(title) can support substring matching for this expression."},{"value":"C","label":"A unique index on the course ID","feedback":"An ID index supports identity lookup, not substring matching in the title."},{"value":"D","label":"A B-tree index on the course price","feedback":"A price index does not help locate substrings in course titles."}],"correctValues":["B"],"hint":"Consider an index designed around overlapping character sequences rather than complete ordered values."},{"id":"q2","type":"true_false","question":"PostgreSQL may choose a sequential scan even when a matching trigram index exists.","options":[{"value":"true","label":"True","feedback":"The planner compares estimated costs; a sequential scan can be cheaper for small tables or queries matching many rows."},{"value":"false","label":"False","feedback":"An index is not automatically selected. PostgreSQL may estimate that scanning the table costs less."}],"correctValues":["true"],"hint":"Think about how the optimizer chooses between reading many index entries and scanning a small table."},{"id":"q3","type":"multi_select","question":"Which statements about deterministic catalog pagination are correct? Select all that apply.","options":[{"value":"A","label":"Adding a unique ID tie-breaker establishes a total sort order","feedback":"A unique final key resolves ordering ties among courses with equal primary sort values."},{"value":"B","label":"Offset pagination can shift when records are inserted between requests","feedback":"New rows can change the offsets of existing records, causing duplicates or omissions across requests."},{"value":"C","label":"Keyset pagination always provides a historical snapshot of the catalog","feedback":"Keyset pagination does not by itself freeze the dataset or guarantee snapshot consistency across requests."},{"value":"D","label":"Keyset pagination can use the previous page's final sort-key values","feedback":"The final key values can define the boundary for retrieving the next page."}],"correctValues":["A","B","D"],"hint":"Separate ordering guarantees, page navigation mechanics, and transaction snapshot guarantees."},{"id":"q4","type":"single_select","question":"Why should a LearnHub search Specification escape user-supplied percent signs and underscores before using SQL LIKE?","options":[{"value":"A","label":"To make the query use a primary-key index","feedback":"Escaping search metacharacters does not select an index or change the indexed column."},{"value":"B","label":"To prevent valid Unicode search text","feedback":"Literal wildcard escaping does not require rejecting Unicode input."},{"value":"C","label":"To treat those characters as literal search text","feedback":"Percent and underscore have wildcard meanings in LIKE; escaping them preserves literal-substring semantics."},{"value":"D","label":"To replace database parameter binding","feedback":"LIKE escaping addresses pattern semantics; parameter binding is still needed to pass user input safely."}],"correctValues":["C"],"hint":"Think about the difference between user text and the pattern language interpreted by SQL LIKE."},{"id":"q5","type":"true_false","question":"A JPA Specification that joins a many-to-one instructor association necessarily multiplies each course into multiple result rows.","options":[{"value":"true","label":"True","feedback":"A many-to-one relationship references at most one instructor per course, so that join does not inherently multiply course rows."},{"value":"false","label":"False","feedback":"A many-to-one join has at most one matching related row for each course; one-to-many joins are the common row-multiplication concern."}],"correctValues":["false"],"hint":"Compare the maximum number of matching rows on the related side for many-to-one versus one-to-many relationships."}]} />

## 19. Completion checklist

{@body const sections=[{name:"Database and migrations",items:["Verified Lesson 11 baseline","Created and applied the pg_trgm Flyway migration","Verified trigram and B-tree indexes","Loaded synthetic development data safely","Ran ANALYZE and EXPLAIN ANALYZE","Recorded query plans and timings"]},{name:"Spring Boot",items:["Updated CourseSpecifications","Verified literal wildcard escaping","Added validated search request","Preserved deterministic sorting","Added instructor fetch graph to paged reads","Implemented optional cursor-based feed"]},{name:"Testing and Angular",items:["Added PostgreSQL search integration tests","Added HTTP contract tests","Added cursor pagination tests","Ran Maven clean verify","Connected Angular catalog filters","Verified Angular loading and error states","Ran Angular production build"]},{name:"Wrap-up",items:["Completed at least two exercises","Reviewed query plans and index costs","Committed verified Lesson 12 changes"]}]}
{@body const [checked,setChecked]=DIL.useState({})}
{@body const total=sections.reduce((n,s)=>n+s.items.length,0)}
{@body const done=Object.values(checked).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 12 progress**
      <text color="secondary" size="xs">Mark items only after local verification.</text>
    </box>
    <title size="lg" tabularNums>{done}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" width={`${done/total*100}%`} height="100%" radius="full"/>
  </box>
  {#each sections as section,i}
    <box gap={2} key={i}>
      **{section.name}**
      {#each section.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!checked[`${i}-${j}`]} onChange={v=>setChecked(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if done===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 12 checklist completed.**
    </row>
  {/if}
</box>

## 20. Final verification and Git checkpoint

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 12 backend verification failed."
}

git diff --check
git status

git add src\main src\test
git diff --cached --stat
git commit -m "Optimize LearnHub catalog search and pagination"
```

Angular:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 12 Angular build failed."
}

git diff --check
git status

git add src\app
git diff --cached --stat
git commit -m "Integrate LearnHub catalog search and pagination"
```

Inspect staged changes before committing. Do not commit database credentials, local `.env` files, or generated build outputs.

### Final review

| Capability | Lesson 12 outcome |
|---|---|
| Case-insensitive search | JPA `LOWER(...) LIKE` |
| Literal substring matching | Escaped `%`, `_`, and backslash |
| Search indexes | PostgreSQL `pg_trgm` with GIN |
| Stable ordering | Whitelisted sort modes with ID tie-breakers |
| Offset pagination | Existing canonical course API |
| Keyset pagination | Separate learning endpoint |
| Database diagnostics | `EXPLAIN (ANALYZE, BUFFERS)` |
| Data volume | Synthetic development fixtures |
| Tests | PostgreSQL Testcontainers and MockMvc |
| Angular | Debounced search and paginated catalog |

The supplied code is a reference implementation continuing the Lesson 11 structure. It has not been executed against your local workspace here, so reconcile any entity fields, Flyway version numbers, and existing Angular routes before applying it.

## 21. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="shield" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 13 — Spring Security Fundamentals**

    We'll introduce Spring Security's filter chain, password hashing, authentication versus authorization, public and protected routes, security-focused integration tests, and safe preparation for session-based login.

    This is especially important because the course creation, update, publish, archive, and delete endpoints are still development-only and must not be exposed without authorization.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 13: Spring Security fundamentals, SecurityFilterChain, authentication versus authorization, BCrypt password hashing, public catalog routes, protected course management routes, secure session preparation, CSRF fundamentals, PostgreSQL user model alignment, MockMvc and Testcontainers security tests, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 12 with experienced-Java pacing.")}>Start Lesson 13 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 12 deliverable:** A LearnHub catalog with indexed substring search, dynamic filters, deterministic sorting, measured PostgreSQL query plans, an optional keyset-pagination feed, and Angular search integration.

The key engineering skill from this lesson is learning to **verify performance with evidence rather than assume an index or ORM annotation made a query faster**.