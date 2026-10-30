# LearnHub Java & Spring Boot Backend — Lesson 6

<text color="secondary" size="sm">REST APIs with Spring MVC: Course CRUD, DTOs, Validation, HTTP Status Codes, Testing, and Angular Integration</text>

<row gap={2} wrap="wrap" align="center">
  <badge>Lesson 6 of 24</badge>
  <badge>Java 21 + Spring Boot 3.5</badge>
  <badge>Experienced Java · Steady Pace</badge>
</row>

In Lesson 5, we migrated LearnHub to Spring Boot, introduced dependency injection, and started our first HTTP server on port `8080`.

Now we'll build the **first real LearnHub REST API** so your Angular frontend can retrieve courses from Spring Boot instead of relying on `json-server`.

By the end of this lesson, you will have a working course API supporting listing, filtering, pagination, individual course retrieval, draft creation, updating, publishing, archiving, and deletion.

We'll preserve the existing Java domain model and service architecture rather than rewriting everything.

## 1. Learning objectives

By completing Lesson 6, you will understand how to:

- Design REST endpoints using resource-oriented URLs.
- Use `@RestController`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, and `@DeleteMapping`.
- Work with `@PathVariable`, `@RequestParam`, `@RequestBody`, and `ResponseEntity`.
- Separate HTTP request/response DTOs from internal domain models.
- Validate incoming requests using Jakarta Bean Validation.
- Return meaningful HTTP status codes and structured errors.
- Test Spring MVC using `MockMvc`.
- Connect Angular to Spring Boot using CORS and compatible JSON responses.

**Estimated duration:** 3–4 hours, including exercises and tests.

## 2. The REST API we're building

<box border radius="xl" padding={3} gap={3}>
  <text weight="medium">LearnHub request flow</text>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    <icon name="monitor-smartphone" size="lg"/>
    **Angular LearnHub frontend**
    <text color="secondary" size="xs">localhost:4200</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring MVC CourseController**
    <text color="secondary" size="xs">localhost:8080/api/v1/courses</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **CourseCatalogService + CourseManagementService**
    <text color="secondary" size="xs">Validation and business rules</text>
  </box>
  <row justify="center"><icon name="arrow-down-up" color="secondary"/></row>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    **CourseRepository**
    <text color="secondary" size="xs">In-memory storage for now; PostgreSQL later</text>
  </box>
</box>

### Endpoint contract

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/v1/courses` | List published courses |
| `GET` | `/api/v1/courses/{id}` | Retrieve a published course |
| `GET` | `/api/v1/courses/categories` | List published categories |
| `POST` | `/api/v1/courses` | Create a draft |
| `PUT` | `/api/v1/courses/{id}` | Replace editable draft fields |
| `PATCH` | `/api/v1/courses/{id}/publish` | Publish a draft |
| `PATCH` | `/api/v1/courses/{id}/archive` | Archive a published course |
| `DELETE` | `/api/v1/courses/{id}` | Delete a course |

For now, write endpoints are **development-only and unauthenticated**. They must not be exposed publicly until we implement Spring Security and instructor ownership rules.

## 3. Prepare the Lesson 5 project

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 5 baseline verification failed."
}
```

Create a checkpoint:

```powershell
git status
git add .
git commit -m "Checkpoint before LearnHub REST API"
```

If there is nothing to commit, continue.

### Create the required directories

```powershell
$root = "$HOME\angular-learning\learnhub-backend"

$directories = @(
    "$root\src\main\java\com\learnhub\web\dto",
    "$root\src\main\java\com\learnhub\web\error",
    "$root\src\test\java\com\learnhub\web"
)

foreach ($directory in $directories) {
    New-Item -ItemType Directory -Force -Path $directory |
        Out-Null
}
```

Our new and modified files will be:

```text
src/
├── main/java/com/learnhub/
│   ├── config/
│   │   └── WebConfig.java                  NEW
│   ├── service/
│   │   └── CourseManagementService.java    UPDATE
│   └── web/
│       ├── CourseController.java           NEW
│       ├── dto/
│       │   ├── CourseResponse.java         NEW
│       │   ├── CoursePageResponse.java     NEW
│       │   └── CourseWriteRequest.java     NEW
│       └── error/
│           ├── ApiError.java               NEW
│           └── GlobalExceptionHandler.java NEW
└── test/java/com/learnhub/web/
    └── CourseControllerTest.java           NEW
```

We will also update `pom.xml` and `application.yml`.

## 4. Add Jakarta Bean Validation

Spring MVC handles HTTP requests, but we need a standard way to validate incoming data.

Open:

`$HOME\angular-learning\learnhub-backend\pom.xml`

Inside `<dependencies>`, add:

```xml
<!-- Validates request DTOs using Jakarta Bean Validation. -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Keep your existing Spring Web, Actuator, and Test dependencies.

Verify:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean compile

if ($LASTEXITCODE -ne 0) {
    throw "Validation dependency setup failed."
}
```

### What Bean Validation gives us

| Annotation | Meaning |
|---|---|
| `@NotBlank` | Text cannot be null, empty, or whitespace |
| `@NotNull` | Value must be supplied |
| `@Size` | Length must fall within limits |
| `@DecimalMin` | Decimal value must meet a minimum |
| `@DecimalMax` | Decimal value must meet a maximum |
| `@Positive` | Numeric value must be greater than zero |
| `@Valid` | Trigger validation of the request object |

Validation at the HTTP boundary does not replace domain validation. We want both layers: useful API errors for clients and enforced invariants inside the application.

## 5. Design the DTOs

Your existing Angular `Course` interface includes fields such as `thumbnail`, `durationHours`, and `lastUpdated`, while our current Java `Course` record does not.

We must avoid pretending that Spring already stores those fields.

For Lesson 6, we'll introduce a **backend course DTO** based on the fields that actually exist in the Lesson 4 Java model. We'll then add an explicit compatibility adapter for Angular.

In Lesson 7 and the PostgreSQL lessons, we'll extend the persisted course model to cover the complete frontend contract.

### 5.1 CourseResponse

File: `src/main/java/com/learnhub/web/dto/CourseResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.model.Course;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;

import java.math.BigDecimal;

/**
 * Public API representation of a course.
 *
 * Deliberately separate from the domain record,
 * so the HTTP contract can evolve independently.
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
        CourseStatus status
) {

    /**
     * Maps a domain Course to its API representation.
     */
    public static CourseResponse from(Course course) {
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
                course.status()
        );
    }
}
</CodeBlock>

**Important Java/Jackson detail:** a record component named `isFree` is serialized as the JSON property `"isFree"` in the usual Jackson record mapping. This matches the Angular property name.

### 5.2 CourseWriteRequest

File: `src/main/java/com/learnhub/web/dto/CourseWriteRequest.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.model.Course;
import com.learnhub.model.CourseLevel;
import com.learnhub.model.CourseStatus;

import jakarta.validation.constraints.DecimalMax;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

import java.math.BigDecimal;

/**
 * Validated input for creating or updating a course.
 *
 * The API controls status transitions separately.
 * Clients cannot directly set PUBLISHED or ARCHIVED.
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
        BigDecimal rating
) {

    /**
     * Creates an immutable draft domain object.
     *
     * The ID is controlled by the server-side path
     * or ID allocation policy, not the request body.
     */
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

We use `Boolean` rather than primitive `boolean` for incoming JSON so `@NotNull` can detect a missing `isFree` property.

The domain record should continue to reject inconsistent combinations such as a free course with a nonzero price.

### 5.3 CoursePageResponse

File: `src/main/java/com/learnhub/web/dto/CoursePageResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web.dto;

import com.learnhub.catalog.PageResult;
import com.learnhub.model.Course;

import java.util.List;

/**
 * API pagination envelope.
 *
 * Uses zero-based page numbering.
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
            PageResult<Course> page
    ) {
        List<CourseResponse> responses =
                page.items()
                        .stream()
                        .map(CourseResponse::from)
                        .toList();

        return new CoursePageResponse(
                responses,
                page.totalItems(),
                page.pageNumber(),
                page.pageSize(),
                page.totalPages()
        );
    }
}
</CodeBlock>

This assumes the `PageResult` record from Lesson 2 exposes `items()`, `totalItems()`, `pageNumber()`, `pageSize()`, and `totalPages()`.

If your Lesson 2 record uses slightly different accessor names, adapt this mapping to your actual `PageResult` implementation rather than introducing a second pagination model.

## 6. Extend CourseManagementService for REST operations

Our Lesson 4 service already supports `createDraft()`, `publish()`, and `archive()`.

We now need two additional capabilities:

1. Allocate an ID when the client creates a course.
2. Update the editable fields of an existing draft.

We'll keep the existing methods so earlier tests continue to work.

Replace:

`src/main/java/com/learnhub/service/CourseManagementService.java`

<CodeBlock language="java" editable>
package com.learnhub.service;

import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.exception.DuplicateCourseException;
import com.learnhub.exception.InvalidCourseStateException;
import com.learnhub.model.Course;
import com.learnhub.model.CourseStatus;
import com.learnhub.repository.CourseRepository;
import com.learnhub.web.dto.CourseWriteRequest;

import org.springframework.stereotype.Service;

import java.util.Comparator;
import java.util.Objects;

/**
 * Application service for course management.
 *
 * Lesson 6 uses in-memory storage and demo-only writes.
 * Authentication, ownership, transactions, and durable
 * ID generation will be introduced in later lessons.
 */
@Service
public final class CourseManagementService {

    private final CourseRepository repository;

    public CourseManagementService(
            CourseRepository repository
    ) {
        this.repository = Objects.requireNonNull(repository);
    }

    /**
     * Retained from Lesson 4.
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
     * Creates a draft with a demo-generated ID.
     *
     * This method is synchronized for this service instance,
     * but does not provide durable or distributed ID safety.
     */
    public synchronized Course createDraft(
            CourseWriteRequest request
    ) {
        Objects.requireNonNull(request);

        long maxId = repository.findAll()
                .stream()
                .map(Course::id)
                .max(Comparator.naturalOrder())
                .orElse(0L);

        long nextId = Math.addExact(maxId, 1L);

        return createDraft(request.toDraft(nextId));
    }

    /**
     * Replaces editable fields of an existing draft.
     *
     * The course ID and DRAFT status are preserved.
     */
    public Course updateDraft(
            long courseId,
            CourseWriteRequest request
    ) {
        Objects.requireNonNull(request);

        Course existing = requireCourse(courseId);

        if (existing.status() != CourseStatus.DRAFT) {
            throw new InvalidCourseStateException(
                    "Only DRAFT courses can be edited."
            );
        }

        return repository.save(request.toDraft(courseId));    }

    /**
     * Publishes a draft.
     */
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

    /**
     * Archives a published course.
     */
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

    /**
     * Deletes a draft or archived course.
     *
     * Published courses must be archived first.
     */
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

    /**
     * Copy the immutable Course record with a new status.
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

### Architecture note

This lesson deliberately places `CourseWriteRequest` in the service's method signature to keep the migration manageable. In a more strictly layered design, the web controller would convert it into an application command such as `CreateCourseCommand`, so the service would have no dependency on the `web` package.

We'll make that separation when introducing persistence and more complex instructor workflows.

Also, the demo ID generator is suitable only for our single-process, in-memory exercise. PostgreSQL sequences or identity columns will replace it.

## 7. Create the REST controller

File: `src/main/java/com/learnhub/web/CourseController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.catalog.CourseFilter;
import com.learnhub.catalog.CourseSort;
import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.model.Course;
import com.learnhub.model.CourseLevel;
import com.learnhub.service.CourseCatalogService;
import com.learnhub.service.CourseManagementService;
import com.learnhub.web.dto.CoursePageResponse;
import com.learnhub.web.dto.CourseResponse;
import com.learnhub.web.dto.CourseWriteRequest;

import jakarta.validation.Valid;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.util.List;
import java.util.Locale;

/**
 * Versioned LearnHub course REST API.
 *
 * GET endpoints expose only published courses.
 *
 * Write endpoints are for local development ONLY
 * until Spring Security and ownership checks exist.
 */
@RestController
@RequestMapping("/api/v1/courses")
public class CourseController {

    private final CourseCatalogService catalogService;
    private final CourseManagementService managementService;

    public CourseController(
            CourseCatalogService catalogService,
            CourseManagementService managementService
    ) {
        this.catalogService = catalogService;
        this.managementService = managementService;
    }

    /**
     * GET /api/v1/courses
     *
     * Example:
     * ?search=angular&page=0&size=10&freeOnly=true
     */
    @GetMapping
    public CoursePageResponse list(
            @RequestParam(defaultValue = "") String search,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) Boolean freeOnly,
            @RequestParam(required = false) CourseLevel level,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size
    ) {
        if (page < 0) {
            throw new IllegalArgumentException(
                    "Page number cannot be negative."
            );
        }

        if (size < 1 || size > 100) {
            throw new IllegalArgumentException(
                    "Page size must be between 1 and 100."
            );
        }

        // Uses the CourseFilter and CourseSort
        // contracts established in Lesson 2.
        CourseFilter filter = new CourseFilter(
                search,
                category,
                freeOnly,
                level
        );

        return CoursePageResponse.from(
                catalogService.search(
                        filter,
                        CourseSort.DEFAULT,
                        page,
                        size
                )
        );
    }

    /**
     * GET /api/v1/courses/categories
     *
     * The explicit path is matched before the
     * variable course-ID route.
     */
    @GetMapping("/categories")
    public List<String> categories() {
        return catalogService.publishedCategories();
    }

    /**
     * GET /api/v1/courses/{id}
     */
    @GetMapping("/{id}")
    public CourseResponse getById(
            @PathVariable long id
    ) {
        Course course = catalogService
                .findPublishedById(id)
                .orElseThrow(
                        () -> new CourseNotFoundException(id)
                );

        return CourseResponse.from(course);
    }

    /**
     * POST /api/v1/courses
     *
     * Returns 201 Created and a Location header.
     */
    @PostMapping
    public ResponseEntity<CourseResponse> create(
            @Valid @RequestBody CourseWriteRequest request
    ) {
        Course created = managementService.createDraft(request);

        URI location = URI.create(
                "/api/v1/courses/" + created.id()
        );

        return ResponseEntity.created(location)
                .body(CourseResponse.from(created));
    }

    /**
     * PUT /api/v1/courses/{id}
     *
     * Full replacement of editable draft fields.
     */
    @PutMapping("/{id}")
    public CourseResponse update(
            @PathVariable long id,
            @Valid @RequestBody CourseWriteRequest request
    ) {
        return CourseResponse.from(
                managementService.updateDraft(id, request)
        );
    }

    /**
     * PATCH /api/v1/courses/{id}/publish
     */
    @PatchMapping("/{id}/publish")
    public CourseResponse publish(
            @PathVariable long id
    ) {
        return CourseResponse.from(
                managementService.publish(id)
        );
    }

    /**
     * PATCH /api/v1/courses/{id}/archive
     */
    @PatchMapping("/{id}/archive")
    public CourseResponse archive(
            @PathVariable long id
    ) {
        return CourseResponse.from(
                managementService.archive(id)
        );
    }

    /**
     * DELETE /api/v1/courses/{id}
     *
     * Successful deletion returns 204 No Content.
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(
            @PathVariable long id
    ) {
        managementService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
</CodeBlock>

Remove the unused `java.util.Locale` import if your IDE flags it. We haven't added custom sorting yet; `CourseSort.DEFAULT` is intentional.

### A key distinction: public reads versus management writes

`GET /api/v1/courses/{id}` hides drafts and archived courses, so it returns 404 for those IDs.

But the management endpoints can still operate on draft and archived courses because they use the management service.

Later, authenticated instructor endpoints will have their own authorization checks and response contracts.

## 8. Design structured API errors

Without a consistent error handler, clients can receive inconsistent error shapes.

We'll introduce a small response contract.

### 8.1 ApiError

File: `src/main/java/com/learnhub/web/error/ApiError.java`

<CodeBlock language="java" editable>
package com.learnhub.web.error;

import java.time.Instant;
import java.util.Map;

/**
 * Consistent error response for the LearnHub API.
 */
public record ApiError(
        Instant timestamp,
        int status,
        String error,
        String message,
        String path,
        Map<String, String> fieldErrors
) {
    public ApiError {
        fieldErrors = Map.copyOf(fieldErrors);
    }
}
</CodeBlock>

### 8.2 GlobalExceptionHandler

File: `src/main/java/com/learnhub/web/error/GlobalExceptionHandler.java`

<CodeBlock language="java" editable>
package com.learnhub.web.error;

import com.learnhub.exception.CourseNotFoundException;
import com.learnhub.exception.DuplicateCourseException;
import com.learnhub.exception.InvalidCourseStateException;

import jakarta.servlet.http.HttpServletRequest;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingServletRequestParameterException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;

import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Converts application and request exceptions into
 * stable, client-friendly HTTP error responses.
 *
 * Avoid exposing stack traces and internal exception
 * implementation details in public responses.
 */
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CourseNotFoundException.class)
    public ResponseEntity<ApiError> notFound(
            CourseNotFoundException exception,
            HttpServletRequest request
    ) {
        return build(
                HttpStatus.NOT_FOUND,
                exception.getMessage(),
                request,
                Map.of()
        );
    }

    @ExceptionHandler({
            DuplicateCourseException.class,
            InvalidCourseStateException.class
    })
    public ResponseEntity<ApiError> conflict(
            RuntimeException exception,
            HttpServletRequest request
    ) {
        return build(
                HttpStatus.CONFLICT,
                exception.getMessage(),
                request,
                Map.of()
        );
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> validation(
            MethodArgumentNotValidException exception,
            HttpServletRequest request
    ) {
        Map<String, String> fields = new LinkedHashMap<>();

        exception.getBindingResult()
                .getFieldErrors()
                .forEach(fieldError ->
                        fields.putIfAbsent(
                                fieldError.getField(),
                                fieldError.getDefaultMessage() == null
                                        ? "Invalid value."
                                        : fieldError.getDefaultMessage()
                        )
                );

        return build(
                HttpStatus.BAD_REQUEST,
                "Request validation failed.",
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
    public ResponseEntity<ApiError> badRequest(
            Exception exception,
            HttpServletRequest request
    ) {
        // Do not echo raw parser errors, which can
        // reveal implementation details.
        String message =
                exception instanceof IllegalArgumentException
                        ? exception.getMessage()
                        : "Invalid request parameters or JSON body.";

        return build(
                HttpStatus.BAD_REQUEST,
                message,
                request,
                Map.of()
        );
    }

    private ResponseEntity<ApiError> build(
            HttpStatus status,
            String message,
            HttpServletRequest request,
            Map<String, String> fields
    ) {
        ApiError body = new ApiError(
                Instant.now(),
                status.value(),
                status.getReasonPhrase(),
                message,
                request.getRequestURI(),
                fields
        );

        return ResponseEntity.status(status).body(body);
    }
}
</CodeBlock>

### Example validation error

A request with an empty title may produce:

```json
{
  "timestamp": "2026-10-28T10:15:30Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Request validation failed.",
  "path": "/api/v1/courses",
  "fieldErrors": {
    "title": "must not be blank"
  }
}
```

The timestamp is illustrative.

We intentionally do not include an all-purpose `Exception` handler yet. Unexpected failures should still be visible in server logs and result in an appropriate server error rather than being mislabeled as client mistakes.

## 9. Configure CORS for Angular development

Your Angular frontend runs at:

`http://localhost:4200`

Your Spring Boot backend runs at:

`http://localhost:8080`

Because they have different origins, browser requests need a CORS policy.

We will allow the Angular development origin only under a Spring `dev` profile.
Create:

`src/main/java/com/learnhub/config/WebConfig.java`

<CodeBlock language="java" editable>
package com.learnhub.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * Development-only CORS policy.
 *
 * Production origins and authentication rules
 * must be configured separately.
 */
@Configuration
@Profile("dev")
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        registry.addMapping("/api/v1/**")
                .allowedOrigins("http://localhost:4200")
                .allowedMethods(
                        "GET",
                        "POST",
                        "PUT",
                        "PATCH",
                        "DELETE",
                        "OPTIONS"
                )
                .allowedHeaders(
                        "Content-Type",
                        "Authorization"
                );
    }
}
</CodeBlock>

Create:

`src/main/resources/application-dev.yml`

<CodeBlock language="yaml" editable>
logging:
  level:
    com.learnhub: DEBUG
</CodeBlock>

Start Spring with the development profile:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

**Security boundary:** CORS controls which browser origins can read responses; it is not authentication or authorization. PowerShell, curl, and other non-browser clients can still call these endpoints.

For that reason, the unauthenticated write API is strictly a local learning feature. Do not deploy it to a publicly reachable environment.

## 10. Test the REST API from PowerShell

Open a second PowerShell terminal while Spring Boot is running.

### 10.1 List published courses

```powershell
$baseUrl = "http://localhost:8080/api/v1/courses"

$result = Invoke-RestMethod -Uri $baseUrl

$result.items |
    Format-Table id, title, category, price, isFree

$result.totalItems
```

Expected: four published courses. Course ID `5` is still a draft.

### 10.2 Search and filter

```powershell
# Search titles, descriptions, instructors, and categories.
Invoke-RestMethod `
  -Uri "$baseUrl`?search=Angular"

# List only free courses.
Invoke-RestMethod `
  -Uri "$baseUrl`?freeOnly=true"

# Select a category.
Invoke-RestMethod `
  -Uri "$baseUrl`?category=Web%20Development"

# Use zero-based pagination.
Invoke-RestMethod `
  -Uri "$baseUrl`?page=0&size=2"
```

The escaped backtick before `?` is harmless here; it prevents PowerShell from interpreting the character in more complex expressions. You can also use `${baseUrl}?search=Angular`.

### 10.3 Retrieve a single course

```powershell
Invoke-RestMethod `
  -Uri "$baseUrl/1"
```

Expected response structure:

```json
{
  "id": 1,
  "title": "Angular Complete Guide",
  "description": "Build modern Angular applications.",
  "instructor": "Alex Morgan",
  "category": "Web Development",
  "price": 49.99,
  "rating": 4.8,
  "isFree": false,
  "level": "BEGINNER",
  "status": "PUBLISHED"
}
```

### 10.4 Verify draft visibility

```powershell
curl.exe -i "$baseUrl/5"
```

Expected HTTP status:

```text
HTTP/1.1 404
```

The draft exists in storage but is not publicly readable.

### 10.5 Create a draft

Prepare a request body:

```powershell
$draft = @{
    title       = "Spring Boot REST API Masterclass"
    description = "Build reliable Java REST APIs."
    instructor  = "Alex Morgan"
    category    = "Web Development"
    price       = 59.99
    isFree      = $false
    level       = "INTERMEDIATE"
    rating      = 4.7
}

$json = $draft | ConvertTo-Json

$created = Invoke-RestMethod `
  -Method Post `
  -Uri $baseUrl `
  -ContentType "application/json" `
  -Body $json

$created
$courseId = $created.id
```

Expected status: `201 Created`.

The new course should have `DRAFT` status and a server-assigned ID, normally `6` with the original seed data.

### 10.6 Update the draft

```powershell
$draft.title = "Spring Boot REST API: Complete Guide"
$draft.price = 64.99

$updated = Invoke-RestMethod `
  -Method Put `
  -Uri "$baseUrl/$courseId" `
  -ContentType "application/json" `
  -Body ($draft | ConvertTo-Json)

$updated
```

Expected: HTTP 200 with the updated draft.

### 10.7 Publish the course

```powershell
$published = Invoke-RestMethod `
  -Method Patch `
  -Uri "$baseUrl/$courseId/publish"

$published.status
```

Expected:

```text
PUBLISHED
```

Now the public endpoint should find it:

```powershell
Invoke-RestMethod -Uri "$baseUrl/$courseId"
```

### 10.8 Archive and delete

```powershell
# Published courses cannot be deleted directly.
curl.exe -i -X DELETE "$baseUrl/$courseId"

# Archive first.
Invoke-RestMethod `
  -Method Patch `
  -Uri "$baseUrl/$courseId/archive"

# Now delete.
Invoke-RestMethod `
  -Method Delete `
  -Uri "$baseUrl/$courseId"

# Confirm the record is gone.
curl.exe -i "$baseUrl/$courseId"
```

Expected sequence: `409 Conflict`, `200 OK`, `204 No Content`, then `404 Not Found`.

### 10.9 Test validation

```powershell
$invalid = @{
    title       = ""
    description = "Invalid course example"
    instructor  = "Alex Morgan"
    category    = "Web Development"
    price       = -10
    isFree      = $false
    level       = "BEGINNER"
    rating      = 7
} | ConvertTo-Json

curl.exe -i `
  -X POST "$baseUrl" `
  -H "Content-Type: application/json" `
  --data-binary $invalid
```

On Windows, passing JSON strings directly through `curl.exe` can be affected by command-line quoting. For a reliable alternative, save the JSON to a file:

```powershell
$invalid | Set-Content -Encoding utf8 .\invalid-course.json

curl.exe -i `
  -X POST "$baseUrl" `
  -H "Content-Type: application/json" `
  --data-binary "@invalid-course.json"
```

Expected: HTTP `400 Bad Request`, with validation details for the invalid fields.

## 11. HTTP status codes: choosing the right response

| Status | Meaning | LearnHub example |
|---|---|---|
| `200 OK` | Successful read or update | Get or update a course |
| `201 Created` | New resource created | Create draft |
| `204 No Content` | Successful operation without body | Delete course |
| `400 Bad Request` | Invalid request | Negative price or invalid enum |
| `401 Unauthorized` | Authentication required | Future protected instructor API |
| `403 Forbidden` | Authenticated but not permitted | Future ownership violation |
| `404 Not Found` | Resource not found or not publicly visible | Unknown or draft course in public GET |
| `409 Conflict` | State or uniqueness conflict | Publishing an already published course |
| `500 Internal Server Error` | Unexpected server failure | Unhandled infrastructure error |

### POST versus PUT versus PATCH

`POST /courses` creates a new resource whose ID is assigned by the server.

`PUT /courses/{id}` replaces the editable fields of an existing draft. Repeating the same request should leave the same final resource representation.

`PATCH /courses/{id}/publish` performs a specific lifecycle transition. In our design, a second publish request returns `409 Conflict` because the course is no longer a draft.

`DELETE /courses/{id}` removes a course, subject to our lifecycle rules.

The HTTP method alone does not enforce permissions. Security checks will be implemented on the server in later lessons.

## 12. Automated Spring MVC tests

We'll use `MockMvc` to test the controller without opening a real network port.

Because our application uses a seeded in-memory repository, tests should avoid depending on mutations made by other tests. The tests below use a fresh Spring application context for each test method, which is slower but keeps the behavior deterministic for this lesson.

Create:

`src/test/java/com/learnhub/web/CourseControllerTest.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.learnhub.LearnHubApplication;

import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.web.servlet.MockMvc;

import java.util.Map;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * End-to-end MVC tests through the Spring test dispatcher.
 *
 * Each test gets fresh in-memory course data.
 */
@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
class CourseControllerTest {

    @Autowired
    private MockMvc mvc;

    @Autowired
    private ObjectMapper objectMapper;

    private String validCourseJson() throws Exception {
        return objectMapper.writeValueAsString(
                Map.of(
                        "title", "Spring Boot Essentials",
                        "description", "Learn Spring MVC.",
                        "instructor", "Alex Morgan",
                        "category", "Web Development",
                        "price", 39.99,
                        "isFree", false,
                        "level", "BEGINNER",
                        "rating", 4.5
                )
        );
    }

    @Test
    void listsPublishedCourses() throws Exception {
        mvc.perform(get("/api/v1/courses"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.totalItems").value(4))
                .andExpect(jsonPath("$.items.length()").value(4));
    }

    @Test
    void returnsPublishedCourse() throws Exception {
        mvc.perform(get("/api/v1/courses/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.title")
                        .value("Angular Complete Guide"))
                .andExpect(jsonPath("$.isFree").value(false));
    }

    @Test
    void hidesDraftCourse() throws Exception {
        mvc.perform(get("/api/v1/courses/5"))
                .andExpect(status().isNotFound());
    }

    @Test
    void filtersFreeCourses() throws Exception {
        mvc.perform(get("/api/v1/courses")
                        .param("freeOnly", "true"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.totalItems").value(2));
    }

    @Test
    void rejectsInvalidPageSize() throws Exception {
        mvc.perform(get("/api/v1/courses")
                        .param("size", "0"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.status").value(400));
    }

    @Test
    void createsDraft() throws Exception {
        mvc.perform(post("/api/v1/courses")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(validCourseJson()))
                .andExpect(status().isCreated())
                .andExpect(header().string(
                        "Location",
                        "/api/v1/courses/6"
                ))
                .andExpect(jsonPath("$.status").value("DRAFT"))
                .andExpect(jsonPath("$.id").value(6));
    }

    @Test
    void rejectsInvalidCourse() throws Exception {
        String invalid = """
                {
                  "title": "",
                  "description": "Example",
                  "instructor": "Alex",
                  "category": "Programming",
                  "price": -5,
                  "isFree": false,
                  "level": "BEGINNER",
                  "rating": 7
                }
                """;

        mvc.perform(post("/api/v1/courses")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalid))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.fieldErrors.title").exists())
                .andExpect(jsonPath("$.fieldErrors.price").exists());
    }

    @Test
    void publishesDraft() throws Exception {
        mvc.perform(patch("/api/v1/courses/5/publish"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status")
                        .value("PUBLISHED"));

        mvc.perform(get("/api/v1/courses/5"))
                .andExpect(status().isOk());
    }

    @Test
    void rejectsPublishingPublishedCourse() throws Exception {
        mvc.perform(patch("/api/v1/courses/1/publish"))
                .andExpect(status().isConflict());
    }

    @Test
    void updatesDraft() throws Exception {
        mvc.perform(put("/api/v1/courses/5")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(validCourseJson()))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.title")
                        .value("Spring Boot Essentials"))
                .andExpect(jsonPath("$.status").value("DRAFT"));
    }

    @Test
    void archivesAndDeletesCourse() throws Exception {
        mvc.perform(patch("/api/v1/courses/1/archive"))
                .andExpect(status().isOk());

        mvc.perform(delete("/api/v1/courses/1"))
                .andExpect(status().isNoContent());

        mvc.perform(get("/api/v1/courses/1"))
                .andExpect(status().isNotFound());
    }

    @Test
    void returnsPublishedCategories() throws Exception {
        mvc.perform(get("/api/v1/courses/categories"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.length()").value(4));
    }
}
</CodeBlock>

This introduces **12 new MVC tests**.

Run:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 6 tests failed."
}
```

<box border radius="lg" padding={3} gap={2}>
  <row align="center" justify="between">    <box gap={1}>
      <text color="secondary" size="xs">Target test count</text>
      <title size="2xl" tabularNums>63</title>
    </box>
    <icon name="test-tube-diagonal" size="2xl" color="secondary"/>
  </row>
  <text size="sm">51 previous tests + 12 new controller tests, assuming you retained the complete earlier test suite.</text>
  <caption>These are expected counts, not results from your computer.</caption>
</box>

## 13. Connect the Angular LearnHub frontend

Our Spring Boot API now exposes real course data, but the JSON contract is not yet identical to the original Angular mock API.

### The compatibility differences

| Field | Existing Angular model | Lesson 6 backend |
|---|---|---|
| `id` | Number | Number |
| `title`, `description` | String | String |
| `instructor`, `category` | String | String |
| `price`, `rating` | Number | JSON decimal number |
| `isFree` | Boolean | Boolean |
| `level` | `"Beginner"`, `"Intermediate"`, `"Advanced"` | `"BEGINNER"`, `"INTERMEDIATE"`, `"ADVANCED"` |
| `thumbnail` | String | Not yet stored |
| `durationHours` | Number | Not yet stored |
| `lastUpdated` | String | Not yet stored |
| List response | Array of courses | Paginated object containing `items` |

The backend is functional, but we must account for these differences explicitly.

### Step 13.1 — Create a frontend API DTO

In a separate PowerShell terminal:

```powershell
cd $HOME\angular-learning\learnhub

New-Item -ItemType Directory -Force `
    src\app\models
```

File: `src/app/models/spring-course-api.model.ts`

<CodeBlock language="typescript" editable>
/**
 * Response returned by Spring Boot Lesson 6.
 *
 * Keep this separate from the frontend's Course model.
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
}

export interface SpringCoursePageDto {
  items: SpringCourseDto[];
  totalItems: number;
  pageNumber: number;
  pageSize: number;
  totalPages: number;
}
</CodeBlock>

### Step 13.2 — Update the Angular CourseApiService

Replace:

`src/app/services/course-api.ts`

<CodeBlock language="typescript" editable>
import { inject, Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { map, Observable } from 'rxjs';

import { Course, CourseLevel } from '../models/course.model';
import {
  SpringCourseDto,
  SpringCoursePageDto
} from '../models/spring-course-api.model';

/**
 * Angular adapter for the Spring Boot course API.
 *
 * Lesson 6 uses temporary presentation metadata for
 * fields the Java backend does not yet persist.
 */
@Injectable({ providedIn: 'root' })
export class CourseApiService {
  private readonly http = inject(HttpClient);

  private readonly apiUrl =
    'http://localhost:8080/api/v1/courses';

  /**
   * Temporary frontend-only display metadata.
   *
   * This is NOT backend course data. Remove it after
   * Spring persists these fields in a later lesson.
   */
  private readonly displayMetadata: Record<number, {
    thumbnail: string;
    durationHours: number;
    lastUpdated: string;
  }> = {
    1: {
      thumbnail:
        'https://placehold.co/600x350/6d28d9/ffffff?text=Angular',
      durationHours: 42.5,
      lastUpdated: '2026-09-15'
    },
    2: {
      thumbnail:
        'https://placehold.co/600x350/2563eb/ffffff?text=Python',
      durationHours: 18,
      lastUpdated: '2026-08-10'
    },
    3: {
      thumbnail:
        'https://placehold.co/600x350/db2777/ffffff?text=UI%2FUX',
      durationHours: 24,
      lastUpdated: '2026-07-20'
    },
    4: {
      thumbnail:
        'https://placehold.co/600x350/0f766e/ffffff?text=Data+Science',
      durationHours: 16,
      lastUpdated: '2026-06-12'
    }
  };

  /**
   * Loads all published courses from the demo catalog.
   *
   * Page size is capped at 100 by the backend.
   * A production implementation must page through
   * all results or expose pagination to the UI.
   */
  getCourses(): Observable<Course[]> {
    const params = new HttpParams()
      .set('page', '0')
      .set('size', '100');

    return this.http.get<SpringCoursePageDto>(
      this.apiUrl,
      { params }
    ).pipe(
      map(response => response.items.map(dto =>
        this.toCourse(dto)
      ))
    );
  }

  searchCourses(term: string): Observable<Course[]> {
    const params = new HttpParams()
      .set('search', term.trim())
      .set('page', '0')
      .set('size', '100');

    return this.http.get<SpringCoursePageDto>(
      this.apiUrl,
      { params }
    ).pipe(
      map(response => response.items.map(dto =>
        this.toCourse(dto)
      ))
    );
  }

  private toCourse(dto: SpringCourseDto): Course {
    const metadata = this.displayMetadata[dto.id];

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

      // Presentation fallbacks for newly created courses.
      thumbnail: metadata?.thumbnail ??
        'https://placehold.co/600x350/64748b/ffffff?text=Course',

      durationHours: metadata?.durationHours ?? 0,

      // Empty means the backend has not supplied a date.
      lastUpdated: metadata?.lastUpdated ?? ''
    };
  }
}
</CodeBlock>

The adapter uses the original demo metadata for courses `1–4`. For newly created courses, it uses a generic thumbnail and explicitly unknown duration/date values. Those are temporary UI fallbacks, not fabricated backend records.

The current `getCourses()` implementation fetches up to 100 published courses. That's enough for our small local catalog, but it is not a complete pagination implementation.

### Step 13.3 — Run both applications

<grid columns={2} gap={3}>
  <grid-item>
    <box border radius="lg" padding={3} gap={2}>
      **Terminal A — Spring Boot**

      ```powershell
      cd $HOME\angular-learning\learnhub-backend

      mvn spring-boot:run `
        "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
      ```

      <text size="xs" color="secondary">Backend: http://localhost:8080</text>
    </box>
  </grid-item>
  <grid-item>
    <box border radius="lg" padding={3} gap={2}>
      **Terminal B — Angular**

      ```powershell
      cd $HOME\angular-learning\learnhub

      ng.cmd serve --open
      ```

      <text size="xs" color="secondary">Frontend: http://localhost:4200</text>
    </box>
  </grid-item>
</grid>

You no longer need to start `json-server` for course catalog reads.

### What should work now?

The Angular catalog should load the four published courses from Spring Boot. Search should use the backend's search endpoint. Course detail pages should continue to work if they read from the existing Angular `CourseService` state.

Wishlist, enrollment progress, and demo authentication still operate using the frontend's existing local services. We have **not yet migrated those features to the backend**.

Also, the existing Angular course detail and enrollment behavior still uses numeric IDs. The new Spring API preserves numeric IDs, so that part of the contract remains compatible.

## 14. Important improvements before production

This lesson intentionally delivers a local development API. It is not a production-ready instructor administration API.

**Authorization:** Every write endpoint is currently open. Before public deployment, Spring Security must authenticate callers and verify instructor ownership. Angular guards alone cannot protect backend resources.

**Persistence:** Courses are stored in memory. Changes disappear when Spring Boot restarts. PostgreSQL will make data durable.

**Concurrency:** The demo ID generator and repository operations do not provide database-grade uniqueness or transactional consistency.

**Validation:** Bean Validation checks request shape and basic constraints. Domain invariants still need to be enforced, and database constraints will provide an additional safety layer.

**Frontend parity:** We still need persisted thumbnails, duration, last-updated timestamps, instructor IDs, lessons, enrollments, reviews, and other fields required by the full LearnHub frontend.

**API pagination:** The backend supports page and size, but the Angular adapter currently reads only the first 100 courses. We'll implement proper client pagination and server-side sorting later.

### Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Browser reports CORS error | Spring not running with `dev` profile | Start with `--spring.profiles.active=dev` |
| Angular shows empty catalog | Backend unavailable or API URL wrong | Test `/api/v1/courses` in PowerShell |
| Compilation fails on `PageResult` accessors | Lesson 2 accessor names differ | Match your actual `PageResult` record |
| HTTP 400 on POST | Invalid JSON, enum, or validation | Inspect `fieldErrors` |
| HTTP 404 for course 5 | It is a draft | Publish it before public GET |
| HTTP 409 on publish | Course is not in DRAFT status | Review lifecycle state |
| Changes disappear after restart | In-memory repository | Expected until PostgreSQL |
| Maven cannot find validation annotations | Missing starter | Verify `spring-boot-starter-validation` |
| Test counts differ | Different earlier test files | Check Surefire reports rather than assuming 63 |

## 15. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md"><icon name="list-filter" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise A — Add sorting to GET /courses**

      Support `sort=price-asc`, `sort=price-desc`, and `sort=rating-desc`. Map the query parameter to the existing `CourseSort` options from Lesson 2, reject unsupported values with HTTP 400, and write three MVC tests.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md"><icon name="shield-check" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise B — Improve validation**

      Add a cross-field validation rule: when `isFree` is true, `price` must be zero. When `isFree` is false, the price must be greater than zero. Return a clear validation response and add tests.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md"><icon name="file-json" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise C — Improve response DTOs**

      Introduce a separate `CourseSummaryResponse` for course cards and `CourseDetailResponse` for the detail page. Explain why different endpoints may need different representations.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md"><icon name="test-tube" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise D — Full CRUD test**

      Write one integration test that creates a draft, updates it, publishes it, archives it, deletes it, and confirms a final 404. Check the `Location` header and response body at each step.
    </box>
  </row>
</box>

## 16. Lesson 6 quiz

<LearningQuiz
  title="Spring MVC REST API Knowledge Check"
  nextQuizQuery="Give me 5 new advanced LearnHub Spring MVC REST API quiz questions covering validation, idempotency, DTO mapping, pagination, and error handling without repeating Lesson 6 questions."
  questions={[
    {
      "id":"q1",
      "type":"single_select",
      "question":"Which HTTP status best represents successful creation of a new LearnHub draft course with a Location header?",
      "options":[
        {"value":"A","label":"200 OK","feedback":"200 indicates success, but 201 specifically communicates that a resource was created."},
        {"value":"B","label":"201 Created","feedback":"201 identifies successful resource creation, and Location can identify the created resource."},
        {"value":"C","label":"202 Accepted","feedback":"202 indicates accepted processing that may not yet be complete."},
        {"value":"D","label":"204 No Content","feedback":"204 has no response body and does not specifically signal resource creation."}
      ],
      "correctValues":["B"],
      "hint":"Consider whether the operation merely succeeded or created a new addressable resource."
    },
    {
      "id":"q2",
      "type":"single_select",
      "question":"A public GET request targets an existing LearnHub course whose status is DRAFT. Which response matches the Lesson 6 public visibility contract?",
      "options":[
        {"value":"A","label":"200 with the draft course","feedback":"The public catalog must not expose draft content."},
        {"value":"B","label":"204 with no content","feedback":"204 indicates a successful request without a body, not the intended hidden-resource behavior."},
        {"value":"C","label":"404 Not Found","feedback":"The public lookup hides drafts and archived courses, treating them as unavailable."},
        {"value":"D","label":"409 Conflict","feedback":"409 is used for conflicting state-changing operations, not this public lookup."}
      ],
      "correctValues":["C"],
      "hint":"Distinguish internal storage existence from public resource visibility."
    },
    {
      "id":"q3",
      "type":"multi_select",
      "question":"Which practices improve the correctness of a LearnHub REST API? Select all that apply.",
      "options":[
        {"value":"A","label":"Use DTOs instead of exposing persistence entities directly","feedback":"DTOs allow the public contract to evolve without exposing internal persistence fields."},
        {"value":"B","label":"Validate request bodies and preserve domain invariants","feedback":"Both request validation and domain validation are needed at different boundaries."},
        {"value":"C","label":"Rely on CORS as the authorization mechanism for course writes","feedback":"CORS does not authenticate users or authorize non-browser clients."},
        {"value":"D","label":"Return consistent structured error responses","feedback":"Stable error shapes make client handling and debugging more reliable."}
      ],
      "correctValues":["A","B","D"],
      "hint":"Evaluate contract boundaries, input correctness, and client error handling separately."
    },
    {
      "id":"q4",
      "type":"true_false",
      "question":"A Spring MVC controller can return a Java record and have it serialized as JSON when a suitable HTTP message converter is configured.",
      "options":[
        {"value":"true","label":"True","feedback":"Spring MVC can use Jackson to serialize record components into JSON response properties."},
        {"value":"false","label":"False","feedback":"A record can be serialized by Jackson; a manually constructed JSON string is not required."}
      ],
      "correctValues":["true"],
      "hint":"Think about the role of HTTP message converters in Spring MVC."
    },
    {
      "id":"q5",
      "type":"true_false",
      "question":"Adding a CORS policy that allows only localhost:4200 prevents curl and PowerShell clients from invoking unauthenticated course write endpoints.",
      "options":[
        {"value":"true","label":"True","feedback":"CORS is enforced by browsers; it is not a server-side authentication or authorization control for arbitrary HTTP clients."},
        {"value":"false","label":"False","feedback":"Non-browser clients can still send requests, so backend security must protect write operations."}
      ],
      "correctValues":["false"],
      "hint":"Consider which clients enforce the browser same-origin policy."
    }
  ]}
/>

## 17. Lesson completion checklist

{@body const groups=[{title:"Project and REST design",items:["Verified Lesson 5 Maven baseline","Added spring-boot-starter-validation","Understood REST methods and HTTP statuses","Created CourseResponse, CourseWriteRequest, and CoursePageResponse"]},{title:"Backend implementation",items:["Extended CourseManagementService","Created CourseController with CRUD endpoints","Created ApiError and GlobalExceptionHandler","Configured development CORS","Verified public draft visibility rules"]},{title:"API testing",items:["GET courses and individual course work","Search, filters, and pagination work","POST creates a draft","PUT updates a draft","PATCH publishes and archives","DELETE removes an eligible course","Invalid requests return 400 or 409 as appropriate","Automated MVC tests pass"]},{title:"Angular and final verification",items:["Created Angular SpringCourseDto","Updated Angular CourseApiService","Angular loads published courses from Spring Boot","Understood missing frontend metadata limitations","Completed at least two exercises","Maven clean verify passes","Committed Lesson 6 changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((s,g)=>s+g.items.length,0)}
{@body const count=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    <box gap={1}>
      **Lesson 6 progress**
      <text color="secondary" size="xs">Mark milestones after verifying them locally.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${100*count/total}%`} radius="full"/>
  </box>
  {#each groups as group,i}
    <box gap={2} key={i}>
      **{group.title}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!done[`${i}-${j}`]} onChange={v=>setDone(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if count===total}
    <row gap={2} align="center">
      <icon name="check-circle" color="success"/>
      **Lesson 6 checklist completed.**
    </row>
  {/if}
</box>

## 18. Final verification and Git checkpoint

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 6 backend verification failed."
}

git status
git add .
git commit -m "Implement LearnHub Spring MVC course REST API"
```

Frontend:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular integration build failed."
}

git status
git add .
git commit -m "Connect Angular LearnHub catalog to Spring Boot"
```

Commit each project only if it is a Git repository and contains the intended changes.

## 19. Next lesson — PostgreSQL and Spring Data JPA

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="database" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 7 — PostgreSQL Database Integration, Spring Data JPA, Hibernate, and Flyway**

    We'll replace the temporary in-memory repository with PostgreSQL, design a `courses` table, introduce JPA entities and Spring Data repositories, manage schema changes with Flyway migrations, and persist the course data that the Angular frontend needs.

    We'll also replace demo ID allocation with database-generated identifiers and begin testing persistence behavior.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 7: PostgreSQL installation on Windows, database setup, Spring Data JPA, Hibernate entities, repository interfaces, Flyway migrations, database-generated course IDs, persistence testing, and Angular course field compatibility. Continue from Lesson 6 with experienced-Java pacing, complete Windows PowerShell commands, exact file paths, documented code, tests, exercises, quiz, and checklist.")}>Start Lesson 7 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 6 deliverable:** A Spring Boot course REST API with public catalog reads, development-only CRUD operations, request validation, structured HTTP errors, automated MVC tests, and an Angular adapter that replaces the original mock course API.

The next major milestone is making LearnHub's course data durable in PostgreSQL.