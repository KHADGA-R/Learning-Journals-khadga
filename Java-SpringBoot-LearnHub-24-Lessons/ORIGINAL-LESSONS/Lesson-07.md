# LearnHub Java & Spring Boot Backend — Lesson 7

<text color="secondary" size="sm">PostgreSQL, Spring Data JPA, Hibernate, Flyway Migrations, Database-Generated IDs, and Angular Course Compatibility</text>

<row gap={2} wrap="wrap">
  <badge>Lesson 7 of 24</badge>
  <badge>Java 21 + Spring Boot 3.5</badge>
  <badge>Experienced Java · Steady Pace</badge>
</row>

In Lesson 6, you built a working REST API for LearnHub courses. It supports listing, searching, creating drafts, updating, publishing, archiving, and deleting courses.

But there's one major limitation: **every course change disappears when the backend restarts**.

In this lesson, we'll replace the in-memory repository with PostgreSQL, introduce Spring Data JPA and Hibernate, and manage the database schema with Flyway. We'll also extend the stored course model to include the fields your Angular frontend needs.

## What you'll build

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    <icon name="monitor-smartphone" size="lg"/>
    **Angular LearnHub**
    <text color="secondary" size="xs">Course catalog, search, and details</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring MVC CourseController**
    <text color="secondary" size="xs">REST requests and response DTOs</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Course services + repository interface**
    <text color="secondary" size="xs">Business rules and persistence abstraction</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring Data JPA + Hibernate**
    <text color="secondary" size="xs">Entities, SQL operations, transactions</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    <icon name="database" size="lg"/>
    **PostgreSQL**
    <text color="secondary" size="xs">Durable course data and generated IDs</text>
  </box>
</box>

By the end, you should be able to restart Spring Boot and find that created courses still exist.

### Learning objectives

- Install and configure PostgreSQL on Windows.
- Create a dedicated LearnHub database and restricted application account.
- Understand entities, persistence contexts, and repositories.
- Introduce Spring Data JPA without discarding your existing domain design.
- Replace in-memory storage with a PostgreSQL adapter.
- Use Flyway to version and apply schema migrations.
- Generate IDs in PostgreSQL rather than calculating `MAX(id) + 1`.
- Persist `thumbnail`, `durationHours`, and `lastUpdated`.
- Test database persistence and restart durability.
- Update the Angular adapter to consume real backend metadata.

**Estimated time:** 4–6 hours, depending on PostgreSQL installation and testing setup.

---

## 1. Verify the Lesson 6 baseline

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 6 verification failed."
}
```

Create a checkpoint:

```powershell
git status
git add .
git commit -m "Checkpoint before PostgreSQL integration"
```

If Git reports nothing to commit, continue.

We will preserve the existing controller, service interfaces, DTOs, and domain model where practical.

### The architectural decision

Our Lesson 4 `Course` is an immutable Java record. Hibernate generally expects persistent entity classes with an accessible no-argument constructor and mutable persistence state.

Rather than turn the domain record into a JPA entity, we'll use **two representations**:

| Representation | Responsibility |
|---|---|
| `Course` domain record | Business rules and immutable domain state |
| `CourseEntity` JPA class | Database mapping and persistence |
| `CourseResponse` DTO | HTTP contract for Angular |

A mapper converts between the domain record and the JPA entity.

This keeps Hibernate-specific annotations out of the domain model and preserves the value of our earlier Java lessons.

## 2. Install PostgreSQL on Windows

<row align="start" gap={3}>
  <AsyncImage query="PostgreSQL elephant logo database software official logo" aspectRatio="1:1" maxWidth="110px"/>
  <box flex="1" gap={2}>
    **<Entity category="software" value="PostgreSQL" disambig="open-source relational database"/>**

    PostgreSQL is a relational database that supports transactions, foreign keys, constraints, indexes, JSON data types, and more.

    We'll use it as LearnHub's primary database.

    <Link url="https://www.postgresql.org/download/windows/" title="Official PostgreSQL Windows downloads"/>
  </box>
</row>

Use a currently supported PostgreSQL major version compatible with your Windows environment. PostgreSQL 17 or 18 is suitable for this course.

### Option A — Install with the official Windows installer

1. Open the official PostgreSQL Windows download page.
2. Download the Windows installer.
3. Run the installer.
4. Include PostgreSQL Server and Command Line Tools.
5. Keep port `5432` unless another PostgreSQL instance already uses it.
6. Set a strong password for the administrative `postgres` account.
7. Finish the installation.

Remember the administrator password; we'll use it once to create the application account.

### Option B — Install using winget

First check availability:

```powershell
winget search PostgreSQL
```

Select the current PostgreSQL server package shown in the results and install it with the exact package ID reported by `winget`.

For example, if the package listing includes PostgreSQL 17:

```powershell
winget install --id PostgreSQL.PostgreSQL.17 -e
```

The package version and installer prompts can change, so verify the available ID before running the command.

### Verify the installation

Open a new PowerShell terminal:

```powershell
Get-Service |
    Where-Object {
        $_.Name -like "postgresql*"
    } |
    Format-Table Name, Status
```

Check whether port `5432` is listening:

```powershell
Get-NetTCPConnection `
  -LocalPort 5432 `
  -State Listen `
  -ErrorAction SilentlyContinue
```

Find `psql.exe` if it is not in your `PATH`:

```powershell
Get-ChildItem `
  "C:\Program Files\PostgreSQL" `
  -Recurse `
  -Filter psql.exe `
  -ErrorAction SilentlyContinue |
  Select-Object FullName
```

For PostgreSQL 17, the executable is commonly located at:

```text
C:\Program Files\PostgreSQL\17\bin\psql.exe
```

Set a PowerShell variable:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql --version
```

Adjust the path to your installed version.

### Connect as the PostgreSQL administrator

```powershell
& $psql -U postgres -h localhost -p 5432
```

Enter the password you chose during installation.

You should see a prompt similar to:

```text
postgres=#
```

## 3. Create the LearnHub database

Run these commands inside `psql`:

```sql
CREATE ROLE learnhub_app
WITH LOGIN PASSWORD 'CHANGE_THIS_TO_A_STRONG_PASSWORD';

CREATE DATABASE learnhub
OWNER learnhub_app
ENCODING 'UTF8';
```

Choose a strong local password instead of the placeholder.

Verify:

```sql
\l
\du
```

Exit:

```sql
\q
```

### Test the application account

From PowerShell:

```powershell
& $psql `
  -U learnhub_app `
  -h localhost `
  -p 5432 `
  -d learnhub
```

Inside `psql`:

```sql
SELECT current_database(), current_user;
```

Expected values:

```text
learnhub | learnhub_app
```

Exit:

```sql
\q
```

### Why a dedicated database account?

Your Spring Boot application should not connect using the PostgreSQL superuser.

The `learnhub_app` account owns the local LearnHub database and can create schema objects for Flyway migrations. In a production deployment, we would normally separate migration privileges from runtime application privileges.

For local development, one dedicated non-superuser account keeps setup manageable.

## 4. Add Spring Data JPA, PostgreSQL, and Flyway

Open:

`$HOME\angular-learning\learnhub-backend\pom.xml`

Keep the Spring Boot parent, Spring Web, Validation, Actuator, and Test dependencies from Lessons 5–6.

Add these dependencies inside `<dependencies>`:

<CodeBlock language="xml" editable>
<!-- JPA repository support and Hibernate ORM. -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- PostgreSQL JDBC driver. -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Flyway schema migration engine. -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>

<!-- Flyway's PostgreSQL database support. -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>

<!-- Database integration testing in isolated containers. -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
</CodeBlock>

Spring Boot's dependency management supplies compatible library versions.

Verify Maven resolution:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn dependency:tree

if ($LASTEXITCODE -ne 0) {
    throw "Dependency resolution failed."
}
```

### JPA versus Hibernate versus Spring Data JPA

| Technology | Responsibility |
|---|---|
| Jakarta Persistence (JPA) | Standard persistence API and mapping annotations |
| Hibernate | ORM implementation that generates SQL and manages entities |
| Spring Data JPA | Repository abstractions built on JPA |
| PostgreSQL JDBC driver | Connects Java to PostgreSQL |
| Flyway | Applies versioned database migrations |

Spring Data JPA does not replace Hibernate, and Hibernate does not replace PostgreSQL. They work together.

## 5. Configure the database connection securely

Do not commit database passwords to `application.yml`.

### Step 5.1 — Set local environment variables

In the PowerShell terminal used to launch Spring Boot:

```powershell
$env:LEARNHUB_DB_URL = "jdbc:postgresql://localhost:5432/learnhub"
$env:LEARNHUB_DB_USER = "learnhub_app"

$securePassword = Read-Host `
  "Enter LearnHub database password" `
  -AsSecureString

$env:LEARNHUB_DB_PASSWORD = [System.Net.NetworkCredential]::new(
  "",
  $securePassword
).Password

Remove-Variable securePassword
```

The password is now available to child processes launched from that terminal. It is not written to a configuration file, although environment variables are still readable by sufficiently privileged local processes.

### Step 5.2 — Update `application.yml`

File: `src/main/resources/application.yml`

<CodeBlock language="yaml" editable>
spring:
  application:
    name: learnhub-backend

  datasource:
    url: ${LEARNHUB_DB_URL}
    username: ${LEARNHUB_DB_USER}
    password: ${LEARNHUB_DB_PASSWORD}

  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true

  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: never

logging:
  level:
    root: INFO
    com.learnhub: DEBUG
    org.hibernate.SQL: DEBUG
```

The important choice is:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

Hibernate checks that the mapped tables and columns exist, but **does not create or modify the schema**.

Flyway owns schema changes.

This prevents Hibernate and Flyway from competing over database structure.

## 6. Create the first Flyway migration

Create the migration directory:

```powershell
cd $HOME\angular-learning\learnhub-backend

New-Item -ItemType Directory -Force `
  src\main\resources\db\migration |
  Out-Null
```

File:

`src/main/resources/db/migration/V1__create_courses.sql`

<CodeBlock language="sql" editable>
-- LearnHub schema migration V1.
--
-- Course IDs are generated by PostgreSQL.
-- The application does not calculate MAX(id) + 1.

CREATE TABLE courses (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,

    title VARCHAR(150) NOT NULL,
    description VARCHAR(3000) NOT NULL,
    instructor VARCHAR(100) NOT NULL,
    category VARCHAR(100) NOT NULL,

    price NUMERIC(12, 2) NOT NULL,
    rating NUMERIC(3, 2) NOT NULL,

    is_free BOOLEAN NOT NULL,
    level VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL,

    thumbnail VARCHAR(1000),
    duration_hours NUMERIC(8, 2),
    last_updated DATE NOT NULL DEFAULT CURRENT_DATE,

    CONSTRAINT chk_courses_price
        CHECK (price >= 0),

    CONSTRAINT chk_courses_rating
        CHECK (rating BETWEEN 0 AND 5),

    CONSTRAINT chk_courses_duration
        CHECK (
            duration_hours IS NULL
            OR duration_hours >= 0
        ),
    CONSTRAINT chk_courses_free_price
        CHECK (
            (is_free = TRUE AND price = 0)
            OR
            (is_free = FALSE AND price > 0)
        ),

    CONSTRAINT chk_courses_level
        CHECK (
            level IN (
                'BEGINNER',
                'INTERMEDIATE',
                'ADVANCED'
            )
        ),

    CONSTRAINT chk_courses_status
        CHECK (
            status IN (
                'DRAFT',
                'PUBLISHED',
                'ARCHIVED'
            )
        )
);

CREATE INDEX idx_courses_status
    ON courses (status);

CREATE INDEX idx_courses_category_status
    ON courses (category, status);

CREATE INDEX idx_courses_level_status
    ON courses (level, status);
</CodeBlock>

### Why use an identity column?

This declaration:

```sql
id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY
```

lets PostgreSQL allocate IDs.

It avoids the race condition in the Lesson 6 implementation:

```java
long nextId = maxId + 1;
```

Two concurrent requests can no longer independently choose the same next ID through application logic.

### Why constraints matter

Java validation catches invalid requests early, but it is not sufficient by itself.

Database constraints protect data when writes originate from different services, scripts, administrative tools, or concurrent transactions.

Our table enforces basic price, rating, level, status, and free-course consistency rules.

## 7. Seed the original LearnHub catalog

We want the first four published courses and fifth draft course to remain available after migration.

Create:

`src/main/resources/db/migration/V2__seed_courses.sql`

<CodeBlock language="sql" editable>
-- Development seed data for the initial LearnHub catalog.
-- IDs 1–5 preserve the lesson's existing sample identifiers.

INSERT INTO courses (
    id,
    title,
    description,
    instructor,
    category,
    price,
    rating,
    is_free,
    level,
    status,
    thumbnail,
    duration_hours,
    last_updated
)
VALUES
(
    1,
    'Angular Complete Guide',
    'Build modern Angular applications.',
    'Alex Morgan',
    'Web Development',
    49.99,
    4.80,
    FALSE,
    'BEGINNER',
    'PUBLISHED',
    'https://placehold.co/600x350?text=Angular',
    42.50,
    DATE '2026-09-15'
),
(
    2,
    'Python Fundamentals',
    'Learn Python programming from the ground up.',
    'Jordan Lee',
    'Programming',
    0.00,
    4.70,
    TRUE,
    'BEGINNER',
    'PUBLISHED',
    'https://placehold.co/600x350?text=Python',
    18.00,
    DATE '2026-08-10'
),
(
    3,
    'UI UX Design Essentials',
    'Learn user interface and user experience design.',
    'Taylor Kim',
    'Design',
    39.99,
    4.60,
    FALSE,
    'INTERMEDIATE',
    'PUBLISHED',
    'https://placehold.co/600x350?text=UI-UX',
    24.00,
    DATE '2026-07-20'
),
(
    4,
    'Data Science Introduction',
    'Explore the fundamentals of data analysis.',
    'Sam Rivera',
    'Data Science',
    0.00,
    4.50,
    TRUE,
    'BEGINNER',
    'PUBLISHED',
    'https://placehold.co/600x350?text=Data-Science',
    16.00,
    DATE '2026-06-12'
),
(
    5,
    'Advanced Spring Boot',
    'Build advanced Spring Boot services.',
    'Alex Morgan',
    'Web Development',
    79.99,
    0.00,
    FALSE,
    'ADVANCED',
    'DRAFT',
    NULL,
    NULL,
    CURRENT_DATE
);

-- Advance the identity sequence beyond explicit seed IDs.
SELECT setval(
    pg_get_serial_sequence('courses', 'id'),
    (SELECT MAX(id) FROM courses),
    TRUE
);
</CodeBlock>

These are **lesson seed records**, not imported real marketplace courses. Their metadata is illustrative.

The explicit IDs preserve the earlier exercise assumptions. The sequence adjustment ensures the next generated ID is greater than `5`.

### Flyway migration rules

Once a migration has been applied to a shared database, do not edit it to change the schema.

Instead, create a new migration:

```text
V3__add_course_indexes.sql
V4__create_enrollments.sql
```

Flyway records applied migrations and their checksums in `flyway_schema_history`.

For this lesson, V1 and V2 are new and have not yet been applied.

## 8. Create the JPA entity

We'll keep the immutable domain `Course` record and introduce a separate persistence class.

Create the directory:

```powershell
New-Item -ItemType Directory -Force `
  src\main\java\com\learnhub\persistence |
  Out-Null
```

File:

`src/main/java/com/learnhub/persistence/CourseEntity.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;

import jakarta.persistence.*;

import java.math.BigDecimal;
import java.time.LocalDate;

/**
 * Hibernate-managed database representation.
 *
 * This class is intentionally separate from
 * the immutable Course domain record.
 */
@Entity
@Table(name = "courses")
public class CourseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 150)
    private String title;

    @Column(nullable = false, length = 3000)
    private String description;

    @Column(nullable = false, length = 100)
    private String instructor;

    @Column(nullable = false, length = 100)
    private String category;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Column(nullable = false, precision = 3, scale = 2)
    private BigDecimal rating;

    @Column(name = "is_free", nullable = false)
    private boolean free;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private CourseLevel level;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private CourseStatus status;

    @Column(length = 1000)
    private String thumbnail;

    @Column(name = "duration_hours", precision = 8, scale = 2)
    private BigDecimal durationHours;

    @Column(name = "last_updated", nullable = false)
    private LocalDate lastUpdated;

    /**
     * Required by JPA.
     */
    protected CourseEntity() {
    }

    /**
     * Factory for a new course.
     *
     * ID remains null so PostgreSQL generates it.
     */
    public static CourseEntity newDraft(
            String title,
            String description,
            String instructor,
            String category,
            BigDecimal price,
            BigDecimal rating,
            boolean free,
            CourseLevel level,
            String thumbnail,
            BigDecimal durationHours
    ) {
        CourseEntity entity = new CourseEntity();

        entity.title = title;
        entity.description = description;
        entity.instructor = instructor;
        entity.category = category;
        entity.price = price;
        entity.rating = rating;
        entity.free = free;
        entity.level = level;
        entity.status = CourseStatus.DRAFT;
        entity.thumbnail = thumbnail;
        entity.durationHours = durationHours;
        entity.lastUpdated = LocalDate.now();

        return entity;
    }

    public void replaceDraftFields(
            String title,
            String description,
            String instructor,
            String category,
            BigDecimal price,
            BigDecimal rating,
            boolean free,
            CourseLevel level,
            String thumbnail,
            BigDecimal durationHours
    ) {
        this.title = title;
        this.description = description;
        this.instructor = instructor;
        this.category = category;
        this.price = price;
        this.rating = rating;
        this.free = free;
        this.level = level;
        this.thumbnail = thumbnail;
        this.durationHours = durationHours;
        this.lastUpdated = LocalDate.now();
    }

    public void changeStatus(CourseStatus newStatus) {
        this.status = newStatus;
        this.lastUpdated = LocalDate.now();
    }

    public Long getId() {
        return id;
    }

    public String getTitle() {
        return title;
    }

    public String getDescription() {
        return description;
    }

    public String getInstructor() {
        return instructor;
    }

    public String getCategory() {
        return category;
    }

    public BigDecimal getPrice() {
        return price;
    }

    public BigDecimal getRating() {
        return rating;
    }

    public boolean isFree() {
        return free;
    }

    public CourseLevel getLevel() {
        return level;
    }

    public CourseStatus getStatus() {
        return status;
    }

    public String getThumbnail() {
        return thumbnail;
    }

    public BigDecimal getDurationHours() {
        return durationHours;
    }

    public LocalDate getLastUpdated() {
        return lastUpdated;
    }
}
</CodeBlock>

### Why `Long` instead of `long`?

Before persistence, the entity has no database-generated ID.

A nullable `Long` distinguishes an unsaved entity (`id == null`) from a persisted one.

### Why `EnumType.STRING`?

Storing `PUBLISHED` is safer and more readable than storing an enum's ordinal position.

If the enum order changes, ordinal-based values can silently acquire different meanings.

### Why use `BigDecimal`?

Course prices and ratings require predictable decimal representation. PostgreSQL's `NUMERIC` type maps naturally to Java's `BigDecimal`.

## 9. Create the Spring Data JPA repository

File:

`src/main/java/com/learnhub/persistence/JpaCourseRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.model.CourseStatus;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

/**
 * Spring Data generates the repository implementation.
 */
public interface JpaCourseRepository
        extends JpaRepository<CourseEntity, Long> {

    List<CourseEntity> findByStatus(
            CourseStatus status
    );

    List<CourseEntity> findByStatusAndCategoryIgnoreCase(
            CourseStatus status,
            String category
    );

    boolean existsByTitleIgnoreCase(
            String title
    );
}
</CodeBlock>

Spring Data JPA provides methods such as:

```java
findAll()
findById(id)
save(entity)
deleteById(id)
count()
```

You do not write SQL for these basic operations.

Spring Data also derives queries from method names, such as `findByStatus()`.

For complex catalog search, we'll later use specifications or explicit JPQL/SQL queries rather than loading every row into memory.

## 10. Preserve the domain repository abstraction

Our existing `CourseRepository` interface is valuable because the catalog service does not need to know whether data comes from a Java map or PostgreSQL.

We will keep it and add one operation for database-generated IDs.

### Step 10.1 — Introduce a creation command

File:

`src/main/java/com/learnhub/model/NewCourse.java`

<CodeBlock language="java" editable>
package com.learnhub.model;

import java.math.BigDecimal;

/**
 * Domain-level input for creating a course.
 *
 * There is deliberately no ID: PostgreSQL owns
 * ID generation.
 */
public record NewCourse(
        String title,
        String description,
        String instructor,
        String category,
        BigDecimal price,
        boolean free,
        CourseLevel level,
        BigDecimal rating,
        String thumbnail,
        BigDecimal durationHours
) {}
</CodeBlock>

### Step 10.2 — Extend CourseRepository

Open:

`src/main/java/com/learnhub/repository/CourseRepository.java`

Retain all existing methods and add:

```java
/**
 * Persists a new draft with a storage-generated ID.
 */
Course create(NewCourse newCourse);
```

Add the import:

```java
import com.learnhub.model.NewCourse;
```

The repository contract should now include at least:

<CodeBlock language="java" editable>
package com.learnhub.repository;

import com.learnhub.model.Course;
import com.learnhub.model.NewCourse;

import java.util.List;
import java.util.Optional;

/**
 * Storage-independent course repository contract.
 */
public interface CourseRepository {

    List<Course> findAll();

    Optional<Course> findById(long id);

    Course save(Course course);

    void deleteById(long id);

    Course create(NewCourse newCourse);
}
</CodeBlock>

If your Lesson 4 interface includes additional methods, retain them and implement them in the PostgreSQL adapter as well.

### Step 10.3 — Keep the in-memory implementation compiling

We still want the old in-memory repository available for plain Java tests.

Add this method to:

`src/main/java/com/learnhub/repository/InMemoryCourseRepository.java`

<CodeBlock language="java" editable>
@Override
public synchronized Course create(NewCourse input) {

    long nextId = findAll().stream()
            .mapToLong(Course::id)
            .max()
            .orElse(0L) + 1L;

    Course draft = new Course(
            nextId,
            input.title(),
            input.description(),
            input.instructor(),
            input.category(),
            input.price(),
            input.free(),
            input.level(),
            input.rating(),
            CourseStatus.DRAFT
    );

    return save(draft);
}
</CodeBlock>

Add the necessary imports for `NewCourse` and `CourseStatus`.

This retains the in-memory behavior for tests. The PostgreSQL implementation will use a database identity column instead.

## 11. Create the JPA adapter

File:

`src/main/java/com/learnhub/persistence/PostgresCourseRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.model.Course;
import com.learnhub.model.CourseStatus;
import com.learnhub.model.NewCourse;
import com.learnhub.repository.CourseRepository;

import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

/**
 * PostgreSQL-backed implementation of the domain
 * repository interface.
 *
 * The adapter translates between immutable domain
 * records and mutable JPA entities.
 */
@Repository
@Primary
@Transactional
public class PostgresCourseRepository
        implements CourseRepository {

    private final JpaCourseRepository jpa;

    public PostgresCourseRepository(
            JpaCourseRepository jpa
    ) {
        this.jpa = jpa;
    }

    @Override
    @Transactional(readOnly = true)
    public List<Course> findAll() {
        return jpa.findAll().stream()
                .map(this::toDomain)
                .toList();
    }

    @Override
    @Transactional(readOnly = true)
    public Optional<Course> findById(long id) {
        return jpa.findById(id)
                .map(this::toDomain);
    }

    @Override
    public Course create(NewCourse input) {
        CourseEntity entity = CourseEntity.newDraft(
                input.title(),
                input.description(),
                input.instructor(),
                input.category(),
                input.price(),
                input.rating(),
                input.free(),
                input.level(),
                input.thumbnail(),
                input.durationHours()
        );

        return toDomain(jpa.saveAndFlush(entity));
    }

    @Override
    public Course save(Course course) {
        CourseEntity entity = jpa.findById(course.id())
                .orElseThrow(
                        () -> new IllegalArgumentException(
                                "Course does not exist: "
                                        + course.id()
                        )
                );

        entity.replaceDraftFields(
                course.title(),
                course.description(),
                course.instructor(),
                course.category(),
                course.price(),
                course.rating(),
                course.free(),
                course.level(),
                entity.getThumbnail(),
                entity.getDurationHours()
        );

        entity.changeStatus(course.status());

        return toDomain(jpa.saveAndFlush(entity));
    }

    @Override
    public void deleteById(long id) {
        jpa.deleteById(id);
    }

    private Course toDomain(CourseEntity entity) {
        return new Course(
                entity.getId(),
                entity.getTitle(),
                entity.getDescription(),
                entity.getInstructor(),
                entity.getCategory(),
                entity.getPrice(),
                entity.isFree(),
                entity.getLevel(),
                entity.getRating(),
                entity.getStatus()
        );
    }
}
</CodeBlock>

### Important limitation of this first adapter

The existing domain `Course` does not yet contain `thumbnail`, `durationHours`, or `lastUpdated`.

The adapter preserves those database values during ordinary `save()` operations, but the HTTP DTOs need a way to read them.

We'll address that with a dedicated course metadata read service in Section 13.

Also, the adapter's `save()` method updates the entity's timestamp even when only the status changes. That's acceptable for the first persistence lesson, but later we'll make audit timestamps more precise.

### Transaction boundary

The adapter uses `@Transactional` for individual repository operations.

For a complete business operation involving multiple repository calls, the transaction should generally be placed at the application service boundary.

We'll introduce that refinement as enrollment and purchase workflows become more complex.

## 12. Switch Spring Boot from in-memory storage to PostgreSQL

In Lesson 5, we registered the seeded in-memory repository using:

`src/main/java/com/learnhub/config/CourseRepositoryConfig.java`

That configuration must no longer create an in-memory repository bean.

**Delete `CourseRepositoryConfig.java`**, or remove its `@Bean` method entirely.

PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

Remove-Item `
  src\main\java\com\learnhub\config\CourseRepositoryConfig.java
```

Do not delete the `CourseRepository` interface or `InMemoryCourseRepository` class.

Our runtime bean will now be:

```text
CourseRepository
    |
    +-- PostgresCourseRepository (@Repository)
            |
            +-- JpaCourseRepository
                    |
                    +-- PostgreSQL
```

Spring Data automatically creates the `JpaCourseRepository` implementation.

### Replace the Lesson 6 ID allocation

Open:

`src/main/java/com/learnhub/service/CourseManagementService.java`

Find the method:

```java
public synchronized Course createDraft(
        CourseWriteRequest request
)
```

Replace that method with:

<CodeBlock language="java" editable>
/**
 * Creates a new draft using a database-generated ID.
 *
 * The web DTO is converted into a domain command.
 * The repository owns persistence and ID allocation.
 */
public Course createDraft(
        CourseWriteRequest request
) {
    Objects.requireNonNull(request);

    NewCourse input = new NewCourse(
            request.title(),
            request.description(),
            request.instructor(),
            request.category(),
            request.price(),
            request.isFree(),
            request.level(),
            request.rating(),
            null,
            null
    );

    return repository.create(input);
}
</CodeBlock>

Add:

```java
import com.learnhub.model.NewCourse;
```

We no longer use:

```java
repository.findAll().stream().max(...)
```

to generate IDs.

The database generates them safely.

### Check the earlier createDraft(Course) method

The Lesson 4 `createDraft(Course)` method can remain for your plain-Java tests.

However, it should not be used for new PostgreSQL-backed course creation, because it expects a caller-supplied ID and calls `save()`.

Our REST controller uses the `CourseWriteRequest` overload, which now delegates to `repository.create()`.

This distinction is temporary; in a later refactor we'll consolidate creation around the domain command.

## 13. Expose persisted Angular metadata

We need the API to return real values for:

- `thumbnail`
- `durationHours`
- `lastUpdated`

Rather than change the existing domain record and every Lesson 1–6 constructor call, we'll introduce a focused read service.

### 13.1 Create CourseMetadata

File:

`src/main/java/com/learnhub/persistence/CourseMetadata.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import java.math.BigDecimal;
import java.time.LocalDate;

/**
 * Persisted presentation metadata for a course.
 */
public record CourseMetadata(
        String thumbnail,
        BigDecimal durationHours,
        LocalDate lastUpdated
) {
}
</CodeBlock>

### 13.2 Create CourseMetadataService

File:

`src/main/java/com/learnhub/persistence/CourseMetadataService.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Map;
import java.util.List;
import java.util.function.Function;
import java.util.stream.Collectors;

/**
 * Reads metadata that is not yet represented
 * by the Lesson 4 domain Course record.
 */
@Service
public class CourseMetadataService {

    private final JpaCourseRepository repository;

    public CourseMetadataService(
            JpaCourseRepository repository
    ) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public CourseMetadata findByCourseId(long id) {
        return repository.findById(id)
                .map(this::toMetadata)
                .orElse(null);
    }

    @Transactional(readOnly = true)
    public Map<Long, CourseMetadata> findAllByIds(
            List<Long> ids
    ) {
        return repository.findAllById(ids)
                .stream()
                .collect(Collectors.toMap(
                        CourseEntity::getId,
                        this::toMetadata
                ));
    }

    private CourseMetadata toMetadata(
            CourseEntity entity
    ) {
        return new CourseMetadata(
                entity.getThumbnail(),
                entity.getDurationHours(),
                entity.getLastUpdated()
        );
    }
}
</CodeBlock>

The bulk lookup avoids a separate metadata query for each course in a catalog page.

### 13.3 Extend CourseResponse

Replace:

`src/main/java/com/learnhub/web/dto/CourseResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.model.Course;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseMetadata;

import java.math.BigDecimal;
import java.time.LocalDate;

/**
 * API representation aligned with Angular LearnHub.
 */
public record CourseResponse(
        long id,
        String title,
        String description,
        String instructor,
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

    public static CourseResponse from(
            Course course,
            CourseMetadata metadata
    ) {
        return new CourseResponse(
                course.id(),
                course.title(),
                course.description(),
                course.instructor(),
                course.category(),
                course.price(),
                course.rating(),
                course.free(),
                course.level(),
                course.status(),
                metadata == null ? null : metadata.thumbnail(),
                metadata == null ? null : metadata.durationHours(),
                metadata == null ? null : metadata.lastUpdated()
        );
    }

    /**
     * Compatibility overload for earlier unit tests.
     */
    public static CourseResponse from(Course course) {        return from(course, null);
    }
}
</CodeBlock>

### 13.4 Extend CoursePageResponse

Replace:

`src/main/java/com/learnhub/web/dto/CoursePageResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.catalog.PageResult;
import com.learnhub.model.Course;
import com.learnhub.persistence.CourseMetadata;

import java.util.List;
import java.util.Map;

/**
 * Paginated course response for Angular.
 */
public record CoursePageResponse(
        List<CourseResponse> items,
        long totalItems,
        int pageNumber,
        int pageSize,
        int totalPages
) {

    public CoursePageResponse {
        items = List.copyOf(items);
    }

    public static CoursePageResponse from(
            PageResult<Course> page,
            Map<Long, CourseMetadata> metadataById
    ) {
        List<CourseResponse> items = page.items()
                .stream()
                .map(course -> CourseResponse.from(
                        course,
                        metadataById.get(course.id())
                ))
                .toList();

        return new CoursePageResponse(
                items,
                page.totalItems(),
                page.pageNumber(),
                page.pageSize(),
                page.totalPages()
        );
    }
}
</CodeBlock>

### 13.5 Update CourseController

Open:

`src/main/java/com/learnhub/web/CourseController.java`

Add imports:

```java
import com.learnhub.persistence.CourseMetadataService;
import com.learnhub.catalog.PageResult;
import java.util.Map;
```

Add a field:

```java
private final CourseMetadataService metadataService;
```

Update the constructor:

```java
public CourseController(
        CourseCatalogService catalogService,
        CourseManagementService managementService,
        CourseMetadataService metadataService
) {
    this.catalogService = catalogService;
    this.managementService = managementService;
    this.metadataService = metadataService;
}
```

Replace the return statement inside the existing `list()` method with:

<CodeBlock language="java" editable>
PageResult<Course> result = catalogService.search(
        filter,
        CourseSort.DEFAULT,
        page,
        size
);

List<Long> ids = result.items()
        .stream()
        .map(Course::id)
        .toList();

Map<Long, CourseMetadata> metadata =
        metadataService.findAllByIds(ids);

return CoursePageResponse.from(result, metadata);
</CodeBlock>

Add:

```java
import com.learnhub.persistence.CourseMetadata;
```

Update the single-course response:

```java
return CourseResponse.from(
        course,
        metadataService.findByCourseId(course.id())
);
```

Update the write endpoints (`create`, `update`, `publish`, and `archive`) so their response mapping uses:

```java
CourseResponse.from(
        course,
        metadataService.findByCourseId(course.id())
)
```

For example, the `publish()` method becomes:

```java
@PatchMapping("/{id}/publish")
public CourseResponse publish(@PathVariable long id) {
    Course course = managementService.publish(id);

    return CourseResponse.from(
            course,
            metadataService.findByCourseId(course.id())
    );
}
```

The remaining endpoints follow the same pattern.

### Important: metadata for newly created courses

At this point, the database stores metadata for the seeded courses, but our Lesson 6 request DTO does not accept thumbnail or duration values.

We'll fix that next.

## 14. Accept thumbnail and duration in write requests

Replace:

`src/main/java/com/learnhub/web/dto/CourseWriteRequest.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.model.Course;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;

import jakarta.validation.constraints.*;

import java.math.BigDecimal;

/**
 * Course creation and draft-update request.
 */
public record CourseWriteRequest(

        @NotBlank
        @Size(max = 150)
        String title,

        @NotBlank
        @Size(max = 3000)
        String description,

        @NotBlank
        @Size(max = 100)
        String instructor,

        @NotBlank
        @Size(max = 100)
        String category,

        @NotNull
        @DecimalMin("0.00")
        BigDecimal price,

        @NotNull
        Boolean isFree,

        @NotNull
        CourseLevel level,

        @NotNull
        @DecimalMin("0.0")
        @DecimalMax("5.0")
        BigDecimal rating,

        @Size(max = 1000)
        String thumbnail,

        @DecimalMin("0.0")
        BigDecimal durationHours
) {

    public Course toDraft(long id) {
        return new Course(
                id,
                title,
                description,
                instructor,
                category,
                price,
                isFree,
                level,
                rating,
                CourseStatus.DRAFT
        );
    }
}
</CodeBlock>

Now update `CourseManagementService.createDraft(CourseWriteRequest)` to pass the new fields into `NewCourse`:

```java
NewCourse input = new NewCourse(
        request.title(),
        request.description(),
        request.instructor(),
        request.category(),
        request.price(),
        request.isFree(),
        request.level(),
        request.rating(),
        request.thumbnail(),
        request.durationHours()
);

return repository.create(input);
```

For draft updates, we also need to persist these fields.

Add this method to `CourseMetadataService`:

<CodeBlock language="java" editable>
@Transactional
public void updateMetadata(
        long courseId,
        String thumbnail,
        BigDecimal durationHours
) {
    CourseEntity entity = repository.findById(courseId)
            .orElseThrow(
                    () -> new IllegalArgumentException(
                            "Course does not exist: " + courseId
                    )
            );

    entity.replaceDraftFields(
            entity.getTitle(),
            entity.getDescription(),
            entity.getInstructor(),
            entity.getCategory(),
            entity.getPrice(),
            entity.getRating(),
            entity.isFree(),
            entity.getLevel(),
            thumbnail,
            durationHours
    );

    repository.save(entity);
}
</CodeBlock>

Add the `BigDecimal` import.

In `CourseController.update()`, after the management service updates the draft, call:

```java
metadataService.updateMetadata(
        id,
        request.thumbnail(),
        request.durationHours()
);
```

Then return the updated `CourseResponse`.

This gives us a working migration path, although the domain update and metadata update currently happen in separate transactions. For stronger atomicity, we'll consolidate them into one application-service transaction in the next architecture refactor.

## 15. Start the application and apply Flyway migrations

Before launching, make sure the old repository configuration has been removed and your PostgreSQL environment variables are set in the current terminal.

```powershell
cd $HOME\angular-learning\learnhub-backend

$env:LEARNHUB_DB_URL
$env:LEARNHUB_DB_USER
```

Do not print the password.

Start Spring Boot:

```powershell
mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

On startup, the expected sequence is:

1. Spring Boot creates the datasource.
2. Flyway connects to PostgreSQL.
3. Flyway applies V1 and V2.
4. Hibernate validates the entity mapping against the schema.
5. Spring creates the repository, services, and controllers.
6. Embedded Tomcat starts on port `8080`.

If the application fails during startup, inspect the first underlying exception rather than only the final `Application run failed` line.

### Inspect the database

In a second PowerShell terminal:

```powershell
& $psql `
  -U learnhub_app `
  -h localhost `
  -d learnhub
```

Run:

```sql
\dt

SELECT installed_rank, version, description, success
FROM flyway_schema_history
ORDER BY installed_rank;

SELECT
    id,
    title,
    status,
    price,
    thumbnail,
    duration_hours,
    last_updated
FROM courses
ORDER BY id;
```

Expected:

- A `courses` table.
- A `flyway_schema_history` table.
- Two successful migrations.
- Five seeded courses.
- Four published courses and one draft.

Exit:

```sql
\q
```

### Verify the API

```powershell
$baseUrl = "http://localhost:8080/api/v1/courses"

$response = Invoke-RestMethod -Uri $baseUrl

$response.items |
    Select-Object id, title, thumbnail,
                  durationHours, lastUpdated |
    Format-Table
```

The seeded courses should now return actual metadata from PostgreSQL.

## 16. Prove that course data survives a restart

Create a new course:

```powershell
$course = @{
    title         = "Java Persistence Masterclass"
    description   = "Learn PostgreSQL, JPA, and Hibernate."
    instructor    = "LearnHub Instructor"
    category      = "Web Development"
    price         = 69.99
    isFree        = $false
    level         = "INTERMEDIATE"
    rating        = 4.6
    thumbnail     = "https://placehold.co/600x350?text=JPA"
    durationHours = 21.5
}

$created = Invoke-RestMethod `
  -Method Post `
  -Uri $baseUrl `
  -ContentType "application/json" `
  -Body ($course | ConvertTo-Json)

$created | Format-List

$id = $created.id
```

Expected: a newly generated ID, ordinarily `6` in a fresh database.

Publish it:

```powershell
Invoke-RestMethod `
  -Method Patch `
  -Uri "$baseUrl/$id/publish"
```

Confirm it is publicly visible:

```powershell
Invoke-RestMethod -Uri "$baseUrl/$id"
```

Now stop Spring Boot with `Ctrl+C`, then restart:

```powershell
mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

In a second terminal:

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:8080/api/v1/courses/$id"
```

<box border radius="lg" padding={3} gap={2}>
  <row align="center" gap={2}>
    <icon name="database" color="success" size="lg"/>
    **Persistence milestone**
  </row>
  If the course is still present after restarting Spring Boot, you have verified that LearnHub is reading durable course data from PostgreSQL rather than rebuilding an in-memory catalog on every launch.
</box>

## 17. Testcontainers: isolated PostgreSQL integration tests

Unit tests should remain fast and independent of infrastructure.

But repository integration tests should use a real PostgreSQL database, because H2 and PostgreSQL do not behave identically.

We'll use Testcontainers to start a temporary PostgreSQL instance.

### Prerequisite: Docker Desktop

<row align="start" gap={3}>
  <AsyncImage query="Docker whale logo software containers" aspectRatio="1:1" maxWidth="90px"/>
  <box flex="1" gap={1}>
    **<Entity category="software" value="Docker Desktop" disambig="container development environment"/>**

    Testcontainers requires an accessible container runtime. On Windows, Docker Desktop with WSL 2 is a common setup.

    <Link url="https://docs.docker.com/desktop/setup/install/windows-install/" title="Docker Desktop installation guide"/>
  </box>
</row>

Verify:

```powershell
docker --version
docker info
```

If Docker is unavailable, you can still run the application against your locally installed PostgreSQL database. The container-backed integration tests will need to wait until a container runtime is configured.

### 17.1 Create a shared PostgreSQL test base

File:
`src/test/java/com/learnhub/PostgresIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub;

import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

/**
 * Shared PostgreSQL integration test infrastructure.
 *
 * Testcontainers starts an isolated database.
 * Flyway applies migrations automatically.
 */
@Testcontainers
public abstract class PostgresIntegrationTest {

    @Container
    @SuppressWarnings("resource")
    protected static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17")
                    .withDatabaseName("learnhub_test")
                    .withUsername("learnhub_test")
                    .withPassword("test_password");

    static {
        POSTGRES.start();
    }

    @DynamicPropertySource
    static void configureDatabase(
            DynamicPropertyRegistry registry
    ) {
        registry.add(
                "spring.datasource.url",
                POSTGRES::getJdbcUrl
        );

        registry.add(
                "spring.datasource.username",
                POSTGRES::getUsername
        );

        registry.add(
                "spring.datasource.password",
                POSTGRES::getPassword
        );
    }
}
</CodeBlock>

Because the container is started in a static initializer, we should not also ask JUnit's Testcontainers extension to manage its lifecycle. For this implementation, remove `@Testcontainers` and `@Container` and keep the explicit static startup:

```java
public abstract class PostgresIntegrationTest {
    protected static final PostgreSQLContainer<?> POSTGRES = ...;

    static {
        POSTGRES.start();
    }

    // @DynamicPropertySource remains unchanged.
}
```

This prevents the extension from stopping a shared container between test classes while Spring may still cache their application contexts.

### 17.2 Create a persistence integration test

File:

`src/test/java/com/learnhub/persistence/CoursePersistenceTest.java`

First create the directory:

```powershell
New-Item -ItemType Directory -Force `
  src\test\java\com\learnhub\persistence |
  Out-Null
```

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;
import com.learnhub.model.NewCourse;
import com.learnhub.repository.CourseRepository;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Verifies PostgreSQL-backed course persistence.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@Transactional
class CoursePersistenceTest extends PostgresIntegrationTest {

    @Autowired
    private CourseRepository repository;

    @Autowired
    private JpaCourseRepository jpa;

    @Test
    void flywaySeedsFiveCourses() {
        assertEquals(5, jpa.count());
    }

    @Test
    void postgresGeneratesCourseIds() {
        NewCourse input = new NewCourse(
                "Database Engineering",
                "Build reliable relational data models.",
                "LearnHub Instructor",
                "Web Development",
                new BigDecimal("39.99"),
                false,
                CourseLevel.INTERMEDIATE,
                new BigDecimal("4.50"),
                null,
                new BigDecimal("12.00")
        );

        var created = repository.create(input);

        assertTrue(created.id() > 5);
        assertEquals(CourseStatus.DRAFT, created.status());

        assertTrue(
                repository.findById(created.id()).isPresent()
        );
    }

    @Test
    void readsPersistedCourseMetadata() {
        var course = jpa.findById(1L).orElseThrow();

        assertNotNull(course.getThumbnail());
        assertNotNull(course.getDurationHours());
        assertNotNull(course.getLastUpdated());
    }
}
</CodeBlock>

### 17.3 Update the existing Spring integration tests

The Lesson 5 `LearnHubApplicationTest` and `LearnHubStatusControllerTest`, and Lesson 6 `CourseControllerTest`, now require PostgreSQL.

Change their class declarations so they extend `PostgresIntegrationTest`:

```java
class LearnHubApplicationTest
        extends PostgresIntegrationTest {
```

```java
class LearnHubStatusControllerTest
        extends PostgresIntegrationTest {
```

```java
class CourseControllerTest
        extends PostgresIntegrationTest {
```

Add the import when the test is in a subpackage:

```java
import com.learnhub.PostgresIntegrationTest;
```

This gives those tests an isolated PostgreSQL database instead of requiring your development database credentials.

### Test isolation warning

The Lesson 6 controller tests use `@DirtiesContext(AFTER_EACH_TEST_METHOD)` to reset the Spring context. With a persistent test container, that does **not** reset database rows.

For reliable test isolation, update write tests to clean up their records, or use a database reset strategy that recreates the schema or rolls back changes between tests.

A practical immediate approach is to use transactional repository tests for persistence and separate isolated MVC tests using mocked services. In Lesson 8, we'll refine the test suite into unit, MVC-slice, and full integration test layers.

Also, the Lesson 5 and Lesson 6 tests that assert exact seed counts or IDs must not run concurrently against the same mutable database unless isolation is enforced.

### Run verification

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 7 verification failed."
}
```

Unlike earlier lessons, **do not assume the test count or test outcomes until you've reconciled the existing test fixtures with the database-backed implementation**.

## 18. Update Angular to use real course metadata

We no longer need the hardcoded metadata lookup introduced in Lesson 6.

### 18.1 Extend the Angular API DTO

Open:

`$HOME\angular-learning\learnhub\src\app\models\spring-course-api.model.ts`

Replace it with:

<CodeBlock language="typescript" editable>
/**
 * Spring Boot Lesson 7 course response.
 */
export interface SpringCourseDto {
  id: number;
  title: string;
  description: string;
  instructor: string;
  category: string;
  price: number;
  rating: number;
  isFree: boolean;

  level: 'BEGINNER' | 'INTERMEDIATE' | 'ADVANCED';
  status: 'DRAFT' | 'PUBLISHED' | 'ARCHIVED';

  thumbnail: string | null;
  durationHours: number | null;
  lastUpdated: string | null;
}

export interface SpringCoursePageDto {
  items: SpringCourseDto[];
  totalItems: number;
  pageNumber: number;
  pageSize: number;
  totalPages: number;
}
</CodeBlock>

### 18.2 Update the Angular mapper

Open:

`$HOME\angular-learning\learnhub\src\app\services\course-api.ts`

Remove the entire `displayMetadata` map.

Replace the `toCourse()` method with:

<CodeBlock language="typescript" editable>
private toCourse(dto: SpringCourseDto): Course {
  const level: CourseLevel =
    dto.level === 'BEGINNER'
      ? 'Beginner'
      : dto.level === 'INTERMEDIATE'
        ? 'Intermediate'
        : 'Advanced';

  return {
    id: dto.id,
    title: dto.title,
    description: dto.description,
    instructor: dto.instructor,
    category: dto.category,
    price: dto.price,
    rating: dto.rating,
    isFree: dto.isFree,
    level,

    // These values now come from PostgreSQL.
    thumbnail: dto.thumbnail ??
      'https://placehold.co/600x350?text=Course',

    durationHours: dto.durationHours ?? 0,
    lastUpdated: dto.lastUpdated ?? ''
  };
}
</CodeBlock>

The fallback thumbnail is only for courses whose optional metadata is null. We no longer maintain a separate ID-to-metadata map.

### 18.3 Verify Angular compilation

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}
```

Start both applications:

<grid columns={2} gap={3}>
  <grid-item>
    <box border radius="lg" padding={3} gap={2}>
      **Backend terminal**

      ```powershell
      cd $HOME\angular-learning\learnhub-backend

      mvn spring-boot:run `
        "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
      ```

      <text color="secondary" size="xs">PostgreSQL must be running and database environment variables must be set.</text>
    </box>
  </grid-item>
  <grid-item>
    <box border radius="lg" padding={3} gap={2}>
      **Frontend terminal**

      ```powershell
      cd $HOME\angular-learning\learnhub

      ng.cmd serve --open
      ```

      <text color="secondary" size="xs">Open http://localhost:4200</text>
    </box>
  </grid-item>
</grid>

Verify that course cards display their images and durations, and that the Angular app retrieves data from Spring Boot.

### What is still incomplete?

We have persisted the fields required by the original course card model, but we have not yet implemented a complete instructor profile, course curriculum, student reviews, enrollments, or authentication.

The frontend may continue to use local demo services for those features until their backend lessons.

## 19. Important engineering review

### Persistence context and dirty checking

When Hibernate loads a `CourseEntity` inside a transaction, it generally tracks the entity as managed.

Changes to a managed entity can be synchronized to the database at flush or transaction commit.

This means the following pattern can work:

```java
@Transactional
public void renameCourse(long id, String title) {
    CourseEntity entity = repository.findById(id)
            .orElseThrow();

    entity.rename(title);
}
```

The `rename()` method is illustrative; we haven't added it to `CourseEntity`.

Hibernate can detect the change and issue an SQL `UPDATE` without an explicit `save()` call.

However, the behavior depends on the entity being managed in an active persistence context.

### `save()` is not the same as SQL INSERT

Spring Data JPA's `save()` may persist a new entity or merge an existing entity depending on its state and identity strategy.

For our new course, `id == null` and the database identity strategy clearly represent a new entity.

### `@Transactional` and service boundaries

The ideal design for course updates is:

```text
@Transactional CourseManagementService.updateDraft()
    |
    +-- Load course
    +-- Check draft status
    +-- Update domain fields
    +-- Update metadata
    +-- Commit one transaction
```

Our current implementation still separates domain and metadata updates. That is an architectural improvement to make before adding more complex workflows.

### Flyway versus Hibernate DDL

Flyway should own production schema changes.

Avoid changing:

```yaml
ddl-auto: validate
```

to:

```yaml
ddl-auto: update
```

as a shortcut for production migrations. Hibernate's automatic schema update is not a replacement for reviewed, versioned database migrations.

### Repository scalability

Our current `CourseCatalogService` still loads courses into memory and applies Java stream operations.

That is fine for five seeded courses, but not for a marketplace with thousands of courses.

Later, we'll move filtering, sorting, and pagination into SQL-backed repository queries using Spring Data JPA.

### A note on migration correctness

Because this course evolves code from earlier lessons, the precise method signatures in your local Lesson 2–4 files matter. In particular, check `CourseFilter`, `CourseSort`, `PageResult`, and `CourseRepository` before copying replacement code.

The lesson supplies the intended architecture and implementation, but it has not been compiled against your actual local project. Resolve signature differences deliberately rather than deleting earlier tests or changing domain invariants to make new code compile.

## 20. Common problems and fixes

| Problem | Likely cause | Fix |
|---|---|---|
| `Connection refused` on port 5432 | PostgreSQL stopped | Check the PostgreSQL Windows service |
| `password authentication failed` | Wrong credentials | Recheck `LEARNHUB_DB_USER` and password |
| `relation "courses" does not exist` | Flyway did not run | Inspect migration logs and schema |
| Flyway checksum mismatch | An applied migration was edited | Restore the original migration and create a new version |
| Hibernate schema validation fails | Entity/table mismatch | Compare `@Column` definitions with SQL |
| Multiple `CourseRepository` beans | Old `@Bean` configuration still active | Remove `CourseRepositoryConfig` |
| Generated IDs collide with seed IDs | Identity sequence not advanced | Check the V2 `setval()` statement |
| Tests fail because Docker is unavailable | Testcontainers cannot start | Start Docker Desktop or defer container-backed tests |
| Tests pass alone but fail together | Shared mutable test database | Isolate or roll back test mutations |
| Angular still shows old metadata | Old adapter map remains | Remove `displayMetadata` and rebuild |
| Course disappears from public catalog | Status is DRAFT or ARCHIVED | Public GET only shows PUBLISHED |

## 21. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="database" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise A — Database constraints**

      Create a Flyway V3 migration that adds a constraint requiring nonblank course titles. Write a test that attempts to persist a blank title and verifies that PostgreSQL rejects it.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="search" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise B — Database-level search**

      Add a Spring Data repository query that searches published courses by title using case-insensitive matching. Compare the generated SQL with the previous in-memory stream filter.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="shield-check" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise C — Transactional draft update**

      Refactor course and metadata updates into one `@Transactional` service method. Add a test that deliberately fails the second operation and verifies that the first operation is rolled back.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="test-tube" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise D — Reliable test isolation**

      Refactor the Lesson 6 MVC tests so that one test's course creation or deletion cannot affect another test's expected seed counts. Run the suite repeatedly and in a different test order.
    </box>
  </row>
</box>

## 22. Lesson 7 quiz

{@body const quiz=[{q:"What should generate IDs for newly created LearnHub courses?",opts:["The Angular client","MAX(id) + 1 in Java","PostgreSQL identity column","The HTTP controller"],a:2,why:"Database-generated identity values avoid application-level ID allocation races."},{q:"Which tool should own versioned schema changes?",opts:["Flyway","Jackson","MockMvc","Angular CLI"],a:0,why:"Flyway records and applies versioned database migrations."},{q:"What does ddl-auto: validate do?",opts:["Drops and recreates tables","Checks mappings against the existing schema","Automatically creates missing indexes","Disables Hibernate"],a:1,why:"Validation checks schema compatibility without managing schema changes."},{q:"Why keep CourseEntity separate from the Course domain record?",opts:["JPA forbids interfaces","To isolate ORM concerns from domain design","To avoid all mapping code","Because records cannot contain methods"],a:1,why:"The separation keeps persistence details out of immutable domain objects."},{q:"Why use Testcontainers for PostgreSQL integration tests?",opts:["To replace all unit tests","To verify behavior against real PostgreSQL in an isolated environment","To run Angular inside Maven","To eliminate Flyway migrations"],a:1,why:"Containerized PostgreSQL gives realistic database behavior without depending on a manually maintained test database."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [checked,setChecked]=DIL.useState(false)}
{@body const score=quiz.filter((q,i)=>answers[i]===q.a).length}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">    **Persistence and JPA knowledge check**
    <text color="secondary" size="xs">{Object.keys(answers).length}/5 answered</text>
  </row>
  {#each quiz as q,i}
    <box key={i} gap={2}>
      <text weight="medium">{i+1}. {q.q}</text>
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setChecked(false)}}>
        {#each q.opts as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if checked}
        <row align="center" gap={2}>
          <icon name={answers[i]===q.a?"check-circle":"alert-circle"} color={answers[i]===q.a?"success":"danger"}/>
          <text size="sm" weight="medium">{answers[i]===q.a?"Correct":"Answer: "+q.opts[q.a]}</text>
        </row>
        <text color="secondary" size="sm">{q.why}</text>
      {/if}
    </box>
    {#if i<quiz.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if checked}
    <box background="surface-secondary" padding={3} radius="md" gap={1}>
      <title size="xl" tabularNums>{score}/5</title>
      <text size="sm">{score===5?"All answers correct.":"Review the explanations and retry."}</text>
    </box>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==quiz.length} onClick={()=>setChecked(true)}>Check answers</button>
    <button variant="outline" color="secondary" onClick={()=>{setAnswers({});setChecked(false)}}>Reset</button>
  </row>
</box>

## 23. Lesson completion checklist

{@body const groups=[{name:"PostgreSQL setup",items:["Installed PostgreSQL on Windows","Created learnhub database and learnhub_app account","Verified connection using psql","Configured database environment variables"]},{name:"Spring persistence",items:["Added Spring Data JPA and PostgreSQL dependencies","Created V1 and V2 Flyway migrations","Created CourseEntity","Created JpaCourseRepository","Created PostgresCourseRepository","Removed the old in-memory Spring bean","Replaced MAX(id) + 1 with database-generated IDs"]},{name:"API and frontend",items:["Exposed thumbnail, durationHours, and lastUpdated","Updated CourseWriteRequest","Updated Angular SpringCourseDto","Removed hardcoded Angular metadata","Verified courses survive a backend restart"]},{name:"Testing and engineering",items:["Configured Testcontainers","Added PostgreSQL integration tests","Reconciled older MVC tests with persistent storage","Verified Maven clean verify locally","Completed at least two exercises","Committed Lesson 7 changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((s,g)=>s+g.items.length,0)}
{@body const count=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    <box gap={1}>
      **Lesson 7 progress**
      <text color="secondary" size="xs">Check each milestone after local verification.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" radius="full" height="8px" clip>
    <box background="rgba(109,40,217,0.85)" radius="full" height="100%" width={`${count/total*100}%`}/>
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
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 7 checklist completed.**
    </row>
  {/if}
</box>

## 24. Final verification and Git checkpoint

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend verification failed."
}

git status
git add .
git commit -m "Persist LearnHub courses with PostgreSQL JPA and Flyway"
```

Frontend:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}

git status
git add .
git commit -m "Use persisted LearnHub course metadata from Spring Boot"
```

Do not commit database passwords, environment files containing secrets, or PostgreSQL data directories.

## 25. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="shield-check" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 8 — Production-Quality Validation, Exception Handling, and Transaction Boundaries**

    We'll improve the current REST API with cross-field validation, consistent problem responses, transactional course workflows, reliable database test isolation, and clear service boundaries.

    This is an important cleanup lesson before implementing users, enrollments, and authentication.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 8: production-quality Bean Validation, cross-field constraints, RFC 9457 Problem Details, @RestControllerAdvice, @Transactional service boundaries, atomic course updates, database test isolation with Testcontainers, and API contract tests. Continue from Lesson 7 with experienced-Java pacing, complete Windows PowerShell commands, exact file paths, documented code, exercises, quiz, and checklist.")}>Start Lesson 8 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 7 deliverable:** LearnHub now has a PostgreSQL-backed course persistence architecture, versioned Flyway migrations, database-generated IDs, and an Angular API contract capable of returning persisted course metadata.

The critical milestone is proving that course changes survive application restarts. Before moving on, make sure the migration, compilation, and integration-test issues identified above are resolved in your actual project.