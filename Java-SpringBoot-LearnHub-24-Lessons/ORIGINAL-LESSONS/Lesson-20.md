# LearnHub Java & Spring Boot — Lesson 20

<text color="secondary" size="sm">Secure checkout · Verified webhooks · PostgreSQL payments · Idempotent fulfillment · Refunds · Angular integration</text>

**Stack:** Java 21 · Spring Boot 3.5.x · PostgreSQL · Flyway · Spring Security · Stripe Checkout · Angular standalone · JUnit 5 · MockMvc · Testcontainers

**Estimated time:** 10–14 hours, including exercises and tests.

## 1. What we're building

Lesson 19 gave instructors a controlled publishing workflow. Lesson 20 introduces the commercial side of LearnHub: students can purchase published paid courses and receive enrollment access **only after the backend verifies payment**.

We'll use <Entity category="software" value="Stripe Checkout" disambig="Stripe-hosted payment checkout"/> as a concrete provider example. The design keeps payment-provider details behind a Java interface so we can test the application without making real payments.

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" padding={3} radius="lg" gap={1}>
    **1. Student chooses a paid course**
    <text size="xs" color="secondary">Angular requests checkout using the authenticated session</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} gap={1}>
    **2. Spring Boot creates a payment attempt**
    <text size="xs" color="secondary">Authoritative course price · Pending database record · Provider checkout session</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} gap={1}>
    **3. Student pays on provider-hosted checkout**
    <text size="xs" color="secondary">Card details never pass through LearnHub's backend</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} gap={1}>
    **4. Provider sends a signed webhook**
    <text size="xs" color="secondary">Verify raw payload signature · Confirm payment status, amount, currency, and ownership</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" border radius="lg" padding={3} gap={1}>
    **5. Backend grants enrollment**
    <text size="xs" color="secondary">Transactional fulfillment · Unique constraints · Safe webhook retries</text>
  </box>
</box>

### Security rule of the day

**A successful redirect to `/checkout/success` is not proof of payment.**

The browser is untrusted. The backend grants paid enrollment only after processing a verified provider event or another trusted server-side confirmation.

## 2. Payment state machine

We'll separate the state of a payment attempt from the state of a student's enrollment.

<box border radius="xl" padding={3} gap={2}>
  <row justify="center" align="center" gap={2}>
    <box background="surface-secondary" radius="lg" padding={3}>
      **PENDING**
    </box>
    <icon name="arrow-right" color="secondary"/>
    <box background="surface-secondary" radius="lg" padding={3}>
      **PAID**
    </box>
  </row>
  <grid columns={2} gap={2}>
    <grid-item>
      <box border radius="lg" padding={3} gap={1}>
        **FAILED / EXPIRED**
        <text size="xs" color="secondary">No entitlement granted</text>
      </box>
    </grid-item>
    <grid-item>
      <box border radius="lg" padding={3} gap={1}>
        **REFUNDED**
        <text size="xs" color="secondary">Entitlement reviewed or revoked by policy</text>
      </box>
    </grid-item>
  </grid>
</box>

We will use these application states:

| State | Meaning | Student access |
|---|---|---|
| `PENDING` | Checkout created, payment not verified | No new access |
| `PAID` | Provider-confirmed successful payment | Active enrollment |
| `FAILED` | Payment attempt failed | No new access |
| `EXPIRED` | Checkout session expired | No new access |
| `REFUNDED` | Full refund verified | Revoked under today's policy |

A partial refund will **not** automatically revoke access. We'll track refund amounts separately and treat full refund as a distinct transition.

### Important domain distinction

A student can have multiple payment attempts for the same course, but should have at most one active enrollment for that course.

Therefore:

- `payment_attempts` can contain several rows per student/course.
- `enrollments` retains its Lesson 9 unique `(student_user_id, course_id)` constraint.
- A payment attempt records what happened financially.
- An enrollment records access.

---

## 3. Prerequisites and baseline

Open PowerShell.

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info
git status

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Fix Lesson 19 backend failures first."
}
```

Angular:

```powershell
cd $HOME\angular-learning\learnhub

node --version
npm --version

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Fix Lesson 19 Angular failures first."
}
```

Inspect the current database schema and migrations:

```powershell
cd $HOME\angular-learning\learnhub-backend

Get-ChildItem src\main\resources\db\migration |
    Sort-Object Name
```

Lesson 19 used `V10` as an example. We'll use **V11** only if it is the next unused Flyway version.

Check the actual Lesson 17 enrollment model:

```powershell
Get-Content src\main\java\com\learnhub\persistence\EnrollmentEntity.java

Get-Content src\main\java\com\learnhub\persistence\JpaEnrollmentRepository.java

Get-Content src\main\java\com\learnhub\service\EnrollmentService.java
```

The payment implementation must reuse the existing enrollment uniqueness and `ACTIVE`/`CANCELLED` lifecycle rather than create a competing entitlement table.

---

## 4. PostgreSQL payment schema

We'll create two tables:

- `payment_attempts` — one row per checkout attempt.
- `payment_webhook_events` — an inbox of verified provider events, deduplicated by event ID.

We'll also track refund totals on the payment attempt.

### Flyway migration

**Create:** `src/main/resources/db/migration/V11__create_payments.sql`

<CodeBlock language="sql" editable>
-- Lesson 20: payments and verified webhook inbox.
-- Rename V11 if your Flyway history differs.

CREATE TABLE payment_attempts (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY
        PRIMARY KEY,

    student_user_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,

    provider VARCHAR(30) NOT NULL DEFAULT 'STRIPE',

    provider_checkout_session_id VARCHAR(255),
    provider_payment_intent_id VARCHAR(255),

    amount_minor BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,

    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',

    refunded_minor BIGINT NOT NULL DEFAULT 0,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_payment_attempt_student
        FOREIGN KEY (student_user_id)
        REFERENCES app_users(id)
        ON DELETE RESTRICT,

    CONSTRAINT fk_payment_attempt_course
        FOREIGN KEY (course_id)
        REFERENCES courses(id)
        ON DELETE RESTRICT,

    CONSTRAINT chk_payment_attempt_amount
        CHECK (amount_minor > 0),

    CONSTRAINT chk_payment_attempt_currency
        CHECK (currency ~ '^[A-Z]{3}$'),

    CONSTRAINT chk_payment_attempt_refund
        CHECK (
            refunded_minor >= 0
            AND refunded_minor <= amount_minor
        ),

    CONSTRAINT chk_payment_attempt_status
        CHECK (
            status IN (
                'PENDING',
                'PAID',
                'FAILED',
                'EXPIRED',
                'REFUNDED'
            )
        )
);

CREATE UNIQUE INDEX uq_payment_checkout_session
    ON payment_attempts(provider, provider_checkout_session_id)
    WHERE provider_checkout_session_id IS NOT NULL;

CREATE UNIQUE INDEX uq_payment_intent
    ON payment_attempts(provider, provider_payment_intent_id)
    WHERE provider_payment_intent_id IS NOT NULL;

CREATE INDEX idx_payment_student_course
    ON payment_attempts(student_user_id, course_id, created_at DESC);

CREATE INDEX idx_payment_course_status
    ON payment_attempts(course_id, status);

CREATE TABLE payment_webhook_events (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY
        PRIMARY KEY,

    provider VARCHAR(30) NOT NULL,

    provider_event_id VARCHAR(255) NOT NULL,

    event_type VARCHAR(120) NOT NULL,

    received_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    processed_at TIMESTAMPTZ,

    CONSTRAINT uq_payment_webhook_event
        UNIQUE (provider, provider_event_id)
);
</CodeBlock>

### Why use integer minor units?

Payment providers commonly represent money in the smallest currency unit. For USD:

\[
\$49.99 \rightarrow 4999\text{ cents}
\]

Never use `double` or `float` for monetary calculations.

### Why store a price snapshot?

The course price can change after checkout starts.

`payment_attempts.amount_minor` captures the amount the server authorized for that checkout attempt. Fulfillment compares the verified provider amount against this snapshot, not the course's current price.

### PostgreSQL uniqueness is the final guard

Webhook deliveries can be duplicated or arrive concurrently. The unique provider event ID prevents the same event from being inserted twice.

However, **event deduplication alone does not guarantee correct fulfillment**. We'll also lock the payment attempt and make state transitions idempotent.

---

## 5. Java dependencies and configuration

Spring Boot already provides Jackson, validation, JDBC, JPA, and transaction support through the dependencies from previous lessons. We'll use Java 21's `HttpClient` for the Stripe REST API and Java's cryptography libraries for webhook verification.

This avoids introducing a payment SDK whose API may differ by version. For a production integration, using Stripe's maintained Java SDK is also a good option.

### Development configuration

**Modify:** `src/main/resources/application.yml`

Merge this section into the existing file:

<CodeBlock language="yaml" editable>
learnhub:
  payments:
    stripe:
      secret-key: ${STRIPE_SECRET_KEY:}
      webhook-secret: ${STRIPE_WEBHOOK_SECRET:}
      api-base: https://api.stripe.com
    public-base-url: ${LEARNHUB_PUBLIC_BASE_URL:http://localhost:4200}
</CodeBlock>

Do not commit live keys.

Set development secrets in the current PowerShell session:

```powershell
$env:STRIPE_SECRET_KEY = "sk_test_REPLACE_WITH_YOUR_KEY"
$env:STRIPE_WEBHOOK_SECRET = "whsec_REPLACE_WITH_YOUR_WEBHOOK_SECRET"
$env:LEARNHUB_PUBLIC_BASE_URL = "http://localhost:4200"
```

Use only Stripe test-mode credentials for this lesson.

### Configuration properties

**Create:** `src/main/java/com/learnhub/payments/PaymentProperties.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "learnhub.payments")
public record PaymentProperties(
        Stripe stripe,
        String publicBaseUrl
) {
    public record Stripe(
            String secretKey,
            String webhookSecret,
            String apiBase
    ) {}
}
</CodeBlock>

Register it in your Spring Boot application or a configuration class:

```java
@EnableConfigurationProperties(PaymentProperties.class)
```

Import:

```java
import com.learnhub.payments.PaymentProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
```

---

## 6. Payment domain model

We'll represent payment state with an enum and keep the database entity responsible for valid transitions.

**Create:** `src/main/java/com/learnhub/payments/PaymentStatus.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

public enum PaymentStatus {
    PENDING,
    PAID,
    FAILED,
    EXPIRED,
    REFUNDED
}
</CodeBlock>

**Create:** `src/main/java/com/learnhub/payments/PaymentAttemptEntity.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import jakarta.persistence.*;
import java.time.OffsetDateTime;

@Entity
@Table(name = "payment_attempts")
public class PaymentAttemptEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "student_user_id", nullable = false)
    private Long studentUserId;

    @Column(name = "course_id", nullable = false)
    private Long courseId;

    @Column(nullable = false)
    private String provider = "STRIPE";

    @Column(name = "provider_checkout_session_id")
    private String checkoutSessionId;

    @Column(name = "provider_payment_intent_id")
    private String paymentIntentId;

    @Column(name = "amount_minor", nullable = false)
    private long amountMinor;

    @Column(nullable = false, length = 3)
    private String currency;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private PaymentStatus status = PaymentStatus.PENDING;

    @Column(name = "refunded_minor", nullable = false)
    private long refundedMinor;

    @Column(name = "created_at", insertable = false, updatable = false)
    private OffsetDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private OffsetDateTime updatedAt = OffsetDateTime.now();

    protected PaymentAttemptEntity() {}

    public PaymentAttemptEntity(
            long studentUserId,
            long courseId,
            long amountMinor,
            String currency
    ) {
        if (amountMinor <= 0) {
            throw new IllegalArgumentException(
                    "Payment amount must be positive."
            );
        }

        this.studentUserId = studentUserId;
        this.courseId = courseId;
        this.amountMinor = amountMinor;
        this.currency = currency;
    }

    public void attachCheckout(String sessionId) {
        if (checkoutSessionId != null
                && !checkoutSessionId.equals(sessionId)) {
            throw new IllegalStateException(
                    "Checkout session already assigned."
            );
        }
        checkoutSessionId = sessionId;
        touch();
    }

    public void markPaid(String intentId) {
        if (status == PaymentStatus.PAID) {
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

    public void markExpired() {
        if (status == PaymentStatus.PENDING) {
            status = PaymentStatus.EXPIRED;
            touch();
        }
    }

    public void markFailed() {
        if (status == PaymentStatus.PENDING) {
            status = PaymentStatus.FAILED;
            touch();
        }
    }

    public void applyRefund(long totalRefundedMinor) {
        if (status != PaymentStatus.PAID
                && status != PaymentStatus.REFUNDED) {
            throw new IllegalStateException(
                    "Cannot refund an unpaid attempt."
            );
        }

        if (totalRefundedMinor < refundedMinor
                || totalRefundedMinor > amountMinor) {
            throw new IllegalArgumentException(
                    "Invalid cumulative refund amount."
            );
        }

        refundedMinor = totalRefundedMinor;

        if (refundedMinor == amountMinor) {
            status = PaymentStatus.REFUNDED;
        }
        touch();
    }

    private void touch() {
        updatedAt = OffsetDateTime.now();
    }

    public Long getId() { return id; }
    public Long getStudentUserId() { return studentUserId; }
    public Long getCourseId() { return courseId; }
    public String getCheckoutSessionId() { return checkoutSessionId; }
    public String getPaymentIntentId() { return paymentIntentId; }
    public long getAmountMinor() { return amountMinor; }
    public String getCurrency() { return currency; }
    public PaymentStatus getStatus() { return status; }
    public long getRefundedMinor() { return refundedMinor; }
}
</CodeBlock>

### JPA repository

**Create:** `src/main/java/com/learnhub/payments/JpaPaymentAttemptRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.Optional;

public interface JpaPaymentAttemptRepository
        extends JpaRepository<PaymentAttemptEntity, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        SELECT p FROM PaymentAttemptEntity p
        WHERE p.id = :id
        """)
    Optional<PaymentAttemptEntity> lockById(
            @Param("id") Long id
    );

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        SELECT p FROM PaymentAttemptEntity p
        WHERE p.provider = 'STRIPE'
          AND p.paymentIntentId = :intentId
        """)
    Optional<PaymentAttemptEntity> lockByPaymentIntentId(
            @Param("intentId") String intentId
    );
}
</CodeBlock>

**Why pessimistic locking here?** Multiple verified events can update the same payment attempt. The row lock serializes those state transitions within PostgreSQL transactions.

---

## 7. Server-authoritative checkout

The Angular application will send only a course ID.

It must not send:

- A price.
- A currency.
- A student ID.
- A claim that the payment succeeded.

Those values must come from trusted server-side state.

### Currency policy

For this lesson, LearnHub sells paid courses in **USD only**. The earlier `courses.price` field does not store a currency, so we must not pretend it supports multi-currency pricing.

We can extend the course model with currency-aware pricing in a future lesson.

### Money conversion

**Create:** `src/main/java/com/learnhub/payments/Money.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import java.math.BigDecimal;
import java.math.RoundingMode;

public final class Money {

    private Money() {}

    public static long usdMinor(BigDecimal amount) {
        if (amount == null || amount.signum() <= 0) {
            throw new IllegalArgumentException(
                    "Paid course price must be positive."
            );
        }

        return amount
                .setScale(2, RoundingMode.UNNECESSARY)
                .movePointRight(2)
                .longValueExact();
    }
}
</CodeBlock>

### Provider interface

**Create:** `src/main/java/com/learnhub/payments/CheckoutProvider.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

public interface CheckoutProvider {

    CheckoutSession createCheckout(
            long paymentAttemptId,
            long amountMinor,
            String currency,
            String courseTitle
    );

    record CheckoutSession(
            String sessionId,
            String checkoutUrl
    ) {}
}
</CodeBlock>

This interface is an important testing seam: integration tests can replace the provider with a fake while still exercising real PostgreSQL transactions.

---

## 8. Stripe-hosted checkout adapter

Stripe's Checkout Sessions API accepts form-encoded parameters and returns a hosted checkout URL.

The adapter below uses an authenticated HTTPS request from Spring Boot. It never exposes the Stripe secret key to Angular.

**Create:** `src/main/java/com/learnhub/payments/StripeCheckoutProvider.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import org.springframework.stereotype.Component;

import java.net.URI;
import java.net.URLEncoder;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.stream.Collectors;

@Component
public class StripeCheckoutProvider implements CheckoutProvider {

    private final PaymentProperties properties;
    private final ObjectMapper mapper;
    private final HttpClient http = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .build();

    public StripeCheckoutProvider(
            PaymentProperties properties,
            ObjectMapper mapper
    ) {
        this.properties = properties;
        this.mapper = mapper;
    }

    @Override
    public CheckoutSession createCheckout(
            long attemptId,
            long amountMinor,
            String currency,
            String title
    ) {
        String secret = properties.stripe().secretKey();

        if (secret == null || secret.isBlank()) {
            throw new IllegalStateException(
                    "Stripe test secret key is not configured."
            );
        }

        Map<String, String> fields = new LinkedHashMap<>();

        fields.put("mode", "payment");
        fields.put("client_reference_id", Long.toString(attemptId));
        fields.put("metadata[attempt_id]", Long.toString(attemptId));
        fields.put("payment_intent_data[metadata][attempt_id]",
                Long.toString(attemptId));

        fields.put("line_items[0][price_data][currency]", currency);
        fields.put("line_items[0][price_data][unit_amount]",
                Long.toString(amountMinor));
        fields.put("line_items[0][price_data][product_data][name]",
                title);
        fields.put("line_items[0][quantity]", "1");

        fields.put("success_url",
                properties.publicBaseUrl()
                        + "/checkout/success?session_id="
                        + "{CHECKOUT_SESSION_ID}");

        fields.put("cancel_url",
                properties.publicBaseUrl()
                        + "/checkout/cancel");

        String form = fields.entrySet().stream()
                .map(e -> encode(e.getKey())
                        + "=" + encode(e.getValue()))
                .collect(Collectors.joining("&"));

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(
                        properties.stripe().apiBase()
                                + "/v1/checkout/sessions"
                ))
                .timeout(Duration.ofSeconds(20))
                .header("Authorization", "Bearer " + secret)
                .header(
                        "Content-Type",
                        "application/x-www-form-urlencoded"
                )
                .header(
                        "Idempotency-Key",
                        "learnhub-checkout-" + attemptId
                )
                .POST(HttpRequest.BodyPublishers.ofString(form))
                .build();

        try {
            HttpResponse<String> response = http.send(
                    request,
                    HttpResponse.BodyHandlers.ofString()
            );

            if (response.statusCode() < 200
                    || response.statusCode() >= 300) {
                // Do not log the full provider response;
                // it may contain sensitive operational data.
                throw new IllegalStateException(
                        "Checkout provider rejected the request."
                );
            }

            JsonNode json = mapper.readTree(response.body());

            String id = json.path("id").asText();
            String url = json.path("url").asText();

            if (!id.startsWith("cs_")
                    || !url.startsWith("https://")) {
                throw new IllegalStateException(
                        "Invalid checkout provider response."
                );
            }

            return new CheckoutSession(id, url);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(
                    "Checkout request interrupted.", e
            );
        } catch (java.io.IOException e) {
            throw new IllegalStateException(
                    "Checkout provider unavailable.", e
            );
        }
    }

    private static String encode(String value) {
        return URLEncoder.encode(
                value, StandardCharsets.UTF_8
        );
    }
}
</CodeBlock>

### Production provider configuration

Never let a browser request choose the Stripe API host or redirect base URL. Both must come from trusted configuration.

The `Idempotency-Key` makes retries of the same checkout attempt safer. It does not eliminate the need for database idempotency.

---

## 9. Transaction boundaries for checkout creation

A payment checkout involves an external network call. We should **not hold a database transaction open while waiting for Stripe**.

We'll use three stages:

1. In a short transaction, create a `PENDING` payment attempt.
2. Outside the transaction, call Stripe.
3. In another short transaction, attach the Checkout Session ID.

If the Stripe request fails, the attempt remains pending until a retry or reconciliation process handles it. That is safer than pretending a timeout means payment definitely failed.

### Transactional persistence component

**Create:** `src/main/java/com/learnhub/payments/PaymentAttemptStore.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class PaymentAttemptStore {

    private final JpaPaymentAttemptRepository repository;

    public PaymentAttemptStore(
            JpaPaymentAttemptRepository repository
    ) {
        this.repository = repository;
    }

    @Transactional
    public long create(
            long studentId,
            long courseId,
            long amountMinor
    ) {
        return repository.saveAndFlush(
                new PaymentAttemptEntity(
                        studentId,
                        courseId,
                        amountMinor,
                        "USD"
                )
        ).getId();
    }

    @Transactional
    public void attach(
            long attemptId,
            String sessionId
    ) {
        var attempt = repository.lockById(attemptId)
                .orElseThrow();

        attempt.attachCheckout(sessionId);
    }
}
</CodeBlock>

### Checkout response

**Create:** `src/main/java/com/learnhub/payments/CheckoutResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

public record CheckoutResponse(
        long paymentAttemptId,
        String checkoutUrl
) {}
</CodeBlock>

### Checkout service

**Create:** `src/main/java/com/learnhub/payments/CheckoutService.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import com.learnhub.model.CourseStatus;
import com.learnhub.persistence.JpaCourseRepository;
import com.learnhub.security.CurrentActor;
import com.learnhub.security.SecurityRole;

import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

@Service
public class CheckoutService {

    private final CurrentActor actor;
    private final JpaCourseRepository courses;
    private final PaymentAttemptStore attempts;
    private final CheckoutProvider provider;
    private final PaidEnrollmentPolicy enrollmentPolicy;

    public CheckoutService(
            CurrentActor actor,
            JpaCourseRepository courses,
            PaymentAttemptStore attempts,
            CheckoutProvider provider,
            PaidEnrollmentPolicy enrollmentPolicy
    ) {
        this.actor = actor;
        this.courses = courses;
        this.attempts = attempts;
        this.provider = provider;
        this.enrollmentPolicy = enrollmentPolicy;
    }

    @PreAuthorize("hasRole('STUDENT')")
    public CheckoutResponse checkout(long courseId) {
        var principal = actor.require();

        if (principal.role() != SecurityRole.STUDENT) {
            throw new AccessDeniedException(
                    "Student account required."
            );
        }

        var course = courses.findById(courseId)
                .orElseThrow(() ->
                        new IllegalArgumentException(
                                "Course unavailable."
                        )
                );

        if (course.getStatus() != CourseStatus.PUBLISHED) {
            throw new IllegalArgumentException(
                    "Course unavailable."
            );
        }

        if (course.isFree()) {
            throw new IllegalArgumentException(
                    "Free courses use direct enrollment."
            );
        }

        enrollmentPolicy.requireNotActive(
                principal.id(),
                courseId
        );

        long amountMinor = Money.usdMinor(
                course.getPrice()
        );

        long attemptId = attempts.create(
                principal.id(),
                courseId,
                amountMinor
        );

        var session = provider.createCheckout(
                attemptId,
                amountMinor,
                "usd",
                course.getTitle()
        );

        attempts.attach(
                attemptId,
                session.sessionId()
        );

        return new CheckoutResponse(
                attemptId,
                session.checkoutUrl()
        );
    }
}
</CodeBlock>

`PaidEnrollmentPolicy` is the eligibility adapter we will define below. It keeps the existing Lesson 17 enrollment rules in one place.

### A checkout race worth understanding

A course might be archived between the initial eligibility check and payment completion. The webhook fulfillment service must therefore apply an explicit policy for already-created checkout attempts.

For this lesson, a verified successful payment for a checkout that was created while the course was published is honored. Future course archiving does not invalidate the historical payment. Student lesson access still follows the archived-course access policy established in Lesson 19.

---
## 10. Enrollment eligibility and fulfillment

Lesson 17 created a unique constraint on `(student_user_id, course_id)`. We'll reuse it.

### Eligibility adapter

**Create:** `src/main/java/com/learnhub/payments/PaidEnrollmentPolicy.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import com.learnhub.persistence.JpaEnrollmentRepository;

import org.springframework.stereotype.Component;

@Component
public class PaidEnrollmentPolicy {

    private final JpaEnrollmentRepository enrollments;

    public PaidEnrollmentPolicy(
            JpaEnrollmentRepository enrollments
    ) {
        this.enrollments = enrollments;
    }

    public void requireNotActive(
            long studentId,
            long courseId
    ) {
        if (enrollments
                .findByStudent_IdAndCourse_IdAndStatus(
                        studentId,
                        courseId,
                        "ACTIVE"
                )
                .isPresent()) {
            throw new IllegalStateException(
                    "Student already has course access."
            );
        }
    }
}
</CodeBlock>

### Transaction-safe enrollment grant

We'll use a native PostgreSQL upsert. It works with the existing enrollment uniqueness constraint and avoids a check-then-insert race.

**Create:** `src/main/java/com/learnhub/payments/PaidEnrollmentWriter.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class PaidEnrollmentWriter {

    private final JdbcTemplate jdbc;

    public PaidEnrollmentWriter(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public void grant(long studentId, long courseId) {
        jdbc.update("""
            INSERT INTO enrollments (
                student_user_id,
                course_id,
                status
            )
            VALUES (?, ?, 'ACTIVE')
            ON CONFLICT (student_user_id, course_id)
            DO UPDATE SET status = 'ACTIVE'
            """,
            studentId,
            courseId
        );
    }

    public void revoke(long studentId, long courseId) {
        jdbc.update("""
            UPDATE enrollments
            SET status = 'CANCELLED'
            WHERE student_user_id = ?
              AND course_id = ?
              AND status = 'ACTIVE'
            """,
            studentId,
            courseId
        );
    }
}
</CodeBlock>

### Important refund and repurchase rule

The simple `revoke()` method above is **not sufficient for production** if a student has multiple successful purchases for the same course.

A refund of Purchase A must not revoke access granted by an independent, still-valid Purchase B.

Before enabling refunds in production, either:

- Introduce an explicit entitlement table linking access grants to payment attempts; or
- Query all successful, unrefunded payment attempts before revoking access.

We'll implement the second approach in the refund service.

---

## 11. Webhook signature verification

A webhook endpoint must not trust an event merely because its JSON says `"paid": true`.

Stripe sends a `Stripe-Signature` header. We'll verify its HMAC-SHA256 signature over the **exact raw request body**.

### Security requirements

- Verify the signature before parsing or processing the event.
- Use the raw UTF-8 payload bytes.
- Enforce a timestamp tolerance.
- Compare signatures in constant time.
- Reject missing or malformed headers.
- Never accept a secret from the request.
- Never log the webhook signing secret.

### StripeWebhookVerifier.java

**Create:** `src/main/java/com/learnhub/payments/StripeWebhookVerifier.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import org.springframework.stereotype.Component;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.HexFormat;
import java.util.List;

@Component
public class StripeWebhookVerifier {

    private static final Duration TOLERANCE =
            Duration.ofMinutes(5);

    private final PaymentProperties properties;
    private final Clock clock;

    public StripeWebhookVerifier(
            PaymentProperties properties
    ) {
        this.properties = properties;
        this.clock = Clock.systemUTC();
    }

    public void verify(
            byte[] rawBody,
            String signatureHeader
    ) {
        String secret =
                properties.stripe().webhookSecret();

        if (secret == null || secret.isBlank()
                || signatureHeader == null) {
            throw new InvalidWebhookSignatureException();
        }

        Long timestamp = null;
        List<byte[]> signatures = new ArrayList<>();

        try {
            for (String item : signatureHeader.split(",")) {
                String[] parts = item.trim().split("=", 2);

                if (parts.length != 2) {
                    continue;
                }

                if ("t".equals(parts[0])) {
                    timestamp = Long.parseLong(parts[1]);
                }

                if ("v1".equals(parts[0])) {
                    signatures.add(
                            HexFormat.of().parseHex(parts[1])
                    );
                }
            }

            if (timestamp == null || signatures.isEmpty()) {
                throw new InvalidWebhookSignatureException();
            }

            Instant signedAt =
                    Instant.ofEpochSecond(timestamp);

            if (Duration.between(
                    signedAt, clock.instant()
            ).abs().compareTo(TOLERANCE) > 0) {
                throw new InvalidWebhookSignatureException();
            }

            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(
                    secret.getBytes(StandardCharsets.UTF_8),
                    "HmacSHA256"
            ));

            mac.update(
                    (timestamp + ".").getBytes(
                            StandardCharsets.UTF_8
                    )
            );
            byte[] expected = mac.doFinal(rawBody);

            boolean valid = signatures.stream().anyMatch(
                    candidate ->
                            MessageDigest.isEqual(
                                    expected, candidate
                            )
            );

            if (!valid) {
                throw new InvalidWebhookSignatureException();
            }

        } catch (InvalidWebhookSignatureException e) {
            throw e;
        } catch (Exception e) {
            throw new InvalidWebhookSignatureException();
        }
    }
}
</CodeBlock>

**Create:** `src/main/java/com/learnhub/payments/InvalidWebhookSignatureException.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

public final class InvalidWebhookSignatureException
        extends RuntimeException {

    public InvalidWebhookSignatureException() {
        super("Invalid webhook signature.");
    }
}
</CodeBlock>

### Why verify before deserialization?

JSON parsers may normalize whitespace or property ordering. Any such transformation would change the bytes and invalidate the signature.

The signature is calculated over the original body, not a reserialized JSON object.

---

## 12. Verified event processing and idempotency

Stripe may deliver the same event more than once, and event types may arrive out of order.

Our processing rules will be:

- `checkout.session.completed`: fulfill only when `payment_status` is `paid`.
- `checkout.session.async_payment_succeeded`: fulfill an asynchronous payment after it succeeds.
- `checkout.session.async_payment_failed`: mark the pending attempt failed.
- `checkout.session.expired`: mark the pending attempt expired.
- `charge.refunded`: record cumulative refund amount and evaluate entitlement revocation.
- Unknown event types: acknowledge safely without changing financial state.

**Never fulfill a Checkout Session merely because its status is `complete`; require its payment status to be `paid`.**

### 12.1 Verified webhook inbox

**Create:** `src/main/java/com/learnhub/payments/WebhookInboxRepository.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class WebhookInboxRepository {

    private final JdbcTemplate jdbc;

    public WebhookInboxRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public boolean claim(String eventId, String type) {
        return jdbc.update("""
            INSERT INTO payment_webhook_events (
                provider, provider_event_id, event_type
            )
            VALUES ('STRIPE', ?, ?)
            ON CONFLICT (provider, provider_event_id)
            DO NOTHING
            """,
            eventId,
            type
        ) == 1;
    }

    public void markProcessed(String eventId) {
        jdbc.update("""
            UPDATE payment_webhook_events
            SET processed_at = CURRENT_TIMESTAMP
            WHERE provider = 'STRIPE'
              AND provider_event_id = ?
            """,
            eventId
        );
    }
}
</CodeBlock>

The inbox claim and payment changes must execute in **one database transaction**. If processing fails, the event claim rolls back too, allowing Stripe's retry to process it.

### 12.2 Refund-safe entitlement revocation

Add this query to `JpaPaymentAttemptRepository`:

<CodeBlock language="java" editable>
@Query("""
    SELECT COUNT(p)
    FROM PaymentAttemptEntity p
    WHERE p.studentUserId = :studentId
      AND p.courseId = :courseId
      AND p.status = :status
    """)
long countByStudentCourseAndStatus(
        @Param("studentId") Long studentId,
        @Param("courseId") Long courseId,
        @Param("status") PaymentStatus status
);
</CodeBlock>

A full refund should revoke access only when no other valid paid attempt remains. However, **a student's access might also come from a free enrollment, an administrative grant, or another entitlement source**. The existing Lesson 17 schema cannot distinguish those sources.

For that reason, the implementation below makes refund revocation conditional on a strict lesson assumption: **paid-course access is granted only by this payment subsystem**. Do not use it for manually granted or promotional entitlements without adding provenance.

### 12.3 Webhook processor

**Create:** `src/main/java/com/learnhub/payments/StripeWebhookProcessor.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class StripeWebhookProcessor {

    private final ObjectMapper mapper;
    private final WebhookInboxRepository inbox;
    private final JpaPaymentAttemptRepository payments;
    private final PaidEnrollmentWriter enrollments;

    public StripeWebhookProcessor(
            ObjectMapper mapper,
            WebhookInboxRepository inbox,
            JpaPaymentAttemptRepository payments,
            PaidEnrollmentWriter enrollments
    ) {
        this.mapper = mapper;
        this.inbox = inbox;
        this.payments = payments;
        this.enrollments = enrollments;
    }

    @Transactional
    public void process(byte[] verifiedPayload) {
        JsonNode event;

        try {
            event = mapper.readTree(verifiedPayload);
        } catch (Exception e) {
            throw new InvalidWebhookSignatureException();
        }

        String eventId = requiredText(event, "id");
        String type = requiredText(event, "type");

        if (!eventId.startsWith("evt_")) {
            throw new IllegalArgumentException(
                    "Invalid event ID."
            );
        }

        if (!inbox.claim(eventId, type)) {
            // Already processed by another delivery.
            return;
        }

        JsonNode object = event.path("data").path("object");

        switch (type) {
            case "checkout.session.completed",
                 "checkout.session.async_payment_succeeded" ->
                    handlePaidCheckout(object);

            case "checkout.session.async_payment_failed" ->
                    handleUnpaidCheckout(object, false);

            case "checkout.session.expired" ->
                    handleUnpaidCheckout(object, true);

            case "charge.refunded" ->
                    handleRefund(object);

            default -> {
                // Ignore unsupported event types safely.
            }
        }

        inbox.markProcessed(eventId);
    }

    private void handlePaidCheckout(JsonNode session) {
        if (!"paid".equals(
                session.path("payment_status").asText()
        )) {
            // A completed checkout may still await payment.
            return;
        }

        long attemptId = parseAttemptId(session);

        var payment = payments.lockById(attemptId)
                .orElseThrow(() ->
                        new IllegalStateException(
                                "Unknown payment attempt."
                        )
                );

        verifyCheckoutIdentity(payment, session);

        String intentId = requiredText(
                session, "payment_intent"
        );

        if (!intentId.startsWith("pi_")) {
            throw new IllegalArgumentException(
                    "Invalid payment intent."
            );
        }

        if (payment.getStatus() == PaymentStatus.PAID) {
            if (!intentId.equals(
                    payment.getPaymentIntentId()
            )) {
                throw new IllegalStateException(
                        "Conflicting payment intent."
                );
            }
            return;
        }

        // A refund or terminal failure must not be
        // silently reversed by an old webhook.
        if (payment.getStatus() != PaymentStatus.PENDING) {
            return;
        }

        payment.attachCheckout(
                requiredText(session, "id")
        );

        payment.markPaid(intentId);

        enrollments.grant(
                payment.getStudentUserId(),
                payment.getCourseId()
        );
    }

    private void handleUnpaidCheckout(
            JsonNode session,
            boolean expired
    ) {
        long attemptId = parseAttemptId(session);

        var payment = payments.lockById(attemptId)
                .orElseThrow();

        verifyCheckoutIdentity(payment, session);

        payment.attachCheckout(
                requiredText(session, "id")
        );

        if (expired) {
            payment.markExpired();
        } else {
            payment.markFailed();
        }
    }

    private void handleRefund(JsonNode charge) {
        String intentId = requiredText(
                charge, "payment_intent"
        );

        var payment = payments
                .lockByPaymentIntentId(intentId)
                .orElseThrow(() ->
                        new IllegalStateException(
                                "Unknown refunded payment."
                        )
                );

        long refunded = charge.path(
                "amount_refunded"
        ).asLong(-1);

        long chargeAmount = charge.path(
                "amount"
        ).asLong(-1);

        String chargeCurrency = requiredText(
                charge, "currency"
        );

        if (chargeAmount != payment.getAmountMinor()
                || !chargeCurrency.equalsIgnoreCase(
                        payment.getCurrency()
                )) {
            throw new IllegalStateException(
                    "Refund amount or currency mismatch."
            );
        }

        payment.applyRefund(refunded);

        if (payment.getStatus() != PaymentStatus.REFUNDED) {
            return;
        }

        long otherPaidAttempts =
                payments.countByStudentCourseAndStatus(
                        payment.getStudentUserId(),
                        payment.getCourseId(),
                        PaymentStatus.PAID
                );

        if (otherPaidAttempts == 0) {
            enrollments.revoke(
                    payment.getStudentUserId(),
                    payment.getCourseId()
            );
        }
    }

    private static void verifyCheckoutIdentity(
            PaymentAttemptEntity payment,
            JsonNode session
    ) {
        String sessionId = requiredText(session, "id");

        if (!sessionId.startsWith("cs_")) {
            throw new IllegalArgumentException(
                    "Invalid checkout session."
            );
        }

        if (payment.getCheckoutSessionId() != null
                && !sessionId.equals(
                        payment.getCheckoutSessionId()
                )) {
            throw new IllegalStateException(
                    "Checkout session mismatch."
            );
        }

        long amount = session.path(
                "amount_total"
        ).asLong(-1);

        String currency = requiredText(
                session, "currency"
        );

        if (amount != payment.getAmountMinor()
                || !currency.equalsIgnoreCase(
                        payment.getCurrency()
                )) {
            throw new IllegalStateException(
                    "Payment amount or currency mismatch."
            );
        }
    }

    private static long parseAttemptId(JsonNode session) {
        String reference = requiredText(
                session, "client_reference_id"
        );

        long id = Long.parseLong(reference);

        if (id <= 0) {
            throw new IllegalArgumentException(
                    "Invalid payment attempt reference."
            );
        }

        String metadataId = session.path("metadata")
                .path("attempt_id").asText("");

        if (!metadataId.isEmpty()
                && !metadataId.equals(reference)) {
            throw new IllegalArgumentException(
                    "Conflicting attempt metadata."
            );
        }

        return id;
    }

    private static String requiredText(
            JsonNode node,
            String field
    ) {
        String value = node.path(field).asText("");

        if (value.isBlank()) {
            throw new IllegalArgumentException(
                    "Missing provider field: " + field
            );
        }

        return value;
    }
}
</CodeBlock>

### Important out-of-order webhook limitation

A refund event can arrive before the checkout-completion event has been processed, leaving no known payment intent mapping.

The processor deliberately throws in that case, causing the transaction to roll back and the provider to retry. This works for transient ordering problems, but it is not a complete reconciliation system.

A production implementation should also have a scheduled reconciliation job that retrieves authoritative provider state, processes unresolved events, and alerts on persistent failures.

### Important concurrency limitation

The refund handler checks for other paid attempts while holding the refunded attempt's row lock. That does not serialize **all** payment attempts for the same student/course.

For strict refund safety, use a shared lock on the student/course entitlement or enrollment row for every grant and revocation, then recompute the effective entitlement under that lock. We'll keep this as a required hardening task before enabling real refunds.

---

## 13. Webhook controller and Spring Security

### 13.1 Webhook controller

**Create:** `src/main/java/com/learnhub/web/StripeWebhookController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.payments.StripeWebhookProcessor;
import com.learnhub.payments.StripeWebhookVerifier;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/webhooks/stripe")
public class StripeWebhookController {

    private final StripeWebhookVerifier verifier;
    private final StripeWebhookProcessor processor;

    public StripeWebhookController(
            StripeWebhookVerifier verifier,
            StripeWebhookProcessor processor
    ) {
        this.verifier = verifier;
        this.processor = processor;
    }

    @PostMapping
    public ResponseEntity<Void> receive(
            @RequestBody byte[] rawBody,
            @RequestHeader(
                    value = "Stripe-Signature",
                    required = false
            )
            String signature
    ) {
        verifier.verify(rawBody, signature);
        processor.process(rawBody);

        return ResponseEntity.ok().build();
    }
}
</CodeBlock>

### 13.2 Checkout controller

**Create:** `src/main/java/com/learnhub/web/CheckoutController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.payments.CheckoutResponse;
import com.learnhub.payments.CheckoutService;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/me/checkout")
public class CheckoutController {

    private final CheckoutService checkout;

    public CheckoutController(CheckoutService checkout) {
        this.checkout = checkout;
    }

    @PostMapping("/courses/{courseId}")
    public CheckoutResponse create(
            @PathVariable long courseId
    ) {
        return checkout.checkout(courseId);
    }
}
</CodeBlock>

### 13.3 SecurityFilterChain changes

**Modify:** `src/main/java/com/learnhub/config/SecurityConfig.java`

Add the webhook and checkout rules before broader matchers:

<CodeBlock language="java" editable>
// Stripe is a server-to-server caller.
// It does not have an Angular session or CSRF cookie.
.requestMatchers(
        HttpMethod.POST,
        "/api/v1/webhooks/stripe"
).permitAll()

// Only students can initiate checkout.
.requestMatchers(
        HttpMethod.POST,
        "/api/v1/me/checkout/courses/*"
).hasRole("STUDENT")
</CodeBlock>

For CSRF, exempt **only the exact webhook endpoint**:

<CodeBlock language="java" editable>
.csrf(csrf -> csrf
    .csrfTokenRepository(
        CookieCsrfTokenRepository.withHttpOnlyFalse()
    )
    .ignoringRequestMatchers(
        new AntPathRequestMatcher(
            "/api/v1/webhooks/stripe",
            "POST"
        )
    )
)
</CodeBlock>

Imports:

```java
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;
```

Merge this into the existing Lesson 14 CSRF configuration; do not accidentally replace the SPA token request handler or session configuration.

### Why the webhook is `permitAll`

Stripe cannot log into LearnHub with a student session. Its authentication mechanism is the verified webhook signature.

`permitAll` means Spring Security does not require a browser session. It does **not** mean the event is trusted without signature verification.

### Invalid signature response

Add a handler to the existing `GlobalExceptionHandler`:

```java
@ExceptionHandler(InvalidWebhookSignatureException.class)
public ResponseEntity<ProblemDetail> invalidWebhook() {
    ProblemDetail problem =
            ProblemDetail.forStatusAndDetail(
                    HttpStatus.BAD_REQUEST,
                    "Invalid webhook request."
            );

    return ResponseEntity.badRequest().body(problem);
}
```

Import the exception from `com.learnhub.payments`.

---

## 14. Angular standalone checkout integration

The Angular application should start checkout and redirect to the provider-hosted URL.

It should **never** collect card numbers or treat a return URL as a payment receipt.

### 14.1 Typed checkout API

**Create:** `learnhub/src/app/core/api/checkout-api.service.ts`

<CodeBlock language="typescript" editable>
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface CheckoutResponse {
  paymentAttemptId: number;
  checkoutUrl: string;
}

@Injectable({ providedIn: 'root' })
export class CheckoutApiService {
  private readonly http = inject(HttpClient);

  create(courseId: number): Observable<CheckoutResponse> {
    return this.http.post<CheckoutResponse>(
      `/api/v1/me/checkout/courses/${courseId}`,
      {}
    );
  }
}
</CodeBlock>

The existing Lesson 16 HTTP interceptor must attach the CSRF header to this `POST`.

### 14.2 Checkout button

**Create:** `learnhub/src/app/features/checkout/checkout-button.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  inject,
  input,
  signal,
} from '@angular/core';

import { CheckoutApiService }
  from '../../core/api/checkout-api.service';

@Component({
  selector: 'app-checkout-button',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <button
      type="button"
      [disabled]="busy()"
      (click)="checkout()"
    >
      {{ busy() ? 'Opening checkout...' : 'Buy course' }}
    </button>

    @if (error()) {
      <p role="alert">{{ error() }}</p>
    }
  `,
})
export class CheckoutButtonComponent {
  readonly courseId = input.required<number>();

  private readonly api = inject(CheckoutApiService);

  readonly busy = signal(false);
  readonly error = signal<string | null>(null);

  checkout(): void {
    if (this.busy()) {
      return;
    }

    this.busy.set(true);
    this.error.set(null);

    this.api.create(this.courseId()).subscribe({
      next: result => {
        try {
          const url = new URL(result.checkoutUrl);

          if (
            url.protocol !== 'https:' ||
            !(
              url.hostname === 'checkout.stripe.com' ||
              url.hostname.endsWith('.checkout.stripe.com')
            )
          ) {
            throw new Error('Unexpected checkout destination.');
          }

          window.location.assign(url.href);
        } catch {
          this.busy.set(false);
          this.error.set(
            'The checkout destination was invalid.'
          );
        }
      },
      error: () => {
        this.busy.set(false);
        this.error.set(
          'Unable to start checkout. Please try again.'
        );
      },
    });
  }
}
</CodeBlock>

The hostname check prevents a compromised or misconfigured API response from redirecting students to an arbitrary domain. If your provider uses another documented checkout host, update the allowlist deliberately.

### 14.3 Success page

The success page should say **payment confirmation is pending** until LearnHub's backend reports a verified paid state.

For that, add a student-scoped payment status endpoint.

**Create:** `src/main/java/com/learnhub/payments/PaymentStatusResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

public record PaymentStatusResponse(
        long paymentAttemptId,
        long courseId,
        PaymentStatus status,
        boolean accessGranted
) {}
</CodeBlock>

Add this read method to a new service:

**Create:** `src/main/java/com/learnhub/payments/MyPaymentService.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import com.learnhub.security.CurrentActor;
import com.learnhub.persistence.JpaEnrollmentRepository;

import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class MyPaymentService {

    private final CurrentActor actor;
    private final JpaPaymentAttemptRepository payments;
    private final JpaEnrollmentRepository enrollments;

    public MyPaymentService(
            CurrentActor actor,
            JpaPaymentAttemptRepository payments,
            JpaEnrollmentRepository enrollments
    ) {
        this.actor = actor;
        this.payments = payments;
        this.enrollments = enrollments;
    }

    @PreAuthorize("hasRole('STUDENT')")
    @Transactional(readOnly = true)
    public PaymentStatusResponse status(long attemptId) {
        var payment = payments.findById(attemptId)
                .orElseThrow(() ->
                        new PaymentNotFoundException()
                );

        if (!payment.getStudentUserId()                .equals(actor.id())) {
            throw new PaymentNotFoundException();
        }

        boolean access = enrollments
                .findByStudent_IdAndCourse_IdAndStatus(
                        actor.id(),
                        payment.getCourseId(),
                        "ACTIVE"
                )
                .isPresent();

        return new PaymentStatusResponse(
                payment.getId(),
                payment.getCourseId(),
                payment.getStatus(),
                access
        );
    }
}
</CodeBlock>

**Create:** `src/main/java/com/learnhub/payments/PaymentNotFoundException.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

public final class PaymentNotFoundException
        extends RuntimeException {

    public PaymentNotFoundException() {
        super("Payment not found.");
    }
}
</CodeBlock>

Map this exception to `404` in the existing error handler.

**Create:** `src/main/java/com/learnhub/web/MyPaymentController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.payments.MyPaymentService;
import com.learnhub.payments.PaymentStatusResponse;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/me/payments")
public class MyPaymentController {

    private final MyPaymentService service;

    public MyPaymentController(MyPaymentService service) {
        this.service = service;
    }

    @GetMapping("/{attemptId}")
    public PaymentStatusResponse status(
            @PathVariable long attemptId
    ) {
        return service.status(attemptId);
    }
}
</CodeBlock>

Add to `SecurityFilterChain`:

```java
.requestMatchers(
        HttpMethod.GET,
        "/api/v1/me/payments/*"
).hasRole("STUDENT")
```

### Important browser flow detail

The success URL contains Stripe's Checkout Session ID, not LearnHub's payment attempt ID.

For a complete success-page experience, add a **student-scoped lookup by Checkout Session ID**, or store the attempt ID in temporary in-memory navigation state before redirecting.

Do not put a Stripe secret or trusted payment claim in browser storage.

A student-scoped lookup can use the unique `provider_checkout_session_id` column, but it must still check that the session belongs to the authenticated student.

---

## 15. Integration testing strategy

The payment subsystem needs tests at three distinct boundaries.

| Boundary | What to test |
|---|---|
| Signature verification | Real HMAC, altered payload, stale timestamp, missing header |
| PostgreSQL fulfillment | Payment transitions, duplicate events, enrollment uniqueness, refunds |
| HTTP security | Student checkout authorization, CSRF, private status endpoints, webhook bypass |

We'll use a fake checkout provider in tests so no real payment API is called.

### 15.1 HMAC signature unit test

**Create:** `src/test/java/com/learnhub/payments/StripeWebhookVerifierTest.java`

<CodeBlock language="java" editable>
package com.learnhub.payments;

import org.junit.jupiter.api.Test;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.HexFormat;

import static org.junit.jupiter.api.Assertions.*;

class StripeWebhookVerifierTest {

    private final String secret = "whsec_test_secret";

    private final StripeWebhookVerifier verifier =
            new StripeWebhookVerifier(
                    new PaymentProperties(
                            new PaymentProperties.Stripe(
                                    "sk_test_unused",
                                    secret,
                                    "https://api.stripe.com"
                            ),
                            "http://localhost:4200"
                    )
            );

    @Test
    void acceptsCorrectSignature() throws Exception {
        byte[] payload = """
            {"id":"evt_test","type":"ping"}
            """.getBytes(StandardCharsets.UTF_8);

        String header = sign(
                payload,
                Instant.now().getEpochSecond()
        );

        assertDoesNotThrow(
                () -> verifier.verify(payload, header)
        );
    }

    @Test
    void rejectsAlteredPayload() throws Exception {
        byte[] original =
                "{\"id\":\"evt_test\"}".getBytes(
                        StandardCharsets.UTF_8
                );

        byte[] altered =
                "{\"id\":\"evt_other\"}".getBytes(
                        StandardCharsets.UTF_8
                );

        String header = sign(
                original,
                Instant.now().getEpochSecond()
        );

        assertThrows(
                InvalidWebhookSignatureException.class,
                () -> verifier.verify(altered, header)
        );
    }

    @Test
    void rejectsOldTimestamp() throws Exception {
        byte[] payload = "{}".getBytes(
                StandardCharsets.UTF_8
        );

        String header = sign(
                payload,
                Instant.now().minusSeconds(600)
                        .getEpochSecond()
        );

        assertThrows(
                InvalidWebhookSignatureException.class,
                () -> verifier.verify(payload, header)
        );
    }

    private String sign(
            byte[] payload,
            long timestamp
    ) throws Exception {
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(
                secret.getBytes(StandardCharsets.UTF_8),
                "HmacSHA256"
        ));

        mac.update(
                (timestamp + ".").getBytes(
                        StandardCharsets.UTF_8
                )
        );

        String hex = HexFormat.of().formatHex(
                mac.doFinal(payload)
        );

        return "t=" + timestamp + ",v1=" + hex;
    }
}
</CodeBlock>

### 15.2 PostgreSQL webhook integration test

**Create:** `src/test/java/com/learnhub/payments/PaymentFulfillmentIntegrationTest.java`

This test uses the real `StripeWebhookProcessor`, `JdbcTemplate`, and PostgreSQL Testcontainers from the earlier lessons.

<CodeBlock language="java" editable>
package com.learnhub.payments;

import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.transaction.annotation.Transactional;

import java.nio.charset.StandardCharsets;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Transactional
class PaymentFulfillmentIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired JdbcTemplate jdbc;
    @Autowired StripeWebhookProcessor processor;

    private long user() {
        return jdbc.queryForObject("""
            INSERT INTO app_users (
                email, display_name, role, enabled
            )
            VALUES (
                'payment-test-student@example.test',
                'Payment Student',
                'STUDENT',
                true
            )
            RETURNING id
            """,
            Long.class
        );
    }

    private long course() {
        return jdbc.queryForObject("""
            INSERT INTO courses (
                title, description, instructor,
                category, price, is_free,
                level, rating, status
            )
            VALUES (
                'Paid Test Course',
                'A paid course',
                'Test Instructor',
                'Development',
                49.99,
                false,
                'BEGINNER',
                0.0,
                'PUBLISHED'
            )
            RETURNING id
            """,
            Long.class
        );
    }

    @Test
    void verifiedPaymentCreatesOneEnrollment() {
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
              "id": "evt_lesson20_paid_1",
              "type": "checkout.session.completed",
              "data": {
                "object": {
                  "id": "cs_test_lesson20",
                  "client_reference_id": "%d",
                  "metadata": {"attempt_id": "%d"},
                  "payment_status": "paid",
                  "payment_intent": "pi_test_lesson20",
                  "amount_total": 4999,
                  "currency": "usd"
                }
              }
            }
            """.formatted(attemptId, attemptId);

        byte[] body = payload.getBytes(
                StandardCharsets.UTF_8
        );

        processor.process(body);
        processor.process(body);

        Integer enrollments = jdbc.queryForObject("""
            SELECT COUNT(*)
            FROM enrollments
            WHERE student_user_id = ?
              AND course_id = ?
              AND status = 'ACTIVE'
            """,
            Integer.class,
            studentId,
            courseId
        );

        assertEquals(1, enrollments);

        String status = jdbc.queryForObject("""
            SELECT status
            FROM payment_attempts
            WHERE id = ?
            """,
            String.class,
            attemptId
        );

        assertEquals("PAID", status);

        Integer eventCount = jdbc.queryForObject("""
            SELECT COUNT(*)
            FROM payment_webhook_events
            WHERE provider_event_id = ?
            """,
            Integer.class,
            "evt_lesson20_paid_1"
        );

        assertEquals(1, eventCount);
    }
}
</CodeBlock>

**Fixture note:** This uses the Lesson 19 publication-date default/backfill assumptions. Because the Lesson 19 migration added a check requiring `published_at` for published courses, the test fixture must explicitly supply `published_at`:

```sql
-- In the course() fixture's INSERT:
-- Add published_at to the column list
-- and CURRENT_TIMESTAMP to the VALUES list.
```

Do this before running the test. Otherwise PostgreSQL correctly rejects the fixture.

### 15.3 MockMvc security tests

Create `src/test/java/com/learnhub/web/PaymentSecurityIntegrationTest.java`, using the established Lesson 14–19 MockMvc and `LearnHubPrincipal` fixtures.

At minimum, implement these assertions:

| Request | Expected |
|---|---|
| Anonymous POST checkout with valid CSRF | `401` |
| Student POST checkout without CSRF | `403` |
| Instructor POST checkout with CSRF | `403` |
| Student POST checkout with CSRF | Authorized; fake provider returns checkout URL |
| Anonymous POST webhook with valid signature | `200` |
| Anonymous POST webhook with invalid signature | `400` |
| Student A GET Student B payment attempt | `404` |
| Duplicate webhook delivery | `200`, no duplicate enrollment |
| Webhook with mismatched amount | No enrollment, non-2xx response |
| Checkout success redirect only | No enrollment created |

### 15.4 Fake provider

For the MockMvc checkout test, replace the real provider with a deterministic fake:

<CodeBlock language="java" editable>
@TestConfiguration
public class FakeCheckoutConfiguration {

    @Bean
    @Primary
    CheckoutProvider fakeCheckoutProvider() {
        return (attemptId, amount, currency, title) ->
                new CheckoutProvider.CheckoutSession(
                        "cs_test_" + attemptId,
                        "https://checkout.stripe.com/test-session"
                );
    }
}
</CodeBlock>

Import this configuration into the relevant test class with `@Import(FakeCheckoutConfiguration.class)`.

This fake must never be active in production.

---

## 16. Run the payment flow locally

### Terminal A — Spring Boot

```powershell
cd $HOME\angular-learning\learnhub-backend

$env:STRIPE_SECRET_KEY = "sk_test_REPLACE_WITH_YOUR_KEY"
$env:STRIPE_WEBHOOK_SECRET = "whsec_REPLACE_WITH_YOUR_WEBHOOK_SECRET"

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

### Terminal B — Angular

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd serve --proxy-config proxy.conf.json
```

### Terminal C — Stripe CLI

Install the official Stripe CLI, authenticate it with a test account, and forward webhook events:

```powershell
stripe login

stripe listen --forward-to `
  http://localhost:8080/api/v1/webhooks/stripe
```

The CLI prints a webhook signing secret. Copy that `whsec_...` value into `STRIPE_WEBHOOK_SECRET` for the Spring Boot process and restart the backend.

The CLI's local forwarding secret is not necessarily the same as a webhook endpoint secret configured in the Stripe Dashboard.

### Test-mode checkout

Use a published paid course and a signed-in student account. Initiate checkout through the Angular button.

On the Stripe-hosted page, use Stripe's documented test card details for a successful test payment. Do not use real card details while learning.

After checkout:

1. Verify the browser returns to `/checkout/success`.
2. Verify Stripe CLI reports a delivered webhook.
3. Verify `payment_attempts.status` becomes `PAID`.
4. Verify exactly one `ACTIVE` enrollment exists.
5. Repeat the webhook delivery and verify no duplicate enrollment appears.
6. Verify the student can open the course player.

### Database inspection

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT id, course_id, status, amount_minor, currency, refunded_minor FROM payment_attempts ORDER BY id DESC LIMIT 20;"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT provider_event_id, event_type, processed_at FROM payment_webhook_events ORDER BY id DESC LIMIT 20;"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT student_user_id, course_id, status FROM enrollments ORDER BY id DESC LIMIT 20;"
```

Do not print or commit payment-provider secrets, full webhook payloads, or sensitive customer information.

---

## 17. Critical hardening before accepting real money

The reference implementation teaches the core architecture, but there are several important integration and production issues to resolve.

### A. Checkout failure handling

If the provider creates a session but the database attachment fails, the session may still be payable.

The event handler's attempt-ID metadata lets it associate a verified event with the original attempt. Add reconciliation for pending attempts with missing session IDs, and ensure provider retries reuse the same idempotency key.

### B. Refund and entitlement provenance

The existing enrollment table does not record whether access came from payment, a promotion, or an administrator.

Before enabling real refunds, add an explicit entitlement-grant model or equivalent provenance. Refund revocation must not remove access granted by another valid source.

### C. Payment event ordering

The simple state machine treats `FAILED` and `EXPIRED` as terminal. Real providers may deliver delayed or out-of-order events. Define transitions using authoritative provider state, and reconcile ambiguous cases rather than assuming delivery order.

### D. Enrollment and refund concurrency

Use one consistent student/course lock for all entitlement grants and revocations. A count of other paid attempts alone is not sufficient under concurrent transactions.

### E. Amount and currency validation

The verified Checkout Session must match the payment attempt's amount and currency. The refund event must also match the original payment. Do not use the current course price to validate an old purchase.

### F. Fulfillment transaction boundaries

The payment transition, enrollment grant, and webhook inbox update must commit together. A failure must roll back all three.

### G. Payment-provider error responses

Do not expose raw provider errors to the frontend. Map expected failures to stable application errors such as:

| Situation | Suggested response |
|---|---|
| Invalid checkout request | `400` |
| Student already enrolled | `409` |
| Course unavailable | `404` |
| Provider temporarily unavailable | `503` |
| Invalid webhook signature | `400` |
| Webhook processing temporarily fails | `500` to permit retry |

Add specific exception classes and handlers rather than allowing generic `IllegalArgumentException` or `IllegalStateException` to become uncontrolled responses.

### H. Test fixture correction

For the Lesson 19 schema, use this complete published-course fixture SQL in the payment integration test:

<CodeBlock language="sql" editable>
INSERT INTO courses (
    title,
    description,
    instructor,
    category,
    price,
    is_free,
    level,
    rating,
    status,
    published_at
)
VALUES (
    'Paid Test Course',
    'A paid course',
    'Test Instructor',
    'Development',
    49.99,
    false,
    'BEGINNER',
    0.0,
    'PUBLISHED',
    CURRENT_TIMESTAMP
)
RETURNING id;
</CodeBlock>

### I. CSRF and session security

Only the exact server-to-server webhook route is exempt from CSRF. The browser checkout endpoint must remain CSRF-protected.

Keep session cookies HttpOnly and appropriately Secure/SameSite in production, and serve the entire application over HTTPS.

### J. Missing paid-course access policy

Lesson 17 intentionally rejected direct enrollment into paid courses. Preserve that restriction. The payment fulfillment service is the only new trusted route that grants paid access.

---

## 18. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="shield-check" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise A — Forged payment webhook**

      Modify the signed payload's amount from `4999` to `1` without recalculating the signature. Verify rejection. Then sign a valid test payload containing the wrong amount and verify fulfillment still refuses it.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="refresh-cw" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise B — Duplicate and concurrent webhooks**

      Process the same event twice, then from two separate transactions. Verify one inbox record, one successful payment transition, and one enrollment.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="database" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise C — Payment provenance**

      Design a `course_entitlements` table linking access to the granting payment attempt. Explain how refunds, promotions, manual grants, and repurchases coexist.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="credit-card" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise D — Payment status page**

      Implement an authenticated student-scoped lookup by Checkout Session ID. Build an Angular success page that shows pending, paid, failed, and expired states without trusting the redirect.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="git-compare" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise E — Refund reconciliation**

      Simulate two successful purchases for the same course, refund one, and prove access remains. Refund both and verify the entitlement policy is applied exactly once.
    </box>
  </row>
</box>

## 19. Knowledge quiz

{@body const questions=[{q:"What is sufficient proof to grant a paid enrollment?",a:["The browser visits /checkout/success","A verified provider payment event matching the stored attempt","Angular sends paymentSucceeded: true"],correct:1,why:"Only trusted server-side confirmation can establish that a payment succeeded."},{q:"Why store amount_minor in payment_attempts?",a:["To snapshot the authorized checkout amount","To calculate Angular CSS styles","To avoid storing the course ID"],correct:0,why:"A payment attempt must retain its original price even when the course price later changes."},{q:"What protects against duplicate provider event delivery?",a:["Only a frontend loading spinner","A unique provider event ID plus transactional, idempotent fulfillment","Disabling retries"],correct:1,why:"The database inbox and idempotent payment transition handle retries safely."},{q:"Why is the webhook endpoint exempt from browser CSRF?",a:["Because all POST routes should be exempt","Because Stripe uses signed server-to-server requests instead of browser sessions","Because it has no security requirements"],correct:1,why:"Webhook signature verification is its authentication mechanism; browser checkout still requires CSRF."},{q:"Why is refund revocation more complicated than changing enrollment to CANCELLED?",a:["A student may have another valid payment or entitlement grant","PostgreSQL cannot update enrollment status","Refunds always occur before payments"],correct:0,why:"Revocation must preserve access granted by other valid sources."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Lesson 20 — Payment security**
    <text color="secondary" size="xs">5 questions</text>
  </row>
  {#each questions as q,i}
    <box key={i} gap={2}>
      **{i+1}. {q.q}**
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setGraded(false);}}>
        {#each q.a as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if graded}
        <row gap={2} align="start">
          <icon name={answers[i]===q.correct?"check-circle":"info"} color={answers[i]===q.correct?"success":"secondary"}/>
          <box flex="1" gap={1}>
            <text size="sm" weight="medium">{answers[i]===q.correct?"Correct":"Correct: "+q.a[q.correct]}</text>
            <text size="sm" color="secondary">{q.why}</text>
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

## 20. Completion checklist

{@body const groups=[{name:"Database and payment domain",items:["Verified Lesson 19 baseline","Selected next unused Flyway version","Created payment_attempts table","Created verified webhook inbox","Implemented payment entity and repository","Implemented amount conversion","Implemented provider interface and Stripe adapter"]},{name:"Security and fulfillment",items:["Implemented server-authoritative checkout","Implemented short transaction boundaries","Implemented webhook signature verification","Implemented idempotent event claim","Implemented verified paid fulfillment","Implemented refund-state processing","Reviewed entitlement provenance and concurrency","Configured exact webhook CSRF exemption","Protected student checkout and payment status APIs"]},{name:"Angular",items:["Created typed checkout API service","Built standalone checkout button","Used provider-hosted checkout","Preserved session and CSRF flow","Designed success-page status confirmation","Verified paid-course navigation"]},{name:"Testing and delivery",items:["Added signature verification tests","Added PostgreSQL fulfillment tests","Tested duplicate webhook delivery","Tested amount and currency mismatch","Tested anonymous and student access","Tested refund scenarios","Ran Maven clean verify","Ran Angular tests and build","Completed two exercises","Completed quiz","Reviewed staged changes and committed"]}]}
{@body const [checks,setChecks]=DIL.useState({})}
{@body const total=groups.reduce((n,g)=>n+g.items.length,0)}
{@body const finished=Object.values(checks).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 20 progress**
      <text color="secondary" size="xs">Mark items only after local verification.</text>
    </box>
    <title size="lg" tabularNums>{finished}/{total}</title>
  </row>
  <box background="surface-tertiary" radius="full" height="8px" clip>
    <box background="rgba(109,40,217,0.85)" radius="full" height="100%" width={`${finished/total*100}%`}/>
  </box>
  {#each groups as group,i}
    <box key={i} gap={2}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!checks[`${i}-${j}`]} onChange={v=>setChecks(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if finished===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 20 checklist completed.**
    </row>
  {/if}
</box>

## 21. Final verification and Git checkpoints

### Backend

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=StripeWebhookVerifierTest,PaymentFulfillmentIntegrationTest,PaymentSecurityIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Payment tests failed."
}

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend verification failed."
}

git diff --check
git status
git diff --stat
```

The named `PaymentSecurityIntegrationTest` must be implemented from the test matrix before running that exact command.

### Angular

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

git diff --check
git status
git diff --stat
```

### Git checkpoints

```powershell
cd $HOME\angular-learning\learnhub-backend

git add src\main src\test
git diff --cached --stat
git commit -m "Add verified checkout and idempotent paid enrollment"
```

```powershell
cd $HOME\angular-learning\learnhub

git add src\app
git diff --cached --stat
git commit -m "Integrate hosted checkout into LearnHub"
```

Review staged changes carefully. Do not commit `.env` files containing real keys, webhook secrets, or payment-provider credentials.

The examples are implementation guidance, not executed changes. In particular, the refund entitlement model, provider reconciliation, exception mapping, and full MockMvc security test class must be completed and verified before processing real money.

## 22. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="test-tube-diagonal" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 21 — Comprehensive Backend Testing**

    We'll build a testing pyramid covering Java unit tests, Mockito, Spring MVC slices, PostgreSQL Testcontainers, transactional integration tests, security tests, payment webhook tests, and Angular-to-backend contract verification.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 21: comprehensive testing strategy, JUnit 5, Mockito, AssertJ, Spring Boot MockMvc and @WebMvcTest, PostgreSQL Testcontainers, Flyway migrations in tests, Spring Security session and CSRF tests, course ownership and enrollment tests, payment webhook signature and idempotency tests, concurrency and transaction rollback tests, Angular HttpTestingController and contract testing, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 20 with experienced-Java pacing.")}>Start Lesson 21 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 20 deliverable:** A secure paid-course checkout architecture with server-authoritative prices, signed webhook verification, transactional payment fulfillment, database idempotency, student-scoped payment status, and Angular hosted-checkout integration.

The central principle is: **payment creates evidence; verified backend fulfillment creates access.**