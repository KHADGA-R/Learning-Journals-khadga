# LearnHub Java & Spring Boot — Lesson 21
<text color="secondary" size="sm">JUnit 5 · Mockito · AssertJ · MockMvc · PostgreSQL Testcontainers · Security · Concurrency · Angular contracts</text>

**Level:** Experienced Java developer  
**Duration:** 10–14 hours, including exercises  
**Goal:** Turn LearnHub's existing functionality into a reliable, repeatable, security-focused automated test suite.

In Lessons 1–20, we built a course marketplace with instructor-owned courses, authenticated student sessions, curriculum, progress tracking, publishing, and paid enrollments.

Lesson 21 is different: we're not primarily adding features. We're building the evidence that those features behave correctly under normal use, malicious requests, retries, database failures, and concurrency.

## 1. The testing architecture

<box border radius="xl" padding={3} gap={2}>
  <box align="center" gap={1}>
    <box background="rgba(59,130,246,0.15)" border radius="md" width="38%" padding={3} align="center">
      **Browser E2E**
      <text size="2xs" color="secondary">Few, critical journeys</text>
    </box>
    <box background="rgba(59,130,246,0.11)" border radius="md" width="57%" padding={3} align="center">
      **HTTP + PostgreSQL integration**
      <text size="2xs" color="secondary">Security, transactions, contracts</text>
    </box>
    <box background="rgba(59,130,246,0.07)" border radius="md" width="78%" padding={3} align="center">
      **MVC slices + repository tests**
      <text size="2xs" color="secondary">Request validation and persistence</text>
    </box>
    <box background="surface-secondary" border radius="md" width="100%" padding={3} align="center">
      **Unit tests**
      <text size="2xs" color="secondary">Domain rules, service orchestration, edge cases</text>
    </box>
  </box>
</box>

We will test at the lowest level that can prove a behavior, but not lower.

For example, a unit test can prove that `Money.usdMinor()` converts dollars to cents correctly. It cannot prove that PostgreSQL's unique enrollment constraint prevents duplicate rows under concurrent requests. That requires a real database.

### What each layer proves

| Test layer | Dependencies | Best for |
|---|---|---|
| JUnit unit | No Spring context | Pure business rules |
| Mockito service unit | Mocked collaborators | Orchestration, failure paths |
| `@WebMvcTest` | MVC slice, mocked services | Validation, JSON, HTTP errors |
| `@DataJpaTest` | JPA + PostgreSQL | Queries, mappings, constraints |
| `@SpringBootTest` + MockMvc | Full Spring application | Security, transactions, endpoints |
| Angular HTTP tests | Mock HTTP backend | URL, method, request/response contract |
| Browser E2E | Real frontend and backend | Session cookies, CSRF, navigation |

**Important:** Green unit tests do not imply that security rules, Flyway migrations, or transactional boundaries are correct.

---

## 2. Baseline and test dependencies

### 2.1 Windows PowerShell setup

```powershell
$backend = Join-Path $HOME "angular-learning\learnhub-backend"
$frontend = Join-Path $HOME "angular-learning\learnhub"

Set-Location $backend

java -version
mvn -version
docker info
git status --short

Get-ChildItem src\main\resources\db\migration |
    Sort-Object Name

mvn clean verify
```

If the baseline fails, capture the failures before introducing new tests. In particular, reconcile the entity and repository signatures from earlier lessons instead of weakening assertions to make tests pass.

### 2.2 Maven test dependencies

**Modify:** `learnhub-backend/pom.xml`

Your Spring Boot 3.5.x parent manages most test dependency versions. Confirm these entries are present; don't add duplicates.

<CodeBlock language="xml" editable>
<dependencies>
    <!-- JUnit Jupiter, AssertJ, Mockito, Spring Test, MockMvc -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- SecurityMockMvcRequestPostProcessors and @WithMockUser -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- PostgreSQL Testcontainers -->
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
</dependencies>
</CodeBlock>

These belong inside your existing `<dependencies>` element, not inside a second one.

### 2.3 Directory structure

We'll organize tests by the component or behavior under test.

```text
learnhub-backend/
└── src/
    └── test/
        ├── java/com/learnhub/
        │   ├── PostgresIntegrationTest.java
        │   ├── payments/
        │   │   ├── MoneyTest.java
        │   │   ├── PaymentAttemptEntityTest.java
        │   │   ├── StripeWebhookVerifierTest.java
        │   │   ├── StripeWebhookProcessorIntegrationTest.java
        │   │   └── PaymentConcurrencyIntegrationTest.java
        │   ├── service/
        │   │   └── CoursePublishingServiceTest.java
        │   ├── web/
        │   │   ├── CheckoutControllerWebMvcTest.java
        │   │   ├── CourseSecurityIntegrationTest.java
        │   │   └── EnrollmentSecurityIntegrationTest.java
        │   └── contract/
        │       └── ApiContractIntegrationTest.java
        └── resources/
            └── application-test.yml
```

### 2.4 Test profile

**Create:** `src/test/resources/application-test.yml`

<CodeBlock language="yaml" editable>
spring:
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate

  flyway:
    enabled: true

logging:
  level:
    org.springframework.security: WARN

learnhub:
  payments:
    stripe:
      secret-key: sk_test_unused_in_tests
      webhook-secret: whsec_lesson21_test_secret
      api-base: https://api.stripe.com
    public-base-url: http://localhost:4200
</CodeBlock>

This file deliberately does not contain database credentials. Testcontainers supplies the database connection dynamically.

The test-mode secret strings are nonproduction placeholders. No test should call the live Stripe API.

---

## 3. Unit testing with JUnit 5 and AssertJ

Start with code that has no external dependencies.

### 3.1 Money conversion tests

**Create:** `src/test/java/com/learnhub/payments/MoneyTest.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.*;

class MoneyTest {

    @Nested
    @DisplayName("Valid USD amounts")
    class ValidAmounts {

        @ParameterizedTest
        @CsvSource({
            "0.01, 1",
            "1.00, 100",
            "19.99, 1999",
            "49.99, 4999",
            "1000.00, 100000"
        })
        void convertsExactly(
                String dollars,
                long expectedCents
        ) {
            long result = Money.usdMinor(
                    new BigDecimal(dollars)
            );

            assertThat(result).isEqualTo(expectedCents);
        }
    }

    @Nested
    @DisplayName("Invalid USD amounts")
    class InvalidAmounts {

        @Test
        void rejectsNull() {
            assertThatThrownBy(
                    () -> Money.usdMinor(null)
            ).isInstanceOf(IllegalArgumentException.class);
        }

        @Test
        void rejectsZero() {
            assertThatThrownBy(
                    () -> Money.usdMinor(BigDecimal.ZERO)
            ).isInstanceOf(IllegalArgumentException.class);
        }

        @Test
        void rejectsNegativeAmount() {
            assertThatThrownBy(
                    () -> Money.usdMinor(
                            new BigDecimal("-5.00")
                    )
            ).isInstanceOf(IllegalArgumentException.class);
        }

        @Test
        void rejectsFractionalCent() {
            assertThatThrownBy(
                    () -> Money.usdMinor(
                            new BigDecimal("1.005")
                    )
            ).isInstanceOf(ArithmeticException.class);
        }

        @Test
        void rejectsOverflow() {
            assertThatThrownBy(
                    () -> Money.usdMinor(
                            new BigDecimal(
                                    "999999999999999999999.00"
                            )
                    )
            ).isInstanceOf(ArithmeticException.class);
        }
    }
}
</CodeBlock>

### Why use `BigDecimal(String)`?

`new BigDecimal("19.99")` preserves the intended decimal value. `new BigDecimal(19.99)` first captures the binary floating-point approximation.

For money, tests should be exact rather than use a tolerance.

### Run only this test

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=MoneyTest" test
```

---

## 4. Domain state-machine tests

Payment state transitions are a good fit for unit tests because we can test the entity without starting Spring.

**Create:** `src/test/java/com/learnhub/payments/PaymentAttemptEntityTest.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class PaymentAttemptEntityTest {

    private PaymentAttemptEntity pending() {
        return new PaymentAttemptEntity(
                101L,
                202L,
                4999L,
                "USD"
        );
    }

    @Test
    void newAttemptStartsPending() {
        var payment = pending();

        assertThat(payment.getStatus())
                .isEqualTo(PaymentStatus.PENDING);

        assertThat(payment.getRefundedMinor())
                .isZero();
    }

    @Test
    void successfulPaymentBecomesPaid() {
        var payment = pending();

        payment.attachCheckout("cs_test_1");
        payment.markPaid("pi_test_1");

        assertThat(payment.getStatus())
                .isEqualTo(PaymentStatus.PAID);

        assertThat(payment.getPaymentIntentId())
                .isEqualTo("pi_test_1");
    }

    @Test
    void repeatedPaidTransitionIsIdempotent() {
        var payment = pending();

        payment.markPaid("pi_test_1");
        payment.markPaid("pi_test_1");

        assertThat(payment.getStatus())
                .isEqualTo(PaymentStatus.PAID);
    }

    @Test
    void expiredAttemptCannotBecomePaid() {
        var payment = pending();
        payment.markExpired();

        assertThatThrownBy(
                () -> payment.markPaid("pi_test_1")
        ).isInstanceOf(IllegalStateException.class);
    }

    @Test
    void partialRefundPreservesPaidStatus() {
        var payment = pending();
        payment.markPaid("pi_test_1");

        payment.applyRefund(1000);

        assertThat(payment.getStatus())
                .isEqualTo(PaymentStatus.PAID);

        assertThat(payment.getRefundedMinor())
                .isEqualTo(1000);
    }

    @Test
    void fullRefundChangesStatus() {
        var payment = pending();
        payment.markPaid("pi_test_1");

        payment.applyRefund(4999);

        assertThat(payment.getStatus())
                .isEqualTo(PaymentStatus.REFUNDED);
    }

    @Test
    void refundCannotExceedPaymentAmount() {
        var payment = pending();
        payment.markPaid("pi_test_1");

        assertThatThrownBy(
                () -> payment.applyRefund(5000)
        ).isInstanceOf(IllegalArgumentException.class);
    }
}
</CodeBlock>

### This test exposes a Lesson 20 defect

The earlier `markPaid()` method returns immediately if the status is already `PAID`, without checking whether the new payment intent ID matches the original.

That makes the entity method too permissive.

**Correct `PaymentAttemptEntity.markPaid()`**:

<CodeBlock language="java" editable>
public void markPaid(String intentId) {
    if (intentId == null || intentId.isBlank()) {
        throw new IllegalArgumentException(
                "Payment intent is required."
        );
    }

    if (status == PaymentStatus.PAID) {
        if (!intentId.equals(paymentIntentId)) {
            throw new IllegalStateException(
                    "Conflicting payment intent."
            );
        }
        return;
    }

    if (status != PaymentStatus.PENDING) {
        throw new IllegalStateException(
                "Payment cannot transition to PAID."
        );
    }

    paymentIntentId = intentId;
    status = PaymentStatus.PAID;
    touch();
}
</CodeBlock>

Add a regression test:

```java
@Test
void paidAttemptRejectsDifferentPaymentIntent() {
    var payment = pending();
    payment.markPaid("pi_original");

    assertThatThrownBy(
            () -> payment.markPaid("pi_different")
    ).isInstanceOf(IllegalStateException.class);
}
```

This is the value of good unit tests: they expose domain inconsistencies before those inconsistencies reach the database.

---

## 5. Mockito: testing service orchestration

Use Mockito when the behavior under test is *how a service coordinates dependencies*, rather than whether SQL, HTTP, or Spring Security works.

For example, `CheckoutService` must reject an unpublished course **before** creating a payment attempt or contacting Stripe.

### 5.1 Checkout rejection test

**Create:** `src/test/java/com/learnhub/payments/CheckoutServiceTest.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.security.CurrentActor;
import com.learnhub.security.LearnHubPrincipal;
import com.learnhub.security.SecurityRole;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CheckoutServiceTest {

    @Mock CurrentActor actor;
    @Mock JpaCourseRepository courses;
    @Mock PaymentAttemptStore attempts;
    @Mock CheckoutProvider provider;
    @Mock PaidEnrollmentPolicy enrollmentPolicy;

    @InjectMocks CheckoutService service;

    @Test
    void unpublishedCourseNeverStartsCheckout() {
        var principal = new LearnHubPrincipal(
                42L,
                "student@example.test",
                "unused",
                SecurityRole.STUDENT,
                true
        );

        CourseEntity draft = mock(CourseEntity.class);

        when(actor.require()).thenReturn(principal);
        when(courses.findById(10L))
                .thenReturn(Optional.of(draft));
        when(draft.getStatus())
                .thenReturn(CourseStatus.DRAFT);

        assertThatThrownBy(
                () -> service.checkout(10L)
        ).isInstanceOf(IllegalArgumentException.class);

        verifyNoInteractions(
                attempts,
                provider,
                enrollmentPolicy
        );
    }
}
</CodeBlock>

**Why this matters:** The test proves an important negative property: rejected requests do not create payment records or contact an external provider.

`@PreAuthorize` is not activated by a plain Mockito-created service. This test verifies the service's explicit checks, **not** method security. We'll test method security through Spring later.

### 5.2 Mockito guidelines

Prefer `@ExtendWith(MockitoExtension.class)` and constructor injection. Avoid loading Spring for a simple mock-based test. Use `verifyNoInteractions()` to prove forbidden side effects, and avoid mocking domain values such as `BigDecimal` or enums.

Also avoid `lenient()` unless you have a specific reason: unnecessary stubs often reveal a poorly focused test.

---

## 6. MockMvc controller slice with `@WebMvcTest`

A controller slice is useful for checking:

- Request mapping and path variables.
- JSON response shapes.
- Bean Validation.
- HTTP status codes.
- Whether the controller calls the correct application service.

It does **not** prove that JPA transactions or PostgreSQL constraints work.

### 6.1 Controller slice test

**Create:** `src/test/java/com/learnhub/web/CheckoutControllerWebMvcTest.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.payments.CheckoutResponse;
import com.learnhub.payments.CheckoutService;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.Mockito.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(CheckoutController.class)
class CheckoutControllerWebMvcTest {

    @Autowired MockMvc mvc;

    @MockitoBean CheckoutService checkoutService;

    @Test
    void createsCheckoutAndReturnsTypedResponse()
            throws Exception {

        when(checkoutService.checkout(42L))
                .thenReturn(new CheckoutResponse(
                        99L,
                        "https://checkout.stripe.com/test"
                ));

        mvc.perform(post(
                    "/api/v1/me/checkout/courses/{id}",
                    42L
                )
                .with(user("student")
                        .roles("STUDENT"))
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath(
                    "$.paymentAttemptId"
            ).value(99))
            .andExpect(jsonPath(
                    "$.checkoutUrl"
            ).value(
                    "https://checkout.stripe.com/test"
            ));

        verify(checkoutService).checkout(42L);
    }
}
</CodeBlock>

### Configuration caveat

The Lesson 14 `SecurityConfig` has dependencies such as `DatabaseUserDetailsService`, the CSRF handler, and the session security-context repository.

A `@WebMvcTest` can fail to initialize if it imports the full security configuration without those dependencies.

For this slice, use a minimal test security filter chain that explicitly permits the tested route to a student and keeps CSRF enabled. Do **not** disable filters globally just to make the test pass.

**Create:** `src/test/java/com/learnhub/web/CheckoutSliceSecurity.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@TestConfiguration
public class CheckoutSliceSecurity {

    @Bean
    SecurityFilterChain testSecurity(HttpSecurity http)
            throws Exception {

        return http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(
                                "/api/v1/me/checkout/courses/*"
                        ).hasRole("STUDENT")
                        .anyRequest().denyAll()
                )
                .build();
    }
}
</CodeBlock>

Import it into the test:

```java
@Import(CheckoutSliceSecurity.class)
```

with:

```java
import org.springframework.context.annotation.Import;
```

If your production `SecurityConfig` is imported automatically into the slice, exclude that configuration for this specific test. Do not create two competing `SecurityFilterChain` beans and assume which one will win.

The slice verifies its own simplified HTTP security contract. **Full security integration tests must use the actual production filter chain.**

---

## 7. PostgreSQL Testcontainers: one reproducible database

For repository and integration tests, PostgreSQL is part of the behavior being tested.

H2 cannot reliably substitute for:

- PostgreSQL `ON CONFLICT`.
- Functional unique indexes.
- `TIMESTAMPTZ`.
- Row locking and concurrent transactions.
- PostgreSQL-specific search operators.
- Flyway migrations using PostgreSQL extensions.

### 7.1 Shared Testcontainers base

If your existing `PostgresIntegrationTest` from Lesson 7 already works, keep it. Otherwise, use this version.

**Create or update:** `src/test/java/com/learnhub/PostgresIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub;

import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;

@ActiveProfiles("test")
public abstract class PostgresIntegrationTest {

    private static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17")
                    .withDatabaseName("learnhub_test")
                    .withUsername("learnhub_test")
                    .withPassword("learnhub_test");

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
</CodeBlock>

The static container starts once per loaded test class hierarchy in this JVM. Docker must be available locally or in CI.

### 7.2 Flyway is part of the test

`spring.flyway.enabled=true` and `spring.jpa.hibernate.ddl-auto=validate` provide an important guarantee:

**The application context should fail if the migrations cannot create a schema compatible with your JPA mappings.**

Do not replace Flyway with Hibernate `create-drop` in integration tests. That would test a different schema from production.

### 7.3 Verify critical constraints

**Create:** `src/test/java/com/learnhub/persistence/DatabaseConstraintIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.persistence;

import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.*;

@SpringBootTest
@Transactional
class DatabaseConstraintIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired JdbcTemplate jdbc;

    @Test
    void emailUniquenessIsCaseInsensitive() {
        jdbc.update("""
            INSERT INTO app_users (
                email, display_name, role, enabled
            )
            VALUES (
                'Lesson21.Unique@example.test',
                'First User',
                'STUDENT',
                true
            )
            """);

        assertThatThrownBy(() ->
                jdbc.update("""
                    INSERT INTO app_users (
                        email, display_name, role, enabled
                    )
                    VALUES (
                        'lesson21.unique@example.test',
                        'Second User',
                        'STUDENT',
                        true
                    )
                    """)
        ).isInstanceOf(
                org.springframework.dao
                        .DataIntegrityViolationException.class
        );
    }

    @Test
    void duplicateWebhookEventIdIsIgnored() {
        String sql = """
            INSERT INTO payment_webhook_events (
                provider, provider_event_id, event_type
            )
            VALUES ('STRIPE', ?, 'ping')
            ON CONFLICT (
                provider, provider_event_id
            ) DO NOTHING
            """;

        int first = jdbc.update(sql, "evt_constraint_test");
        int second = jdbc.update(sql, "evt_constraint_test");

        assertThat(first).isEqualTo(1);
        assertThat(second).isZero();
    }
}
</CodeBlock>

**PostgreSQL transaction caveat:** After a unique-constraint violation, the current PostgreSQL transaction is marked aborted. In this example, the test method ends immediately after the assertion, allowing Spring to roll it back. Do not attempt more SQL in the same transaction after the violation.

For complex constraint tests, use `TransactionTemplate` with separate transactions or JDBC savepoints.

---

## 8. Spring Security: test the real boundaries

Security tests should distinguish these outcomes:

| Situation | Expected |
|---|---|
| Anonymous request to protected endpoint | `401` |
| Authenticated student tries instructor operation | `403` |
| Authenticated instructor accesses another instructor's private course | `404` or `403`, according to the endpoint policy |
| Authenticated student POST without CSRF | `403` |
| Authorized owner POST with CSRF | Business response |
| Student A reads Student B's payment | `404` |
| Forged webhook signature | `400` |
| Valid webhook, no browser session | Accepted for processing |

Lesson 15 used a deliberate distinction: instructor-scoped reads should generally return `404` for non-owned resources to avoid exposing their existence. For writes, a consistent `403` or non-disclosing `404` policy is acceptable, but it must be tested and documented.

### 8.1 Full-stack security test

**Create:** `src/test/java/com/learnhub/web/PaymentSecurityIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class PaymentSecurityIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired MockMvc mvc;

    @Test
    void anonymousStudentCheckoutIsUnauthorized()
            throws Exception {

        mvc.perform(post(
                    "/api/v1/me/checkout/courses/1"
                )
                .with(csrf()))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void studentCheckoutWithoutCsrfIsForbidden()
            throws Exception {

        mvc.perform(post(
                    "/api/v1/me/checkout/courses/1"
                )
                .with(user("student")
                        .roles("STUDENT")))
            .andExpect(status().isForbidden());
    }

    @Test
    void instructorCannotStartStudentCheckout()
            throws Exception {

        mvc.perform(post(
                    "/api/v1/me/checkout/courses/1"
                )
                .with(user("instructor")
                        .roles("INSTRUCTOR"))
                .with(csrf()))
            .andExpect(status().isForbidden());
    }

    @Test
    void anonymousPaymentStatusIsUnauthorized()
            throws Exception {

        mvc.perform(get(
                    "/api/v1/me/payments/1"
                ))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void forgedWebhookIsRejected()
            throws Exception {

        mvc.perform(post(
                    "/api/v1/webhooks/stripe"
                )
                .header(
                        "Stripe-Signature",
                        "t=1,v1=0000"
                )
                .contentType("application/json")
                .content("""
                    {"id":"evt_fake","type":"ping"}
                    """))
            .andExpect(status().isBadRequest());
    }
}
</CodeBlock>

These tests exercise the production `SecurityFilterChain`. They deliberately stop before valid checkout creation, so they do not need a real Stripe API.

### 8.2 `@WithMockUser` versus a real application principal

`@WithMockUser` creates a Spring Security test identity. It does **not** create a corresponding `app_users` database row.

That is enough for HTTP role checks, but not enough for services that call:

```java
CurrentActor.require()
```

and expect a `LearnHubPrincipal` containing a real user ID.

For successful ownership and checkout tests, use the Lesson 14 principal fixture:

```java
.with(user(new LearnHubPrincipal(
        studentId,
        email,
        encodedPassword,
        SecurityRole.STUDENT,
        true
)))
.with(csrf())
```

Create the matching database user in the Testcontainers database first.

### 8.3 Test session restoration

The most important authentication test uses a real login, not a mock identity.

Test this sequence:

1. GET `/api/v1/security/csrf` and retain cookies.
2. POST `/api/v1/auth/login` with the CSRF cookie and header.
3. Capture the authenticated `MockHttpSession`.
4. GET `/api/v1/auth/me` using that session.
5. Assert the returned ID and role match the database user.
6. POST `/api/v1/auth/logout` with a valid CSRF token.
7. Repeat GET `/api/v1/auth/me` using the invalidated session and verify access is denied.

The session test should use a test user with a BCrypt hash stored in PostgreSQL.

**Do not use `@WithMockUser` for this scenario.** That would bypass the authentication flow you are trying to prove.

---

## 9. Course ownership and enrollment tests

These are high-value regression tests because the application has several entry points for modifying courses and granting access.

### 9.1 Ownership matrix

| Actor | Own draft | Another instructor's draft | Published public course |
|---|---|---|---|
| Anonymous | Cannot edit | Cannot edit | Can read |
| Student | Cannot edit | Cannot edit | Can read |
| Instructor A | Can edit A's draft | Cannot edit B's draft | Can read |
| Instructor B | Cannot edit A's draft | Can edit B's draft | Can read |
| Admin | Policy-defined maintenance access | Policy-defined maintenance access | Can read |

Use parameterized tests for role-only behavior. Use separate integration tests for ownership, because ownership depends on the authenticated principal ID and the `courses.instructor_user_id` foreign key.

For each forbidden mutation, assert **both** the HTTP result and the unchanged database row.

### 9.2 Enrollment invariants

Test these cases against PostgreSQL:
- Student can enroll in a published free course.
- Student cannot directly enroll in a paid course without a verified payment.
- Instructor cannot use a student-only enrollment endpoint.
- Duplicate enrollment attempts do not create duplicate rows.
- A student cannot read another student's enrollment.
- An unenrolled student cannot read protected lesson content.
- Completing a lesson twice does not create duplicate progress.
- A fully refunded paid enrollment follows the selected entitlement policy.

### A crucial regression test

A student sends a request that contains another student's ID:

```json
{
  "studentId": 999,
  "courseId": 42
}
```

The backend must **ignore or reject the client-supplied student ID**. It must use the authenticated principal ID.

In Lesson 17, student-scoped endpoints were designed to derive identity from the server. Preserve that rule across wishlist, enrollment, progress, and payments.

---

## 10. Payment webhook tests: signature, amount, and retries

The Lesson 20 signature verifier already has unit tests for correct signatures, modified payloads, and expired timestamps.

Now add the cases that distinguish a validly signed event from a valid business transaction.

### 10.1 Test matrix

| Event | Signature | Amount | Expected |
|---|---|---|---|
| Successful payment | Valid | Matches | `PAID`, enrollment granted |
| Successful payment | Invalid | Matches | Rejected before parsing |
| Successful payment | Valid | Wrong | Transaction rolls back |
| Completed but unpaid | Valid | Matches | No enrollment |
| Duplicate event ID | Valid | Matches | No duplicate fulfillment |
| Same attempt, different intent | Valid | Matches | Conflict, no replacement |
| Expired checkout | Valid | Matches | `EXPIRED`, no access |
| Full refund | Valid | Matches | Refund policy applied |
| Partial refund | Valid | Matches | Payment remains `PAID` |
| Unknown event type | Valid | N/A | Safely acknowledged |

### 10.2 A regression test for amount mismatch

Add this method to the Lesson 20 `PaymentFulfillmentIntegrationTest`, using its existing user, course, and attempt fixtures.

<CodeBlock language="java" editable>
@Test
void mismatchedAmountDoesNotGrantEnrollment() {
    long studentId = user();
    long courseId = course();

    long attemptId = jdbc.queryForObject("""
        INSERT INTO payment_attempts (
            student_user_id,
            course_id,
            amount_minor,
            currency
        )
        VALUES (?, ?, 4999, 'USD')
        RETURNING id
        """,
        Long.class,
        studentId,
        courseId
    );

    String payload = """
        {
          "id": "evt_wrong_amount_21",
          "type": "checkout.session.completed",
          "data": {
            "object": {
              "id": "cs_test_wrong_amount",
              "client_reference_id": "%d",
              "metadata": {"attempt_id": "%d"},
              "payment_status": "paid",
              "payment_intent": "pi_test_wrong_amount",
              "amount_total": 1,
              "currency": "usd"
            }
          }
        }
        """.formatted(attemptId, attemptId);

    assertThatThrownBy(() ->
            processor.process(
                    payload.getBytes(
                            java.nio.charset.StandardCharsets.UTF_8
                    )
            )
    ).isInstanceOf(IllegalStateException.class);
}
</CodeBlock>

**Important transaction correction:** Do not annotate this particular test with `@Transactional` if you intend to query the database after the expected failure.

Instead, create the fixtures in committed transactions, call the processor in its own transaction, and then query the database in a fresh transaction.

A failed PostgreSQL transaction can remain aborted until rollback, and an outer test transaction can conceal whether the production service correctly commits or rolls back.

### 10.3 Strong rollback assertion

After the failed webhook, verify:

```sql
SELECT status
FROM payment_attempts
WHERE id = ?;
```

Expected: `PENDING`.

```sql
SELECT COUNT(*)
FROM enrollments
WHERE student_user_id = ?
  AND course_id = ?;
```

Expected: `0`.

```sql
SELECT COUNT(*)
FROM payment_webhook_events
WHERE provider_event_id = 'evt_wrong_amount_21';
```

Expected: `0`, because the inbox claim and fulfillment transaction rolled back together.

### 10.4 Test valid signatures through HTTP

A direct call to `StripeWebhookProcessor.process()` does not verify the HMAC signature; it assumes the payload has already passed verification.

At least one MockMvc test must:

1. Construct a payload.
2. Compute its HMAC using the test webhook secret.
3. POST the raw payload and `Stripe-Signature` header.
4. Assert the HTTP response.
5. Query the database to verify the effect.

This proves the entire security boundary, not just the internal processor.

---

## 11. Concurrency testing: prove database invariants

A concurrency test should exercise **independent transactions**. Two calls inside one `@Transactional` test method do not establish that PostgreSQL locking works.

For Lesson 20, the key invariant is:

\[
\text{Successful payment attempts} \geq 1
\quad\Longrightarrow\quad
\text{Enrollment rows per student/course} \leq 1
\]

The database unique constraint is the ultimate protection.

### 11.1 Two concurrent enrollment grants

**Create:** `src/test/java/com/learnhub/payments/PaymentConcurrencyIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.transaction.support.TransactionTemplate;

import java.util.concurrent.*;

import static org.assertj.core.api.Assertions.*;

@SpringBootTest
class PaymentConcurrencyIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired JdbcTemplate jdbc;
    @Autowired PaidEnrollmentWriter writer;
    @Autowired TransactionTemplate transactions;

    @Test
    void concurrentGrantsProduceOneEnrollment()
            throws Exception {

        long studentId = transactions.execute(status ->
                jdbc.queryForObject("""
                    INSERT INTO app_users (
                        email, display_name, role, enabled
                    )
                    VALUES (
                        'concurrent-21@example.test',
                        'Concurrent Student',
                        'STUDENT',
                        true
                    )
                    RETURNING id
                    """,
                    Long.class
                )
        );

        long courseId = transactions.execute(status ->
                jdbc.queryForObject("""
                    INSERT INTO courses (
                        title, description, instructor,
                        category, price, is_free,
                        level, rating, status, published_at
                    )
                    VALUES (
                        'Concurrency Course',
                        'Concurrency test course',
                        'Test Instructor',
                        'Testing',
                        49.99,
                        false,
                        'BEGINNER',
                        0.0,
                        'PUBLISHED',
                        CURRENT_TIMESTAMP
                    )
                    RETURNING id
                    """,
                    Long.class
                )
        );

        ExecutorService pool =
                Executors.newFixedThreadPool(2);

        CountDownLatch ready = new CountDownLatch(2);
        CountDownLatch start = new CountDownLatch(1);

        Callable<Void> grant = () -> {
            ready.countDown();

            if (!start.await(10, TimeUnit.SECONDS)) {
                throw new TimeoutException(
                        "Start barrier timed out."
                );
            }

            transactions.executeWithoutResult(status ->
                    writer.grant(studentId, courseId)
            );

            return null;
        };

        try {
            Future<Void> first = pool.submit(grant);
            Future<Void> second = pool.submit(grant);

            assertThat(
                    ready.await(10, TimeUnit.SECONDS)
            ).isTrue();

            start.countDown();

            first.get(15, TimeUnit.SECONDS);
            second.get(15, TimeUnit.SECONDS);

            Integer count = jdbc.queryForObject("""
                SELECT COUNT(*)
                FROM enrollments
                WHERE student_user_id = ?
                  AND course_id = ?
                """,
                Integer.class,
                studentId,
                courseId
            );

            assertThat(count).isEqualTo(1);

        } finally {
            start.countDown();
            pool.shutdownNow();
        }
    }
}
</CodeBlock>

This test proves that concurrent `grant()` calls cannot create duplicate enrollment rows. It does **not** prove that a concurrent refund and payment fulfillment preserve the correct entitlement. That requires a separate test for the shared entitlement-locking policy discussed in Lesson 20.

### Why use a start barrier?

Without the barrier, the first task may finish before the second begins. The test could pass without exercising meaningful concurrency.

Even with a barrier, thread scheduling is nondeterministic. For deeper locking tests, coordinate transactions using explicit database locks or test-specific synchronization points.

### Test isolation

This example uses a unique fixture email. When running repeatedly against a reused container, generate deterministic per-test fixture IDs or clean the test data in a separate transaction.

Avoid relying on a global database cleanup that can interfere with tests running in parallel.

---

## 12. Transaction rollback tests

A useful rollback test injects a failure **after** an earlier database write, then verifies that the earlier write did not commit.

For the payment webhook processor, the relevant sequence is:

```text
Claim event ID
    ↓
Lock payment attempt
    ↓
Validate payment
    ↓
Mark payment PAID
    ↓
Grant enrollment
    ↓
Mark event processed
    ↓
COMMIT
```

If the enrollment grant throws, the entire transaction must roll back.

### Recommended test

Use a Spring test replacement for `PaidEnrollmentWriter` that throws an exception after the payment has been marked paid.

Run the processor outside a test-managed transaction, then query PostgreSQL and assert:

- Payment status remains `PENDING`.
- No enrollment exists.
- The webhook inbox event row does not exist.
- Retrying the same event with the real writer succeeds.

This is stronger than merely asserting that an exception was thrown.

**Spring testing detail:** `@MockitoBean` can replace the writer in a dedicated full-context test. Avoid applying it to a shared base class because that would affect unrelated integration tests.

---

## 13. Angular `HttpTestingController` contract tests

Angular tests should verify the actual HTTP request made by a typed service.

### 13.1 Checkout API test

**Create:** `learnhub/src/app/core/api/checkout-api.service.spec.ts`

<CodeBlock language="typescript" editable>
import { TestBed } from '@angular/core/testing';

import {
  provideHttpClient,
} from '@angular/common/http';

import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';

import { CheckoutApiService }
  from './checkout-api.service';

describe('CheckoutApiService', () => {
  let service: CheckoutApiService;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });

    service = TestBed.inject(CheckoutApiService);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    http.verify();
  });

  it('creates checkout using only the course ID', () => {
    let checkoutUrl = '';

    service.create(42).subscribe(response => {
      checkoutUrl = response.checkoutUrl;
    });

    const request = http.expectOne(
      '/api/v1/me/checkout/courses/42'
    );

    expect(request.request.method).toBe('POST');
    expect(request.request.body).toEqual({});

    // No browser-controlled price, currency, or student ID.
    expect(request.request.body.price).toBeUndefined();
    expect(request.request.body.studentId).toBeUndefined();

    request.flush({
      paymentAttemptId: 99,
      checkoutUrl:
        'https://checkout.stripe.com/test-session',
    });

    expect(checkoutUrl).toContain(
      'checkout.stripe.com'
    );
  });

  it('surfaces a provider failure as an HTTP error', () => {
    let status = 0;

    service.create(42).subscribe({
      next: () => fail('Expected an error'),
      error: error => status = error.status,
    });

    const request = http.expectOne(
      '/api/v1/me/checkout/courses/42'
    );

    request.flush(
      {
        title: 'Checkout unavailable',
        status: 503,
      },
      {
        status: 503,
        statusText: 'Service Unavailable',
      }
    );

    expect(status).toBe(503);
  });
});
</CodeBlock>

### 13.2 CSRF interceptor tests

A common mistake is to test the checkout API with `provideHttpClient()` and assume that proves CSRF works.

It doesn't. The test above checks the HTTP contract only.

For CSRF, configure the actual Lesson 16 interceptor and test that an unsafe same-origin request includes the expected header when a CSRF token is available.

Angular's built-in XSRF behavior also depends on browser cookie access, so use a browser-level E2E test for the complete cookie flow. Unit tests cannot fully reproduce the browser's cookie and SameSite rules.

### 13.3 Auth restoration test

Add a test for the Lesson 14–16 `AuthService`:

- A fresh application starts with unknown authentication state.
- GET `/api/v1/auth/me` returns a student user.
- The auth signal updates to that user.
- A `401` response clears the authenticated state.
- Instructor route guards reject students.
- A user with `INSTRUCTOR` role can enter instructor routes.

Do not trust a role copied from `localStorage` as authorization evidence. The backend remains authoritative.

---

## 14. Cross-stack API contract testing

TypeScript interfaces and Java records can drift apart.

For example, Angular may expect:

```typescript
interface PaymentStatusResponse {
  paymentAttemptId: number;
  courseId: number;
  status: 'PENDING' | 'PAID' | 'FAILED' | 'EXPIRED' | 'REFUNDED';
  accessGranted: boolean;
}
```

while the backend accidentally changes `accessGranted` to `enrolled`.

Both projects may compile independently, but the browser breaks at runtime.

### Recommended contract workflow

1. Define a versioned OpenAPI contract for `/api/v1`.
2. Generate or validate TypeScript types against that contract.
3. Run backend MockMvc tests against documented response examples.
4. Run Angular HTTP tests using those same examples.
5. Fail CI when a breaking API change is introduced without updating consumers.

We'll implement OpenAPI documentation formally in Lesson 22.

### Contract inventory

| API | Method | Key assertion |
|---|---|---|
| `/api/v1/courses` | GET | Public catalog excludes drafts |
| `/api/v1/auth/me` | GET | Authenticated identity and role |
| `/api/v1/instructor/dashboard` | GET | Instructor-owned aggregates only |
| `/api/v1/instructor/courses/{id}/readiness` | GET | Readiness issues and version |
| `/api/v1/me/wishlist` | GET | Current student's wishlist only |
| `/api/v1/me/enrollments` | GET | Current student's enrollments only |
| `/api/v1/me/checkout/courses/{id}` | POST | Server-authoritative checkout |
| `/api/v1/me/payments/{id}` | GET | Student-owned payment status |
| `/api/v1/webhooks/stripe` | POST | Signed events only |

Use the actual route names from Lessons 15–20 where they differ. Avoid maintaining two competing API contracts.

---

## 15. Test execution strategy

A reliable suite needs different commands for fast feedback and full verification.

### Fast unit tests

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=MoneyTest,PaymentAttemptEntityTest,StripeWebhookVerifierTest,CheckoutServiceTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Unit tests failed."
}
```

### PostgreSQL integration tests

```powershell
cd $HOME\angular-learning\learnhub-backend

docker info

mvn "-Dtest=DatabaseConstraintIntegrationTest,PaymentFulfillmentIntegrationTest,PaymentConcurrencyIntegrationTest,PaymentSecurityIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Integration tests failed."
}
```

If you renamed a class, update the test selection. Do not interpret “no tests matching pattern” as a successful verification.

### Full backend verification

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend verification failed."
}
```

### Angular tests and build

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd test --watch=false

if ($LASTEXITCODE -ne 0) {
    throw "Angular tests failed."
}

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}
```

### Important Maven lifecycle distinction

Maven Surefire normally executes unit tests in the `test` phase. Maven Failsafe conventionally executes integration tests in the `integration-test` and `verify` phases.

Our existing `*IntegrationTest` classes may currently run through Surefire because of their names or explicit configuration. Check your actual Maven configuration instead of assuming `mvn verify` discovers every integration test.

For a larger project, introduce Failsafe and use a consistent naming convention such as `*IT.java`.

---

## 16. Engineering review: fixes uncovered by testing

A good testing lesson should reveal defects, not merely confirm happy paths.

| Issue from earlier lessons | Why it matters | Required action |
|---|---|---|
| Payment intent can be silently replaced in entity method | Financial identity integrity | Reject conflicting intent IDs |
| Refund may revoke an unrelated access grant | Entitlement correctness | Track grant provenance |
| Concurrent refund and purchase lack shared entitlement locking | Race condition | Lock one entitlement aggregate |
| Payment webhook event may arrive out of order | Eventual consistency | Add reconciliation |
| Checkout can be created before database attachment succeeds | External side-effect gap | Reconcile orphaned sessions |
| Course version bumps mix JDBC and managed JPA entities | Stale persistence context | Standardize locking/version protocol |
| Older course routes may bypass ownership checks | Authorization bypass | Audit every write route |
| Success redirect is not proof of payment | Unauthorized access risk | Wait for verified backend fulfillment |
| `@WithMockUser` lacks application principal ID | Misleading security tests | Use real principal or login |
| Test transaction can conceal commits and rollbacks | False confidence | Use independent transactions |

### Test prioritization

Before enabling real payments, I would treat these as release blockers: signature verification, amount/currency matching, enrollment idempotency, refund entitlement provenance, concurrent grant/revoke correctness, and a complete session/CSRF test.

---

## 17. Hands-on exercises

<box gap={3}>
  <row gap={3} align="start">
    <icon name="test-tube" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise A — Parameterized publishing rules**

      Write JUnit parameterized tests for every allowed and forbidden course lifecycle transition: `DRAFT → PUBLISHED`, `PUBLISHED → ARCHIVED`, and all invalid transitions. Include expected-version conflicts.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="shield-check" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise B — Real session and CSRF**

      Register a student, log in, restore the session, attempt a protected POST without CSRF, then retry with CSRF. Verify logout invalidates the session.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="database" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise C — Enrollment concurrency**

      Start two independent transactions that attempt to enroll the same student in the same course. Verify the unique database constraint and the application's intended idempotent response.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="refresh-cw" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise D — Webhook retry and rollback**

      Inject a failure during enrollment creation, verify complete rollback, then retry the same signed event and prove it succeeds exactly once.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="file-code" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise E — Angular contract drift**

      Intentionally rename a JSON response field in a test fixture. Make the contract test fail, then restore the correct field and verify the typed service.
    </box>
  </row>
</box>

## 18. Interactive knowledge quiz

{@body const questions=[{q:"Which test best proves a PostgreSQL unique constraint works under concurrent requests?",options:["Mockito unit test","Testcontainers with independent transactions","Angular component test"],correct:1,why:"Only a real database exercising independent transactions can establish its locking and uniqueness behavior."},{q:"What does @WithMockUser fail to provide for LearnHub?",options:["Any Spring Security authority","A database-backed LearnHubPrincipal with a real user ID","A way to test role matchers"],correct:1,why:"It creates a mock Spring Security identity, not necessarily the application principal or database record."},{q:"Why should rollback tests generally avoid an outer @Transactional test?",options:["It can hide actual commit and rollback boundaries","It disables SQL constraints","It makes JSON serialization fail"],correct:0,why:"The test-managed transaction may roll back everything at the end, obscuring whether the application service committed correctly."},{q:"What should a successful payment redirect do?",options:["Immediately create enrollment in Angular","Ask the backend for verified payment and access status","Trust a success=true query parameter"],correct:1,why:"The browser redirect is not trusted proof of payment."},{q:"What does a duplicate webhook test need to assert?",options:["Only that HTTP returns 200","Only that the event JSON parses","That event processing and enrollment side effects occur at most once"],correct:2,why:"Idempotency concerns durable state and side effects, not merely response codes."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    **Lesson 21 — Testing mastery**
    <text color="secondary" size="xs">5 questions</text>
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
        <row gap={2} align="start">
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
    <title size="lg">{questions.filter((q,i)=>answers[i]===q.correct).length} / 5 correct</title>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==questions.length} onClick={()=>setGraded(true)}>Check answers</button>
    <button variant="outline" color="secondary" onClick={()=>{setAnswers({});setGraded(false);}}>Reset</button>
  </row>
</box>

## 19. Lesson completion checklist

{@body const groups=[{name:"Java unit and service tests",items:["Confirmed Maven test dependencies","Created test profile","Tested Money conversion edge cases","Tested payment state transitions","Fixed conflicting payment-intent handling","Added Mockito service test","Verified forbidden side effects"]},{name:"Spring and PostgreSQL",items:["Configured PostgreSQL Testcontainers","Verified Flyway migrations in tests","Tested database unique constraints","Added @WebMvcTest controller slice","Tested production SecurityFilterChain","Tested real session restoration and logout","Tested course ownership isolation","Tested free versus paid enrollment rules"]},{name:"Payments and concurrency",items:["Tested webhook HMAC verification","Tested signed HTTP webhook delivery","Tested amount and currency mismatch","Tested duplicate webhook events","Tested transaction rollback","Tested concurrent enrollment grants","Reviewed refund entitlement concurrency"]},{name:"Angular and delivery",items:["Added checkout HttpTestingController tests","Tested authentication restoration","Tested instructor route guards","Reviewed CSRF browser coverage","Reviewed API contract inventory","Ran Maven clean verify","Ran Angular tests and build","Completed at least two exercises","Completed knowledge quiz","Reviewed Git diff and committed"]}]}
{@body const [checked,setChecked]=DIL.useState({})}
{@body const total=groups.reduce((n,g)=>n+g.items.length,0)}
{@body const completed=Object.values(checked).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    <box gap={1}>
      **Lesson 21 verification progress**
      <text color="secondary" size="xs">Mark items only after running them locally.</text>
    </box>
    <title size="lg" tabularNums>{completed}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${100*completed/total}%`} radius="full"/>
  </box>
  {#each groups as group,i}
    <box key={i} gap={2}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!checked[`${i}-${j}`]} onChange={v=>setChecked(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if completed===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **All Lesson 21 checklist items complete.**
    </row>
  {/if}
</box>

## 20. Git checkpoint

Review the backend changes:

```powershell
cd $HOME\angular-learning\learnhub-backend

git diff --check
git status --short
git diff --stat

git add pom.xml src\main src\test
git diff --cached --stat
git commit -m "Build comprehensive backend security and integration tests"
```

Review the Angular changes:

```powershell
cd $HOME\angular-learning\learnhub

git diff --check
git status --short
git diff --stat

git add src\app
git diff --cached --stat
git commit -m "Add Angular API contract and auth integration tests"
```

Only commit after the tests pass. These are instructional examples; I have not executed them against your local LearnHub workspace.

## 21. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" radius="lg" padding={3}>
    <icon name="activity" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 22 — API Documentation, Observability, and Production Diagnostics**

    We'll introduce OpenAPI documentation, Swagger UI, actuator health checks, structured logging, correlation IDs, Micrometer metrics, request tracing, and secure operational endpoints.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 22: OpenAPI 3 and Swagger UI, API contract documentation, Spring Boot Actuator health and readiness probes, Micrometer metrics, structured JSON logging, correlation IDs, distributed tracing concepts, security-safe error reporting, PostgreSQL and payment observability, Angular API contract alignment, MockMvc and Testcontainers tests, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 21 with experienced-Java pacing.")}>Start Lesson 22 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 21 deliverable:** A layered automated test suite covering pure Java rules, service orchestration, HTTP contracts, Spring Security, real PostgreSQL behavior, payment verification, transaction rollback, concurrency, and Angular integration.

**Core principle:** A test suite is trustworthy when it proves both that authorized operations succeed and that forbidden, duplicated, stale, or failed operations leave the system in a correct state.