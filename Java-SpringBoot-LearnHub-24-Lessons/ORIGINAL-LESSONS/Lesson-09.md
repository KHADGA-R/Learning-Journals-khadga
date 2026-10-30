# LearnHub Java & Spring Boot Backend — Lesson 9

<text color="secondary" size="sm">SQL and PostgreSQL Deep Dive: Relational Modeling, Joins, Indexes, Transactions, Isolation, Flyway, and Query Plans</text>

**Track:** Java 21 · Spring Boot 3.5.x · PostgreSQL · Flyway · Testcontainers  
**Estimated time:** 4–6 hours, including SQL labs and tests.

In Lesson 8, we strengthened validation, transaction boundaries, and error handling. Today we'll work closer to the database.

Rather than add more controllers, we'll design the relational foundation LearnHub will eventually need for instructors, students, courses, and enrollments.

By the end of Lesson 9, you'll be able to explain **why a query is correct, why an index helps or doesn't help, and what happens when two transactions access the same data concurrently**.

## 1. Learning objectives

You'll practice:

- Relational modeling, primary keys, foreign keys, and uniqueness.
- One-to-many and many-to-many relationships.
- Inner joins, left joins, aggregation, and `HAVING`.
- Index design and `EXPLAIN (ANALYZE, BUFFERS)`.
- PostgreSQL transactions, row locks, and isolation levels.
- Safe Flyway schema evolution.
- Integration tests that verify database constraints.
- SQL exercises based on the actual LearnHub domain.

Our Lesson 9 changes are intentionally **database-focused**. We won't replace your existing Spring Data repositories or expose new user/enrollment REST endpoints yet.

## 2. The database model

<box border radius="xl" padding={3} gap={2}>
  <text weight="medium">LearnHub relational model — Lesson 9</text>
  <svg viewBox="0 0 340 315" width="100%" xmlns="http://www.w3.org/2000/svg">
    {#each [{x:8,y:10,w:140,h:91,t:"app_users",fields:["PK id","UQ email","role"]},{x:192,y:10,w:140,h:91,t:"courses",fields:["PK id","FK instructor_user_id","status, price"]},{x:102,y:205,w:145,h:91,t:"enrollments",fields:["PK id","FK student_user_id","FK course_id"]}] as b}
      <rect x={b.x} y={b.y} width={b.w} height={b.h} rx="9" fill="none" stroke="currentColor" strokeOpacity=".35"/>
      <rect x={b.x} y={b.y} width={b.w} height="28" rx="9" fill="currentColor" fillOpacity=".09"/>
      <text x={b.x+10} y={b.y+19} fill="currentColor" fontSize="12" fontWeight="600">{b.t}</text>
      {#each b.fields as f,i}
        <text x={b.x+10} y={b.y+45+i*15} fill="currentColor" fontSize="10.5">{f}</text>
      {/each}
    {/each}
    <path d="M148 49 H192" fill="none" stroke="currentColor" strokeOpacity=".6" strokeWidth="1.5"/>
    <text x="154" y="40" fill="currentColor" fontSize="10">1 : N</text>
    <path d="M65 101 V155 H143 V205" fill="none" stroke="currentColor" strokeOpacity=".6" strokeWidth="1.5"/>
    <path d="M262 101 V155 H207 V205" fill="none" stroke="currentColor" strokeOpacity=".6" strokeWidth="1.5"/>
    <text x="76" y="148" fill="currentColor" fontSize="10">student</text>
    <text x="225" y="148" fill="currentColor" fontSize="10">course</text>
    <text x="12" y="311" fill="currentColor" fontSize="10" opacity=".7">An enrollment connects one student to one course.</text>
  </svg>
  <text color="secondary" size="xs">Conceptual diagram. Actual constraints and relationships are defined in the SQL migration below.</text>
</box>

### Relationship design

| Relationship | Cardinality | Implementation |
|---|---|---|
| Instructor → courses | One-to-many | `courses.instructor_user_id` |
| Student → enrollments | One-to-many | `enrollments.student_user_id` |
| Course → enrollments | One-to-many | `enrollments.course_id` |
| Students ↔ courses | Many-to-many | Through `enrollments` |

Why not simply store a list of student IDs in a `courses` column?

Because enrollment is a domain concept with its own properties: enrollment date, completion status, payment entitlement, and progress. It deserves a separate table.

### Important scope boundary

These are **schema foundations**, not a complete account or enrollment implementation. Authentication, authorization, and enrollment APIs come later. The existing `courses.instructor` text field remains for compatibility with the current Java code.

---

## 3. Prepare your environment

Start with your Lesson 8 backend.

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info
git status
```

Run the baseline build:

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 8 baseline failed. Resolve it first."
}
```

Create a checkpoint if needed:

```powershell
git add .
git commit -m "Checkpoint before LearnHub SQL deep dive"
```

### Connect to PostgreSQL

Locate `psql.exe` if needed:

```powershell
Get-ChildItem "C:\Program Files\PostgreSQL" `
  -Recurse -Filter psql.exe `
  -ErrorAction SilentlyContinue |
  Select-Object FullName
```

Set the path according to your installation:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -p 5432 -d learnhub
```

Inside `psql`, enable readable output:

```sql
\x auto
\timing on
\pset null '(NULL)'
```

Inspect the existing schema:

```sql
\dt
\d courses

SELECT version, description, success
FROM flyway_schema_history
ORDER BY installed_rank;
```

Your Lesson 7 database should already contain the `courses` table and Flyway history.

## 4. SQL foundations: keys and constraints

### Primary key

A primary key uniquely identifies a row and cannot be null.

```sql
id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY
```

PostgreSQL generates an ID when the application omits it.

### Foreign key

A foreign key requires a referenced row to exist.

```sql
instructor_user_id BIGINT
    REFERENCES app_users(id)
```

This protects referential integrity. A course cannot reference a nonexistent instructor account.

### Unique constraint

An enrollment should be unique for a student/course pair:

```sql
UNIQUE (student_user_id, course_id)
```

That prevents duplicate enrollments even when two requests race.

### Check constraint

A check constraint restricts allowed values:

```sql
CHECK (role IN ('STUDENT', 'INSTRUCTOR', 'ADMIN'))
```

It complements Java validation.

### Referential actions

Consider deleting a course with enrollments.

- `ON DELETE CASCADE` would automatically delete the enrollments.
- `ON DELETE RESTRICT` or `NO ACTION` prevents deleting a referenced course unless dependent records are handled first.
- `ON DELETE SET NULL` preserves dependent rows but removes the reference, if the column permits null.

For LearnHub, we'll **restrict deletion of enrolled courses** rather than silently erase students' enrollment records.

## 5. Create the Lesson 9 Flyway migration

We will add two tables and an optional instructor relationship to `courses`.

We will **not** create real user accounts yet. The user records in this lesson are database fixtures for learning SQL, not authenticated accounts.

### Migration plan

| Version | Description |
|---|---|
| V1 | Create `courses` |
| V2 | Seed original course catalog |
| V3 | Create users and enrollments, add instructor relationship |

If you've already created a V3 migration in an optional earlier exercise, use the **next unused version number** for this migration. Flyway migration versions must be unique.

Create:

`src/main/resources/db/migration/V3__create_users_and_enrollments.sql`

<CodeBlock language="sql" editable>
-- Lesson 9: Relational foundations for LearnHub.
--
-- These tables are schema preparation only.
-- They do not implement authentication or enrollment APIs.

CREATE TABLE app_users (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,

    email VARCHAR(254) NOT NULL,
    display_name VARCHAR(120) NOT NULL,
    role VARCHAR(20) NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_app_users_role
        CHECK (role IN ('STUDENT', 'INSTRUCTOR', 'ADMIN')),

    CONSTRAINT chk_app_users_email_not_blank
        CHECK (length(btrim(email)) > 0),

    CONSTRAINT chk_app_users_display_name_not_blank
        CHECK (length(btrim(display_name)) > 0)
);

-- Email comparison is case-insensitive for uniqueness.
CREATE UNIQUE INDEX uq_app_users_email_lower
    ON app_users (lower(email));

-- Nullable during migration from instructor display names
-- to real instructor identities.
ALTER TABLE courses
    ADD COLUMN instructor_user_id BIGINT;

ALTER TABLE courses
    ADD CONSTRAINT fk_courses_instructor
    FOREIGN KEY (instructor_user_id)
    REFERENCES app_users(id)
    ON DELETE RESTRICT;

CREATE TABLE enrollments (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,

    student_user_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,

    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',

    CONSTRAINT fk_enrollments_student
        FOREIGN KEY (student_user_id)
        REFERENCES app_users(id)
        ON DELETE RESTRICT,

    CONSTRAINT fk_enrollments_course
        FOREIGN KEY (course_id)
        REFERENCES courses(id)
        ON DELETE RESTRICT,

    CONSTRAINT uq_enrollments_student_course
        UNIQUE (student_user_id, course_id),

    CONSTRAINT chk_enrollments_status
        CHECK (status IN ('ACTIVE', 'COMPLETED', 'CANCELLED'))
);

-- PostgreSQL does not automatically index every
-- referencing column in a foreign key.
CREATE INDEX idx_courses_instructor_user_id
    ON courses (instructor_user_id);

CREATE INDEX idx_enrollments_course_id
    ON enrollments (course_id);

-- The unique constraint already supplies an index
-- beginning with student_user_id.
-- Do not add a redundant single-column index yet.
</CodeBlock>

### Why is `instructor_user_id` nullable?

Our five seeded courses have instructor display names, not authenticated user IDs. Making the new column immediately `NOT NULL` would break the migration.

A safe future migration will follow this sequence:

1. Create instructor accounts.
2. Associate existing courses with their instructor accounts.
3. Verify every course has a valid instructor.
4. Add `NOT NULL` when the application is ready to enforce it.

This is an example of **expand → backfill → constrain**, a common production migration pattern.

### Why no password column?

We're designing relational structure, not authentication. Password hashing, account lifecycle, and credentials belong to the security lessons.

### Why no automatic seed users?

Production schema migrations should not silently create fake students or instructors. We'll put SQL practice data in a separate lab script, not a Flyway migration.

## 6. Apply the migration

Ensure your database environment variables are configured, then start Spring Boot:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

Flyway should apply the new migration before Hibernate validates the schema.

In another terminal, connect using `psql`:

```powershell
& $psql -U learnhub_app -h localhost -d learnhub
```

Inspect:

```sql
\dt
\d app_users
\d enrollments
\d courses

SELECT version, description, success
FROM flyway_schema_history
ORDER BY installed_rank;
```

Expected: `app_users` and `enrollments` exist, and the new migration appears in Flyway history.

**Compatibility note:** We have not added a JPA relationship to `CourseEntity` yet. Hibernate's `ddl-auto: validate` tolerates extra database columns, so the existing course API can continue to work while the schema evolves.

---

## 7. Create a safe SQL practice environment

We want realistic joins and enrollment queries without modifying the development catalog.

We'll create a separate schema named `lesson9_lab`, containing copies of the relevant table structures and isolated practice records.

Create:

`sql/lesson09/setup_lab.sql`

PowerShell:

```powershell
New-Item -ItemType Directory -Force sql\lesson09 |
    Out-Null
```

<CodeBlock language="sql" editable>
-- Isolated SQL practice schema.
-- Do not run against production.

DROP SCHEMA IF EXISTS lesson9_lab CASCADE;
CREATE SCHEMA lesson9_lab;

CREATE TABLE lesson9_lab.users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email VARCHAR(254) NOT NULL UNIQUE,
    name VARCHAR(120) NOT NULL,
    role VARCHAR(20) NOT NULL
        CHECK (role IN ('STUDENT', 'INSTRUCTOR'))
);

CREATE TABLE lesson9_lab.courses (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title VARCHAR(150) NOT NULL,
    instructor_id BIGINT NOT NULL
        REFERENCES lesson9_lab.users(id),
    price NUMERIC(12, 2) NOT NULL CHECK (price >= 0),
    status VARCHAR(20) NOT NULL
        CHECK (status IN ('DRAFT', 'PUBLISHED'))
);

CREATE TABLE lesson9_lab.enrollments (
    student_id BIGINT NOT NULL
        REFERENCES lesson9_lab.users(id),
    course_id BIGINT NOT NULL
        REFERENCES lesson9_lab.courses(id),
    enrolled_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (student_id, course_id)
);

INSERT INTO lesson9_lab.users (email, name, role)
VALUES
    ('alex@example.test', 'Alex Morgan', 'INSTRUCTOR'),
    ('sarah@example.test', 'Sarah Lee', 'INSTRUCTOR'),
    ('maya@example.test', 'Maya Patel', 'STUDENT'),
    ('jordan@example.test', 'Jordan Chen', 'STUDENT'),
    ('riley@example.test', 'Riley Brooks', 'STUDENT');

INSERT INTO lesson9_lab.courses
    (title, instructor_id, price, status)
VALUES
    ('Angular Complete Guide', 1, 49.99, 'PUBLISHED'),
    ('Spring Boot Fundamentals', 1, 59.99, 'PUBLISHED'),
    ('Python for Beginners', 2, 0.00, 'PUBLISHED'),
    ('Advanced Hibernate', 2, 79.99, 'DRAFT');

INSERT INTO lesson9_lab.enrollments
    (student_id, course_id)
VALUES
    (3, 1),
    (3, 3),
    (4, 1),
    (4, 2),
    (5, 3);
</CodeBlock>

This script is **destructive only to the `lesson9_lab` schema**: rerunning it drops and recreates that practice schema. Never use the lab script in a production database.

Run it from PowerShell:

```powershell
& $psql `
  -U learnhub_app `
  -h localhost `
  -d learnhub `
  -v ON_ERROR_STOP=1 `
  -f ".\sql\lesson09\setup_lab.sql"

if ($LASTEXITCODE -ne 0) {
    throw "SQL lab setup failed."
}
```

The `ON_ERROR_STOP=1` option ensures that a SQL error produces a failing process exit status rather than quietly continuing.

## 8. SQL lab: joins

### 8.1 INNER JOIN — find enrolled students

```sql
SELECT
    u.name AS student,
    c.title AS course
FROM lesson9_lab.enrollments e
JOIN lesson9_lab.users u
    ON u.id = e.student_id
JOIN lesson9_lab.courses c
    ON c.id = e.course_id
ORDER BY u.name, c.title;
```

Expected: five enrollment rows.

An inner join returns only rows with matching records on both sides.

### 8.2 LEFT JOIN — include courses with no enrollments

```sql
SELECT
    c.title,
    COUNT(e.student_id) AS enrollment_count
FROM lesson9_lab.courses c
LEFT JOIN lesson9_lab.enrollments e
    ON e.course_id = c.id
GROUP BY c.id, c.title
ORDER BY c.id;
```

Expected:

| Course | Enrollments |
|---|---:|
| Angular Complete Guide | 2 |
| Spring Boot Fundamentals | 1 |
| Python for Beginners | 2 |
| Advanced Hibernate | 0 |

The `LEFT JOIN` preserves the draft course with zero enrollments.

**Common mistake:** using `COUNT(*)` would report `1` for an unmatched course row. `COUNT(e.student_id)` correctly reports zero because that column is null in the unmatched joined row.

### 8.3 Find students who have not enrolled

```sql
SELECT
    u.id,
    u.name
FROM lesson9_lab.users u
LEFT JOIN lesson9_lab.enrollments e
    ON e.student_id = u.id
WHERE u.role = 'STUDENT'
  AND e.student_id IS NULL;
```

With the current lab fixtures, this returns no rows. All three students have at least one enrollment.

Try inserting another student with no enrollment, then rerun the query.

### 8.4 Join through multiple relationships

Find each instructor's students:

```sql
SELECT DISTINCT
    instructor.name AS instructor,
    student.name AS student
FROM lesson9_lab.courses c
JOIN lesson9_lab.users instructor
    ON instructor.id = c.instructor_id
JOIN lesson9_lab.enrollments e
    ON e.course_id = c.id
JOIN lesson9_lab.users student
    ON student.id = e.student_id
ORDER BY instructor.name, student.name;
```

`DISTINCT` matters because one student may enroll in multiple courses taught by the same instructor.

## 9. Aggregation, GROUP BY, and HAVING

### Count published courses by instructor

```sql
SELECT
    u.name AS instructor,
    COUNT(*) AS published_courses
FROM lesson9_lab.courses c
JOIN lesson9_lab.users u
    ON u.id = c.instructor_id
WHERE c.status = 'PUBLISHED'
GROUP BY u.id, u.name
ORDER BY published_courses DESC, instructor;
```

### Find courses with at least two enrollments

```sql
SELECT
    c.title,
    COUNT(e.student_id) AS enrollment_count
FROM lesson9_lab.courses c
JOIN lesson9_lab.enrollments e
    ON e.course_id = c.id
GROUP BY c.id, c.title
HAVING COUNT(e.student_id) >= 2
ORDER BY enrollment_count DESC, c.title;
```

Expected: Angular Complete Guide and Python for Beginners.

**`WHERE` versus `HAVING`:** `WHERE` filters input rows before grouping; `HAVING` filters groups after aggregation.

### Calculate enrollment totals by category

Our lab courses intentionally have no category column. Use the real `courses` table to explore category distribution:

```sql
SELECT
    category,
    COUNT(*) AS course_count,
    ROUND(AVG(price), 2) AS average_price
FROM courses
WHERE status = 'PUBLISHED'
GROUP BY category
ORDER BY course_count DESC, category;
```

This is a read-only query against the real development schema.

---

## 10. Indexes and query performance

An index helps PostgreSQL locate matching rows without scanning every row, but it has costs:

- Additional disk storage.
- More work on inserts, updates, and deletes.
- Maintenance and vacuum overhead.
- Potentially no benefit for very small tables or low-selectivity queries.

### 10.1 Inspect existing indexes

```sql
SELECT
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'public'
  AND tablename IN ('courses', 'app_users', 'enrollments')
ORDER BY tablename, indexname;
```

You should see indexes created by primary keys, unique constraints, and our Flyway migrations.

### 10.2 Read an execution plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, price
FROM courses
WHERE status = 'PUBLISHED'
ORDER BY id
LIMIT 10;
```

Important plan fields:

| Field | Meaning |
|---|---|
| `Seq Scan` | Reads table pages sequentially |
| `Index Scan` | Uses an index to locate rows and fetch table data |
| `Index Only Scan` | May satisfy columns from an index without fetching every heap row |
| `actual time` | Measured execution time for plan nodes |
| `rows` | Rows produced or estimated |
| `Buffers: shared hit` | Pages found in PostgreSQL's shared buffer cache |
| `Buffers: shared read` | Pages read into shared buffers |

With only five or six courses, a sequential scan may be faster than using an index. That is normal, not a database failure.

### 10.3 Composite indexes

Suppose a common query is:

```sql
SELECT id, title, price
FROM courses
WHERE status = 'PUBLISHED'
  AND category = 'Web Development'
ORDER BY id
LIMIT 20;
```

An index such as `(status, category, id)` may be useful because it matches the filtering and ordering pattern.

Our existing Lesson 7 index is `(category, status)`. It may help filtering but does not necessarily satisfy the ordering requirement.

**Don't create indexes solely from intuition.** Measure representative queries with realistic data and inspect their plans.

### 10.4 An optional index experiment

Inside the isolated lab schema:

```sql
CREATE INDEX idx_lab_courses_status_price
ON lesson9_lab.courses (status, price);
```

Then:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, price
FROM lesson9_lab.courses
WHERE status = 'PUBLISHED'
ORDER BY price;
```

The planner may still prefer a sequential scan because the table is tiny.

You can experiment with:

```sql
BEGIN;
SET LOCAL enable_seqscan = off;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, price
FROM lesson9_lab.courses
WHERE status = 'PUBLISHED'
ORDER BY price;

ROLLBACK;
```

This is an **educational experiment**, not a production tuning recommendation. Forcing a plan can demonstrate index access, but it does not prove that the forced plan is faster.

## 11. Transactions and ACID

A transaction groups database operations into a unit of work.

The four ACID properties are:

| Property | Meaning in LearnHub |
|---|---|
| Atomicity | Course and metadata updates commit together or roll back |
| Consistency | Database constraints remain satisfied |
| Isolation | Concurrent transactions observe data according to the chosen isolation level |
| Durability | Committed changes survive application restart |

### 11.1 Practice COMMIT and ROLLBACK

Run this inside `psql`:

```sql
BEGIN;

INSERT INTO lesson9_lab.users
    (email, name, role)
VALUES
    ('temporary@example.test', 'Temporary Student', 'STUDENT');

SELECT *
FROM lesson9_lab.users
WHERE email = 'temporary@example.test';

ROLLBACK;
```

Verify the row is gone:

```sql
SELECT *
FROM lesson9_lab.users
WHERE email = 'temporary@example.test';
```

Now repeat with `COMMIT` instead of `ROLLBACK`.

The row should remain until you delete it or reset the lab.

### 11.2 Savepoints

Savepoints let you roll back part of a transaction:

```sql
BEGIN;

INSERT INTO lesson9_lab.users
    (email, name, role)
VALUES
    ('student-a@example.test', 'Student A', 'STUDENT');

SAVEPOINT before_second_student;

INSERT INTO lesson9_lab.users
    (email, name, role)
VALUES
    ('student-b@example.test', 'Student B', 'STUDENT');

ROLLBACK TO SAVEPOINT before_second_student;

COMMIT;
```

Student A is committed. Student B is not.

Savepoints are useful in specific workflows, but they should not be used to hide unexpected business failures.

## 12. Isolation levels and concurrent access

PostgreSQL supports these SQL isolation levels:

| Level | PostgreSQL behavior |
|---|---|
| Read Uncommitted | Behaves like Read Committed |
| Read Committed | Each statement sees a snapshot of committed data |
| Repeatable Read | Transaction uses a stable snapshot; serialization failures are possible |
| Serializable | Prevents nonserializable outcomes, with retries required for some conflicts |

PostgreSQL's default is **Read Committed**.

### Two-terminal experiment

Open two separate `psql` sessions, A and B.

First, reset the lab if you've changed the sample data:

```powershell
& $psql -U learnhub_app -h localhost -d learnhub `
  -v ON_ERROR_STOP=1 `
  -f ".\sql\lesson09\setup_lab.sql"
```

### Session A

```sql
BEGIN;

UPDATE lesson9_lab.courses
SET price = 54.99
WHERE id = 1;

-- Leave this transaction open.
```

### Session B

```sql
SELECT price
FROM lesson9_lab.courses
WHERE id = 1;
```

Session B sees the last committed value, normally `49.99`, not Session A's uncommitted `54.99`.

Now, in Session B, try:

```sql
BEGIN;

SET LOCAL lock_timeout = '3s';

UPDATE lesson9_lab.courses
SET price = 59.99
WHERE id = 1;
```

The second update waits for Session A's row lock. If Session A does not finish within three seconds, PostgreSQL cancels Session B's statement with a lock-timeout error.

After a statement error inside an explicit PostgreSQL transaction, issue:

```sql
ROLLBACK;
```

Then in Session A:

```sql
COMMIT;
```

This demonstrates row-level write contention.

### Why this matters for LearnHub

Imagine two students attempting to enroll in the same limited-capacity course, or two instructors editing the same draft.

Transactions alone don't automatically implement business rules for capacity or conflict detection. Those workflows may require row locks, optimistic versioning, unique constraints, or retry logic.

## 13. A practical uniqueness race

Our `enrollments` table has:

```sql
UNIQUE (student_user_id, course_id)
```

Suppose two concurrent requests both try to insert the same enrollment.

Both may check for existence and initially see no matching row. But the database's unique constraint still prevents both inserts from committing successfully.

This illustrates why a pattern like:

```java
if (!repository.existsByStudentAndCourse(studentId, courseId)) {
    repository.save(enrollment);
}
```

is **not sufficient by itself** to prevent duplicates under concurrency.

The database constraint is the final safeguard.

In the future enrollment service, we'll combine application checks with database uniqueness and a clear API response.

## 14. Add PostgreSQL constraint integration tests

We'll test the **real V3 schema** using Testcontainers, without relying on manually inserted development data.

The tests use JDBC because we're verifying SQL constraints directly, not JPA mapping behavior.

### 14.1 Test dependencies

Your Lesson 7 `pom.xml` should already include:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

Keep the existing Testcontainers JUnit dependency if already present.

### 14.2 Create the test

**File:** `src/test/java/com/learnhub/persistence/LearnHubSchemaTest.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.*;
import static org.springframework.dao.support.DataAccessUtils.*;

import org.springframework.dao.DataIntegrityViolationException;

/**
 * Tests PostgreSQL constraints created by Flyway V3.
 *
 * Uses the shared Testcontainers database.
 * Each test rolls back its own inserts.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@Transactional
class LearnHubSchemaTest extends PostgresIntegrationTest {

    @Autowired
    private JdbcTemplate jdbc;

    private long insertUser(
            String email,
            String name,
            String role
    ) {
        Long id = jdbc.queryForObject(
                """
                INSERT INTO app_users
                    (email, display_name, role)
                VALUES (?, ?, ?)
                RETURNING id
                """,
                Long.class,
                email,
                name,
                role
        );

        return id;
    }

    @Test
    void allowsValidStudentAndInstructor() {
        long studentId = insertUser(
                "student9@example.test",
                "Student Nine",
                "STUDENT"
        );

        long instructorId = insertUser(
                "instructor9@example.test",
                "Instructor Nine",
                "INSTRUCTOR"
        );

        assertTrue(studentId > 0);
        assertTrue(instructorId > 0);
    }

    @Test
    void rejectsInvalidRole() {
        assertThrows(
                DataIntegrityViolationException.class,
                () -> insertUser(
                        "invalid9@example.test",
                        "Invalid Role",
                        "SUPERUSER"
                )
        );
    }

    @Test
    void rejectsDuplicateEmailIgnoringCase() {
        insertUser(
                "Example9@LearnHub.test",
                "First User",
                "STUDENT"
        );

        assertThrows(
                DataIntegrityViolationException.class,
                () -> insertUser(
                        "example9@learnhub.test",
                        "Second User",
                        "STUDENT"
                )
        );
    }
    @Test
    void rejectsEnrollmentWithMissingStudent() {
        assertThrows(
                DataIntegrityViolationException.class,
                () -> jdbc.update(
                        """
                        INSERT INTO enrollments
                            (student_user_id, course_id)
                        VALUES (?, ?)
                        """,
                        99999999L,
                        1L
                )
        );
    }

    @Test
    void rejectsDuplicateEnrollment() {
        long studentId = insertUser(
                "duplicate9@example.test",
                "Duplicate Test Student",
                "STUDENT"
        );

        jdbc.update(
                """
                INSERT INTO enrollments
                    (student_user_id, course_id)
                VALUES (?, ?)
                """,
                studentId,
                1L
        );

        assertThrows(
                DataIntegrityViolationException.class,
                () -> jdbc.update(
                        """
                        INSERT INTO enrollments
                            (student_user_id, course_id)
                        VALUES (?, ?)
                        """,
                        studentId,
                        1L
                )
        );
    }

    @Test
    void preventsDeletingEnrolledCourse() {
        long studentId = insertUser(
                "delete9@example.test",
                "Deletion Test Student",
                "STUDENT"
        );

        jdbc.update(
                """
                INSERT INTO enrollments
                    (student_user_id, course_id)
                VALUES (?, ?)
                """,
                studentId,
                1L
        );

        assertThrows(
                DataIntegrityViolationException.class,
                () -> jdbc.update(
                        "DELETE FROM courses WHERE id = ?",
                        1L
                )
        );
    }
}
</CodeBlock>

Remove the unused static import from `DataAccessUtils` if your IDE flags it.

### A PostgreSQL testing detail

When PostgreSQL raises a constraint violation inside a transaction, that transaction becomes aborted until rollback.

That's fine for these individual tests because Spring rolls back the test transaction afterward. **Do not try to execute another SQL statement in the same transaction after the expected failure**, unless you use a savepoint or a separate transaction.

Also note that `JdbcTemplate.queryForObject()` with `INSERT ... RETURNING` may depend on the overload and driver behavior. If your Spring JDBC setup does not accept this usage, use `jdbc.queryForObject(sql, Long.class, args...)` as shown above or a `PreparedStatement` with generated keys.

### What these six tests verify

They test actual database guarantees:

- Valid user roles.
- Rejection of invalid roles.
- Case-insensitive email uniqueness.
- Foreign-key integrity.
- Duplicate enrollment prevention.
- Restricted deletion of enrolled courses.

They do **not** prove that a student is authenticated or that an instructor is authorized. Those remain application-security concerns.

## 15. A production-oriented migration review

Before proceeding, inspect the V3 migration for future operational considerations.

**Email uniqueness:** `lower(email)` prevents case-only duplicates. It does not normalize Unicode, trim whitespace, or define a complete email-address identity policy. We'll handle normalization and validation in the account layer.

**Role integrity:** the database ensures a role string belongs to the allowed set, but the foreign key does not ensure `courses.instructor_user_id` references a user with role `INSTRUCTOR`. That's a business rule for the service layer or a more specialized database design.

**Enrollment eligibility:** the schema permits enrollment in a draft or paid course without proof of payment. Future enrollment services must enforce publishing and entitlement rules.

**Deletion:** existing Lesson 8 course deletion code may now encounter a foreign-key violation when deleting a course with enrollments. The current general error handler does not translate that specific persistence failure into a stable conflict response. We'll address that when enrollment APIs are introduced.

**Indexing:** the composite enrollment uniqueness constraint already supports lookup by student ID and course ID together. We added a separate course-ID index because that column is second in the unique index.

**Migrations:** never modify an already-applied V3 migration. If you need to change constraints later, create V4 or the next unused migration.

## 16. Run verification

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=LearnHubSchemaTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 9 schema tests failed."
}
```

Then run the complete suite:

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 9 verification failed."
}
```

These are **six new expected tests**, not a claim that they've passed on your machine.

Inspect the reports:

```powershell
Get-ChildItem target\surefire-reports -Filter *.txt |
    ForEach-Object {
        Get-Content $_.FullName |
            Select-String "Tests run:|Failures:|Errors:"
    }
```

For this lesson, no Angular code changes are required. The existing course API contract should remain unchanged.

## 17. Hands-on SQL exercises

Complete these against `lesson9_lab`, not your main application tables.

<box gap={3}>
  <row gap={3} align="start">
    <icon name="table" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise A — Student enrollment report**

      Write a query returning each student, their total enrollment count, and their total enrolled course price. Include students with zero enrollments. Sort by enrollment count descending.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="search" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise B — Courses without students**

      Return every course with zero enrollments using both `LEFT JOIN ... IS NULL` and `NOT EXISTS`. Compare the results and execution plans.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="gauge" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise C — Index investigation**

      Create an index on `(status, price)` in the lab schema. Compare plans before and after the index, then explain why a tiny table might still use a sequential scan.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="lock-keyhole" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise D — Concurrent enrollment**

      Open two database sessions and attempt to insert the same student/course enrollment concurrently. Observe blocking, uniqueness enforcement, and transaction rollback.
    </box>
  </row>
</box>

### Exercise A — Reference solution

Try solving it before expanding the solution.

{@body const [showSolution,setShowSolution]=DIL.useState(false)}
<box border radius="lg" padding={3} gap={2}>
  <button block color="secondary" variant="outline" onClick={()=>setShowSolution(v=>!v)}>
    {showSolution?"Hide reference solution":"Show reference solution"}
    <icon name={showSolution?"chevron-up":"chevron-down"} inline/>
  </button>
  {#if showSolution}
    <CodeBlock language="sql">
SELECT
    u.name AS student,
    COUNT(e.course_id) AS enrollment_count,
    COALESCE(SUM(c.price), 0) AS total_course_price
FROM lesson9_lab.users u
LEFT JOIN lesson9_lab.enrollments e
    ON e.student_id = u.id
LEFT JOIN lesson9_lab.courses c
    ON c.id = e.course_id
WHERE u.role = 'STUDENT'
GROUP BY u.id, u.name
ORDER BY enrollment_count DESC, student;
    </CodeBlock>
    <text color="secondary" size="sm">The LEFT JOIN preserves students without enrollments. COALESCE turns a null sum into zero. This sums listed prices, not verified payment amounts.</text>
  {/if}
</box>

## 18. Lesson 9 knowledge quiz

{@body const quiz=[{q:"Which database constraint prevents a student from enrolling in the same course twice?",opts:["CHECK on course price","UNIQUE(student_user_id, course_id)","Foreign key on instructor_user_id","DEFAULT CURRENT_TIMESTAMP"],a:1,why:"A composite unique constraint enforces uniqueness across the student/course pair."},{q:"Which join preserves courses that have zero enrollments?",opts:["INNER JOIN","CROSS JOIN","LEFT JOIN from courses to enrollments","JOIN without an ON clause"],a:2,why:"A left join retains every course even when no enrollment row matches."},{q:"Why might PostgreSQL choose a sequential scan despite an available index?",opts:["Indexes never work on small tables","A sequential scan can be cheaper for a small table or broad query","Flyway disables indexes","PostgreSQL only uses indexes for primary keys"],a:1,why:"The query planner estimates the cost of different plans and chooses the cheaper one."},{q:"What does PostgreSQL do when a transaction violates a foreign-key constraint?",opts:["Silently deletes the referencing row","Ignores the constraint","Raises an error and leaves the transaction aborted until rollback or savepoint recovery","Automatically creates the missing parent"],a:2,why:"Constraint violations fail the statement and require transaction recovery."},{q:"Why do we keep instructor_user_id nullable in the initial migration?",opts:["Foreign keys must always be nullable","Existing seeded courses have no linked user records yet","JPA cannot map nonnull columns","Nullable keys are faster"],a:1,why:"A phased migration lets us add the relationship before backfilling existing data and enforcing NOT NULL."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
{@body const score=quiz.filter((q,i)=>answers[i]===q.a).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **SQL and PostgreSQL knowledge check**
    <text size="xs" color="secondary">{Object.keys(answers).length} of 5 answered</text>
  </row>
  {#each quiz as q,i}
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
        <text size="sm" color="secondary">{q.why}</text>
      {/if}
    </box>
    {#if i<quiz.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if graded}
    <box background="surface-secondary" radius="md" padding={3} gap={1}>
      <title size="xl" tabularNums>{score}/5</title>
      <text size="sm">{score===5?"All answers correct.":"Review the explanations and retry."}</text>
    </box>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==5} onClick={()=>setGraded(true)}>Check answers</button>
    <button color="secondary" variant="outline" onClick={()=>{setAnswers({});setGraded(false)}}>Reset</button>
  </row>
</box>

## 19. Lesson completion checklist

{@body const groups=[{name:"Database modeling",items:["Verified Lesson 8 baseline","Explained primary and foreign keys","Created V3 migration","Applied Flyway migration successfully","Inspected tables and constraints"]},{name:"SQL practice",items:["Created isolated lesson9_lab schema","Practiced INNER JOIN and LEFT JOIN","Practiced GROUP BY and HAVING","Compared query plans with EXPLAIN ANALYZE","Inspected indexes and their tradeoffs"]},{name:"Transactions and concurrency",items:["Practiced COMMIT and ROLLBACK","Practiced SAVEPOINT","Observed row-lock contention","Explained PostgreSQL isolation levels","Understood unique constraints under concurrency"]},{name:"Testing and completion",items:["Added six schema integration tests","Verified database constraint behavior","Ran Maven clean verify","Completed at least two SQL exercises","Committed Lesson 9 changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((s,g)=>s+g.items.length,0)}
{@body const count=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 9 progress**
      <text color="secondary" size="xs">Mark items only after verifying them locally.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${100*count/total}%`} radius="full"/>
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
      **Lesson 9 checklist completed.**
    </row>
  {/if}
</box>

## 20. Final Git checkpoint

Run the final verification:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 9 verification failed."
}

git status
git add `
  src\main\resources\db\migration `
  src\test\java\com\learnhub\persistence\LearnHubSchemaTest.java `
  sql\lesson09

git commit -m "Add LearnHub relational schema and PostgreSQL SQL labs"
```

The lab SQL contains only synthetic example identities. Do not commit real student records or database credentials.

## 21. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="network" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 10 — Spring Data JPA Relationships, Hibernate Fetching, and Transactions**

    We'll map the new relational schema into Java entities, explore `@ManyToOne` and `@OneToMany`, understand lazy loading and the N+1 query problem, and build efficient repository queries.

    We'll also examine why directly serializing JPA entities into Angular API responses is risky and how DTO projections help.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 10: Spring Data JPA relationships, @ManyToOne and @OneToMany, Hibernate lazy loading, N+1 query problems, entity graphs, JPQL, DTO projections, pagination, transaction boundaries, Testcontainers integration tests, and Angular-safe response DTOs. Continue from Lesson 9 with experienced-Java pacing, exact file paths, complete code, Windows PowerShell commands, exercises, quiz, and checklist.")}>Start Lesson 10 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 9 deliverable:** LearnHub now has a relational schema foundation for users, instructor ownership, and enrollments, plus a safe SQL practice environment and database constraint tests.

The key milestone is understanding that **data integrity belongs in the database as well as the application**, while authorization and business workflows remain responsibilities of the backend services.