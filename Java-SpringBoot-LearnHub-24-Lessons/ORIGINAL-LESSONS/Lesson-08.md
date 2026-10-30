# LearnHub Java & Spring Boot Backend — Lesson 8

<text color="secondary" size="sm">Production-Quality Validation, Problem Details, Transaction Boundaries, and Reliable Database Tests</text>

**Track:** Java 21 · Spring Boot 3.5.x · PostgreSQL · Flyway · JUnit 5 · Testcontainers  
**Duration:** Approximately 4–6 hours, including refactoring and verification.

In Lesson 7, you introduced PostgreSQL, Hibernate entities, Flyway migrations, database-generated course IDs, and persisted course metadata.

Today we'll address the engineering weaknesses that become important once a REST API has real data:

- A request can be structurally valid but violate business rules.
- An error response needs a stable contract that Angular can understand.
- Updating a course and its metadata in separate transactions can leave inconsistent data.
- Tests sharing a database can pass individually and fail when run together.
- Persistence errors should not leak SQL details or stack traces to API clients.

Our goal is to make LearnHub's course API **consistent, atomic, and testable** before adding user accounts and enrollments.

## 1. What changes in Lesson 8?

<box border radius="xl" padding={3} gap={2}>
  <text weight="medium">Before — Lesson 7</text>
  <box background="surface-secondary" radius="md" padding={3} gap={1}>
    **CourseController**
    <text color="secondary" size="xs">Calls management service, then metadata service</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <grid columns={2} gap={2}>
    <grid-item>
      <box border radius="md" padding={3} align="center" gap={1}>
        **Update course**
        <text color="secondary" size="xs">Transaction A</text>
      </box>
    </grid-item>
    <grid-item>
      <box border radius="md" padding={3} align="center" gap={1}>
        **Update metadata**
        <text color="secondary" size="xs">Transaction B</text>
      </box>
    </grid-item>
  </grid>
  <text color="secondary" size="xs">If B fails after A commits, the database may contain a partial update.</text>
  <divider/>
  <text weight="medium">After — Lesson 8</text>
  <box background="surface-secondary" radius="md" padding={3} align="center">
    **CourseController**
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="md" padding={3} gap={1} align="center">
    **@Transactional CourseManagementService**
    <text color="secondary" size="xs">One business operation, one transaction</text>
    <row gap={2} align="center">
      <icon name="check-circle" color="success"/>
      <text size="sm">Update domain fields + metadata together</text>
    </row>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box background="surface-secondary" radius="md" padding={3} align="center">
    **PostgreSQL — commit all or roll back all**
  </box>
</box>

We'll also standardize errors using <Entity category="scientific_concept" value="RFC 9457 Problem Details" disambig="HTTP API machine-readable error response standard"/>.

## 2. Prepare the project

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info

git status
```

Create a Git checkpoint:

```powershell
git add .
git commit -m "Checkpoint before LearnHub Lesson 8"
```

If there's nothing to commit, continue.

Run the current build:

```powershell
mvn clean compile

if ($LASTEXITCODE -ne 0) {
    throw "Resolve Lesson 7 compilation issues before Lesson 8."
}
```

### Important migration assumptions

Lesson 7 added `CourseEntity`, `JpaCourseRepository`, `PostgresCourseRepository`, `CourseMetadataService`, and `CourseManagementService`.

We'll continue from that architecture, but correct two design issues along the way:

1. **The service should not depend on HTTP DTOs.** We'll introduce an application-level command.
2. **The controller should not perform separate database updates.** The service will own the complete transaction.

We'll also fix an important persistence mismatch: the original `Course` record requires ratings to have one decimal place, while Lesson 7 stored ratings with two decimal places. This lesson will keep the domain's one-decimal rating invariant and align request validation accordingly.

## 3. Project files

Create the new packages:

```powershell
$root = "$HOME\angular-learning\learnhub-backend"

$dirs = @(
    "$root\src\main\java\com\learnhub\application",
    "$root\src\main\java\com\learnhub\validation",
    "$root\src\main\java\com\learnhub\web\error",
    "$root\src\test\java\com\learnhub\validation",
    "$root\src\test\java\com\learnhub\persistence",
    "$root\src\test\java\com\learnhub\web"
)

foreach ($dir in $dirs) {
    New-Item -ItemType Directory -Force -Path $dir |
        Out-Null
}
```

| File | Action |
|---|---|
| `application/CourseUpsertCommand.java` | New |
| `validation/ValidCoursePricing.java` | New |
| `validation/CoursePricingValidator.java` | New |
| `web/dto/CourseWriteRequest.java` | Replace |
| `service/CourseManagementService.java` | Replace |
| `persistence/PostgresCourseRepository.java` | Update |
| `persistence/CourseMetadataService.java` | Simplify |
| `web/CourseController.java` | Update |
| `web/error/GlobalExceptionHandler.java` | Replace |
| `test/.../CourseValidationTest.java` | New |
| `test/.../CourseTransactionTest.java` | New |
| `test/.../CourseApiContractTest.java` | New |

We'll preserve the existing Flyway migrations rather than editing migrations that may already have been applied.

## 4. Build cross-field Bean Validation

Field-level validation can check that `price >= 0`, but it cannot independently establish whether `price` agrees with `isFree`.

Our business rule is:

| `isFree` | `price` | Valid? |
|---|---:|---|
| `true` | `0.00` | Yes |
| `true` | `29.99` | No |
| `false` | `29.99` | Yes |
| `false` | `0.00` | No |
| Either | Negative | No |

We'll implement a reusable class-level constraint.

### 4.1 Create `@ValidCoursePricing`

**File:** `src/main/java/com/learnhub/validation/ValidCoursePricing.java`

```java
package com.learnhub.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Validates the relationship between a course's
 * free flag and its monetary price.
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = CoursePricingValidator.class)
public @interface ValidCoursePricing {

    String message() default
            "Free courses must cost zero; paid courses must cost more than zero.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

### 4.2 Create the validator

**File:** `src/main/java/com/learnhub/validation/CoursePricingValidator.java`

```java
package com.learnhub.validation;

import com.learnhub.web.dto.CourseWriteRequest;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

import java.math.BigDecimal;

/**
 * Cross-field validator for course pricing.
 *
 * Null checks are left to @NotNull, so clients
 * receive the appropriate missing-field errors.
 */
public final class CoursePricingValidator
        implements ConstraintValidator<
                ValidCoursePricing,
                CourseWriteRequest
        > {

    @Override
    public boolean isValid(
            CourseWriteRequest request,
            ConstraintValidatorContext context
    ) {
        if (request == null) {
            return true;
        }

        Boolean free = request.isFree();
        BigDecimal price = request.price();

        if (free == null || price == null) {
            return true;
        }

        boolean valid = free
                ? price.signum() == 0
                : price.signum() > 0;

        if (!valid) {
            context.disableDefaultConstraintViolation();

            context.buildConstraintViolationWithTemplate(
                    "Price must be zero for free courses and positive for paid courses."
            )
                    .addPropertyNode("price")
                    .addConstraintViolation();
        }

        return valid;
    }
}
```

This validator reports the violation against the `price` field rather than only at the object level. That makes it easier for Angular to display a message beside the price input.

### 4.3 Replace `CourseWriteRequest`

**File:** `src/main/java/com/learnhub/web/dto/CourseWriteRequest.java`

```java
package com.learnhub.web.dto;

import com.learnhub.application.CourseUpsertCommand;
import com.learnhub.model.CourseLevel;
import com.learnhub.validation.ValidCoursePricing;

import jakarta.validation.constraints.*;

import java.math.BigDecimal;

@ValidCoursePricing
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
        @Digits(integer = 10, fraction = 2)
        BigDecimal price,

        @NotNull
        Boolean isFree,

        @NotNull
        CourseLevel level,

        @NotNull
        @DecimalMin("0.0")
        @DecimalMax("5.0")
        @Digits(integer = 1, fraction = 1)
        BigDecimal rating,

        @Size(max = 1000)
        String thumbnail,

        @DecimalMin("0.0")
        @Digits(integer = 6, fraction = 2)
        BigDecimal durationHours
) {

    /**
     * Convert HTTP input into an application command.
     * The service no longer needs to import web DTOs.
     */
    public CourseUpsertCommand toCommand() {
        return new CourseUpsertCommand(
                title,
                description,
                instructor,
                category,
                price,
                isFree,
                level,
                rating,
                thumbnail,
                durationHours
        );
    }
}
```

The `@Digits` annotations align the request with the current domain and database precision. A rating of `4.85` is rejected; `4.8` is accepted.

**One remaining limitation:** this validation layer doesn't automatically apply to callers that bypass HTTP. We'll also validate the application command at the service boundary.

## 5. Introduce an application-level command

**File:** `src/main/java/com/learnhub/application/CourseUpsertCommand.java`

```java
package com.learnhub.application;

import com.learnhub.model.CourseLevel;

import java.math.BigDecimal;
import java.util.Objects;

/**
 * Application input independent of Spring MVC.
 *
 * Both create and draft-update operations use
 * the same set of editable fields.
 */
public record CourseUpsertCommand(
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
) {
    public CourseUpsertCommand {
        title = requireText(title, "Title");
        description = requireText(description, "Description");
        instructor = requireText(instructor, "Instructor");
        category = requireText(category, "Category");

        Objects.requireNonNull(price, "Price is required.");
        Objects.requireNonNull(level, "Level is required.");
        Objects.requireNonNull(rating, "Rating is required.");

        if (price.signum() < 0 || price.scale() > 2
                || price.precision() - price.scale() > 10) {
            throw new IllegalArgumentException(
                    "Price must be nonnegative and fit NUMERIC(12,2)."
            );
        }

        if (free != (price.signum() == 0)) {
            throw new IllegalArgumentException(
                    "Free status must match course price."
            );
        }

        if (rating.compareTo(BigDecimal.ZERO) < 0
                || rating.compareTo(new BigDecimal("5.0")) > 0
                || rating.scale() > 1) {
            throw new IllegalArgumentException(
                    "Rating must be between 0 and 5 with at most one decimal place."
            );
        }

        if (thumbnail != null && thumbnail.length() > 1000) {
            throw new IllegalArgumentException(
                    "Thumbnail must be at most 1000 characters."
            );
        }

        if (durationHours != null
                && (durationHours.signum() < 0
                || durationHours.scale() > 2
                || durationHours.precision() - durationHours.scale() > 6)) {
            throw new IllegalArgumentException(
                    "Duration must fit NUMERIC(8,2) and cannot be negative."
            );
        }
    }

    private static String requireText(
            String value,
            String field
    ) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException(
                    field + " is required."
            );
        }

        return value.trim();
    }
}
```

This is an example of **defense in depth**:

- Bean Validation produces useful HTTP field errors.
- The application command rejects invalid data from non-HTTP callers.
- The existing domain record enforces core business invariants.
- PostgreSQL enforces constraints on stored values.

No single layer replaces the others.

## 6. Make course updates atomic

We will keep the existing `CourseRepository` abstraction for reads and legacy unit tests, but add a focused write operation to the PostgreSQL adapter.

The new operation will update both core course fields and metadata on the **same managed JPA entity**.

### 6.1 Add an atomic update method to the PostgreSQL adapter

**File:** `src/main/java/com/learnhub/persistence/PostgresCourseRepository.java`

Keep the existing class, constructor, `findAll()`, `findById()`, `create()`, `save()`, and `deleteById()` methods from Lesson 7.

Add these imports:

```java
import com.learnhub.application.CourseUpsertCommand;
import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.exception.InvalidCourseStateException;
```

Add the following method inside the class:

```java
/**
 * Updates the core course fields and metadata together.
 *
 * The caller's @Transactional boundary encloses this
 * operation and any subsequent business steps.
 */
public Course updateDraft(
        long courseId,
        CourseUpsertCommand command
) {
    CourseEntity entity = jpa.findById(courseId)
            .orElseThrow(
                    () -> new CourseNotFoundException(courseId)
            );

    if (entity.getStatus() != CourseStatus.DRAFT) {
        throw new InvalidCourseStateException(
                "Only DRAFT courses can be edited."
        );
    }

    entity.replaceDraftFields(
            command.title(),
            command.description(),
            command.instructor(),
            command.category(),
            command.price(),
            command.rating(),
            command.free(),
            command.level(),
            command.thumbnail(),
            command.durationHours()
    );

    // The entity is managed by Hibernate.
    // Dirty checking persists all changed columns.
    return toDomain(entity);
}
```

Because `CourseEntity.replaceDraftFields()` updates all editable fields, we no longer need a second metadata write.

The database update occurs when Hibernate flushes the persistence context, normally during transaction commit.

### 6.2 Replace CourseManagementService

**File:** `src/main/java/com/learnhub/service/CourseManagementService.java`

```java
package com.learnhub.service;

import com.learnhub.application.CourseUpsertCommand;
import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.exception.DuplicateCourseException;
import com.learnhub.exception.InvalidCourseStateException;
import com.learnhub.model.Course;
import com.learnhub.model.CourseStatus;
import com.learnhub.model.NewCourse;
import com.learnhub.persistence.PostgresCourseRepository;
import com.learnhub.repository.CourseRepository;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Objects;

/**
 * Coordinates course-management business operations.
 *
 * Public methods define transaction boundaries.
 * Runtime exceptions trigger rollback by default.
 */
@Service
public class CourseManagementService {

    private final CourseRepository repository;
    private final PostgresCourseRepository postgresRepository;

    public CourseManagementService(
            CourseRepository repository,
            PostgresCourseRepository postgresRepository
    ) {
        this.repository = Objects.requireNonNull(repository);
        this.postgresRepository =
                Objects.requireNonNull(postgresRepository);
    }

    /**
     * Legacy Lesson 4 entry point.
     *
     * New REST course creation uses createDraft(command).
     */
    @Transactional
    public Course createDraft(Course course) {
        Objects.requireNonNull(course);

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
     * Creates a new draft with a PostgreSQL-generated ID.
     */
    @Transactional
    public Course createDraft(CourseUpsertCommand command) {
        Objects.requireNonNull(command);

        NewCourse input = new NewCourse(
                command.title(),
                command.description(),
                command.instructor(),
                command.category(),
                command.price(),
                command.free(),
                command.level(),
                command.rating(),
                command.thumbnail(),
                command.durationHours()
        );

        return repository.create(input);
    }

    /**
     * Atomically updates core fields and metadata.
     */
    @Transactional
    public Course updateDraft(
            long courseId,
            CourseUpsertCommand command
    ) {
        Objects.requireNonNull(command);

        return postgresRepository.updateDraft(
                courseId,
                command
        );
    }

    @Transactional
    public Course publish(long courseId) {
        Course existing = requireCourse(courseId);

        if (existing.status() != CourseStatus.DRAFT) {
            throw new InvalidCourseStateException(
                    "Only DRAFT courses can be published."
            );
        }

        return repository.save(
                withStatus(existing, CourseStatus.PUBLISHED)
        );
    }

    @Transactional
    public Course archive(long courseId) {
        Course existing = requireCourse(courseId);

        if (existing.status() != CourseStatus.PUBLISHED) {
            throw new InvalidCourseStateException(
                    "Only PUBLISHED courses can be archived."
            );
        }

        return repository.save(
                withStatus(existing, CourseStatus.ARCHIVED)
        );
    }

    @Transactional
    public void delete(long courseId) {
        Course existing = requireCourse(courseId);

        if (existing.status() == CourseStatus.PUBLISHED) {
            throw new InvalidCourseStateException(
                    "Archive a published course before deleting it."
            );
        }

        repository.deleteById(courseId);
    }

    private Course requireCourse(long courseId) {
        return repository.findById(courseId)
                .orElseThrow(
                        () -> new CourseNotFoundException(courseId)
                );
    }

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
```

### A compatibility issue to resolve

This implementation intentionally injects `PostgresCourseRepository` for the atomic update. That is a **transitional design**: it works with PostgreSQL, but older plain-Java tests that construct `CourseManagementService` with a single repository argument will no longer compile.

For a cleaner design, the next refactor should move `updateDraft(courseId, command)` into the `CourseRepository` interface and implement it in both adapters. That removes the persistence-specific dependency from the service and restores simple unit-test construction.

For Lesson 8, prefer making that interface change before running the complete legacy test suite. Do not delete earlier tests merely because the constructor changed.

### 6.3 Simplify CourseMetadataService

**File:** `src/main/java/com/learnhub/persistence/CourseMetadataService.java`

Keep these read methods:

```java
findByCourseId(long id)
findAllByIds(List<Long> ids)
```

Remove the Lesson 7 `updateMetadata()` method entirely.

Metadata writes now belong to the single atomic course update operation.

### 6.4 Update the controller

**File:** `src/main/java/com/learnhub/web/CourseController.java`

Replace the `create()` and `update()` methods with:

```java
@PostMapping
public ResponseEntity<CourseResponse> create(
        @Valid @RequestBody CourseWriteRequest request
) {
    Course created = managementService.createDraft(
            request.toCommand()
    );

    URI location = URI.create(
            "/api/v1/courses/" + created.id()
    );

    return ResponseEntity.created(location)
            .body(CourseResponse.from(
                    created,
                    metadataService.findByCourseId(created.id())
            ));
}

@PutMapping("/{id}")
public CourseResponse update(
        @PathVariable long id,
        @Valid @RequestBody CourseWriteRequest request
) {
    Course updated = managementService.updateDraft(
            id,
            request.toCommand()
    );

    return CourseResponse.from(
            updated,
            metadataService.findByCourseId(updated.id())
    );
}
```

Retain the other endpoints from Lesson 7.

Remove the old controller call to:

```java
metadataService.updateMetadata(...)
```

The controller now coordinates HTTP input and output, not database writes.

### Transaction behavior

`@Transactional` defaults to rollback for unchecked exceptions (`RuntimeException` and `Error`).

It does not automatically roll back for every checked exception.

Also, Spring's proxy-based transactions normally apply when a method is called through the Spring-managed proxy, not when one method calls another transactional method on `this`.

These distinctions matter when designing service workflows.

## 7. Return RFC 9457 Problem Details

RFC 9457 defines a standard JSON format for HTTP API errors.

Instead of a custom `ApiError` shape, we'll use Spring's `ProblemDetail`.

A typical response looks like:

```json
{
  "type": "https://learnhub.example/problems/validation",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more request fields are invalid.",
  "instance": "/api/v1/courses",
  "fieldErrors": {
    "price": "Price must be zero for free courses and positive for paid courses."
  }
}
```

The `type` URL is an illustrative problem identifier. You should replace it with a real documentation URL before production; no page is assumed to exist at that address.

### Replace GlobalExceptionHandler

**File:** `src/main/java/com/learnhub/web/error/GlobalExceptionHandler.java`

```java
package com.learnhub.web.error;

import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.exception.DuplicateCourseException;
import com.learnhub.exception.InvalidCourseStateException;

import jakarta.servlet.http.HttpServletRequest;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingServletRequestParameterException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;

import java.net.URI;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Converts known application failures into RFC 9457
 * Problem Details responses.
 */
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CourseNotFoundException.class)
    public ResponseEntity<ProblemDetail> notFound(
            CourseNotFoundException exception,
            HttpServletRequest request
    ) {
        return problem(
                HttpStatus.NOT_FOUND,
                "Course not found",
                exception.getMessage(),
                "not-found",
                request,
                Map.of()
        );
    }

    @ExceptionHandler({
            DuplicateCourseException.class,
            InvalidCourseStateException.class
    })
    public ResponseEntity<ProblemDetail> conflict(
            RuntimeException exception,
            HttpServletRequest request
    ) {
        return problem(
                HttpStatus.CONFLICT,
                "Course state conflict",
                exception.getMessage(),
                "conflict",
                request,
                Map.of()
        );
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> validation(
            MethodArgumentNotValidException exception,
            HttpServletRequest request
    ) {
        Map<String, String> fields = new LinkedHashMap<>();

        exception.getBindingResult()
                .getFieldErrors()
                .forEach(error -> fields.putIfAbsent(
                        error.getField(),
                        error.getDefaultMessage() == null
                                ? "Invalid value."
                                : error.getDefaultMessage()
                ));

        return problem(
                HttpStatus.BAD_REQUEST,
                "Validation failed",
                "One or more request fields are invalid.",
                "validation",
                request,
                fields
        );
    }

    @ExceptionHandler({
            IllegalArgumentException.class,
            HttpMessageNotReadableException.class,
            MethodArgumentTypeMismatchException.class,
            MissingServletRequestParameterException.class
    })
    public ResponseEntity<ProblemDetail> badRequest(
            Exception exception,
            HttpServletRequest request
    ) {
        return problem(
                HttpStatus.BAD_REQUEST,
                "Invalid request",
                "The request contains invalid input.",
                "bad-request",
                request,
                Map.of()
        );
    }

    private ResponseEntity<ProblemDetail> problem(
            HttpStatus status,
            String title,
            String detail,
            String type,
            HttpServletRequest request,
            Map<String, String> fieldErrors
    ) {
        ProblemDetail body = ProblemDetail.forStatusAndDetail(
                status,
                detail
        );

        body.setTitle(title);

        body.setType(URI.create(
                "https://learnhub.example/problems/" + type
        ));

        body.setInstance(URI.create(
                request.getRequestURI()
        ));

        if (!fieldErrors.isEmpty()) {
            body.setProperty("fieldErrors", fieldErrors);
        }

        return ResponseEntity.status(status)
                .body(body);
    }
}
```

### Why not catch every exception?

A broad handler that turns all failures into `400 Bad Request` would hide server bugs.

Unexpected database or programming errors should result in a server error and be logged. They should not be mislabeled as invalid client input.

### Important compatibility change

Your Angular client must now read:

```typescript
error.error.detail
error.error.fieldErrors
```

rather than the old custom error response's `message` field.

The `application/problem+json` media type identifies the standardized response.

## 8. Verify validation and error behavior

Start the backend using your existing PostgreSQL environment variables:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

Open another PowerShell terminal.

### Test inconsistent pricing

```powershell
$baseUrl = "http://localhost:8080/api/v1/courses"

$invalid = @{
    title         = "Invalid Free Course"
    description   = "A deliberately invalid request."
    instructor    = "LearnHub Instructor"
    category      = "Programming"
    price         = 29.99
    isFree        = $true
    level         = "BEGINNER"
    rating        = 4.5
    thumbnail     = $null
    durationHours = 5.0
}

$invalid |
    ConvertTo-Json |
    Set-Content -Encoding utf8 .\invalid-course.json

curl.exe -i `
  -X POST $baseUrl `
  -H "Content-Type: application/json" `
  --data-binary "@invalid-course.json"
```
Expected:

- HTTP `400 Bad Request`
- Content type `application/problem+json`
- `fieldErrors.price` describing the inconsistent pricing

### Test invalid decimal precision

Change the rating:

```powershell
$invalid.price = 0
$invalid.rating = 4.85

$invalid |
    ConvertTo-Json |
    Set-Content -Encoding utf8 .\invalid-course.json
```

Repeat the POST request.

Expected: HTTP 400, with a rating precision error.

### Test an unknown course

```powershell
curl.exe -i `
  http://localhost:8080/api/v1/courses/999999
```

Expected: HTTP 404 with a Problem Details response.

### Test an invalid lifecycle transition

```powershell
curl.exe -i `
  -X PATCH `
  http://localhost:8080/api/v1/courses/1/publish
```

The original course `1` is already published.

Expected: HTTP `409 Conflict`.

## 9. Refine the repository boundary before testing

The implementation in Section 6 works as a PostgreSQL-specific transition, but we can make it cleaner **without breaking the Lesson 4 constructor-based unit tests**.

Use the following refinement as the final implementation for Lesson 8. It supersedes the two-repository constructor in Section 6.

### 9.1 Extend CourseRepository

**File:** `src/main/java/com/learnhub/repository/CourseRepository.java`

Add this method alongside the existing methods:

```java
/**
 * Atomically replaces the editable fields of a draft.
 *
 * Implementations must reject missing or non-draft courses.
 */
Course updateDraft(
        long courseId,
        CourseUpsertCommand command
);
```

Add:

```java
import com.learnhub.application.CourseUpsertCommand;
```

### 9.2 Update the PostgreSQL adapter

The `PostgresCourseRepository.updateDraft()` method from Section 6 already has the required signature. Add `@Override` above it.

### 9.3 Update the in-memory repository

**File:** `src/main/java/com/learnhub/repository/InMemoryCourseRepository.java`

Add:

```java
@Override
public synchronized Course updateDraft(
        long courseId,
        CourseUpsertCommand command
) {
    Course existing = findById(courseId)
            .orElseThrow(
                    () -> new CourseNotFoundException(courseId)
            );

    if (existing.status() != CourseStatus.DRAFT) {
        throw new InvalidCourseStateException(
                "Only DRAFT courses can be edited."
        );
    }

    Course updated = new Course(
            courseId,
            command.title(),
            command.description(),
            command.instructor(),
            command.category(),
            command.price(),
            command.free(),
            command.level(),
            command.rating(),
            CourseStatus.DRAFT
    );

    return save(updated);
}
```

Add imports:

```java
import com.learnhub.application.CourseUpsertCommand;
import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.exception.InvalidCourseStateException;
```

This adapter has no persisted metadata, so it updates only the fields represented by the existing `Course` domain record. That is appropriate for the earlier plain-Java tests.

### 9.4 Simplify CourseManagementService

In the Section 6 service, remove:

```java
private final PostgresCourseRepository postgresRepository;
```

Remove the `PostgresCourseRepository` import.

Replace the constructor with:

```java
public CourseManagementService(
        CourseRepository repository
) {
    this.repository = Objects.requireNonNull(repository);
}
```

Replace the `updateDraft()` method with:

```java
@Transactional
public Course updateDraft(
        long courseId,
        CourseUpsertCommand command
) {
    Objects.requireNonNull(command);

    return repository.updateDraft(
            courseId,
            command
    );
}
```

The final architecture is now:

```text
CourseManagementService
        |
        v
CourseRepository interface
        |
        +-- InMemoryCourseRepository
        |
        +-- PostgresCourseRepository
                    |
                    v
             JpaCourseRepository
```

The Spring service has only one constructor dependency again, making the earlier unit tests easier to retain.

### 9.5 One important limitation of the legacy creation method

The original `createDraft(Course)` method passes an already-assigned ID to `repository.save()`. Lesson 7's PostgreSQL adapter treats `save()` as an update and therefore rejects a brand-new ID.

For the REST API, always use:

```java
createDraft(CourseUpsertCommand command)
```

The legacy overload remains useful for in-memory unit tests. In a later cleanup, it should either be removed or explicitly modeled as an import operation with different persistence semantics.

This distinction should be documented rather than silently ignored.

## 10. Make Testcontainers deterministic

Lesson 7 introduced Testcontainers, but also exposed a test isolation problem: a fresh Spring Application Context does not automatically mean a fresh database.

We'll use one PostgreSQL container for the test JVM and rollback transactions for synchronous MVC tests.

### 10.1 Replace the shared test base

**File:** `src/test/java/com/learnhub/PostgresIntegrationTest.java`

```java
package com.learnhub;

import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import org.testcontainers.containers.PostgreSQLContainer;

/**
 * Shared PostgreSQL test infrastructure.
 *
 * Starts one container for the JVM.
 * Do not combine this explicit lifecycle with
 * @Container/@Testcontainers extension management.
 */
public abstract class PostgresIntegrationTest {

    protected static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17")
                    .withDatabaseName("learnhub_test")
                    .withUsername("learnhub_test")
                    .withPassword("test_password");

    static {
        POSTGRES.start();
    }

    @DynamicPropertySource
    static void databaseProperties(
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
```

### 10.2 Use rollback for synchronous MVC tests

In your existing Lesson 6 test:

`src/test/java/com/learnhub/web/CourseControllerTest.java`

Keep `@SpringBootTest` and `@AutoConfigureMockMvc`, but replace `@DirtiesContext` with:

```java
import org.springframework.transaction.annotation.Transactional;

@Transactional
class CourseControllerTest
        extends PostgresIntegrationTest {
    // Existing MockMvc tests.
}
```

This approach works for ordinary synchronous `MockMvc` calls executed on the test thread. The test transaction rolls back after each test method.

It does **not** provide automatic rollback for requests sent to a real server on a different thread, such as `TestRestTemplate` with `RANDOM_PORT`.

For those tests, avoid modifying shared seed records or use a separately reset database.

### 10.3 Avoid depending on fixed generated IDs

A PostgreSQL sequence is not rolled back when a transaction rolls back.

That means a test may create a course with ID `6`, roll back the insert, and the next test may create a course with ID `7`.

Both behaviors are correct.

Therefore, replace brittle assertions such as:

```java
.andExpect(jsonPath("$.id").value(6))
```

with assertions that check the generated ID is positive, and capture it dynamically when subsequent requests need it.

Also avoid hardcoding a `Location` header ending in `/6`.

## 11. Test cross-field validation

**File:** `src/test/java/com/learnhub/validation/CourseValidationTest.java`

```java
package com.learnhub.validation;

import com.learnhub.model.CourseLevel;
import com.learnhub.web.dto.CourseWriteRequest;

import jakarta.validation.Validation;
import jakarta.validation.Validator;
import jakarta.validation.ValidatorFactory;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;

import static org.junit.jupiter.api.Assertions.*;

class CourseValidationTest {

    private static ValidatorFactory factory;
    private static Validator validator;

    @BeforeAll
    static void setup() {
        factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
    }

    @AfterAll
    static void close() {
        factory.close();
    }

    private CourseWriteRequest request(
            BigDecimal price,
            boolean free
    ) {
        return new CourseWriteRequest(
                "Java Fundamentals",
                "Learn core Java.",
                "LearnHub Instructor",
                "Programming",
                price,
                free,
                CourseLevel.BEGINNER,
                new BigDecimal("4.5"),
                null,
                new BigDecimal("10.00")
        );
    }

    @Test
    void acceptsFreeCourseWithZeroPrice() {
        assertTrue(
                validator.validate(
                        request(new BigDecimal("0.00"), true)
                ).isEmpty()
        );
    }

    @Test
    void acceptsPaidCourseWithPositivePrice() {
        assertTrue(
                validator.validate(
                        request(new BigDecimal("29.99"), false)
                ).isEmpty()
        );
    }

    @Test
    void rejectsFreeCourseWithPositivePrice() {
        var violations = validator.validate(
                request(new BigDecimal("29.99"), true)
        );

        assertTrue(
                violations.stream().anyMatch(
                        violation -> violation.getPropertyPath()
                                .toString()
                                .equals("price")
                )
        );
    }

    @Test
    void rejectsPaidCourseWithZeroPrice() {
        assertFalse(
                validator.validate(
                        request(new BigDecimal("0.00"), false)
                ).isEmpty()
        );
    }

    @Test
    void rejectsRatingWithTooManyDecimalPlaces() {
        CourseWriteRequest input = new CourseWriteRequest(
                "Java Fundamentals",
                "Learn core Java.",
                "LearnHub Instructor",
                "Programming",
                new BigDecimal("19.99"),
                false,
                CourseLevel.BEGINNER,
                new BigDecimal("4.85"),
                null,
                null
        );

        assertFalse(
                validator.validate(input).isEmpty()
        );
    }
}
```

These five tests run without Spring Boot or PostgreSQL.

## 12. Test transaction rollback against PostgreSQL

We need a test that proves an update is not committed if the business operation fails.

We'll create a test-only service that performs a valid update and then throws an unchecked exception.

### 12.1 Test-only transactional workflow

**File:** `src/test/java/com/learnhub/persistence/FailingCourseWorkflow.java`

```java
package com.learnhub.persistence;

import com.learnhub.application.CourseUpsertCommand;
import com.learnhub.repository.CourseRepository;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Test-only bean that deliberately fails after an update.
 */
@Service
public class FailingCourseWorkflow {

    private final CourseRepository repository;

    public FailingCourseWorkflow(
            CourseRepository repository
    ) {
        this.repository = repository;
    }

    @Transactional
    public void updateThenFail(
            long courseId,
            CourseUpsertCommand command
    ) {
        repository.updateDraft(courseId, command);

        throw new IllegalStateException(
                "Simulated failure after update."
        );
    }
}
```

This class lives under `src/test/java`, so it is not packaged into the production JAR.

### 12.2 Transaction integration test

**File:** `src/test/java/com/learnhub/persistence/CourseTransactionTest.java`

```java
package com.learnhub.persistence;

import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;
import com.learnhub.application.CourseUpsertCommand;
import com.learnhub.model.CourseLevel;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.math.BigDecimal;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Verifies rollback across the PostgreSQL adapter.
 */
@SpringBootTest(
        classes = {
                LearnHubApplication.class,
                FailingCourseWorkflow.class
        }
)
class CourseTransactionTest
        extends PostgresIntegrationTest {

    @Autowired
    private FailingCourseWorkflow workflow;

    @Autowired
    private JpaCourseRepository jpa;

    @Test
    void rollsBackCoreFieldsAndMetadataTogether() {
        // Seed course 5 is a draft.
        CourseEntity before = jpa.findById(5L)
                .orElseThrow();

        String originalTitle = before.getTitle();
        String originalThumbnail = before.getThumbnail();

        CourseUpsertCommand command =
                new CourseUpsertCommand(
                        "Changed But Rolled Back",
                        "This update should not persist.",
                        "LearnHub Instructor",
                        "Web Development",
                        new BigDecimal("79.99"),
                        false,
                        CourseLevel.ADVANCED,
                        new BigDecimal("4.8"),
                        "https://example.com/changed.png",
                        new BigDecimal("20.00")
                );

        assertThrows(
                IllegalStateException.class,
                () -> workflow.updateThenFail(5L, command)
        );

        CourseEntity after = jpa.findById(5L)
                .orElseThrow();

        assertEquals(originalTitle, after.getTitle());
        assertEquals(
                originalThumbnail,
                after.getThumbnail()
        );
    }
}
```

This test deliberately does not annotate the test method with `@Transactional`. The workflow opens its own transaction, throws an exception, and Spring rolls it back. A subsequent repository read checks the committed database state.

The test assumes the original seed course `5` remains a draft and that no other test modifies it permanently.

## 13. Add API contract tests

An API contract test checks more than the status code. It verifies that clients receive the expected JSON structure, content type, and error properties.

**File:** `src/test/java/com/learnhub/web/CourseApiContractTest.java`

```java
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

@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class CourseApiContractTest
        extends PostgresIntegrationTest {

    @Autowired
    private MockMvc mvc;

    @Test
    void coursePageContainsAngularCompatibleFields()
            throws Exception {

        mvc.perform(get("/api/v1/courses")
                        .param("page", "0")
                        .param("size", "10"))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(
                        MediaType.APPLICATION_JSON
                ))
                .andExpect(jsonPath("$.items").isArray())
                .andExpect(jsonPath("$.totalItems").isNumber())
                .andExpect(jsonPath("$.pageNumber").value(0))
                .andExpect(jsonPath("$.items[0].id").isNumber())
                .andExpect(jsonPath("$.items[0].title").isString())
                .andExpect(jsonPath("$.items[0].isFree").isBoolean())
                .andExpect(jsonPath("$.items[0].level").isString())
                .andExpect(jsonPath("$.items[0].thumbnail").exists())
                .andExpect(jsonPath("$.items[0].durationHours").isNumber())
                .andExpect(jsonPath("$.items[0].lastUpdated").isString());
    }

    @Test
    void invalidPricingReturnsProblemDetails()
            throws Exception {

        String json = """
                {
                  "title": "Invalid Free Course",
                  "description": "Cross-field validation example.",
                  "instructor": "LearnHub Instructor",
                  "category": "Programming",
                  "price": 25.00,
                  "isFree": true,
                  "level": "BEGINNER",
                  "rating": 4.5,
                  "durationHours": 10.00
                }
                """;

        mvc.perform(post("/api/v1/courses")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(json))
                .andExpect(status().isBadRequest())
                .andExpect(content().contentTypeCompatibleWith(
                        MediaType.valueOf("application/problem+json")
                ))
                .andExpect(jsonPath("$.status").value(400))
                .andExpect(jsonPath("$.title")
                        .value("Validation failed"))
                .andExpect(jsonPath("$.type").exists())
                .andExpect(jsonPath("$.instance")
                        .value("/api/v1/courses"))
                .andExpect(jsonPath("$.fieldErrors.price").exists());
    }

    @Test
    void unknownCourseReturnsProblemDetails()
            throws Exception {

        mvc.perform(get("/api/v1/courses/999999"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.status").value(404))
                .andExpect(jsonPath("$.title")
                        .value("Course not found"));
    }

    @Test
    void invalidStateTransitionReturnsConflict()
            throws Exception {

        mvc.perform(patch("/api/v1/courses/1/publish"))
                .andExpect(status().isConflict())
                .andExpect(jsonPath("$.status").value(409))
                .andExpect(jsonPath("$.title")
                        .value("Course state conflict"));
    }
}
```

These tests cover the public course response and the new Problem Details contract.

### Fix the older Lesson 6 assertions

In `CourseControllerTest.java`, the earlier custom error assertions may still expect:

```java
jsonPath("$.fieldErrors.title")
```

That remains valid when a field error exists.

But assertions expecting a custom `message` or `error` property must change to the RFC 9457 fields:

```java
jsonPath("$.detail")
jsonPath("$.title")
jsonPath("$.status")
```

Also update the older test's exact generated ID and `Location` assertions as described in Section 10.

### A test-suite limitation worth addressing

Some Lesson 5–7 tests use `@SpringBootTest(webEnvironment = RANDOM_PORT)` and `TestRestTemplate`. Those requests execute in server threads and are not covered by a transaction on the test thread.

Keep those tests read-only, or use an explicit database cleanup/reset strategy.

For the Lesson 8 CRUD and validation tests, synchronous `MockMvc` plus rollback is a practical approach.

## 14. Verify the full build

First, run only the new validation tests:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=CourseValidationTest" test
```

Then run the new database-backed tests:

```powershell
mvn "-Dtest=CourseTransactionTest,CourseApiContractTest" test
```

Finally:

```powershell
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 8 verification failed."
}
```

The new files introduce **10 tests**: five validation tests, one transaction rollback test, and four API contract tests.

The complete suite's count depends on which earlier tests you've retained and how you've adapted them to PostgreSQL. Do not treat a test-count target as proof that the build is correct.

### Inspect Surefire results

```powershell
Get-ChildItem target\surefire-reports -Filter *.txt |
    ForEach-Object {
        Get-Content $_.FullName |
            Select-String "Tests run:|Failures:|Errors:"
    }
```

## 15. Angular: handle Problem Details

Your Angular frontend can now display server-side validation errors consistently.

For example, in a course editor component:

```typescript
import { HttpErrorResponse } from '@angular/common/http';

interface ApiProblem {
  type?: string;
  title?: string;
  status?: number;
  detail?: string;
  instance?: string;
  fieldErrors?: Record<string, string>;
}

function readApiError(error: HttpErrorResponse): {
  message: string;
  fields: Record<string, string>;
} {
  const problem = error.error as ApiProblem | null;

  return {
    message:
      problem?.detail ??
      'The request could not be completed.',

    fields: problem?.fieldErrors ?? {}
  };
}
```

A form can display `fields.price` beside its price input, while using the general `message` for a page-level notification.

Don't blindly render arbitrary server error HTML. Treat server messages as text.

## 16. Important engineering considerations

### Transactions do not replace authorization

A transaction can ensure a course update is atomic. It cannot determine whether the caller owns that course.

Our course write endpoints remain **local-development-only and unauthenticated**. Do not expose them publicly until Spring Security and instructor ownership checks are implemented.

### Rollback does not reset database sequences

PostgreSQL identity sequences can advance even when the inserting transaction rolls back. Tests must not assume consecutive generated IDs.

### A successful HTTP response should follow a successful commit

Our service transaction commits before control returns to the controller. The controller then reads metadata and builds the HTTP response.

That is appropriate for this lesson, though more complex workflows may benefit from returning a fully populated application result rather than performing an additional read.

### Test isolation has several levels

| Test type | Preferred isolation |
|---|---|
| Domain unit test | Plain Java objects |
| Bean Validation test | Validator without Spring Boot |
| MVC slice test | Mocked services, no real database |
| Repository integration test | PostgreSQL Testcontainers, rollback |
| Full HTTP integration test | Dedicated database or explicit cleanup |
| End-to-end test | Isolated application environment |

Use the lightest test level that can prove the behavior.

### Known refinement for the next architecture pass

The `CourseCatalogService` still filters an in-memory list of all courses after reading from PostgreSQL. It works for the lesson's small catalog but does not scale well.

We'll eventually move pagination, filtering, and sorting into database queries.

## 17. Hands-on exercises

<box gap={3}>
  <row gap={3} align="start">
    <icon name="shield-check" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise A — Additional cross-field rules**

      Add validation requiring a nonblank thumbnail URL for published courses. Consider why this rule belongs to the publishing operation rather than the draft creation request.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="database" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise B — Transaction rollback**

      Write another test that updates a draft, forces a persistence exception during flush, and verifies that neither its title nor its metadata changes are committed.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="file-json" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise C — Problem Details extensions**

      Add a stable `errorCode` property such as `COURSE_STATE_CONFLICT` or `COURSE_NOT_FOUND`. Update Angular to branch on the code rather than comparing human-readable messages.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="test-tube" size="lg" color="secondary"/>
    <box flex="1" gap={1}>
      **Exercise D — MVC test isolation**

      Convert one full-context controller test into a `@WebMvcTest` using mocked service dependencies. Compare its startup time and dependencies with the Testcontainers-backed integration test.
    </box>
  </row>
</box>

## 18. Knowledge quiz

{@body const quiz=[{q:"What is the primary purpose of class-level Bean Validation?",opts:["Replace database constraints","Validate relationships between multiple fields","Generate database tables","Start a Spring transaction"],correct:1,why:"Class-level validation is useful when a rule depends on more than one property."},{q:"Which Spring type represents RFC 9457 error responses?",opts:["ResponseEntity only","ProblemDetail","Pageable","EntityManager"],correct:1,why:"Spring's ProblemDetail models the standard error representation."},{q:"Where should a transaction begin for a multi-step business operation?",opts:["At the application service boundary","Inside Angular","In every DTO getter","Only in the HTTP error handler"],correct:0,why:"A service-level transaction can cover all database changes belonging to one business operation."},{q:"What happens to a PostgreSQL identity sequence after an insert rolls back?",opts:["It always resets","The sequence may still advance","The entire table is dropped","Flyway re-runs automatically"],correct:1,why:"Sequence allocation is not rolled back with the transaction."},{q:"Why can a RANDOM_PORT HTTP test behave differently from a transactional MockMvc test?",opts:["HTTP requests execute on a separate server thread","MockMvc never uses Spring MVC","PostgreSQL cannot support HTTP tests","Flyway disables transactions"],correct:0,why:"A transaction on the test thread does not automatically encompass a request processed on a separate server thread."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
{@body const score=quiz.filter((q,i)=>answers[i]===q.correct).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Lesson 8 knowledge check**
    <text color="secondary" size="xs">{Object.keys(answers).length} of {quiz.length} answered</text>
  </row>
  {#each quiz as q,i}
    <box gap={2} key={i}>
      <text weight="medium">{i+1}. {q.q}</text>
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setGraded(false)}}>
        {#each q.opts as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if graded}
        <row align="center" gap={2}>
          <icon name={answers[i]===q.correct?"check-circle":"alert-circle"} color={answers[i]===q.correct?"success":"danger"}/>
          <text size="sm" weight="medium">{answers[i]===q.correct?"Correct":"Answer: "+q.opts[q.correct]}</text>
        </row>
        <text size="sm" color="secondary">{q.why}</text>
      {/if}
    </box>
    {#if i<quiz.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if graded}
    <box background="surface-secondary" radius="md" padding={3} gap={1}>
      <title size="xl" tabularNums>{score}/{quiz.length}</title>
      <text size="sm">{score===quiz.length?"All answers correct.":"Review the explanations and retry."}</text>
    </box>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==quiz.length} onClick={()=>setGraded(true)}>Check answers</button>
    <button color="secondary" variant="outline" onClick={()=>{setAnswers({});setGraded(false)}}>Reset</button>
  </row>
</box>

## 19. Lesson completion checklist

{@body const groups=[{name:"Validation",items:["Created @ValidCoursePricing","Implemented cross-field pricing validation","Updated CourseWriteRequest","Introduced CourseUpsertCommand","Verified invalid prices and rating precision"]},{name:"Architecture and transactions",items:["Added atomic updateDraft to repository interface","Updated both repository implementations","Moved transaction boundaries to CourseManagementService","Removed separate metadata writes","Preserved constructor-based unit testability"]},{name:"HTTP and tests",items:["Implemented Problem Details responses","Verified 400, 404, and 409 contracts","Configured Testcontainers lifecycle","Added Bean Validation tests","Added rollback integration test","Added API contract tests","Reconciled older MVC test fixtures"]},{name:"Completion",items:["Verified Maven clean verify","Tested API from PowerShell","Updated Angular error handling","Completed at least two exercises","Committed Lesson 8 changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((s,g)=>s+g.items.length,0)}
{@body const count=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 8 progress**
      <text color="secondary" size="xs">Mark items after local verification.</text>
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
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 8 checklist completed.**
    </row>
  {/if}
</box>

## 20. Final verification and Git checkpoint

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 8 backend verification failed."
}

git status
git add .
git commit -m "Add transactional course updates and Problem Details"
```

If you updated the Angular frontend's error handling, verify and commit that project separately:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular verification failed."
}

git status
git add .
git commit -m "Handle Spring Boot Problem Details responses"
```

## 21. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="database" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 9 — SQL and PostgreSQL Deep Dive**

    We'll strengthen the relational foundation before moving into authentication and enrollment models: primary and foreign keys, joins, constraints, indexes, transactions, query plans, and PostgreSQL data modeling for LearnHub.

    We'll also examine the current `courses` schema and design the future relationships between users, instructors, courses, and enrollments.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 9: SQL and PostgreSQL deep dive with LearnHub database modeling, primary and foreign keys, joins, indexes, EXPLAIN ANALYZE, transactions, isolation levels, Flyway migrations, realistic SQL exercises, tests, Windows PowerShell and psql commands, quiz, and checklist. Continue from Lesson 8 with experienced-Java pacing and exact file paths.")}>Start Lesson 9 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 8 deliverable:** LearnHub now has cross-field request validation, a standardized Problem Details error contract, atomic course updates, and stronger PostgreSQL integration tests.

The essential verification is that **invalid data is rejected, partial updates roll back, and API errors remain predictable for Angular**. These are foundational guarantees for the user accounts, enrollments, and payment workflows we'll build later.