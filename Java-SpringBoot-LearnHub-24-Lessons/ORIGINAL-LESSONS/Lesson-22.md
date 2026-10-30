# LearnHub Java & Spring Boot — Lesson 22
<text color="secondary" size="sm">OpenAPI 3 · Swagger UI · Spring Boot Actuator · Micrometer · Structured Logging · Correlation IDs · Tracing · API Contract Testing</text>

**Level:** Experienced Java developer  
**Estimated duration:** 10–14 hours  
**Starting point:** Lesson 21's tested LearnHub backend and Angular standalone application  
**Target stack:** Java 21, Spring Boot 3.5.x, Spring Security 6, PostgreSQL, Flyway, Angular standalone

## 1. What we're building

LearnHub now has public catalog endpoints, instructor-owned course management, authenticated sessions, student enrollments, a course player, and payment processing.

The next challenge is answering operational questions:

- What endpoints exist, and what do they return?
- Is the application healthy and ready to accept traffic?
- Why did a request fail?
- How many checkout attempts are succeeding?
- Which request triggered a database error?
- Can we diagnose a failure without exposing session cookies, passwords, or payment secrets?

By the end of Lesson 22, you'll have a documented API, observable request flow, health probes, application metrics, and regression tests for the operational contract.

### Architecture

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    **Angular standalone frontend**
    <text size="xs" color="secondary">Same-origin /api/v1 requests · session cookie · CSRF</text>
  </box>
  <row justify="center" align="center" gap={2}>
    <icon name="arrow-down" color="secondary"/>
    <text size="xs" color="secondary">X-Request-ID</text>
  </row>
  <box border radius="lg" padding={3} gap={2}>
    <box align="center" gap={1}>
      **Spring Boot application**
      <text size="xs" color="secondary">Security filters · MVC · services · JPA</text>
    </box>
    <grid columns={2} gap={2}>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2} align="center" gap={1}>
          <icon name="book-open" color="secondary"/>
          **OpenAPI**
          <text size="2xs" color="secondary">API contract</text>
        </box>
      </grid-item>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2} align="center" gap={1}>
          <icon name="heart-pulse" color="secondary"/>
          **Actuator**
          <text size="2xs" color="secondary">Health/readiness</text>
        </box>
      </grid-item>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2} align="center" gap={1}>
          <icon name="chart-no-axes-column" color="secondary"/>
          **Micrometer**
          <text size="2xs" color="secondary">Metrics</text>
        </box>
      </grid-item>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2} align="center" gap={1}>
          <icon name="logs" color="secondary"/>
          **JSON logs + MDC**
          <text size="2xs" color="secondary">Request diagnostics</text>
        </box>
      </grid-item>
    </grid>
  </box>
  <row justify="center">
    <icon name="arrow-down" color="secondary"/>
  </row>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    **PostgreSQL · payment provider**
    <text size="xs" color="secondary">External dependencies, measured without logging secrets</text>
  </box>
</box>

The key distinction is:

**Documentation describes the contract. Logs explain individual events. Metrics reveal patterns. Traces connect work across components. Health probes tell infrastructure whether the application can serve traffic.**

None of these replaces automated tests.

---

## 2. First: reconcile the Lesson 20–21 baseline

Before adding dependencies, establish a clean baseline.

### PowerShell

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

if ($LASTEXITCODE -ne 0) {
    throw "Fix the Lesson 21 baseline before Lesson 22."
}
```

This course has evolved incrementally. Some earlier examples introduced overlapping controller methods, repository signatures, or migrations. Keep the working implementations already in your repository; don't paste an older class over a newer one.

No new Flyway migration is needed for Lesson 22. Observability should not modify your domain schema.

---

## 3. Maven dependencies

**File:** `learnhub-backend/pom.xml`

Confirm the Actuator starter is present and add OpenAPI and Prometheus support.

For Spring Boot **3.5.x**, use the springdoc **2.x** starter family. The example below pins `2.8.14`; verify the selected release against your project before upgrading dependencies.

<CodeBlock language="xml" editable>
<!-- Add inside your existing <dependencies> element. -->

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.14</version>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<!-- Explicit JSON console logging support for Logback. -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>8.1</version>
</dependency>
</CodeBlock>

Do not duplicate `spring-boot-starter-actuator` if Lesson 5 already added it.

The Spring Boot parent manages the Micrometer dependency versions. The springdoc and Logstash encoder versions are explicitly pinned.

### Verify dependency resolution

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn dependency:tree `
  "-Dincludes=org.springdoc:*,io.micrometer:*,net.logstash.logback:*"

mvn clean test
```

If your dependency policy requires a different approved springdoc or Logstash encoder version, use that version consistently across local development and CI.

---

## 4. OpenAPI 3: document the real contract

OpenAPI describes operations, parameters, request bodies, response schemas, security requirements, and error responses.

Swagger UI renders the OpenAPI document into a browsable interface.

For LearnHub, the distinction between **cookie authentication** and **CSRF protection** matters. The browser authenticates with `JSESSIONID`; unsafe requests additionally require `X-XSRF-TOKEN`.

### 4.1 OpenAPI configuration

**Create:** `src/main/java/com/learnhub/config/OpenApiConfig.java`

<CodeBlock language="java" editable>
package com.learnhub.config;

import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.security.SecurityScheme;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Bean
    OpenAPI learnHubOpenApi() {
        return new OpenAPI()
                .info(new Info()
                        .title("LearnHub REST API")
                        .version("v1")
                        .description("""
                            Course marketplace API.

                            Browser authentication uses a server-side
                            session cookie. Unsafe browser requests
                            also require a CSRF token.

                            A successful checkout redirect does not
                            prove payment. Access is granted only
                            after verified server-side fulfillment.
                            """))
                .components(new Components()
                        .addSecuritySchemes(
                                "sessionCookie",
                                new SecurityScheme()
                                        .type(
                                                SecurityScheme.Type.APIKEY
                                        )
                                        .in(
                                                SecurityScheme.In.COOKIE
                                        )
                                        .name("JSESSIONID")
                        )
                        .addSecuritySchemes(
                                "csrfHeader",
                                new SecurityScheme()
                                        .type(
                                                SecurityScheme.Type.APIKEY
                                        )
                                        .in(
                                                SecurityScheme.In.HEADER
                                        )
                                        .name("X-XSRF-TOKEN")
                        ));
    }
}
</CodeBlock>

The OpenAPI schemes are descriptive. Swagger UI's browser requests still obey cookie and CSRF rules; declaring a security scheme does not automatically perform login or retrieve tokens.

### 4.2 Document a public catalog operation

Instead of rewriting your existing `CourseController`, annotate its current catalog method.

**File:** `src/main/java/com/learnhub/web/CourseController.java`

Add imports:

<CodeBlock language="java" editable>
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
</CodeBlock>

At the controller class:

```java
@Tag(
    name = "Courses",
    description = "Public catalog and course discovery"
)
```

Above the existing GET catalog method:

<CodeBlock language="java" editable>
@Operation(
    summary = "Search published courses",
    description = """
        Returns published courses only.

        Supports search, category, free/paid filtering,
        level, sorting, and pagination.

        Draft and archived courses are never exposed
        through the public catalog.
        """
)
@ApiResponses({
    @ApiResponse(
        responseCode = "200",
        description = "Published course page"
    ),
    @ApiResponse(
        responseCode = "400",
        description = "Invalid search or pagination parameters"
    )
})
</CodeBlock>

Retain the actual controller method signature from Lessons 11–12.

### 4.3 Document the authenticated checkout endpoint

**File:** `src/main/java/com/learnhub/web/CheckoutController.java`

Add these annotations above its existing `create()` method:

<CodeBlock language="java" editable>
@Operation(
    summary = "Create a hosted checkout session",
    description = """
        The authenticated student provides only a course ID.

        LearnHub determines the student, price, and currency
        from trusted server-side state.

        This endpoint does not grant enrollment.
        """,
    security = {
        @SecurityRequirement(name = "sessionCookie"),
        @SecurityRequirement(name = "csrfHeader")
    }
)
@ApiResponses({
    @ApiResponse(
        responseCode = "200",
        description = "Hosted checkout session created"
    ),
    @ApiResponse(
        responseCode = "401",
        description = "Authentication required"
    ),
    @ApiResponse(
        responseCode = "403",
        description = "Role or CSRF requirement failed"
    ),
    @ApiResponse(
        responseCode = "404",
        description = "Course unavailable"
    ),
    @ApiResponse(
        responseCode = "409",
        description = "Student already has access"
    ),
    @ApiResponse(
        responseCode = "503",
        description = "Payment provider unavailable"
    )
})
</CodeBlock>

Imports:

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
```

The response codes above are the **target contract**. In Lesson 20, some domain failures still used generic exceptions. Implement the corresponding exception mappings before treating these response codes as guaranteed.

### 4.4 Document the webhook separately

A Stripe webhook is authenticated by its signature, not by a browser session.

Document:

- `Stripe-Signature` as a required header.
- The body as a provider event payload.
- `200` for accepted or already-processed events.
- `400` for invalid signatures.
- A retriable non-2xx response for transient processing failures.

Do not add `sessionCookie` or `csrfHeader` requirements to the webhook operation.

### 4.5 Document Problem Details

The existing Lesson 8 error handler returns RFC 9457-style `ProblemDetail` responses.

A documented example:

<CodeBlock language="json" editable>
{
  "type": "about:blank",
  "title": "Conflict",
  "status": 409,
  "detail": "Student already has course access.",
  "instance": "/api/v1/me/checkout/courses/42",
  "requestId": "f4d7c31e3b884bb8"
}
</CodeBlock>

The `requestId` extension will be added later in this lesson. Do not document internal SQL statements, exception stack traces, payment-provider credentials, or user password hashes.

---

## 5. Expose Swagger UI safely

Springdoc commonly exposes:

| Endpoint | Purpose |
|---|---|
| `/v3/api-docs` | OpenAPI JSON |
| `/v3/api-docs.yaml` | OpenAPI YAML |
| `/swagger-ui/index.html` | Interactive API documentation |

For LearnHub, we'll enable interactive documentation in the `dev` and `test` profiles, not on the public production surface.

### Configuration

**File:** `src/main/resources/application.yml`

Merge these settings:

<CodeBlock language="yaml" editable>
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
</CodeBlock>

**File:** `src/main/resources/application-dev.yml`

<CodeBlock language="yaml" editable>
springdoc:
  api-docs:
    enabled: true
  swagger-ui:
    enabled: true
    path: /swagger-ui.html
    operations-sorter: alpha
    tags-sorter: alpha
</CodeBlock>

**File:** `src/test/resources/application-test.yml`

Add:

```yaml
springdoc:
  api-docs:
    enabled: true
  swagger-ui:
    enabled: false
```

### Spring Security rules

Your Lesson 14 `SecurityConfig` ends with `.anyRequest().denyAll()`. Consequently, merely installing springdoc will not make its routes accessible.

Add these rules **before** `.anyRequest().denyAll()`:

<CodeBlock language="java" editable>
.requestMatchers(
        "/v3/api-docs",
        "/v3/api-docs/**",
        "/swagger-ui.html",
        "/swagger-ui/**"
).permitAll()
</CodeBlock>

This matcher is safe only when documentation is disabled in production as configured above. For stricter isolation, create a dedicated development-only security chain or restrict access at the ingress.

Do not globally disable Spring Security to make Swagger UI work.

### Run locally

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

Open:

`http://localhost:8080/swagger-ui/index.html`

Or inspect the JSON:

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:8080/v3/api-docs" |
  ConvertTo-Json -Depth 10 |
  Set-Content .\openapi-dev.json
```

`openapi-dev.json` is a generated contract artifact. Review it for unintended endpoints or internal schemas before publishing it.

---

## 6. Spring Boot Actuator: health, liveness, readiness

A process being alive is not the same as being ready.

**Liveness** answers whether the application process should continue running. **Readiness** answers whether it should receive new requests.

A database outage can make LearnHub unready without necessarily meaning the JVM should be restarted.

### 6.1 Actuator configuration

**File:** `src/main/resources/application.yml`

Merge this section:

<CodeBlock language="yaml" editable>
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: never

  endpoints:
    web:
      exposure:
        include: health,info,prometheus

  metrics:
    tags:
      application: learnhub-backend

  health:
    db:
      enabled: true
</CodeBlock>

By default, Actuator shares the application's port. For production, a separate management port or private ingress can reduce exposure.

### 6.2 Health endpoints

```powershell
Invoke-RestMethod http://localhost:8080/actuator/health

Invoke-RestMethod http://localhost:8080/actuator/health/liveness

Invoke-RestMethod http://localhost:8080/actuator/health/readiness
```

Typical responses:

<grid columns={2} gap={2}>
  <grid-item>
    <box border radius="lg" padding={3} gap={1}>
      <row align="center" gap={2}>
        <icon name="check-circle" color="success"/>
        **Healthy**
      </row>
      <CodeBlock language="json">{"status":"UP"}</CodeBlock>
    </box>
  </grid-item>
  <grid-item>
    <box border radius="lg" padding={3} gap={1}>
      <row align="center" gap={2}>
        <icon name="alert-circle" color="danger"/>
        **Unhealthy**
      </row>
      <CodeBlock language="json">{"status":"DOWN"}</CodeBlock>
    </box>
  </grid-item>
</grid>

### 6.3 Include PostgreSQL in readiness

The default readiness group may not include the database health indicator. Since LearnHub requires PostgreSQL for normal operation, explicitly include it.

Update the health configuration:

<CodeBlock language="yaml" editable>
management:
  endpoint:
    health:
      probes:
        enabled: true
      add-additional-paths: true
      # Additional main-port paths: /livez and /readyz
      show-details: never
      group:
        readiness:
          include: readinessState,db
</CodeBlock>

Merge this into the existing `management.endpoint.health` configuration rather than replacing the whole management section.

The readiness endpoint will now report `DOWN` when the database health check fails, assuming the `db` health contributor is registered.

**Avoid including Stripe in liveness.** A third-party payment outage should not cause an endless cycle of JVM restarts.

### 6.4 Protect operational endpoints

In `SecurityConfig`, add explicit rules:

<CodeBlock language="java" editable>
.requestMatchers(
        HttpMethod.GET,
        "/actuator/health",
        "/actuator/health/**",
        "/livez",
        "/readyz"
).permitAll()
</CodeBlock>

For `/actuator/prometheus`, prefer a private management network or an authenticated monitoring identity. Do not expose it publicly just because it is not customer data.

If you use a separate management port, secure that port through deployment networking and the applicable management security configuration.

---

## 7. Correlation IDs: connect a request to its logs

Imagine Angular displays:

> Checkout unavailable. Reference: `7f3a9c...`

That reference should let an operator find the corresponding backend logs without exposing a stack trace to the student.

We'll add an `X-Request-ID` response header and store the ID in Logback's Mapped Diagnostic Context (MDC).

### 7.1 Request ID filter

**Create:** `src/main/java/com/learnhub/observability/RequestCorrelationFilter.java`

<CodeBlock language="java" editable>
package com.learnhub.observability;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;
import java.util.regex.Pattern;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestCorrelationFilter
        extends OncePerRequestFilter {

    public static final String HEADER = "X-Request-ID";
    public static final String ATTRIBUTE = "learnhub.requestId";

    private static final Pattern SAFE_ID =
            Pattern.compile("[A-Za-z0-9_-]{8,64}");

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain
    ) throws ServletException, IOException {

        String supplied = request.getHeader(HEADER);

        String requestId =
                supplied != null
                && SAFE_ID.matcher(supplied).matches()
                        ? supplied
                        : UUID.randomUUID().toString();

        request.setAttribute(ATTRIBUTE, requestId);
        response.setHeader(HEADER, requestId);

        // Preserve any MDC values established by other filters.
        String previous = MDC.get("requestId");

        try {
            MDC.put("requestId", requestId);
            chain.doFilter(request, response);
        } finally {
            if (previous == null) {
                MDC.remove("requestId");
            } else {
                MDC.put("requestId", previous);
            }
        }
    }
}
</CodeBlock>

### Why validate incoming IDs?

A caller might submit a request ID containing newlines, control characters, or enormous strings. Blindly copying that into logs creates a log-injection and resource-exhaustion risk.

This implementation accepts a restricted format and generates a new ID otherwise.

For production systems behind a trusted gateway, a stronger pattern is to let the gateway create or overwrite request IDs. The value is a **diagnostic identifier, not an authentication token**.

### 7.2 Test the filter

**Create:** `src/test/java/com/learnhub/observability/RequestCorrelationFilterTest.java`

<CodeBlock language="java" editable>
package com.learnhub.observability;

import jakarta.servlet.FilterChain;

import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.slf4j.MDC;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class RequestCorrelationFilterTest {

    private final RequestCorrelationFilter filter =
            new RequestCorrelationFilter();

    @Test
    void propagatesValidRequestIdAndCleansMdc()
            throws Exception {

        var request = new MockHttpServletRequest();
        var response = new MockHttpServletResponse();

        request.addHeader(
                "X-Request-ID",
                "lesson22-request-123"
        );

        FilterChain chain = (req, res) ->
                assertThat(MDC.get("requestId"))
                        .isEqualTo("lesson22-request-123");

        filter.doFilter(request, response, chain);

        assertThat(response.getHeader("X-Request-ID"))
                .isEqualTo("lesson22-request-123");

        assertThat(MDC.get("requestId")).isNull();
    }

    @Test
    void replacesInvalidRequestId() throws Exception {
        var request = new MockHttpServletRequest();
        var response = new MockHttpServletResponse();

        request.addHeader(
                "X-Request-ID",
                "bad\ninjected"
        );

        filter.doFilter(
                request,
                response,
                mock(FilterChain.class)
        );

        assertThat(response.getHeader("X-Request-ID"))
                .isNotEqualTo("bad\ninjected")
                .isNotBlank();
    }
}
</CodeBlock>

This filter runs early, including for requests rejected by Spring Security. That makes the request ID useful for `401` and `403` responses too.

---

## 8. Structured JSON logging

Plain-text logs are useful for development. Structured logs are easier to search and aggregate across multiple application instances.

A typical production log entry might look like:

<CodeBlock language="json" editable>
{
  "@timestamp": "2026-10-28T14:30:00Z",
  "level": "INFO",
  "logger_name": "com.learnhub.payments.CheckoutService",
  "message": "Checkout session created",
  "requestId": "lesson22-request-123",
  "courseId": 42,
  "paymentAttemptId": 99,
  "service": "learnhub-backend"
}
</CodeBlock>

Notice what's absent: passwords, session cookies, payment-provider secrets, card details, and raw webhook payloads.

### 8.1 Production Logback configuration

**Create:** `src/main/resources/logback-spring.xml`

<CodeBlock language="xml" editable>
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <springProperty
        scope="context"
        name="appName"
        source="spring.application.name"
        defaultValue="learnhub-backend"/>

    <appender name="CONSOLE_JSON"
              class="ch.qos.logback.core.ConsoleAppender">
        <encoder
            class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>
                {"service":"${appName}"}
            </customFields>
        </encoder>
    </appender>

    <appender name="CONSOLE_TEXT"
              class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{HH:mm:ss.SSS} %-5level
                [%X{requestId:-none}]
                %logger{36} - %msg%n
            </pattern>
        </encoder>
    </appender>

    <springProfile name="dev,test">
        <root level="INFO">
            <appender-ref ref="CONSOLE_TEXT"/>
        </root>
    </springProfile>

    <springProfile name="!dev &amp; !test">
        <root level="INFO">
            <appender-ref ref="CONSOLE_JSON"/>
        </root>
    </springProfile>

</configuration>
</CodeBlock>

The encoder includes MDC fields such as `requestId` automatically.

In a real deployment, test that your log collector parses these JSON lines as expected.

### 8.2 Log meaningful state changes

In `CheckoutService`, add:

```java
private static final org.slf4j.Logger log =
        org.slf4j.LoggerFactory.getLogger(
                CheckoutService.class
        );
```

After successfully attaching the Checkout Session ID:

<CodeBlock language="java" editable>
log.info(
        "Checkout session created: attemptId={}, courseId={}",
        attemptId,
        courseId
);
</CodeBlock>

In the webhook processor, after a successful transaction, log a provider event ID and a safe event type—not the raw payload.

**Be careful with transaction timing:** A log line emitted inside a transaction can say an operation succeeded even if the transaction subsequently rolls back. For audit-grade events, use a transactional outbox or emit a post-commit event.

### Logging rules

| Safe to log | Do not log |
|---|---|
| Request ID | `JSESSIONID` |
| Course ID | CSRF token |
| Payment attempt ID | Stripe secret key |
| Provider event ID | Webhook signing secret |
| Event type | Full Authorization header |
| HTTP status | Raw request body |
| Duration | Password or password hash |

Email addresses and student IDs are personal information; avoid putting them into general-purpose operational logs unless there's a justified, controlled need.

---

## 9. Micrometer: application and business metrics

Logs answer, “What happened to this request?”

Metrics answer, “How frequently is this happening, and is the pattern changing?”

Spring Boot Actuator already instruments HTTP server requests. We'll add metrics for checkout creation and webhook fulfillment.

### 9.1 Design metrics with low-cardinality labels

A good metric:

```text
learnhub.checkout.attempts{outcome="created"}
```

A dangerous metric:

```text
learnhub.checkout.attempts{studentId="1234567"}
```

Do not use user IDs, course IDs, request IDs, checkout session IDs, or provider event IDs as metric tags. They can create millions of unique time series.

### 9.2 Payment metrics component

**Create:** `src/main/java/com/learnhub/observability/PaymentMetrics.java`

<CodeBlock language="java" editable>
package com.learnhub.observability;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;

import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class PaymentMetrics {

    private final Counter checkoutCreated;
    private final Counter checkoutRejected;
    private final Counter webhookProcessed;
    private final Counter webhookDuplicate;
    private final Timer checkoutProviderLatency;

    public PaymentMetrics(MeterRegistry registry) {
        checkoutCreated = Counter.builder(
                "learnhub.checkout.attempts"
        ).tag("outcome", "created")
         .register(registry);

        checkoutRejected = Counter.builder(
                "learnhub.checkout.attempts"
        ).tag("outcome", "rejected")
         .register(registry);

        webhookProcessed = Counter.builder(
                "learnhub.webhook.events"
        ).tag("outcome", "processed")
         .register(registry);

        webhookDuplicate = Counter.builder(
                "learnhub.webhook.events"
        ).tag("outcome", "duplicate")
         .register(registry);

        checkoutProviderLatency = Timer.builder(
                "learnhub.checkout.provider.duration"
        ).description(
                "Latency of checkout provider calls"
        ).register(registry);
    }

    public void checkoutCreated() {
        checkoutCreated.increment();
    }

    public void checkoutRejected() {
        checkoutRejected.increment();
    }

    public void webhookProcessed() {
        webhookProcessed.increment();
    }

    public void webhookDuplicate() {
        webhookDuplicate.increment();
    }

    public void recordProviderLatency(long elapsedNanos) {
        checkoutProviderLatency.record(
                elapsedNanos,
                TimeUnit.NANOSECONDS
        );
    }
}
</CodeBlock>

### 9.3 Instrument the external provider call

In `CheckoutService`, inject `PaymentMetrics` as an additional constructor dependency.

Around the existing provider call:

<CodeBlock language="java" editable>
long startNanos = System.nanoTime();

try {
    var session = provider.createCheckout(
            attemptId,
            amountMinor,
            "usd",
            course.getTitle()    );

    attempts.attach(attemptId, session.sessionId());

    metrics.checkoutCreated();

    return new CheckoutResponse(
            attemptId,
            session.checkoutUrl()
    );

} catch (RuntimeException exception) {
    metrics.checkoutRejected();
    throw exception;

} finally {
    metrics.recordProviderLatency(
            System.nanoTime() - startNanos
    );
}
</CodeBlock>

This replaces the corresponding provider-call-and-return section of Lesson 20; don't leave both implementations in the method.

**Metric semantics:** `checkoutCreated` means the provider session was created and attached successfully. `checkoutRejected` means this execution failed during the measured portion. It does not necessarily mean Stripe failed or that no payable session exists.

A production metric should distinguish `provider_error`, `database_attach_error`, and `validation_rejection` without introducing sensitive tags.

### 9.4 Transaction-aware webhook metrics

Do not increment a “processed successfully” counter immediately after `inbox.markProcessed()` inside a transaction. The database commit may still fail.

Instead, register a callback after a successful commit.

**Create:** `src/main/java/com/learnhub/observability/AfterCommitMetrics.java`

<CodeBlock language="java" editable>
package com.learnhub.observability;

import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionSynchronization;
import org.springframework.transaction.support.TransactionSynchronizationManager;

@Component
public class AfterCommitMetrics {

    private final PaymentMetrics metrics;

    public AfterCommitMetrics(PaymentMetrics metrics) {
        this.metrics = metrics;
    }

    public void webhookProcessed() {
        if (!TransactionSynchronizationManager
                .isSynchronizationActive()) {
            throw new IllegalStateException(
                    "Active transaction required."
            );
        }

        TransactionSynchronizationManager
                .registerSynchronization(
                        new TransactionSynchronization() {
                            @Override
                            public void afterCommit() {
                                metrics.webhookProcessed();
                            }
                        }
                );
    }
}
</CodeBlock>

Inject `AfterCommitMetrics` into `StripeWebhookProcessor` and call `afterCommitMetrics.webhookProcessed()` after `inbox.markProcessed(eventId)`.

This avoids reporting success for a transaction that later rolls back. Metrics are still operational telemetry, not a durable financial audit ledger.

### 9.5 Prometheus output

With the registry dependency and the endpoint exposed:

```powershell
Invoke-WebRequest `
  -Uri "http://localhost:8080/actuator/prometheus" |
  Select-Object -ExpandProperty Content |
  Select-String "learnhub_checkout|learnhub_webhook"
```

Micrometer metric names are normalized for Prometheus. A counter such as `learnhub.checkout.attempts` will typically appear with a `_total` suffix in Prometheus output.

Example:

```text
learnhub_checkout_attempts_total{outcome="created",...} 7.0
learnhub_checkout_attempts_total{outcome="rejected",...} 1.0
```

Exact output names and labels depend on your Micrometer and Prometheus registry versions.

---

## 10. Protect Prometheus and operational metadata

Actuator endpoints can expose operational details even when they don't expose business records.

For local development, you can use a dedicated management security policy. In production, the preferred approach is to restrict monitoring endpoints to an internal network and authenticated monitoring infrastructure.

A minimal application-level policy for the main port:

<CodeBlock language="java" editable>
.requestMatchers(
        HttpMethod.GET,
        "/actuator/prometheus"
).hasRole("ADMIN")
</CodeBlock>

Place this rule before `.anyRequest().denyAll()`.

**Important:** This is a training-friendly fallback, not an ideal production Prometheus scrape setup. An internal scraper generally should not need a human admin browser session. Use a dedicated service identity or private management endpoint in production.

Also avoid exposing `/actuator/env`, `/actuator/configprops`, `/actuator/heapdump`, or `/actuator/loggers` publicly.

---

## 11. Security-safe Problem Details with request IDs

A production error response should help the frontend and operator identify a failure without revealing implementation details.

We'll attach the correlation ID to error responses.

### 11.1 Error response helper

**Create:** `src/main/java/com/learnhub/web/error/ProblemFactory.java`

<CodeBlock language="java" editable>
package com.learnhub.web.error;

import com.learnhub.observability.RequestCorrelationFilter;

import jakarta.servlet.http.HttpServletRequest;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.stereotype.Component;

@Component
public class ProblemFactory {

    public ProblemDetail create(
            HttpStatus status,
            String safeDetail,
            HttpServletRequest request
    ) {
        ProblemDetail problem =
                ProblemDetail.forStatusAndDetail(
                        status,
                        safeDetail
                );

        problem.setTitle(status.getReasonPhrase());

        Object requestId = request.getAttribute(
                RequestCorrelationFilter.ATTRIBUTE
        );

        if (requestId instanceof String value) {
            problem.setProperty("requestId", value);
        }

        return problem;
    }
}
</CodeBlock>

### 11.2 Integrate with the existing error handler

**File:** `src/main/java/com/learnhub/web/error/GlobalExceptionHandler.java`

Inject `ProblemFactory` into the existing handler and use it for domain errors.

For example, a payment not found handler:

<CodeBlock language="java" editable>
@ExceptionHandler(PaymentNotFoundException.class)
public ResponseEntity<ProblemDetail> paymentNotFound(
        PaymentNotFoundException exception,
        HttpServletRequest request
) {
    ProblemDetail problem = problemFactory.create(
            HttpStatus.NOT_FOUND,
            "Payment not found.",
            request
    );

    return ResponseEntity.status(
            HttpStatus.NOT_FOUND
    ).body(problem);
}
</CodeBlock>

Relevant imports:

```java
import com.learnhub.payments.PaymentNotFoundException;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
```

Do not add a second `@ExceptionHandler` for an exception already handled in the class. Update the existing method instead.

### 11.3 Generic unexpected failures

A generic `500` handler should use a fixed public message and log the exception server-side.

<CodeBlock language="java" editable>
@ExceptionHandler(Exception.class)
public ResponseEntity<ProblemDetail> unexpected(
        Exception exception,
        HttpServletRequest request
) {
    log.error(
            "Unhandled application error",
            exception
    );

    ProblemDetail problem = problemFactory.create(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred.",
            request
    );

    return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(problem);
}
</CodeBlock>

This assumes your existing handler already has an SLF4J logger and `ProblemFactory` injected.

**Do not turn all exceptions into `500`.** Preserve specific handlers for validation (`400`), authentication (`401`), access denial (`403`), missing resources (`404`), state conflicts (`409`), and temporary provider failures (`503`).

Also note that Spring Security may reject a request **before it reaches MVC**. Those `401` and `403` responses come from the security filter chain, not necessarily `GlobalExceptionHandler`.

To give them consistent JSON Problem Details, configure the security authentication entry point and access-denied handler separately. The correlation response header still works even if those handlers return no body.

---

## 12. Distributed tracing: concepts and boundaries

A request ID helps correlate logs for one inbound request. Distributed tracing goes further: it connects spans across services and external calls.

<box border radius="xl" padding={3} gap={2}>
  <text weight="medium" size="sm">Illustrative trace: creating checkout</text>
  <box gap={1}>
    <row justify="between">
      <text size="xs" weight="medium">HTTP POST /checkout</text>
      <text color="secondary" size="2xs" tabularNums>220 ms</text>
    </row>
    <box background="rgba(59,130,246,0.18)" height="16px" radius="sm" width="100%"/>
  </box>
  <box gap={1}>
    <row justify="between">
      <text size="xs">PostgreSQL: create payment attempt</text>
      <text color="secondary" size="2xs" tabularNums>12 ms</text>
    </row>
    <row>
      <box width="8%"/>
      <box background="rgba(20,184,166,0.5)" height="13px" radius="sm" width="6%"/>
    </row>
  </box>
  <box gap={1}>
    <row justify="between">
      <text size="xs">Stripe HTTPS request</text>
      <text color="secondary" size="2xs" tabularNums>175 ms</text>
    </row>
    <row>
      <box width="15%"/>
      <box background="rgba(245,158,11,0.55)" height="13px" radius="sm" width="80%"/>
    </row>
  </box>
  <box gap={1}>
    <row justify="between">
      <text size="xs">PostgreSQL: attach session ID</text>
      <text color="secondary" size="2xs" tabularNums>9 ms</text>
    </row>
    <row>
      <box width="95%"/>
      <box background="rgba(20,184,166,0.5)" height="13px" radius="sm" width="4%"/>
    </row>
  </box>
  <caption>Illustrative durations, not measured LearnHub performance.</caption>
</box>

A trace has a trace ID; each unit of work is a span with its own span ID. Parent-child relationships show how the work fits together.

### 12.1 Optional tracing dependencies

For Spring Boot 3.5.x, Micrometer Tracing is the relevant integration layer.

Add only when you have a tracing backend or a local collector to receive spans:

<CodeBlock language="xml" editable>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>

<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
</CodeBlock>

### 12.2 Optional tracing configuration

<CodeBlock language="yaml" editable>
management:
  tracing:
    sampling:
      probability: 0.1

  otlp:
    tracing:
      endpoint: ${OTEL_TRACES_ENDPOINT:http://localhost:4318/v1/traces}
</CodeBlock>

This samples approximately 10% of eligible traces. Do not enable an exporter against a nonexistent collector and expect traces to appear.

### 12.3 A subtle Java `HttpClient` limitation

Lesson 20's Stripe adapter uses Java's `java.net.http.HttpClient`.

Spring's automatic HTTP client observation may not instrument that client in the same way as a Spring-managed `RestClient` or `WebClient`.

For complete outbound tracing, prefer a Spring-managed HTTP client with observation support or explicitly create an observation/span around the provider call.

Never attach a raw Stripe secret, payment payload, or session cookie as a span attribute.

### 12.4 Webhooks are separate traces

Stripe delivers a webhook later, often to a different application instance. The checkout request and webhook request are not automatically one synchronous trace.

Use the durable `paymentAttemptId` and provider event ID to connect the business events. Those identifiers belong in controlled logs or audit records, not high-cardinality metric tags.

---

## 13. Angular: typed errors and correlation IDs

Angular should present a helpful message and a request reference, not raw server exceptions.

### 13.1 Problem Details type

**Create:** `learnhub/src/app/core/api/problem-detail.model.ts`

<CodeBlock language="typescript" editable>
export interface ApiProblem {
  type?: string;
  title?: string;
  status?: number;
  detail?: string;
  instance?: string;
  requestId?: string;
}

export interface ApiFailure {
  status: number;
  message: string;
  requestId: string | null;
}
</CodeBlock>

### 13.2 HTTP error interceptor

**Create:** `learnhub/src/app/core/http/api-error.interceptor.ts`

<CodeBlock language="typescript" editable>
import {
  HttpErrorResponse,
  HttpInterceptorFn,
} from '@angular/common/http';

import { catchError, throwError } from 'rxjs';

import {
  ApiFailure,
  ApiProblem,
} from '../api/problem-detail.model';

function safeMessage(status: number): string {
  switch (status) {
    case 0:
      return 'Unable to reach LearnHub.';
    case 400:
      return 'Please check your request.';
    case 401:
      return 'Please sign in.';
    case 403:
      return 'You do not have permission.';
    case 404:
      return 'The requested item was not found.';
    case 409:
      return 'This action conflicts with the current state.';
    case 503:
      return 'The service is temporarily unavailable.';
    default:
      return 'Something went wrong. Please try again.';
  }
}

export const apiErrorInterceptor: HttpInterceptorFn =
  (request, next) => {
    return next(request).pipe(
      catchError((error: unknown) => {
        if (!(error instanceof HttpErrorResponse)) {
          return throwError(() => error);
        }

        const body: ApiProblem =
          error.error &&
          typeof error.error === 'object'
            ? error.error as ApiProblem
            : {};

        const failure: ApiFailure = {
          status: error.status,
          message: safeMessage(error.status),
          requestId:
            error.headers.get('X-Request-ID') ??
            (
              typeof body.requestId === 'string'
                ? body.requestId
                : null
            ),
        };

        return throwError(() => failure);
      })
    );
  };
</CodeBlock>

The interceptor intentionally uses controlled messages rather than displaying arbitrary `detail` values from an unexpected server response.

### 13.3 Register the interceptor

**File:** `learnhub/src/app/app.config.ts`

Merge into your existing HTTP provider configuration:

<CodeBlock language="typescript" editable>
provideHttpClient(
  withXsrfConfiguration({
    cookieName: 'XSRF-TOKEN',
    headerName: 'X-XSRF-TOKEN',
  }),
  withInterceptors([
    apiErrorInterceptor,
  ])
)
</CodeBlock>

Imports:

```typescript
import {
  provideHttpClient,
  withInterceptors,
  withXsrfConfiguration,
} from '@angular/common/http';

import { apiErrorInterceptor }
  from './core/http/api-error.interceptor';
```

If Lesson 16 already registered authentication or CSRF interceptors, retain them and append this interceptor in the appropriate order. Do not call `provideHttpClient()` twice merely to add an interceptor.

### 13.4 Display a reference

In a standalone component:

<CodeBlock language="typescript" editable>
errorMessage = signal<string | null>(null);
requestId = signal<string | null>(null);

// Inside an HTTP subscription:
error: (failure: ApiFailure) => {
  this.errorMessage.set(failure.message);
  this.requestId.set(failure.requestId);
}
</CodeBlock>

Template:

<CodeBlock language="html" editable>
@if (errorMessage()) {
  <p role="alert">{{ errorMessage() }}</p>

  @if (requestId()) {
    <small>
      Support reference: {{ requestId() }}
    </small>
  }
}
</CodeBlock>

**CORS note:** With the Lesson 16 same-origin proxy, the browser can read `X-Request-ID` normally. If you deploy the frontend and API on separate origins, explicitly expose the response header through CORS configuration.

---

## 14. MockMvc: test the documentation and diagnostics contract

We'll verify that OpenAPI is generated, health endpoints are reachable, and correlation IDs appear in responses.

### 14.1 Full application test

**Create:** `src/test/java/com/learnhub/observability/OperationalApiIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.observability;

import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class OperationalApiIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired MockMvc mvc;

    @Test
    void openApiDocumentsCheckoutEndpoint()
            throws Exception {

        mvc.perform(get("/v3/api-docs"))
            .andExpect(status().isOk())
            .andExpect(jsonPath(
                    "$.info.title"
            ).value("LearnHub REST API"))
            .andExpect(jsonPath(
                    "$.paths['/api/v1/me/checkout/courses/{courseId}'].post"
            ).exists());
    }

    @Test
    void healthEndpointDoesNotExposeDetails()
            throws Exception {

        mvc.perform(get("/actuator/health"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("UP"))
            .andExpect(jsonPath("$.components").doesNotExist());
    }

    @Test
    void validRequestIdIsEchoed()
            throws Exception {

        mvc.perform(get("/actuator/health")
                .header(
                        "X-Request-ID",
                        "lesson22-test-request"
                ))
            .andExpect(status().isOk())
            .andExpect(header().string(
                    "X-Request-ID",
                    "lesson22-test-request"
            ));
    }

    @Test
    void invalidRequestIdIsReplaced()
            throws Exception {

        mvc.perform(get("/actuator/health")
                .header("X-Request-ID", "bad id"))
            .andExpect(status().isOk())
            .andExpect(header().exists("X-Request-ID"))
            .andExpect(header().string(
                    "X-Request-ID",
                    org.hamcrest.Matchers.not("bad id")
            ));
    }
}
</CodeBlock>

This test uses the `test` profile inherited from `PostgresIntegrationTest`, where OpenAPI is enabled.

The exact OpenAPI path key depends on the `@PathVariable` name and method signature. If your controller uses `id` instead of `courseId`, update the assertion to match the real public contract.

### 14.2 Test the readiness dependency

Create a separate integration test that simulates PostgreSQL being unavailable and verifies readiness changes to a non-healthy status.

Do **not** stop the shared static Testcontainers instance during an ordinary test class. That would break other tests sharing the Spring context.

A safer strategy is a dedicated application context with a controllable health indicator, or a dedicated container used exclusively by that test.

### 14.3 Test Prometheus access

Add security assertions:

```java
@Test
void anonymousCannotReadPrometheus()
        throws Exception {

    mvc.perform(get("/actuator/prometheus"))
        .andExpect(status().isUnauthorized());
}
```

Use the production security chain. A security rule that accidentally permits every `/actuator/**` path would cause this test to fail.

---

## 15. Micrometer unit test

We can verify custom metrics without starting Spring Boot.

**Create:** `src/test/java/com/learnhub/observability/PaymentMetricsTest.java`

<CodeBlock language="java" editable>
package com.learnhub.observability;

import io.micrometer.core.instrument.simple.SimpleMeterRegistry;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class PaymentMetricsTest {

    @Test
    void recordsCheckoutOutcomes() {
        try (var registry = new SimpleMeterRegistry()) {
            var metrics = new PaymentMetrics(registry);

            metrics.checkoutCreated();
            metrics.checkoutCreated();
            metrics.checkoutRejected();

            assertThat(
                    registry.get(
                            "learnhub.checkout.attempts"
                    )
                    .tag("outcome", "created")
                    .counter()
                    .count()
            ).isEqualTo(2.0);

            assertThat(
                    registry.get(
                            "learnhub.checkout.attempts"
                    )
                    .tag("outcome", "rejected")
                    .counter()
                    .count()
            ).isEqualTo(1.0);
        }
    }

    @Test
    void recordsProviderLatency() {
        try (var registry = new SimpleMeterRegistry()) {
            var metrics = new PaymentMetrics(registry);

            metrics.recordProviderLatency(
                    25_000_000L
            );

            assertThat(
                    registry.get(
                            "learnhub.checkout.provider.duration"
                    )
                    .timer()
                    .count()
            ).isEqualTo(1);
        }
    }
}
</CodeBlock>

This test proves metric registration and counting. It does not prove that your Prometheus scraper can reach the application in production.

---

## 16. Angular interceptor tests

**Create:** `learnhub/src/app/core/http/api-error.interceptor.spec.ts`

<CodeBlock language="typescript" editable>
import { TestBed } from '@angular/core/testing';

import {
  HttpClient,
  provideHttpClient,
  withInterceptors,
} from '@angular/common/http';

import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';

import { apiErrorInterceptor }
  from './api-error.interceptor';

import { ApiFailure }
  from '../api/problem-detail.model';

describe('apiErrorInterceptor', () => {
  let client: HttpClient;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(
          withInterceptors([apiErrorInterceptor])
        ),
        provideHttpClientTesting(),
      ],
    });

    client = TestBed.inject(HttpClient);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    http.verify();
  });

  it('preserves a request reference', () => {
    let failure: ApiFailure | null = null;

    client.get('/api/v1/courses/999').subscribe({
      error: error => {
        failure = error as ApiFailure;
      },
    });

    const request = http.expectOne(
      '/api/v1/courses/999'
    );

    request.flush(
      {
        status: 404,
        title: 'Not Found',
        requestId: 'backend-request-123',
      },
      {
        status: 404,
        statusText: 'Not Found',
      }
    );

    expect(failure).toEqual({
      status: 404,
      message: 'The requested item was not found.',
      requestId: 'backend-request-123',
    });
  });

  it('does not expose arbitrary server details', () => {
    let message = '';

    client.get('/api/v1/courses').subscribe({
      error: (failure: ApiFailure) => {
        message = failure.message;
      },
    });

    const request = http.expectOne(
      '/api/v1/courses'
    );

    request.flush(
      {
        detail:
          'Internal SQL statement and stack trace',
      },
      {
        status: 500,
        statusText: 'Internal Server Error',
      }
    );

    expect(message).toBe(
      'Something went wrong. Please try again.'
    );

    expect(message).not.toContain('SQL');
  });
});
</CodeBlock>

### Why this test matters

The frontend must not turn backend implementation details into user-visible error messages. A request ID provides a safer troubleshooting reference.

---

## 17. Local verification: full operational workflow

### Terminal A — Backend

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

### Terminal B — Angular

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd serve --proxy-config proxy.conf.json
```

### Terminal C — Health and API documentation

```powershell
$base = "http://localhost:8080"

Invoke-RestMethod "$base/actuator/health"
Invoke-RestMethod "$base/actuator/health/liveness"
Invoke-RestMethod "$base/actuator/health/readiness"

$api = Invoke-RestMethod "$base/v3/api-docs"

$api.info.title
$api.info.version
$api.paths.PSObject.Properties.Name
```

### Verify correlation IDs

```powershell
$response = Invoke-WebRequest `
  -Uri "http://localhost:8080/actuator/health" `
  -Headers @{
      "X-Request-ID" = "lesson22-powershell-001"
  }

$response.Headers["X-Request-ID"]
```

Expected:

```text
lesson22-powershell-001
```

### Verify Prometheus access control

```powershell
try {
    Invoke-WebRequest `
      -Uri "http://localhost:8080/actuator/prometheus" `
      -ErrorAction Stop
} catch {
    Write-Host "Monitoring endpoint requires authorization."
}
```

For an anonymous request, the intended response is `401`.

### Test execution

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=RequestCorrelationFilterTest,PaymentMetricsTest,OperationalApiIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 22 backend tests failed."
}

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend verification failed."
}
```

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

## 18. Production-readiness review

Before deploying these changes, verify the following design decisions.

| Concern | Production requirement |
|---|---|
| Swagger UI | Disabled publicly or explicitly protected |
| Health | Minimal response, no sensitive component details |
| Readiness | Includes essential database dependency |
| Prometheus | Private network or dedicated monitoring authentication |
| Logs | Structured, searchable, retention-controlled |
| Correlation ID | Validated, bounded, not trusted as identity |
| Metrics | Low-cardinality labels |
| Tracing | Configured exporter, sampling, sensitive-data filtering |
| Problem Details | Stable error codes, no stack traces |
| Payments | Durable audit records separate from telemetry |
| Angular | Safe messages and request references |
| Security | No session cookies, tokens, or credentials in logs |

### Three important refinements

**1. Correlation IDs and distributed tracing are complementary.** When tracing is enabled, consider adding the trace ID to Problem Details as a separate field, but never treat either identifier as a secret or authorization credential.

**2. Checkout metrics need precise naming.** A successfully created Checkout Session is not a successful payment. Use distinct metrics for checkout creation, verified payment fulfillment, and refund completion.

**3. Operational observability must not change payment correctness.** If a metrics registry or log collector is unavailable, payment processing should continue. Financial state belongs in PostgreSQL, not in Prometheus counters or log lines.

---

## 19. Hands-on exercises

<box gap={3}>
  <row gap={3} align="start">
    <icon name="book-open" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise A — Complete the OpenAPI contract**

      Document the instructor dashboard, curriculum, wishlist, enrollment, and payment-status endpoints. Verify role requirements, DTO schemas, and error responses.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="heart-pulse" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise B — Readiness failure**

      In an isolated test context, simulate PostgreSQL unavailability. Prove readiness becomes unhealthy while the application liveness endpoint remains healthy.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="chart-no-axes-column" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise C — Payment fulfillment metrics**

      Add counters for verified paid events, failed events, duplicate deliveries, and refunds. Record successful fulfillment only after transaction commit.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="shield-check" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise D — Sensitive-data audit**

      Search the codebase for logging of Authorization headers, cookies, passwords, provider keys, raw webhook payloads, and user identifiers. Remove unsafe logging and add regression tests where practical.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="file-code" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise E — OpenAPI contract snapshot**

      Export the generated OpenAPI document and compare it against a reviewed baseline. Fail the build when an unapproved breaking API change occurs.
    </box>
  </row>
</box>

## 20. Interactive knowledge quiz

{@body const qs=[{q:"What is the difference between liveness and readiness?",o:["Liveness tests the database; readiness tests CSS","Liveness indicates the process is alive; readiness indicates it can serve traffic","They are always identical"],a:1,e:"A database outage may make LearnHub unready without requiring a JVM restart."},{q:"Which metric tag is appropriate?",o:["studentId","paymentAttemptId","outcome=created"],a:2,e:"Bounded labels keep metric cardinality manageable. IDs create an unbounded number of time series."},{q:"What should Angular display for an unexpected backend failure?",o:["The full SQL exception","A safe message and request reference","The session cookie"],a:1,e:"Request IDs help operators diagnose failures without disclosing sensitive internals."},{q:"When should a verified payment fulfillment success counter increment?",o:["Before the database transaction starts","After successful commit","When Angular reaches the success page"],a:1,e:"A rolled-back payment must not be counted as durably fulfilled."},{q:"How is a Stripe webhook authenticated?",o:["With the student's JSESSIONID","With the verified Stripe-Signature over the raw body","With the X-XSRF-TOKEN cookie"],a:1,e:"A webhook is a server-to-server request and uses provider signature verification rather than browser session authentication."}]}
{@body const [ans,setAns]=DIL.useState({})}
{@body const [show,setShow]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">    **Lesson 22 — Observability**
    <text size="xs" color="secondary">5 questions</text>
  </row>
  {#each qs as q,i}
    <box key={i} gap={2}>
      **{i+1}. {q.q}**
      <radio-group direction="col" value={ans[i]===undefined?"":String(ans[i])} onChange={v=>{setAns(old=>({...old,[i]:Number(v)}));setShow(false);}}>
        {#each q.o as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if show}
        <row align="start" gap={2}>
          <icon name={ans[i]===q.a?"check-circle":"info"} color={ans[i]===q.a?"success":"secondary"}/>
          <box flex="1" gap={1}>
            <text size="sm" weight="medium">{ans[i]===q.a?"Correct":"Correct: "+q.o[q.a]}</text>
            <text size="sm" color="secondary">{q.e}</text>
          </box>
        </row>
      {/if}
    </box>
    {#if i<qs.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if show}
    <title size="lg">{qs.filter((q,i)=>ans[i]===q.a).length} / 5 correct</title>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(ans).length!==qs.length} onClick={()=>setShow(true)}>Check answers</button>
    <button color="secondary" variant="outline" onClick={()=>{setAns({});setShow(false);}}>Reset</button>
  </row>
</box>

## 21. Lesson completion checklist

{@body const sections=[{name:"OpenAPI",items:["Added springdoc dependency","Configured OpenAPI metadata","Documented public course catalog","Documented authenticated checkout","Documented webhook signature requirement","Verified generated OpenAPI JSON","Restricted Swagger UI by environment"]},{name:"Health and observability",items:["Configured liveness and readiness","Included PostgreSQL in readiness","Protected Prometheus endpoint","Implemented request correlation filter","Added structured JSON logging","Reviewed sensitive logging","Added Micrometer payment metrics","Used after-commit fulfillment metrics","Reviewed optional distributed tracing"]},{name:"Frontend and tests",items:["Created typed Problem Details model","Added safe HTTP error interceptor","Displayed request references","Added MockMvc operational tests","Added correlation filter tests","Added Micrometer unit tests","Added Angular interceptor tests","Ran Maven clean verify","Ran Angular tests and build","Completed at least two exercises","Completed quiz","Reviewed and committed Git changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const count=sections.reduce((n,s)=>n+s.items.length,0)}
{@body const checked=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 22 progress**
      <text size="xs" color="secondary">Check off only locally verified work.</text>
    </box>
    <title size="lg" tabularNums>{checked}/{count}</title>
  </row>
  <box background="surface-tertiary" radius="full" height="8px" clip>
    <box background="rgba(109,40,217,0.85)" radius="full" height="100%" width={`${checked/count*100}%`}/>
  </box>
  {#each sections as s,i}
    <box key={i} gap={2}>
      **{s.name}**
      {#each s.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!done[`${i}-${j}`]} onChange={v=>setDone(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if checked===count}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 22 checklist complete.**
    </row>
  {/if}
</box>

## 22. Git checkpoint

Backend:

```powershell
cd $HOME\angular-learning\learnhub-backend

git diff --check
git status --short
git diff --stat

git add pom.xml src\main src\test
git diff --cached --stat

git commit -m "Document LearnHub API and add operational observability"
```

Angular:

```powershell
cd $HOME\angular-learning\learnhub

git diff --check
git status --short
git diff --stat

git add src\app
git diff --cached --stat

git commit -m "Add safe API diagnostics and correlation-aware errors"
```

These are implementation instructions, not changes executed in your local workspace. Verify your existing Spring Security, Actuator, and logging configurations before merging the examples.

## 23. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="container" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 23 — Docker, CI/CD, and Deployment**

    We'll containerize the Java 21 backend and Angular frontend, create a PostgreSQL-backed Docker Compose environment, manage secrets and Flyway migrations, add GitHub Actions CI, implement health-gated deployments, and plan production rollout and rollback procedures.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 23: Docker multi-stage builds for Java 21 Spring Boot and Angular standalone, PostgreSQL Docker Compose, environment variables and secrets, Flyway migrations, production security and reverse proxy, GitHub Actions CI/CD, Testcontainers in CI, image publishing, health and readiness checks, deployment strategies, backups and rollback, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 22 with experienced-Java pacing.")}>Start Lesson 23 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 22 deliverable:** A documented LearnHub API with protected operational endpoints, PostgreSQL-aware readiness, correlation-aware structured logs, payment metrics, safe frontend error reporting, and automated observability contract tests.

**Core principle:** Observability should make failures understandable without weakening security, exposing sensitive data, or changing the correctness of business transactions.