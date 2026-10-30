# LearnHub Java & Spring Boot — Lesson 24
<text color="secondary" size="sm">Final Capstone · Full-Stack Integration · Security · Payments · Testing · Deployment · Portfolio</text>

<row align="center" gap={2} wrap="wrap">
  <badge color="success">Lesson 24 of 24</badge>
  <text color="secondary" size="xs">Experienced Java · 16–24 hours · Java 21 / Spring Boot 3.5.x / Angular / PostgreSQL</text>
</row>

## 1. Your final project

For the past 23 lessons, we've developed LearnHub from Java fundamentals into a full-stack course marketplace.

The capstone is not about adding more features. It's about proving that the features work **together**, including when requests fail, transactions race, or external services become unavailable.

Your final deliverable is a LearnHub release candidate with:

- A public Angular storefront and course catalog.
- Student registration, login, wishlist, enrollment, curriculum, and learning progress.
- Instructor-owned courses, draft editing, readiness checks, publishing, archiving, and dashboard analytics.
- Server-authoritative paid checkout, signed payment webhooks, idempotent fulfillment, and refund-aware access.
- PostgreSQL persistence, Flyway migrations, tests, observability, and a containerized deployment.
- A portfolio-quality README and a documented release decision.

### Final architecture

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    <icon name="users" size="lg"/>
    **Browser: Student / Instructor**
    <text color="secondary" size="xs">Angular standalone · typed APIs · route guards</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} gap={2}>
    <row justify="between" align="center">
      **Nginx**
      <text color="secondary" size="xs">HTTPS at trusted edge</text>
    </row>
    <grid columns={2} gap={2}>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2} align="center" gap={1}>
          **`/`**
          <text color="secondary" size="xs">Angular SPA</text>
        </box>
      </grid-item>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2} align="center" gap={1}>
          **`/api/v1/`**
          <text color="secondary" size="xs">Spring Boot API</text>
        </box>
      </grid-item>
    </grid>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} gap={2}>
    <box align="center" gap={1}>
      **Spring Boot application**
      <text color="secondary" size="xs">Session authentication · CSRF · authorization</text>
    </box>
    <grid columns={2} gap={2}>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2}>
          **Catalog & Courses**
          <text color="secondary" size="xs">Search · Ownership · Publishing</text>
        </box>
      </grid-item>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2}>
          **Learning**
          <text color="secondary" size="xs">Enrollment · Curriculum · Progress</text>
        </box>
      </grid-item>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2}>
          **Payments**
          <text color="secondary" size="xs">Checkout · Webhooks · Refunds</text>
        </box>
      </grid-item>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2}>
          **Operations**
          <text color="secondary" size="xs">Actuator · Metrics · Logs</text>
        </box>
      </grid-item>
    </grid>
  </box>
  <grid columns={2} gap={2}>
    <grid-item>
      <box align="center" gap={2}>
        <icon name="arrow-down" color="secondary"/>
        <box border radius="lg" padding={3} width="100%" align="center">
          <icon name="database"/>
          **PostgreSQL**
          <text color="secondary" size="xs">Flyway · JPA · Transactions</text>
        </box>
      </box>
    </grid-item>
    <grid-item>
      <box align="center" gap={2}>
        <icon name="arrow-down" color="secondary"/>
        <box border radius="lg" padding={3} width="100%" align="center">
          <icon name="credit-card"/>
          **Payment provider**
          <text color="secondary" size="xs">Hosted checkout · Signed events</text>
        </box>
      </box>
    </grid-item>
  </grid>
</box>

### The final engineering standard

A feature is complete only when its **contract, authorization, persistence, failure behavior, tests, and deployment configuration** agree.

That standard guides the entire lesson.

---

## 2. Workspace and release baseline

We'll continue with the existing three-directory layout.

```text
$HOME\angular-learning\
├── learnhub\                Angular frontend
├── learnhub-backend\        Java 21 / Spring Boot backend
└── learnhub-deploy\         Docker Compose and deployment scripts
```

### PowerShell: establish your baseline

```powershell
$workspace = Join-Path $HOME "angular-learning"

$frontend = Join-Path $workspace "learnhub"
$backend  = Join-Path $workspace "learnhub-backend"
$deploy   = Join-Path $workspace "learnhub-deploy"

Set-Location $backend

java -version
mvn -version
docker info

git status --short
git log -1 --oneline

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend baseline is not passing."
}

Set-Location $frontend

node --version
npm --version

git status --short
git log -1 --oneline

npm ci
npm run build

if ($LASTEXITCODE -ne 0) {
    throw "Frontend baseline is not passing."
}
```

**Do not begin a production deployment while the baseline fails.** This capstone is intended to uncover integration defects, not hide them.

### Release-candidate directory

Create a place for nonsecret release evidence:

```powershell
$releaseDir = Join-Path $workspace "learnhub-release"

New-Item -ItemType Directory -Force `
    $releaseDir, `
    (Join-Path $releaseDir "evidence"), `
    (Join-Path $releaseDir "docs") |
    Out-Null
```

We'll use it for the architecture decision record, release checklist, and portfolio documentation.

---

## 3. Contract reconciliation: one source of truth

Earlier lessons introduced API routes incrementally. Before writing more code, audit the actual controllers.

For example, Lesson 11 used `POST /courses/{id}/publish`, while Lesson 16 illustrated `PATCH /courses/{id}/publish`. Your final application must have **one canonical contract**.

### 3.1 Inventory controller routes

```powershell
cd $HOME\angular-learning\learnhub-backend

Get-ChildItem src\main\java -Recurse -Filter "*Controller.java" |
    Select-String `
        -Pattern '@RequestMapping|@GetMapping|@PostMapping|@PutMapping|@PatchMapping|@DeleteMapping' |
    Select-Object Path, LineNumber, Line
```

Export OpenAPI from the running backend as a second source of evidence:

```powershell
# Run the backend in the dev profile first.
Invoke-RestMethod `
    "http://localhost:8080/v3/api-docs" |
    ConvertTo-Json -Depth 100 |
    Set-Content `
        (Join-Path $releaseDir "evidence\openapi.json")
```

### 3.2 Canonical capstone contract

This is the **target** route inventory. Compare it with your real controller mappings and adapt where necessary.

| Domain | Endpoint | Access |
|---|---|---|
| Catalog | `GET /api/v1/courses` | Public |
| Course details | `GET /api/v1/courses/{id}` | Public, published only |
| Registration | `POST /api/v1/auth/register` | Anonymous + CSRF |
| Login | `POST /api/v1/auth/login` | Anonymous + CSRF |
| Session | `GET /api/v1/auth/me` | Authenticated |
| Logout | `POST /api/v1/auth/logout` | Authenticated + CSRF |
| Instructor courses | `GET /api/v1/instructor/courses` | Instructor/admin |
| Instructor course detail | `GET /api/v1/instructor/courses/{id}` | Owner/admin |
| Draft creation | `POST /api/v1/courses` | Instructor/admin |
| Draft update | `PUT /api/v1/courses/{id}` | Owner/admin |
| Publish | `POST /api/v1/courses/{id}/publish` | Owner/admin |
| Archive | `POST /api/v1/courses/{id}/archive` | Owner/admin |
| Wishlist | `GET /api/v1/me/wishlist` | Authenticated |
| Wishlist save | `PUT /api/v1/me/wishlist/{courseId}` | Authenticated + CSRF |
| Free enrollment | `POST /api/v1/me/enrollments` | Student + CSRF |
| My enrollments | `GET /api/v1/me/enrollments` | Student |
| Course player | `GET /api/v1/me/courses/{id}/curriculum` | Enrolled student |
| Lesson completion | `PUT /api/v1/me/lessons/{id}/completion` | Enrolled student + CSRF |
| Paid checkout | `POST /api/v1/me/checkout/courses/{id}` | Student + CSRF |
| Payment status | `GET /api/v1/me/payments/{id}` | Payment owner |
| Stripe webhook | `POST /api/v1/webhooks/stripe` | Valid signature |

Do not add aliases merely to make old frontend code pass. Fix the frontend service to use the selected backend route.

### 3.3 Check for contract drift

Search the Angular API services:

```powershell
cd $HOME\angular-learning\learnhub

Get-ChildItem src\app -Recurse `
    -Include "*.service.ts","*.api.ts" |
    Select-String `
        -Pattern '/api/v1/' |
    Select-Object Path, LineNumber, Line
```

Look especially for:

- `POST` versus `PATCH` publishing.
- `free` versus `freeOnly` catalog query parameters.
- `id` versus `courseId` path variables.
- `isFree` versus `free` JSON fields.
- Instructor draft reads accidentally using the public course endpoint.
- Checkout bodies containing browser-controlled prices.
- Duplicate auth or error interceptors.

**Capstone gate:** No known contract drift remains between the Java controllers, OpenAPI document, and Angular API services.

---

## 4. PostgreSQL and Flyway schema audit

Before validating user journeys, verify the database foundation.

### 4.1 Review migration history

```powershell
cd $HOME\angular-learning\learnhub-backend

Get-ChildItem `
    src\main\resources\db\migration `
    -Filter "V*.sql" |
    Sort-Object Name |
    Select-Object Name
```

Across the course, the schema evolved to include users, courses, enrollments, wishlists, curriculum, progress, payment attempts, webhook events, and entitlements.

The exact migration versions after Lesson 17 depend on your implemented files. **Never renumber or edit a migration already applied to a shared database.** Create a new migration for corrections.

### 4.2 Final schema audit SQL

**Create:** `learnhub-release/docs/schema-audit.sql`

<CodeBlock language="sql" editable>
-- Lesson 24: Read-only schema and integrity audit.
-- Run against a staging copy or development database.

-- 1. Flyway history
SELECT installed_rank,
       version,
       description,
       type,
       success
FROM flyway_schema_history
ORDER BY installed_rank;

-- 2. Duplicate normalized emails
SELECT lower(email) AS normalized_email,
       COUNT(*) AS occurrences
FROM app_users
GROUP BY lower(email)
HAVING COUNT(*) > 1;

-- 3. Duplicate student/course enrollments
SELECT student_user_id,
       course_id,
       COUNT(*) AS occurrences
FROM enrollments
GROUP BY student_user_id, course_id
HAVING COUNT(*) > 1;

-- 4. Courses missing instructor ownership
SELECT id, title, status
FROM courses
WHERE instructor_user_id IS NULL
ORDER BY id;

-- 5. Enrollment status distribution
SELECT status, COUNT(*)
FROM enrollments
GROUP BY status
ORDER BY status;

-- 6. Public catalog state distribution
SELECT status, COUNT(*)
FROM courses
GROUP BY status
ORDER BY status;

-- 7. Verify important foreign keys and unique constraints
SELECT conrelid::regclass AS table_name,
       conname,
       contype,
       pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE conrelid IN (
    'courses'::regclass,
    'app_users'::regclass,
    'enrollments'::regclass
)
ORDER BY table_name, conname;
</CodeBlock>

### Important interpretation

Courses with `instructor_user_id IS NULL` may be legacy seeded courses from Lesson 7, not necessarily a production defect. Decide whether they must be assigned to real instructor accounts before launch.

A missing foreign key or unique constraint is a release blocker when application correctness depends on it.

### 4.3 Run the audit with PostgreSQL

From the deployment directory:

```powershell
cd $HOME\angular-learning\learnhub-deploy

docker compose ps

docker cp `
  "$releaseDir\docs\schema-audit.sql" `
  "$(docker compose ps -q db):/tmp/schema-audit.sql"

docker compose exec -T db sh -c `
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -v ON_ERROR_STOP=1 -f /tmp/schema-audit.sql'
```

The audit is read-only. Keep the output as release evidence, but redact user emails or other sensitive records before sharing it.

---

## 5. Final domain invariants

These rules should hold regardless of whether the request originates from Angular, a REST client, a scheduled task, or a webhook.

| Domain | Invariant | Enforcement |
|---|---|---|
| Identity | The server determines the acting user | Spring Security principal |
| Course ownership | Only the owner or an authorized admin manages a course | Service policy + database FK |
| Public catalog | Only `PUBLISHED` courses are visible | Repository query |
| Publishing | A course must pass curriculum readiness checks | Transactional lifecycle service |
| Enrollment | One enrollment per student/course | PostgreSQL unique constraint |
| Lesson access | Only eligible enrolled students see protected content | Enrollment/entitlement service |
| Progress | Completion is idempotent and scoped to the student | Unique constraint + access check |
| Checkout | Amount and currency come from server state | Checkout service |
| Payment fulfillment | Only verified, paid events grant access | Signature + amount verification |
| Webhook replay | Repeated events cannot repeat fulfillment | Unique event ID + transaction |
| Refund | Revocation follows entitlement provenance | Payment/entitlement policy |
| Reporting | Instructor sees only owned-course data | Owner-scoped aggregation |
| Operations | Secrets never appear in logs or metrics | Logging and telemetry policy |

### One especially important rule

**Never use Angular route guards as the only authorization mechanism.**

A student can bypass Angular and send HTTP requests directly. Every protected backend operation must enforce identity and authorization independently.

---

## 6. Student and instructor acceptance journeys

Before implementing the final regression tests, write down the behaviors we expect.

### Student journey

<list marker="number" connector="solid" gap={2}>
  <list-item>Register an account, retrieve a CSRF token, and log in through a server-side session.</list-item>
  <list-item>Search published courses. Verify draft and archived courses are absent.</list-item>
  <list-item>Save a course to the wishlist and verify it remains after refreshing the page.</list-item>
  <list-item>Enroll in a published free course. Repeated enrollment must not create duplicate records.</list-item>
  <list-item>Open the course player, read an enrolled-only lesson, and mark it complete.</list-item>
  <list-item>Start checkout for a paid course. The browser sends the course ID, never a price or student ID.</list-item>
  <list-item>Complete a test-mode payment. Wait for verified webhook fulfillment, then confirm course access.</list-item>
  <list-item>Log out and verify protected requests are rejected.</list-item>
</list>

### Instructor journey

<list marker="number" connector="solid" gap={2}>
  <list-item>Sign in with an instructor account provisioned through a trusted administrative process.</list-item>
  <list-item>Create a draft. Confirm it is not publicly visible.</list-item>
  <list-item>Add sections and lessons, then verify ordering and readiness checks.</list-item>
  <list-item>Publish using the expected optimistic-lock version.</list-item>
  <list-item>Confirm the published course appears in the public catalog.</list-item>
  <list-item>Review dashboard analytics restricted to owned courses.</list-item>
  <list-item>Attempt to modify another instructor's course and verify rejection without database changes.</list-item>
  <list-item>Archive a course and verify its public visibility and existing-student access follow the documented policy.</list-item>
</list>

**Exercise:** Perform both journeys in the browser before reading the implementation sections below. Record every discrepancy as a reproducible bug, not a vague observation.

---

## 7. Capstone implementation: centralize learning access

A frequent architectural problem is checking enrollment differently in the curriculum controller, lesson-content controller, and progress service.

Instead, use a single authorization policy.

We'll use this final rule:

- `ACTIVE` or `COMPLETED` enrollment grants learning access.
- `CANCELLED` enrollment does not.
- Existing students retain access to an archived course **unless their entitlement has been revoked**.
- Draft courses are not available to students.
- A paid course requires a currently valid paid entitlement, not merely a historical enrollment row.

This last condition is critical for refund correctness.

### 7.1 A small entitlement abstraction

**Create:** `learnhub-backend/src/main/java/com/learnhub/learning/LearningEntitlement.java`

<CodeBlock language="java" editable>
package com.learnhub.learning;

/**
 * A normalized authorization decision.
 *
 * Domain services should not need to understand the
 * payment provider's webhook or refund state machine.
 */
public record LearningEntitlement(
        long studentId,
        long courseId,
        boolean allowed,
        String reason
) {
    public static LearningEntitlement allow(
            long studentId,
            long courseId
    ) {
        return new LearningEntitlement(
                studentId,
                courseId,
                true,
                "ACCESS_GRANTED"
        );
    }

    public static LearningEntitlement deny(
            long studentId,
            long courseId,
            String reason
    ) {
        return new LearningEntitlement(
                studentId,
                courseId,
                false,
                reason
        );
    }
}
</CodeBlock>

### 7.2 Enrollment repository query

Add a read query to your existing `JpaEnrollmentRepository`.

<CodeBlock language="java" editable>
@Query("""
    SELECT COUNT(e) > 0
    FROM EnrollmentEntity e
    WHERE e.student.id = :studentId
      AND e.course.id = :courseId
      AND e.status IN ('ACTIVE', 'COMPLETED')
    """)
boolean hasActiveLearningEnrollment(
        @Param("studentId") Long studentId,
        @Param("courseId") Long courseId
);
</CodeBlock>

Imports:

```java
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
```

This assumes the Lesson 17 enrollment entity still stores `status` as a `String`. If Lesson 18 introduced an enum, use enum parameters or the correct enum literals instead.

### 7.3 Paid entitlement boundary

**Create:** `learnhub-backend/src/main/java/com/learnhub/learning/PaidAccessLookup.java`

<CodeBlock language="java" editable>
package com.learnhub.learning;

/**
 * Implement using the paid entitlement model from
 * Lesson 20, not payment redirect query parameters.
 */
public interface PaidAccessLookup {

    /**
     * Returns true only when the student's paid
     * entitlement is currently valid.
     */
    boolean hasValidEntitlement(
            long studentId,
            long courseId
    );
}
</CodeBlock>

This interface deliberately does not guess the exact payment entity names from Lesson 20. Your implementation must read the authoritative entitlement state, including refund/revocation rules.

Do **not** implement it as `return true` or as “a successful payment once existed.”

### 7.4 Learning access policy

**Create:** `learnhub-backend/src/main/java/com/learnhub/learning/LearningAccessPolicy.java`

<CodeBlock language="java" editable>
package com.learnhub.learning;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.persistence.JpaEnrollmentRepository;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class LearningAccessPolicy {

    private final JpaCourseRepository courses;
    private final JpaEnrollmentRepository enrollments;
    private final PaidAccessLookup paidAccess;

    public LearningAccessPolicy(
            JpaCourseRepository courses,
            JpaEnrollmentRepository enrollments,
            PaidAccessLookup paidAccess
    ) {
        this.courses = courses;
        this.enrollments = enrollments;
        this.paidAccess = paidAccess;
    }

    @Transactional(readOnly = true)
    public LearningEntitlement evaluate(
            long studentId,
            long courseId
    ) {
        CourseEntity course = courses.findById(courseId)
                .orElse(null);

        if (course == null
                || course.getStatus() == CourseStatus.DRAFT) {
            return LearningEntitlement.deny(
                    studentId,
                    courseId,
                    "COURSE_UNAVAILABLE"
            );
        }

        boolean enrolled =
                enrollments.hasActiveLearningEnrollment(
                        studentId,
                        courseId
                );

        if (!enrolled) {
            return LearningEntitlement.deny(
                    studentId,
                    courseId,
                    "NOT_ENROLLED"
            );
        }

        if (!course.isFree()
                && !paidAccess.hasValidEntitlement(
                        studentId,
                        courseId
                )) {
            return LearningEntitlement.deny(
                    studentId,
                    courseId,
                    "PAID_ACCESS_REVOKED"
            );
        }

        return LearningEntitlement.allow(
                studentId,
                courseId
        );
    }

    @Transactional(readOnly = true)
    public void requireAccess(
            long studentId,
            long courseId
    ) {
        LearningEntitlement decision =
                evaluate(studentId, courseId);

        if (!decision.allowed()) {
            // Non-disclosing response for protected content.
            throw new LearningContentNotFoundException();
        }
    }
}
</CodeBlock>

**Create:** `learnhub-backend/src/main/java/com/learnhub/learning/LearningContentNotFoundException.java`

<CodeBlock language="java" editable>
package com.learnhub.learning;

public final class LearningContentNotFoundException
        extends RuntimeException {

    public LearningContentNotFoundException() {
        super("Learning content not found.");
    }
}
</CodeBlock>

Map this exception to `404` in your existing `GlobalExceptionHandler`.

<CodeBlock language="java" editable>
@ExceptionHandler(LearningContentNotFoundException.class)
public ResponseEntity<ProblemDetail> learningContentNotFound(
        LearningContentNotFoundException exception,
        HttpServletRequest request
) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(problemFactory.create(
                    HttpStatus.NOT_FOUND,
                    "Learning content not found.",
                    request
            ));
}
</CodeBlock>

Reuse the Lesson 22 `ProblemFactory` and its existing imports.

### Integration point

Every protected lesson-content read and lesson-completion write must call:

```java
learningAccessPolicy.requireAccess(
        currentActor.id(),
        courseId
);
```

Resolve `courseId` through the lesson's actual section/course relationship. Never accept the course ID supplied by a client as proof that a lesson belongs to that course.

This policy is intentionally read-oriented. A concurrent refund and content read may race; for financial state transitions, use the shared entitlement-locking protocol from Lesson 20.

---

## 8. Capstone unit tests for learning access

**Create:** `src/test/java/com/learnhub/learning/LearningAccessPolicyTest.java`

<CodeBlock language="java" editable>
package com.learnhub.learning;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.persistence.JpaEnrollmentRepository;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class LearningAccessPolicyTest {

    @Mock JpaCourseRepository courses;
    @Mock JpaEnrollmentRepository enrollments;
    @Mock PaidAccessLookup paidAccess;

    @InjectMocks LearningAccessPolicy policy;

    @Test
    void freeEnrolledStudentHasAccess() {
        CourseEntity course = mock(CourseEntity.class);

        when(courses.findById(10L))
                .thenReturn(Optional.of(course));
        when(course.getStatus())
                .thenReturn(CourseStatus.PUBLISHED);
        when(course.isFree()).thenReturn(true);

        when(enrollments.hasActiveLearningEnrollment(
                5L, 10L
        )).thenReturn(true);

        LearningEntitlement decision =
                policy.evaluate(5L, 10L);

        assertThat(decision.allowed()).isTrue();

        verifyNoInteractions(paidAccess);
    }

    @Test
    void paidCourseRequiresValidEntitlement() {
        CourseEntity course = mock(CourseEntity.class);

        when(courses.findById(20L))
                .thenReturn(Optional.of(course));
        when(course.getStatus())
                .thenReturn(CourseStatus.PUBLISHED);
        when(course.isFree()).thenReturn(false);

        when(enrollments.hasActiveLearningEnrollment(
                5L, 20L
        )).thenReturn(true);

        when(paidAccess.hasValidEntitlement(
                5L, 20L
        )).thenReturn(false);

        assertThatThrownBy(() ->
                policy.requireAccess(5L, 20L)
        ).isInstanceOf(
                LearningContentNotFoundException.class
        );
    }

    @Test
    void draftIsUnavailableEvenWhenEnrollmentExists() {
        CourseEntity course = mock(CourseEntity.class);

        when(courses.findById(30L))
                .thenReturn(Optional.of(course));
        when(course.getStatus())
                .thenReturn(CourseStatus.DRAFT);

        assertThat(policy.evaluate(5L, 30L).allowed())
                .isFalse();

        verifyNoInteractions(enrollments, paidAccess);
    }

    @Test
    void archivedCourseRetainsEligibleStudentAccess() {
        CourseEntity course = mock(CourseEntity.class);

        when(courses.findById(40L))
                .thenReturn(Optional.of(course));
        when(course.getStatus())
                .thenReturn(CourseStatus.ARCHIVED);
        when(course.isFree()).thenReturn(true);

        when(enrollments.hasActiveLearningEnrollment(
                5L, 40L
        )).thenReturn(true);

        assertThat(policy.evaluate(5L, 40L).allowed())
                .isTrue();
    }
}
</CodeBlock>

These tests verify domain decisions, not actual database isolation. Add the integration tests in Section 12 to prove the repository query and protected endpoints use the same rules.

---

## 9. Payment and refund consistency: final audit

The most important capstone failure scenario is a race between payment fulfillment and refund processing.

Imagine:

1. A student purchases a paid course.
2. The payment webhook arrives and grants access.
3. A refund webhook arrives and revokes that access.
4. A delayed duplicate payment webhook arrives.

The duplicate must **not** restore access after a full refund.

### Recommended entitlement state model

<box border radius="lg" padding={3} gap={2}>
  <row justify="center">
    <box background="surface-secondary" radius="md" padding={3} align="center">
      **NO_ACCESS**
      <text size="xs" color="secondary">No valid purchase</text>
    </box>
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row justify="center">
    <box border radius="md" padding={3} align="center">
      **ACTIVE**
      <text size="xs" color="secondary">Verified payment fulfilled</text>
    </box>
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row justify="center">
    <box background="surface-secondary" radius="md" padding={3} align="center">
      **REVOKED**
      <text size="xs" color="secondary">Full refund / chargeback policy</text>
    </box>
  </row>
  <caption>Access may become ACTIVE again only through a new valid entitlement or an explicitly authorized reinstatement—not through replaying an old event.</caption>
</box>

### 9.1 Invariants to test

| Scenario | Expected outcome |
|---|---|
| Invalid webhook signature | No payment state change |
| Correct signature, wrong amount | No fulfillment |
| Correct amount, wrong currency | No fulfillment |
| Same event ID delivered twice | One durable processing result |
| Different event IDs for same payment | No duplicate entitlement |
| Full refund | Revoke only the entitlement tied to that payment |
| Partial refund | Follow explicit partial-refund policy |
| Refund then duplicate paid event | Remain revoked |
| New valid purchase after refund | Access may be restored |
| Concurrent grant and revoke | Deterministic final state |

### 9.2 PostgreSQL locking approach

For concurrent financial transitions, lock a shared row representing the student's entitlement for the course.

For example, the transaction boundary should be conceptually:

```sql
BEGIN;

SELECT id, status, source_payment_attempt_id
FROM course_entitlements
WHERE student_user_id = :student_id
  AND course_id = :course_id
FOR UPDATE;

-- Validate event and payment provenance.
-- Apply the allowed state transition.
-- Record event processing atomically.

COMMIT;
```

The table name and columns above are a **target design**, not a claim about your existing Lesson 20 migration. Adapt the query to your implemented entitlement schema.

**Avoid a subtle locking bug:** If the entitlement row does not yet exist, `SELECT ... FOR UPDATE` locks nothing. For first-time concurrent grants, use a unique `(student_user_id, course_id)` constraint and a consistent row-creation/locking protocol, or lock a stable parent row.

### 9.3 Webhook verification boundary

A safe processing pipeline is:

```text
Raw HTTP request body
    ↓
Verify Stripe-Signature against configured secret
    ↓
Parse supported event
    ↓
Claim provider event ID idempotently
    ↓
Load and lock payment / entitlement state
    ↓
Verify amount, currency, and payment identity
    ↓
Apply allowed transition
    ↓
Commit database transaction
    ↓
Return successful acknowledgement
```

Do not trust the Angular success page as evidence of payment. Do not log the raw webhook body. Do not acknowledge a transient processing failure as successful unless the event has been durably queued for later processing.

---

## 10. Publishing and optimistic locking

Publishing must be more than setting `status = 'PUBLISHED'`.

The course must be owned by the actor, pass readiness checks, and have the expected version.

### 10.1 Final publishing rules

A course is publishable only if:

- Its status is `DRAFT`.
- The authenticated actor owns it or has authorized administrative access.
- Required metadata is valid.
- It has at least one section and at least one publishable lesson.
- Lesson ordering and required content are valid.
- The expected optimistic-lock version matches the current version.

### 10.2 Transactional service pattern

The following illustrates the final orchestration. Reuse your Lesson 19 service and entity names rather than creating duplicate publishing services.

<CodeBlock language="java" editable>
@Transactional
public CourseDetailsResponse publish(
        long courseId,
        long expectedVersion
) {
    CourseEntity course = courses.findById(courseId)
            .orElseThrow(
                    () -> new CourseNotFoundException(courseId)
            );

    ownershipPolicy.requireCanManage(
            currentActor.require(),
            course
    );

    if (course.getVersion() != expectedVersion) {
        throw new CourseConflictException(
                "Course was modified by another request."
        );
    }

    if (course.getStatus() != CourseStatus.DRAFT) {
        throw new CourseConflictException(
                "Only draft courses can be published."
        );
    }

    var readiness = readinessService.evaluate(courseId);

    if (!readiness.ready()) {
        throw new CourseConflictException(
                "Course does not meet publishing requirements."
        );
    }

    course.changeStatus(CourseStatus.PUBLISHED);

    courses.flush();

    return CourseDetailsResponse.from(course);
}</CodeBlock>

This assumes Lesson 19 added `@Version`, `getVersion()`, and a readiness service. The `readiness.ready()` method is an integration point; use your actual readiness DTO API.

**Concurrency caveat:** Checking the version in Java is not enough by itself. The entity must have a correctly mapped JPA `@Version` field so concurrent updates are rejected during flush. Curriculum edits must participate in the same version/readiness protocol; otherwise a lesson could be removed between validation and publishing.

### 10.3 Required tests

Use two independent transactions:

- Both read version `7`.
- First request publishes and commits.
- Second request attempts to publish with version `7`.
- Second request receives `409`.
- The database contains one valid published state.

Also verify that an instructor cannot publish another instructor's draft, even with the correct version.

---

## 11. Security hardening checklist

### Session and CSRF

Use the Lesson 14 session-based security configuration. Confirm that:

- `JSESSIONID` is `HttpOnly`.
- Production cookies are `Secure`.
- SameSite is explicitly configured.
- Login changes the session ID.
- Logout invalidates the session.
- Unsafe browser requests require CSRF.
- Stripe webhooks are exempt from browser CSRF **only** on their exact route and require signature verification.
- Authentication and authorization errors do not expose secrets.

### Content authorization

For every lesson endpoint, verify the lesson's actual course relationship before granting access. An attacker must not be able to combine a course they own with a lesson ID from another course.

### Ownership

Never accept `instructorUserId`, `studentUserId`, or `ownerId` from Angular as authoritative. Resolve the actor from the authenticated principal.

### Production surface

Keep these inaccessible from the public internet:

- PostgreSQL port 5432.
- Actuator metrics and sensitive operational endpoints.
- Swagger UI unless explicitly authorized.
- Internal administrative APIs.
- Debug logging and stack traces.

### One important Docker correction

Lesson 23's sample Nginx image uses the default root-managed Nginx startup. The Java backend runs as a non-root user, but the frontend image was not fully hardened.

For a production release, consider an unprivileged Nginx image with a nonprivileged listening port, a read-only root filesystem where feasible, and explicit writable temporary directories.

Also ensure the production deployment uses HTTPS. The local Compose example's HTTP profile is for development only.

---

## 12. Final automated test suite

We'll combine the testing layers from Lesson 21 with the operational and deployment checks from Lessons 22–23.

### 12.1 Testing matrix

| Layer | Examples | Tools |
|---|---|---|
| Pure domain | Money, lifecycle, progress percentage | JUnit 5, AssertJ |
| Service orchestration | Checkout rejection, ownership, access policy | Mockito |
| MVC contracts | Validation, JSON, status codes | MockMvc, `@WebMvcTest` |
| Security | Sessions, CSRF, role boundaries | Spring Security Test |
| Persistence | Constraints, migrations, locking | PostgreSQL Testcontainers |
| Payments | Signatures, retries, refunds, rollback | MockMvc, Testcontainers |
| Angular | HTTP contracts, route guards, error handling | HttpTestingController |
| Deployment | Container health, API routing | Docker Compose, smoke scripts |

### 12.2 Capstone access integration test

This test assumes the Testcontainers base class from Lesson 21 and the `LearningAccessPolicy` from Section 7.

**Create:** `src/test/java/com/learnhub/learning/LearningAccessIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.learning;

import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.transaction.support.TransactionTemplate;

import static org.assertj.core.api.Assertions.*;

@SpringBootTest
class LearningAccessIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired JdbcTemplate jdbc;
    @Autowired TransactionTemplate transactions;
    @Autowired LearningAccessPolicy policy;

    @Test
    void studentCannotReadAnotherStudentsEnrollment() {
        long courseId = 2L;

        long enrolledStudent = createStudent(
                "capstone-enrolled@example.test"
        );

        long otherStudent = createStudent(
                "capstone-other@example.test"
        );

        transactions.executeWithoutResult(status ->
                jdbc.update("""
                    INSERT INTO enrollments (
                        student_user_id,
                        course_id,
                        status
                    )
                    VALUES (?, ?, 'ACTIVE')
                    """,
                    enrolledStudent,
                    courseId
                )
        );

        assertThat(
                policy.evaluate(
                        enrolledStudent,
                        courseId
                ).allowed()
        ).isTrue();

        assertThat(
                policy.evaluate(
                        otherStudent,
                        courseId
                ).allowed()
        ).isFalse();
    }

    private long createStudent(String email) {
        return transactions.execute(status ->
                jdbc.queryForObject("""
                    INSERT INTO app_users (
                        email,
                        display_name,
                        role,
                        enabled
                    )
                    VALUES (?, 'Capstone Student',
                            'STUDENT', true)
                    RETURNING id
                    """,
                    Long.class,
                    email
                )
        );
    }
}
</CodeBlock>

**Fixture assumption:** This test uses course ID `2`, which earlier seed data treated as a published free course. If your database seeds differ, create a dedicated test course instead of relying on a magic ID. This test also requires the `PaidAccessLookup` implementation to exist in the application context.

A production-quality integration suite should create its own fixture records, use unique test identifiers, and clean them up without interfering with parallel tests.

### 12.3 Required HTTP security tests

At minimum, verify these behaviors through the real Spring Security filter chain:

<box border radius="lg" padding={3} gap={2}>
  <text weight="medium" size="sm">Security regression matrix</text>
  <table>
    <table-row><table-cell>Anonymous GET protected curriculum</table-cell><table-cell align="end">401</table-cell></table-row>
    <table-row><table-cell>Student POST without CSRF</table-cell><table-cell align="end">403</table-cell></table-row>
    <table-row><table-cell>Student modifies instructor draft</table-cell><table-cell align="end">403</table-cell></table-row>
    <table-row><table-cell>Instructor edits another instructor's draft</table-cell><table-cell align="end">403/404</table-cell></table-row>
    <table-row><table-cell>Unenrolled student reads lesson</table-cell><table-cell align="end">404</table-cell></table-row>
    <table-row><table-cell>Student reads another student's payment</table-cell><table-cell align="end">404</table-cell></table-row>
    <table-row><table-cell>Forged Stripe webhook</table-cell><table-cell align="end">400</table-cell></table-row>
    <table-row><table-cell>Duplicate valid webhook</table-cell><table-cell align="end">2xx, no duplicate effects</table-cell></table-row>
  </table>
</box>

For instructor ownership, select and document either `403` or a non-disclosing `404` consistently. The capstone contract should not permit both outcomes nondeterministically.

### 12.4 Failure-injection tests

A passing happy-path suite is not enough. Add tests that deliberately fail:

- Payment provider unavailable after a payment-attempt row is created.
- Database transaction rolls back during webhook fulfillment.
- Duplicate payment event arrives concurrently.
- Full refund arrives before a delayed payment event.
- Instructor publishes with a stale course version.
- PostgreSQL is unavailable during readiness checks.
- Browser sends a valid session cookie but no CSRF token.
- An API response field changes without updating Angular types.

Record both the observed HTTP response and the resulting database state.

---

## 13. Angular end-to-end integration

The final Angular application should not maintain separate, conflicting versions of backend state.

### Recommended ownership of state

| State | Authoritative source |
|---|---|
| Authentication | Spring Security session |
| Course catalog | Backend catalog API |
| Draft ownership | Backend instructor API |
| Wishlist | PostgreSQL through wishlist API |
| Enrollment | Backend enrollment API |
| Lesson completion | Backend progress API |
| Payment status | Verified backend payment state |
| Checkout price | Backend course/payment configuration |

Angular signals and services cache or display this state. They do not become a second authority.

### 13.1 Final route inventory

```typescript
// Illustrative app.routes.ts fragment.
// Merge with your existing standalone routes.

export const routes: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./features/catalog/backend-catalog-page.component')
        .then(m => m.BackendCatalogPageComponent),
  },
  {
    path: 'wishlist',
    canActivate: [authGuard],
    loadComponent: () =>
      import('./features/wishlist/wishlist-page.component')
        .then(m => m.WishlistPageComponent),
  },
  {
    path: 'my-learning',
    canActivate: [studentGuard],
    loadComponent: () =>
      import('./features/learning/my-learning-page.component')
        .then(m => m.MyLearningPageComponent),
  },
  {
    path: 'learn/:courseId',
    canActivate: [studentGuard],
    loadComponent: () =>
      import('./features/learning/course-player.component')
        .then(m => m.CoursePlayerComponent),
  },
  {
    path: 'instructor',
    canActivate: [instructorGuard],
    loadComponent: () =>
      import('./features/instructor/instructor-dashboard.component')
        .then(m => m.InstructorDashboardComponent),
  },
];
```

Use the actual component filenames from Lessons 16–20. These imports are integration targets, not replacements for your complete existing route tree.

### 13.2 Add a payment-status polling boundary

The Angular success page should not mark the student as enrolled simply because the payment provider redirected the browser.

A safe pattern is to poll the backend payment-status endpoint for a limited period, stop on a terminal state, and offer a manual retry if fulfillment is delayed.

**Create:** `learnhub/src/app/core/api/payment-status-poller.service.ts`

<CodeBlock language="typescript" editable>
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

import {
  Observable,
  timer,
  switchMap,
  takeWhile,
  take,
} from 'rxjs';

export type PaymentState =
  | 'PENDING'
  | 'PAID'
  | 'FAILED'
  | 'EXPIRED'
  | 'REFUNDED';

export interface PaymentStatusDto {
  paymentAttemptId: number;
  courseId: number;
  status: PaymentState;
  accessGranted: boolean;
}

@Injectable({ providedIn: 'root' })
export class PaymentStatusPollerService {
  private readonly http = inject(HttpClient);

  /**
   * Poll at most 12 times, every 2 seconds.
   * A terminal response is emitted before polling stops.
   */
  watch(attemptId: number): Observable<PaymentStatusDto> {
    return timer(0, 2000).pipe(
      take(12),
      switchMap(() =>
        this.http.get<PaymentStatusDto>(
          `/api/v1/me/payments/${attemptId}`
        )
      ),
      takeWhile(
        response =>
          response.status === 'PENDING',
        true
      )
    );
  }
}
</CodeBlock>

**Important refinements:** The example assumes sequential HTTP responses complete within the polling interval. In production, use `exhaustMap` or an equivalent non-overlapping request strategy to prevent concurrent polls when requests are slow. Also cancel polling when the component is destroyed.

The payment-status DTO must match your actual Lesson 20 backend response. A `PAID` status and `accessGranted` should be interpreted separately if entitlement processing is asynchronous.

### 13.3 Angular contract test

**Create:** `learnhub/src/app/core/api/payment-status-poller.service.spec.ts`

<CodeBlock language="typescript" editable>
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';

import { PaymentStatusPollerService }
  from './payment-status-poller.service';

describe('PaymentStatusPollerService', () => {
  let service: PaymentStatusPollerService;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });

    service = TestBed.inject(PaymentStatusPollerService);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());

  it('reads payment status from the backend', () => {
    let accessGranted = false;

    const subscription = service.watch(99).subscribe(
      response => {
        accessGranted = response.accessGranted;
      }
    );

    const request = http.expectOne(
      '/api/v1/me/payments/99'
    );

    expect(request.request.method).toBe('GET');

    request.flush({
      paymentAttemptId: 99,
      courseId: 42,
      status: 'PAID',
      accessGranted: true,
    });

    expect(accessGranted).toBeTrue();

    subscription.unsubscribe();
  });
});
</CodeBlock>

The first timer emission occurs immediately. The test checks the initial request and terminal response without waiting for subsequent intervals.

---

## 14. Capstone verification scripts

A release should be reproducible from a clean checkout.

### 14.1 Backend verification

**Create:** `learnhub-release/verify-backend.ps1`

<CodeBlock language="powershell" editable>
param(
    [string]$Backend = (
        Join-Path $HOME "angular-learning\learnhub-backend"
    )
)

$ErrorActionPreference = "Stop"

Set-Location $Backend

Write-Host "=== Java and Maven ==="
java -version
mvn -version

Write-Host "=== Docker ==="
docker info | Out-Null

if ($LASTEXITCODE -ne 0) {
    throw "Docker is unavailable."
}

Write-Host "=== Git whitespace check ==="
git diff --check

if ($LASTEXITCODE -ne 0) {
    throw "Git diff check failed."
}

Write-Host "=== Maven verification ==="
mvn -B -ntp clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Maven verification failed."
}

Write-Host "=== Docker image build ==="
docker build `
    -t learnhub-backend:capstone `
    .

if ($LASTEXITCODE -ne 0) {
    throw "Backend Docker build failed."
}

Write-Host "Backend verification passed."
</CodeBlock>

### 14.2 Frontend verification

**Create:** `learnhub-release/verify-frontend.ps1`

<CodeBlock language="powershell" editable>
param(
    [string]$Frontend = (
        Join-Path $HOME "angular-learning\learnhub"
    )
)

$ErrorActionPreference = "Stop"

Set-Location $Frontend

Write-Host "=== Install dependencies ==="
npm ci

if ($LASTEXITCODE -ne 0) {
    throw "npm ci failed."
}

Write-Host "=== Angular tests ==="
npm test -- --watch=false

if ($LASTEXITCODE -ne 0) {
    throw "Angular tests failed."
}

Write-Host "=== Production build ==="
npm run build -- --configuration production

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}

Write-Host "=== Docker image ==="
docker build `
    -t learnhub-frontend:capstone `
    .

if ($LASTEXITCODE -ne 0) {
    throw "Frontend Docker build failed."
}

Write-Host "Frontend verification passed."
</CodeBlock>

### 14.3 Deployment verification

**Create:** `learnhub-release/verify-deployment.ps1`

<CodeBlock language="powershell" editable>
param(
    [string]$Deploy = (
        Join-Path $HOME "angular-learning\learnhub-deploy"
    ),
    [string]$Origin = "http://localhost:8088"
)

$ErrorActionPreference = "Stop"

Set-Location $Deploy

docker compose config --quiet

if ($LASTEXITCODE -ne 0) {
    throw "Invalid Compose configuration."
}

docker compose up -d --build

if ($LASTEXITCODE -ne 0) {
    throw "Compose startup failed."
}

Write-Host "Waiting for backend readiness..."

$healthy = $false

for ($attempt = 1; $attempt -le 30; $attempt++) {
    $containerId = docker compose ps -q backend

    if ($containerId) {
        $health = docker inspect `
            --format '{{.State.Health.Status}}' `
            $containerId

        if ($health -eq "healthy") {
            $healthy = $true
            break
        }
    }

    Start-Sleep -Seconds 5
}

if (-not $healthy) {
    docker compose logs --tail=100 backend
    throw "Backend did not become healthy."
}

Write-Host "Checking public catalog..."

$catalog = Invoke-RestMethod `
    "$Origin/api/v1/courses?page=0&size=5"

if ($null -eq $catalog.items) {
    throw "Unexpected catalog contract."
}

Write-Host "Checking frontend..."

$response = Invoke-WebRequest `
    "$Origin/" `
    -UseBasicParsing

if ($response.StatusCode -ne 200) {
    throw "Frontend unavailable."
}

Write-Host "Deployment smoke checks passed."
</CodeBlock>

### Run the complete verification

```powershell
cd $HOME\angular-learning\learnhub-release

.\verify-backend.ps1
.\verify-frontend.ps1
.\verify-deployment.ps1
```

These scripts are starting points for your release gate. They do not replace a staging test of login, CSRF, instructor publishing, enrollment, checkout, and webhook processing.

---

## 15. Performance and operational review

### 15.1 Query performance

Inspect the catalog and dashboard queries with `EXPLAIN (ANALYZE, BUFFERS)` on realistic staging data.

Review:

- Search performance for common keywords.
- Offset pagination on large course catalogs.
- Instructor dashboard aggregation query plans.
- N+1 queries when loading course instructors or curriculum.
- Database connection pool saturation.
- Slow enrollment/progress writes.
- Lock contention during payment fulfillment.

Do not run aggressive load tests against a production payment provider.

### 15.2 Application performance

A reasonable first set of **illustrative targets**, to be validated against your infrastructure:

| Operation | Suggested starting target |
|---|---|
| Public catalog, p95 | Under 300 ms |
| Public course detail, p95 | Under 200 ms |
| Curriculum read, p95 | Under 300 ms |
| Instructor dashboard, p95 | Under 500 ms |
| Health/readiness probe | Under 100 ms |
| Checkout initiation | Depends on payment-provider latency |
| Webhook processing | Fast, idempotent, retry-safe |

These are engineering goals, not measurements of your current application.

### 15.3 Observability

From Lesson 22, confirm:

- Every request has an `X-Request-ID`.
- Unexpected errors return safe Problem Details.
- PostgreSQL participates in readiness.
- Payment metrics distinguish checkout creation from verified fulfillment.
- Webhook counters do not claim success before commit.
- Logs do not contain passwords, cookies, or payment secrets.
- Monitoring endpoints are private.

---

## 16. Portfolio README

Your README should show what the application does, how it's built, and how you verified it. Avoid claiming production readiness or test coverage that you haven't measured.

Below is a standalone draft you can copy into the repository that presents the full LearnHub project.

<WritingBlock id="58321" variant="document"># LearnHub — Full-Stack Course Marketplace

LearnHub is a full-stack course marketplace built with **Java 21, Spring Boot, Angular, PostgreSQL, and Docker**.

The project demonstrates backend architecture, REST API design, database persistence, secure session authentication, instructor ownership, student learning workflows, payment processing, automated testing, and containerized deployment.

## Features

### Students
- Browse and search published courses.
- Register, sign in, and manage a wishlist.
- Enroll in eligible free courses.
- Purchase paid courses through hosted checkout.
- Access enrolled course lessons and track completion.
- View learning progress and enrollment history.

### Instructors
- Create and manage owned draft courses.
- Organize sections and lessons.
- Validate publishing readiness.
- Publish and archive courses.
- Review instructor-scoped course analytics.

### Backend and platform
- Spring Security session authentication and CSRF protection.
- Role-based authorization and course ownership checks.
- PostgreSQL persistence with Flyway migrations.
- Transactional enrollment and payment workflows.
- Verified payment webhooks and idempotent processing.
- OpenAPI documentation, Actuator health checks, and Micrometer metrics.
- Dockerized Angular, Spring Boot, and PostgreSQL services.

## Technology stack

| Layer | Technology |
|---|---|
| Frontend | Angular, TypeScript, standalone components |
| Backend | Java 21, Spring Boot 3.5.x |
| Security | Spring Security, sessions, CSRF |
| Database | PostgreSQL, Spring Data JPA, Hibernate |
| Migrations | Flyway |
| Payments | Hosted checkout and signed webhooks |
| Testing | JUnit 5, Mockito, AssertJ, MockMvc, Testcontainers, Angular HTTP testing |
| Deployment | Docker, Docker Compose, Nginx, GitHub Actions |
| Observability | Spring Boot Actuator, Micrometer, structured logs |

## Architecture

The Angular frontend is served through Nginx, which forwards `/api/v1/` requests to the Spring Boot backend. PostgreSQL stores application data and financial state. Flyway manages database schema evolution.

The backend uses controllers, application services, repositories, and domain policies to separate HTTP concerns from business rules.

Authentication is session-based. Student and instructor authorization is enforced on the server.

## Local development

### Requirements
- Java 21
- Maven
- Node.js and npm
- Docker Desktop

### Backend

```powershell
cd learnhub-backend
mvn clean verify
mvn spring-boot:run "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

### Frontend

```powershell
cd learnhub
npm ci
ng.cmd serve --proxy-config proxy.conf.json
```

### Docker Compose

```powershell
cd learnhub-deploy
Copy-Item .env.example .env
# Configure local-only credentials.
docker compose up -d --build
```

Open the locally configured frontend URL.

## Testing

```powershell
# Backend
mvn clean verify

# Frontend
npm test -- --watch=false
npm run build
```

The backend test suite includes unit tests, security tests, API integration tests, and PostgreSQL Testcontainers coverage.

## Security design

- User identity comes from the authenticated server-side principal.
- Instructors can manage only authorized courses.
- Students cannot access protected lesson content without valid enrollment and entitlement.
- Payment amounts and currencies are determined by the backend.
- Webhook signatures are verified before financial state changes.
- Duplicate webhook events cannot grant duplicate access.
- Sensitive credentials are supplied through runtime configuration, not source code.

## Deployment and operations

The deployment configuration uses Docker Compose for local production-like testing and includes health checks, PostgreSQL persistence, and an Nginx reverse proxy.

Production deployment requires HTTPS, managed secrets, database backups, monitoring, and an approved rollback process.

## API documentation

In the development profile, OpenAPI documentation is available at:

`http://localhost:8080/swagger-ui/index.html`

## Engineering decisions

Key decisions include:

- PostgreSQL constraints as a final concurrency safeguard.
- Transactional payment fulfillment and idempotent webhook processing.
- Optimistic locking for course publishing.
- Separate public course reads and instructor-owned draft reads.
- Explicit entitlement checks for protected learning content.
- Safe error responses with request correlation IDs.

## Project status

**Portfolio capstone / release candidate.**

Add verified test results, screenshots, deployment details, and known limitations before presenting this as production-ready.

## Future improvements

- Email verification and password recovery.
- Managed media storage and signed video delivery.
- Automated contract generation for Angular.
- More extensive load testing.
- Distributed tracing and production alerting.
- Multi-instance deployment and database recovery automation.
</WritingBlock>

### Suggested portfolio screenshots

<grid columns={2} gap={3}>
  <grid-item>
    <AsyncImage query="modern online learning platform course catalog web dashboard UI screenshot design" aspectRatio="16:10" radius="lg"/>
    <text size="sm" weight="medium">Public catalog</text>
    <caption>Show search, filtering, course cards, and pagination.</caption>
  </grid-item>
  <grid-item>
    <AsyncImage query="online education instructor dashboard analytics courses management web UI" aspectRatio="16:10" radius="lg"/>
    <text size="sm" weight="medium">Instructor dashboard</text>
    <caption>Show owned courses, readiness, and analytics.</caption>
  </grid-item>
  <grid-item>
    <AsyncImage query="e learning course video player curriculum sidebar progress dashboard web interface" aspectRatio="16:10" radius="lg"/>
    <text size="sm" weight="medium">Course player</text>
    <caption>Show curriculum and lesson completion.</caption>
  </grid-item>
  <grid-item>
    <AsyncImage query="developer continuous integration GitHub Actions successful workflow tests dashboard" aspectRatio="16:10" radius="lg"/>
    <text size="sm" weight="medium">Engineering evidence</text>
    <caption>Show passing CI, test reports, and deployment health.</caption>
  </grid-item>
</grid>

<caption>These are visual references for screenshot composition, not screenshots of your actual LearnHub application. Replace them with genuine project screenshots.</caption>

---

## 17. Final project rubric

Use this rubric to evaluate the finished capstone. Each category receives 0–5 points.

{@body const criteria=[{name:"Architecture and API contracts",weight:15,description:"Clear layers, stable DTOs, consistent REST routes, OpenAPI alignment."},{name:"Authentication and authorization",weight:20,description:"Sessions, CSRF, role boundaries, ownership, student data isolation."},{name:"PostgreSQL and data integrity",weight:15,description:"Flyway, constraints, transactional consistency, concurrency."},{name:"Course and learning workflows",weight:15,description:"Drafts, publishing, curriculum, enrollment, progress."},{name:"Payments and refunds",weight:15,description:"Verified webhooks, idempotency, entitlement correctness."},{name:"Testing and CI",weight:10,description:"Automated regression coverage and passing pipeline."},{name:"Deployment and observability",weight:5,description:"Docker, health, logs, metrics, recovery."},{name:"Portfolio documentation",weight:5,description:"Clear README, architecture, evidence, limitations."}]}
{@body const [scores,setScores]=DIL.useState({})}
{@body const totalScore=criteria.reduce((sum,c,i)=>sum+(scores[i]??0)*c.weight/5,0)}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    <box gap={1}>
      **LearnHub capstone score**
      <text color="secondary" size="xs">Self-assessment · 0 = absent, 5 = verified and robust</text>
    </box>
    <title size="xl" tabularNums>{Math.round(totalScore)}/100</title>
  </row>
  <box background="surface-tertiary" height="9px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${totalScore}%`} radius="full"/>
  </box>
  {#each criteria as c,i}
    <box key={i} gap={2}>
      <row justify="between" align="center">
        **{c.name}**
        <text size="xs" color="secondary">{c.weight} points</text>
      </row>
      <text size="xs" color="secondary">{c.description}</text>
      <row align="center" gap={3}>
        <box flex="1">
          <slider min={0} max={5} step={1} value={scores[i]??0} onChange={v=>setScores(old=>({...old,[i]:v}))}/>
        </box>
        <text tabularNums weight="medium">{scores[i]??0}/5</text>
      </row>
    </box>
    {#if i<criteria.length-1}<divider color="subtle"/>{/if}
  {/each}
  <divider/>
  {#if totalScore>=90}
    **Assessment:** Strong release candidate, subject to independent security and deployment review.
  {:else if totalScore>=75}
    **Assessment:** Solid portfolio project; address remaining weaknesses before a real launch.
  {:else if totalScore>=50}
    **Assessment:** Functional foundation, but important integration or reliability work remains.
  {:else}
    **Assessment:** Continue integration and verification before declaring the capstone complete.
  {/if}
  <button color="secondary" variant="outline" block onClick={()=>setScores({})}>Reset assessment</button>
</box>

A high score does not override a critical security or payment defect. Treat such defects as release blockers regardless of the total.

---

## 18. Capstone exercises

<box gap={3}>
  <row gap={3} align="start">
    <icon name="shield-check" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise A — Authorization attack simulation**

      Create two students and two instructors. Attempt cross-user reads, cross-instructor edits, unauthorized lesson access, and CSRF-free mutations. Verify HTTP responses and unchanged database state.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="credit-card" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise B — Payment event disorder**

      Deliver signed test events in unusual orders: duplicate completion, refund before delayed completion, and two concurrent deliveries. Verify entitlement correctness and audit history.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="database" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise C — Database recovery**

      Back up a staging database, restore it into a fresh database, validate Flyway history, and run the capstone smoke tests against the restored instance.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="gauge" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise D — Performance investigation**

      Seed a realistic course catalog, measure the slowest query, inspect its execution plan, add or refine an index, and document the before/after results.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="git-merge" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise E — Release rehearsal**

      Deploy a staging release, run student and instructor acceptance journeys, introduce a reversible application defect, roll back the image, and confirm data remains intact.
    </box>
  </row>
</box>

## 19. Final knowledge quiz

{@body const questions=[{q:"What is the authoritative source of a student's identity in a protected backend request?",options:["The studentId in the JSON body","The authenticated server-side principal","The Angular route parameter"],correct:1,why:"Client-supplied IDs can be manipulated. Server-side authentication establishes the actor."},{q:"Can a successful checkout redirect grant paid course access?",options:["Yes, if the URL contains success=true","Only after verified backend payment fulfillment and valid entitlement","Yes, if Angular has the course price"],correct:1,why:"A redirect is not payment proof; the backend must verify and persist the transaction."},{q:"Why must a published course's curriculum validation coordinate with optimistic locking?",options:["To prevent publishing content that changes concurrently","To make Angular faster","To disable Hibernate validation"],correct:0,why:"A stale readiness decision can publish invalid content unless relevant concurrent edits are detected or serialized."},{q:"Which condition should cause a readiness probe to fail but not necessarily liveness?",options:["A required PostgreSQL dependency is unavailable","A user enters the wrong password","An Angular component is hidden"],correct:0,why:"The process may be alive while unable to serve database-dependent requests."},{q:"What is the safest rollback when an application release fails but the schema remains compatible?",options:["Delete all Flyway history","Restore the previous immutable application image","Drop the production database"],correct:1,why:"Reverting the image is safer than reversing a database migration when schema compatibility is preserved."},{q:"What is the most important test for duplicate payment webhooks?",options:["Both return HTTP 200 only","Only one durable fulfillment and no duplicate entitlement","The JSON parses twice"],correct:1,why:"Idempotency is about durable state and side effects, not merely response codes."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    **Final LearnHub assessment**
    <text color="secondary" size="xs">6 questions</text>
  </row>
  {#each questions as q,i}
    <box key={i} gap={2}>
      **{i+1}. {q.q}**
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setGraded(false);}}>
        {#each q.options as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if graded}
        <row align="start" gap={2}>
          <icon name={answers[i]===q.correct?"check-circle":"info"} color={answers[i]===q.correct?"success":"secondary"}/>
          <box flex="1" gap={1}>
            <text size="sm" weight="medium">{answers[i]===q.correct?"Correct":"Correct: "+q.options[q.correct]}</text>
            <text color="secondary" size="sm">{q.why}</text>
          </box>
        </row>
      {/if}
    </box>
    {#if i<questions.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if graded}
    <title size="lg">{questions.filter((q,i)=>answers[i]===q.correct).length} / 6 correct</title>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==questions.length} onClick={()=>setGraded(true)}>Grade assessment</button>
    <button variant="outline" color="secondary" onClick={()=>{setAnswers({});setGraded(false);}}>Reset</button>
  </row>
</box>

## 20. Graduation checklist

{@body const groups=[{name:"Architecture and database",items:["Canonical API routes documented","Angular contracts match OpenAPI","Flyway migrations validate from a fresh database","Critical foreign keys and unique constraints verified","Course ownership and optimistic locking tested","Curriculum readiness and ordering tested"]},{name:"Student and instructor workflows",items:["Registration, login, and logout work","Session restoration and CSRF tested","Public catalog excludes private drafts","Wishlist persists across refresh","Free enrollment is correct and idempotent","Protected lessons enforce enrollment","Progress is isolated and idempotent","Instructor dashboard is ownership-safe","Publishing and archiving follow lifecycle rules"]},{name:"Payments and security",items:["Checkout price is server-authoritative","Webhook signature verified over raw body","Amount and currency validated","Duplicate events cannot duplicate fulfillment","Full refund entitlement policy tested","Concurrent grant and revoke tested","Cross-user and cross-instructor attacks rejected","Production secrets and cookies reviewed"]},{name:"Testing and deployment",items:["JUnit and Mockito tests pass","MockMvc security tests pass","PostgreSQL Testcontainers tests pass","Angular HTTP tests pass","Maven clean verify passes","Angular production build passes","Docker Compose starts successfully","Readiness and smoke tests pass","Database backup restored successfully","CI pipeline passes","Rollback rehearsal completed"]},{name:"Portfolio and delivery",items:["README updated with verified features","Architecture diagram included","Genuine application screenshots captured","Known limitations documented","At least two capstone exercises completed","Final quiz completed","Rubric self-assessment completed","Git changes reviewed and committed"]}]}
{@body const [checked,setChecked]=DIL.useState({})}
{@body const total=groups.reduce((n,g)=>n+g.items.length,0)}
{@body const complete=Object.values(checked).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    <box gap={1}>
      **LearnHub graduation progress**
      <text color="secondary" size="xs">Mark items only after verifying them in your environment.</text>
    </box>
    <title size="lg" tabularNums>{complete}/{total}</title>
  </row>
  <box background="surface-tertiary" height="9px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${100*complete/total}%`} radius="full"/>
  </box>
  {#each groups as group,i}
    <box key={i} gap={2}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!checked[`${i}-${j}`]} onChange={v=>setChecked(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if complete===total}
    <row align="center" gap={2}>
      <icon name="graduation-cap" color="success" size="lg"/>      **All capstone verification items marked complete.**
    </row>
  {/if}
</box>

## 21. Final Git checkpoint

### Backend

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify
if ($LASTEXITCODE -ne 0) {
    throw "Do not release a failing backend."
}

git diff --check
git status --short
git diff --stat

git add src pom.xml README.md
git diff --cached --stat
git commit -m "Complete LearnHub backend capstone hardening"
```

### Angular

```powershell
cd $HOME\angular-learning\learnhub

npm test -- --watch=false
if ($LASTEXITCODE -ne 0) {
    throw "Angular tests failed."
}

npm run build
if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}

git diff --check
git status --short

git add src README.md
git diff --cached --stat
git commit -m "Complete LearnHub full-stack capstone"
```

### Release evidence

```powershell
cd $HOME\angular-learning\learnhub-release

Get-ChildItem -Recurse

# Review before committing. Do not include secrets,
# private user data, or database backups.
```

If you're maintaining the course journal in `KHADGA-R/Learning-Journals-khadga`, add a final `Java-SpringBoot-LearnHub-24-Lessons/README.md` that links to your implementation repositories, summarizes the architecture, and records the actual verification results.

## 22. Final release decision

A release should have a written decision, not just a green build.

<box border radius="xl" padding={3} gap={3}>
  {@body const [decision,setDecision]=DIL.useState("hold")}
  {@body const [notes,setNotes]=DIL.useState("")}
  **LearnHub release-candidate review**
  <radio-group direction="col" value={decision} onChange={setDecision}>
    <radio value="hold">Hold — critical gaps or verification failures remain</radio>
    <radio value="staging">Approve staging — core tests pass; production gates remain</radio>
    <radio value="production">Approve production candidate — all required gates independently verified</radio>
  </radio-group>
  <label>Evidence, blockers, or remaining risks</label>
  <textarea rows={3} value={notes} onChange={setNotes} placeholder="Example: PostgreSQL restore verified; webhook concurrency test pending; staging only."/>
  <button block onClick={()=>GenUI.copy(`LearnHub Capstone Release Decision\nDecision: ${decision}\nEvidence / Risks: ${notes || "Not recorded"}\nVerification: Must be supported by actual test and deployment evidence.`)}>
    <icon name="copy" inline/> Copy release decision
  </button>
</box>

I have not executed the code or tests against your local repositories. Some integration snippets depend on the exact entity, migration, and endpoint signatures you implemented in Lessons 18–20. Those must be reconciled before considering the release candidate complete.

---

## 23. What you've learned across 24 lessons

<box gap={2}>
  <row align="center" gap={3}>
    <box background="surface-secondary" radius="lg" padding={3}>
      <icon name="coffee" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Lessons 1–4: Java foundations**

      Java 21, records, collections, streams, OOP, interfaces, exceptions, testing, and SOLID design.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="center" gap={3}>
    <box background="surface-secondary" radius="lg" padding={3}>
      <icon name="server" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Lessons 5–12: Backend architecture**

      Spring Boot, REST, DTOs, validation, PostgreSQL, Flyway, JPA, CRUD, search, and pagination.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="center" gap={3}>
    <box background="surface-secondary" radius="lg" padding={3}>
      <icon name="shield-check" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Lessons 13–16: Security and full-stack integration**

      Spring Security, sessions, CSRF, roles, instructor ownership, Angular services, and route guards.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="center" gap={3}>
    <box background="surface-secondary" radius="lg" padding={3}>
      <icon name="graduation-cap" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Lessons 17–20: Marketplace workflows**

      Wishlist, enrollment, curriculum, progress, publishing, analytics, checkout, webhooks, and refunds.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="center" gap={3}>
    <box background="surface-secondary" radius="lg" padding={3}>
      <icon name="rocket" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Lessons 21–24: Production engineering**

      Comprehensive testing, observability, Docker, CI/CD, deployment, recovery, and final system validation.
    </box>
  </row>
</box>

<box align="center" padding={{top:4,bottom:3}} gap={2}>
  <icon name="graduation-cap" size="3xl" color="default"/>
  <title size="xl" textAlign="center">LearnHub — 24-Lesson Course Complete</title>
  <text color="secondary" textAlign="center">The curriculum is complete. The final engineering milestone is a verified, documented LearnHub release candidate.</text>
</box>

The most valuable outcome is not simply having written a Spring Boot application. It's being able to explain **why the system is secure, how its data remains consistent, how failures are handled, how correctness is tested, and how a release can be deployed and recovered safely.**