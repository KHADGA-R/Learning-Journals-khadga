# LearnHub Java & Spring Boot Backend — Lesson 5

<text color="secondary" size="sm">Spring Boot Setup, Maven, Dependency Injection, Application Context, Configuration, and First HTTP Server</text>

<row align="center" gap={2} wrap="wrap">
  <badge>Lesson 5 of 24</badge>
  <badge>Experienced Java Track</badge>
  <badge>Hands-on Migration</badge>
</row>

In Lessons 1–4, you built a plain Java 21 application with a validated `Course` model, repository abstraction, catalog search, publishing workflows, typed exceptions, and automated tests.

Today, we'll **convert that existing Maven project into a Spring Boot application without starting over**.

By the end, LearnHub will run as an HTTP server on port `8080`, use Spring's dependency injection container to assemble its services, and expose a first diagnostic endpoint. We'll keep the course catalog logic intact so we can build real REST APIs in the following lessons.

## Learning objectives

- Understand what Spring Boot adds to the Spring Framework.
- Convert the existing `pom.xml` to a Spring Boot build.
- Use Spring Initializr as a reference without overwriting your project.
- Introduce `@SpringBootApplication`, `@Service`, `@Repository`, and `@Bean`.
- Understand constructor injection, bean lifecycles, and the Application Context.
- Configure the application using `application.yml`.
- Start and inspect a local embedded HTTP server.
- Write Spring integration tests while retaining the previous Java unit tests.

<text color="secondary" size="sm">Estimated time: 2–3 hours, including migration, troubleshooting, and exercises.</text>

## 1. Understand the architecture change

<box border radius="lg" padding={3} gap={2}>
  <text weight="medium">Before — Plain Java, Lesson 4</text>
  <box background="surface-secondary" padding={3} radius="md" align="center" gap={1}>
    **Main.java**
    <text color="secondary" size="xs">Manually constructs repository and services</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border padding={3} radius="md" align="center">
    **CourseCatalogService + CourseManagementService**
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" padding={3} radius="md" align="center">
    **InMemoryCourseRepository**
  </box>
  <divider/>
  <text weight="medium">After — Spring Boot, Lesson 5</text>
  <box background="surface-secondary" padding={3} radius="md" align="center" gap={1}>
    **Spring Boot Application**
    <text color="secondary" size="xs">Starts the embedded web server and Application Context</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <grid columns={2} gap={2}>
    <grid-item>
      <box border padding={3} radius="md" align="center" gap={1}>
        **Catalog Service**
        <text color="secondary" size="xs">Spring-managed bean</text>
      </box>
    </grid-item>
    <grid-item>
      <box border padding={3} radius="md" align="center" gap={1}>
        **Management Service**
        <text color="secondary" size="xs">Spring-managed bean</text>
      </box>
    </grid-item>
  </grid>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border padding={3} radius="md" align="center" gap={1}>
    **CourseRepository**
    <text color="secondary" size="xs">Injected interface</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" padding={3} radius="md" align="center" gap={1}>
    **InMemoryCourseRepository**
    <text color="secondary" size="xs">Registered with @Repository</text>
  </box>
</box>

The central change is that **Spring creates and wires application objects**. Our business logic remains plain Java, but Spring manages its dependencies and lifecycle.

## 2. Prepare the Lesson 4 project

Open Windows PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
mvn clean verify
```

You should be using JDK 21 and the existing Maven project.

If you followed every provided test file, the expected Lesson 4 baseline is 46 tests.

Create a Git checkpoint:

```powershell
git status
git add .
git commit -m "Checkpoint LearnHub backend before Spring Boot"
```

If Git reports nothing to commit, that's fine.

### Confirm the project root

```powershell
Get-Location
Get-ChildItem
Get-ChildItem src\main\java\com\learnhub -Recurse -File
```

Your root should contain:

```text
learnhub-backend/
├── pom.xml
├── src/
│   ├── main/java/com/learnhub/
│   └── test/java/com/learnhub/
└── .gitignore
```

Do not delete your existing domain, catalog, repository, or service files.

## 3. Spring Framework versus Spring Boot

The **Spring Framework** provides the foundational programming model: dependency injection, bean lifecycle management, web MVC, transactions, data access integration, and more.

**Spring Boot** builds on Spring to make applications easier to configure, launch, and operate.

| Spring concept | Role in LearnHub |
|---|---|
| Application Context | Stores and manages application beans |
| Bean | An object managed by Spring |
| Dependency Injection | Supplies dependencies to classes |
| Component scanning | Finds annotated classes |
| Auto-configuration | Configures common infrastructure from dependencies and settings |
| Starter dependency | Convenient group of compatible libraries |
| Embedded server | Runs the HTTP application without a separate servlet-container installation |
| Externalized configuration | Supplies ports, environment settings, and application properties |

Spring Boot does not replace Java design principles. Our repository interface, domain validation, and service boundaries remain important.

## 4. Spring Initializr: reference and project generation

<row align="start" gap={3}>
  <AsyncImage query="Spring Initializr start.spring.io project generator interface Spring Boot Java Maven dependencies" aspectRatio="5:3" maxWidth="260px"/>
  <box flex="1" gap={2}>
    <text weight="medium"><Entity category="software" value="Spring Initializr" disambig="Spring Boot project generation service"/></text>
    <text size="sm">Spring Initializr generates a starting project with a compatible build file, application entry point, and selected dependencies.</text>
    <text size="sm"><Link url="https://start.spring.io/" title="Open Spring Initializr"/></text>
  </box>
</row>

For a new project, you would typically select:

| Setting | Value |
|---|---|
| Project | Maven |
| Language | Java |
| Spring Boot | Stable version compatible with Java 21 |
| Group | `com.learnhub` |
| Artifact | `learnhub-backend` |
| Packaging | Jar |
| Java | 21 |
| Dependencies | Spring Web, Spring Boot Actuator, Spring Boot Starter Test |

However, **we already have a working Maven project**. We will update it directly rather than generate a new project and risk losing earlier lessons.

### Optional: generate a reference project

From PowerShell, you can use Spring Initializr to generate a separate reference project. The following URL requests Spring Boot's currently offered default version, so inspect the generated `pom.xml` before using it.

```powershell
cd $HOME\angular-learning

Invoke-WebRequest `
  -Uri "https://start.spring.io/starter.zip?type=maven-project&language=java&groupId=com.learnhub&artifactId=learnhub-reference&name=learnhub-reference&packageName=com.learnhub.reference&packaging=jar&javaVersion=21&dependencies=web,actuator" `
  -OutFile "learnhub-reference.zip"

Expand-Archive `
  -Path "learnhub-reference.zip" `
  -DestinationPath "learnhub-reference" `
  -Force
```

This is optional and separate from your real backend project.

For the rest of this lesson, work in:

```powershell
cd $HOME\angular-learning\learnhub-backend
```

## 5. Convert `pom.xml` to Spring Boot

For a reproducible lesson, we'll use **Spring Boot 3.5.7**, a Java 21-compatible release in the Spring Boot 3.x line. We'll deliberately pin the version rather than rely on whatever version Spring Initializr selects.

This migration uses Spring Boot 3.x APIs. A later upgrade to Spring Boot 4 should be treated as a separate, tested dependency migration.

Replace the entire file:

`$HOME\angular-learning\learnhub-backend\pom.xml`

<CodeBlock language="xml" editable>
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
           http://maven.apache.org/POM/4.0.0
           https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- Spring Boot manages compatible dependency
         and Maven plugin versions. -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.7</version>
        <relativePath/>
    </parent>

    <groupId>com.learnhub</groupId>
    <artifactId>learnhub-backend</artifactId>
    <version>0.1.0-SNAPSHOT</version>

    <name>LearnHub Backend</name>
    <description>
        Java 21 and Spring Boot learning platform backend
    </description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>

        <!-- Spring MVC, JSON support, and embedded Tomcat. -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Application health and operational endpoints. -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- JUnit Jupiter, Spring Test, Mockito, AssertJ, etc. -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <!-- Repackage the JAR with dependencies
                 and a Spring Boot executable launcher. -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
</CodeBlock>

### What changed?

We replaced the manually configured compiler, JUnit, Surefire, and JAR plugin versions with Spring Boot's managed configuration.

The Spring Boot parent supplies compatible versions and defaults. The Spring Boot Maven plugin produces an executable application archive containing the dependencies required to launch the server.

We also added two runtime starters:

- `spring-boot-starter-web` for Spring MVC and the embedded HTTP server.
- `spring-boot-starter-actuator` for operational health information.

We are **not adding JPA, PostgreSQL, or Spring Security yet**. Those belong to later lessons.

### Verify Maven can resolve the new dependencies

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn dependency:tree
```

Then compile:

```powershell
mvn clean compile
```

The first build may take longer while Maven downloads Spring libraries.

## 6. Create the Spring Boot application entry point

Keep the existing `Main.java` as a plain-Java console demonstration, but create a new Spring Boot entry point.

File: `src/main/java/com/learnhub/LearnHubApplication.java`

<CodeBlock language="java" editable>
package com.learnhub;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Main entry point for the LearnHub Spring Boot server.
 *
 * @SpringBootApplication combines:
 * - @SpringBootConfiguration
 * - @EnableAutoConfiguration
 * - @ComponentScan
 *
 * Component scanning starts in com.learnhub and
 * includes its subpackages.
 */
@SpringBootApplication
public class LearnHubApplication {

    public static void main(String[] args) {

        SpringApplication.run(
                LearnHubApplication.class,
                args
        );
    }
}
</CodeBlock>

### Why put this class in `com.learnhub`?

By default, component scanning starts at the package containing the application class.

Because our classes live under packages such as:

```text
com.learnhub.repository
com.learnhub.service
com.learnhub.web
com.learnhub.config
```

Spring can discover them automatically.

If we put the application class in an unrelated package, component scanning might not find our beans.

### How the Application Context works

<box border radius="lg" padding={3} gap={2}>
  <box background="surface-secondary" radius="md" padding={3} align="center" gap={1}>
    **SpringApplication.run()**
    <text color="secondary" size="xs">Bootstraps the application</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="md" padding={3} gap={2}>
    <text weight="medium" textAlign="center">Application Context</text>
    <grid columns={2} gap={2}>
      {#each ["CourseRepository","CourseCatalogService","CourseManagementService","HealthController"] as bean}
        <grid-item>
          <box background="surface-secondary" radius="sm" padding={2}>
            <text size="xs" textAlign="center">{bean}</text>
          </box>
        </grid-item>
      {/each}
    </grid>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" radius="md" padding={3} align="center" gap={1}>
    **Embedded Tomcat**
    <text color="secondary" size="xs">Accepts HTTP requests on port 8080</text>
  </box>
</box>

Spring scans for components, creates the necessary beans, resolves their constructor dependencies, and starts the embedded web server.

## 7. Register the repository using `@Repository`

We need Spring to know how to create `InMemoryCourseRepository`.

Open:

`src/main/java/com/learnhub/repository/InMemoryCourseRepository.java`

Add this import:

```java
import org.springframework.stereotype.Repository;
```

And add the annotation above the class:

```java
@Repository
public final class InMemoryCourseRepository
        implements CourseRepository {
```

**But there's an important issue:** our repository has two constructors:

```java
public InMemoryCourseRepository()
public InMemoryCourseRepository(Collection<Course> initialCourses)
```

We want Spring to use a seeded repository, not an empty one. Instead of relying on constructor selection or modifying the repository for framework-specific concerns, we'll register the repository through an explicit `@Bean` factory.

Therefore, **do not keep `@Repository` on `InMemoryCourseRepository` for the final implementation of this lesson**. The example above demonstrates where the stereotype goes, but we'll use the configuration approach below.

This avoids two competing bean registrations and preserves the repository as a plain Java class.

### Step 7.1 — Create a configuration class

Create the directory:

```powershell
New-Item -ItemType Directory -Force `
    src\main\java\com\learnhub\config
```

File: `src/main/java/com/learnhub/config/CourseRepositoryConfig.java`

<CodeBlock language="java" editable>
package com.learnhub.config;

import com.learnhub.data.SampleCourses;
import com.learnhub.repository.CourseRepository;
import com.learnhub.repository.InMemoryCourseRepository;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * Defines infrastructure beans for the local demo.
 *
 * When PostgreSQL is introduced, this configuration
 * can be replaced by a persistence adapter.
 */
@Configuration
public class CourseRepositoryConfig {

    /**
     * Registers one seeded repository instance
     * in the Spring Application Context.
     *
     * The default bean scope is singleton.
     */
    @Bean
    public CourseRepository courseRepository() {

        return new InMemoryCourseRepository(
                SampleCourses.create()
        );
    }
}
</CodeBlock>

This is the configuration we will use for the rest of Lesson 5.

### `@Repository` versus `@Bean`

| Approach | Typical use |
|---|---|
| `@Repository` | Spring discovers a repository implementation by component scanning |
| `@Bean` | A configuration method constructs and registers an object |
| `@Configuration` | Declares a class containing bean definitions |
| `@Component` | General-purpose Spring-managed component |

We chose `@Bean` because we want to seed the repository using an existing plain Java constructor.

When Spring Data JPA is introduced, repository registration will work differently.

## 8. Register services using `@Service`

Spring should now create `CourseCatalogService` and `CourseManagementService` and inject the repository.

### Step 8.1 — Update CourseCatalogService

File: `src/main/java/com/learnhub/service/CourseCatalogService.java`

Add:

```java
import org.springframework.stereotype.Service;
```

Annotate the class:

```java
@Servicepublic final class CourseCatalogService {
```

Keep its existing constructor:

```java
public CourseCatalogService(
        CourseRepository repository
) {
    this.repository = Objects.requireNonNull(repository);
}
```

Do not remove the `search()`, `findPublishedById()`, `publishedCategories()`, or `summarize()` methods from Lesson 4.

### Step 8.2 — Update CourseManagementService

File: `src/main/java/com/learnhub/service/CourseManagementService.java`

Add:

```java
import org.springframework.stereotype.Service;
```

Annotate the class:

```java
@Service
public final class CourseManagementService {
```

Keep its constructor and existing `createDraft()`, `publish()`, and `archive()` methods unchanged.

### Why constructor injection?

Spring sees a single constructor requiring `CourseRepository` and finds the bean created by `CourseRepositoryConfig`.

Conceptually, Spring performs the equivalent of:

```java
CourseRepository repository =
        new InMemoryCourseRepository(SampleCourses.create());

CourseCatalogService catalogService =
        new CourseCatalogService(repository);

CourseManagementService managementService =
        new CourseManagementService(repository);
```

But now Spring manages those instances.

You do not need `@Autowired` on a class's only constructor.

### Important: avoid duplicate beans

At the end of this step, the intended bean registration is:

| Type | Registration |
|---|---|
| `CourseRepository` | One `@Bean` factory |
| `CourseCatalogService` | `@Service` |
| `CourseManagementService` | `@Service` |
| `InMemoryCourseRepository` | No class annotation |

If you annotate `InMemoryCourseRepository` with `@Repository` **and** keep the factory method, Spring may register multiple repository candidates and dependency injection may become ambiguous.

## 9. Configure the application with `application.yml`

Create the resources directory:

```powershell
cd $HOME\angular-learning\learnhub-backend

New-Item -ItemType Directory -Force `
    src\main\resources
```

File: `src/main/resources/application.yml`

<CodeBlock language="yaml" editable>
spring:
  application:
    name: learnhub-backend

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

info:
  app:
    name: LearnHub Backend
    description: Java 21 and Spring Boot learning platform
    version: 0.1.0

logging:
  level:
    root: INFO
    com.learnhub: DEBUG
</CodeBlock>

### Configuration explained

| Property | Purpose |
|---|---|
| `spring.application.name` | Identifies the application |
| `server.port` | Selects the HTTP port |
| `management.endpoints.web.exposure.include` | Controls which Actuator endpoints are exposed over HTTP |
| `management.endpoint.health.show-details` | Controls visibility of detailed health information |
| `logging.level.root` | Default log verbosity |
| `logging.level.com.learnhub` | Log verbosity for our application code |

We deliberately expose only `health` and `info`.

Actuator's health endpoint is useful for local development, but it does not prove that every LearnHub feature or external dependency works.

### Spring configuration precedence

A setting in `application.yml` can be overridden by higher-priority configuration sources, including command-line arguments and environment variables.

For example, to launch on port `8081`:

```powershell
mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--server.port=8081"
```

Or when running the packaged JAR:

```powershell
java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar `
  --server.port=8081
```

This allows us to use different settings in development, testing, and production without editing source code for every environment.

## 10. Create your first LearnHub HTTP endpoint

Spring Boot's Actuator already provides a health endpoint. We'll also create a small custom endpoint to confirm that our catalog service is correctly injected.

This is not yet the full courses REST API. We'll implement that in the next lesson.

Create the web package:

```powershell
New-Item -ItemType Directory -Force `
    src\main\java\com\learnhub\web
```

### Step 10.1 — Define the response DTO

File: `src/main/java/com/learnhub/web/LearnHubStatusResponse.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

/**
 * Immutable response object for the local diagnostic API.
 *
 * Spring's JSON converter serializes record components
 * into JSON properties.
 */
public record LearnHubStatusResponse(
        String application,
        String status,
        long publishedCourses,
        long freeCourses,
        long paidCourses
) {
}
</CodeBlock>

### Step 10.2 — Implement the controller

File: `src/main/java/com/learnhub/web/LearnHubStatusController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.catalog.CatalogSummary;
import com.learnhub.service.CourseCatalogService;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Simple diagnostic controller.
 *
 * Demonstrates:
 * - Spring MVC request mapping
 * - Constructor injection
 * - Service delegation
 * - Automatic JSON serialization
 *
 * This endpoint is for local learning, not a
 * substitute for operational health checks.
 */
@RestController
@RequestMapping("/api/v1")
public class LearnHubStatusController {

    private final CourseCatalogService catalogService;

    public LearnHubStatusController(
            CourseCatalogService catalogService
    ) {
        this.catalogService = catalogService;
    }

    /**
     * GET /api/v1/status
     *
     * Returns a small snapshot of public catalog counts.
     */
    @GetMapping("/status")
    public LearnHubStatusResponse status() {

        CatalogSummary summary =
                catalogService.summarize();

        return new LearnHubStatusResponse(
                "LearnHub Backend",
                "RUNNING",
                summary.totalCourses(),
                summary.freeCourses(),
                summary.paidCourses()
        );
    }
}
</CodeBlock>

### What happens when the browser requests this endpoint?

<box border radius="lg" padding={3} gap={2}>
  <row align="center" justify="between">
    <text weight="medium">GET /api/v1/status</text>
    <badge color="success">200 OK</badge>
  </row>
  <box background="surface-secondary" radius="md" padding={3} gap={1}>
    <text size="xs" color="secondary">1. HTTP request</text>
    **LearnHubStatusController**
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" radius="md" padding={3} gap={1}>
    <text size="xs" color="secondary">2. Business logic</text>
    **CourseCatalogService.summarize()**
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" radius="md" padding={3} gap={1}>
    <text size="xs" color="secondary">3. Data access</text>
    **CourseRepository.findAll()**
  </box>
  <row justify="center"><icon name="arrow-up" color="secondary"/></row>
  <box border radius="md" padding={3} gap={1}>
    <text size="xs" color="secondary">4. JSON response</text>
    <text size="sm">Published course counts returned to the client</text>
  </box>
</box>

The controller delegates to the service rather than reaching directly into repository internals. This is the architectural boundary we'll preserve throughout the course.

## 11. Start the Spring Boot server

From PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run
```

Look for output similar to:

```text
Tomcat started on port 8080 (http)
Started LearnHubApplication
```

The exact log wording may differ.

Leave this terminal running.

### Test from a second PowerShell terminal

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:8080/api/v1/status"
```

Expected values:

<box border radius="lg" padding={3} gap={2}>
  <row align="center" justify="between">
    **GET /api/v1/status**
    <badge color="success">Expected 200</badge>
  </row>
  <CodeBlock language="json">
{
  "application": "LearnHub Backend",
  "status": "RUNNING",
  "publishedCourses": 4,
  "freeCourses": 2,
  "paidCourses": 2
}
</CodeBlock>
  <caption>Expected response with the original five sample courses, one of which remains a draft.</caption>
</box>

### Test Actuator

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:8080/actuator/health"
```

Expected:

```json
{
  "status": "UP"
}
```

For a more detailed HTTP inspection:

```powershell
curl.exe -i http://localhost:8080/api/v1/status
curl.exe -i http://localhost:8080/actuator/health
```

To stop the application, press `Ctrl+C` in the terminal running Maven.

## 12. Understand bean scopes and lifecycle

Spring-managed beans have a scope. The default is `singleton`, meaning one bean instance per Spring Application Context.

For this lesson, our repository and services are singleton beans.

| Scope | Meaning | Typical use |
|---|---|---|
| Singleton | One bean instance per context | Stateless services |
| Prototype | New bean instance when requested from the container | Special-purpose stateful objects |
| Request | One bean instance per HTTP request | Request-scoped context |
| Session | One bean instance per HTTP session | Session-specific web state |

### Why singleton services should usually be stateless

Our `CourseCatalogService` stores only a repository reference:

```java
private final CourseRepository repository;
```

It does not store the current user's search term, current page number, or authentication details in mutable instance fields.

That is good because multiple HTTP requests can invoke the same singleton service concurrently.

The in-memory repository is stateful, but it synchronizes individual operations. Its limitations around multi-step atomicity still apply.

### Constructor injection versus field injection

Prefer:

```java
@Service
public class ExampleService {

    private final CourseRepository repository;

    public ExampleService(CourseRepository repository) {
        this.repository = repository;
    }
}
```

Over:

```java
@Autowired
private CourseRepository repository;
```

Constructor injection makes required dependencies explicit, supports immutable fields, and allows plain Java unit tests to construct the service without Spring.

### Spring does not manage every object

This remains an ordinary Java object:

```java
Course course = new Course(...);
```

Spring does not need to manage domain records, DTOs, or every helper object.

Beans are most useful for application services, infrastructure adapters, controllers, and shared configuration.

## 13. Write Spring Boot integration tests

Our previous 46 tests should remain useful because the domain and service logic is still plain Java.

Now we'll add tests that verify Spring's configuration and HTTP layer.

Create the test directory:

```powershell
New-Item -ItemType Directory -Force `
    src\test\java\com\learnhub\web
```

### Step 13.1 — Application Context test

File: `src/test/java/com/learnhub/LearnHubApplicationTest.java`

<CodeBlock language="java" editable>
package com.learnhub;

import com.learnhub.repository.CourseRepository;
import com.learnhub.service.CourseCatalogService;
import com.learnhub.service.CourseManagementService;

import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Verifies that Spring can create the Application Context
 * and resolve our repository and service dependencies.
 */
@SpringBootTest
class LearnHubApplicationTest {

    @Autowired
    private CourseRepository repository;

    @Autowired
    private CourseCatalogService catalogService;

    @Autowired
    private CourseManagementService managementService;

    @Test
    void applicationContextLoads() {

        assertNotNull(repository);
        assertNotNull(catalogService);
        assertNotNull(managementService);
    }

    @Test
    void seededRepositoryContainsFiveCourses() {

        assertEquals(5, repository.findAll().size());
    }

    @Test
    void catalogExposesOnlyPublishedCourses() {

        assertEquals(
                4,
                catalogService.summarize().totalCourses()
        );
    }
}
</CodeBlock>

This verifies that the Spring container can assemble our application.

### Step 13.2 — HTTP endpoint integration test

File: `src/test/java/com/learnhub/web/LearnHubStatusControllerTest.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.LearnHubApplication;

import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import static org.junit.jupiter.api.Assertions.*;
/**
 * Starts the application on a random HTTP port and
 * tests the endpoint through the actual web stack.
 */
@SpringBootTest(
        classes = LearnHubApplication.class,
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT
)
class LearnHubStatusControllerTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void statusEndpointReturnsCatalogCounts() {

        ResponseEntity<LearnHubStatusResponse> response =
                restTemplate.getForEntity(
                        "/api/v1/status",
                        LearnHubStatusResponse.class
                );

        assertEquals(
                HttpStatus.OK,
                response.getStatusCode()
        );

        LearnHubStatusResponse body = response.getBody();

        assertNotNull(body);

        assertEquals(
                "LearnHub Backend",
                body.application()
        );

        assertEquals("RUNNING", body.status());
        assertEquals(4, body.publishedCourses());
        assertEquals(2, body.freeCourses());
        assertEquals(2, body.paidCourses());
    }

    @Test
    void actuatorHealthEndpointIsAvailable() {

        ResponseEntity<String> response =
                restTemplate.getForEntity(
                        "/actuator/health",
                        String.class
                );

        assertEquals(
                HttpStatus.OK,
                response.getStatusCode()
        );

        assertNotNull(response.getBody());

        assertTrue(
                response.getBody().contains("\"UP\"")
        );
    }
}
</CodeBlock>

`RANDOM_PORT` prevents tests from requiring port 8080 to be free.

The `TestRestTemplate` sends actual HTTP requests to the test server, rather than directly invoking controller methods.

### Step 13.3 — Run all tests

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean test
```

<box border radius="lg" padding={3} gap={2}>
  <row justify="between" align="center">
    <box gap={1}>
      <text color="secondary" size="xs">Expected total tests</text>
      <title size="2xl" tabularNums>51</title>
    </box>
    <icon name="test-tube-diagonal" size="2xl" color="secondary"/>
  </row>
  <grid columns={2} gap={2}>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={2} gap={1}>
        **46 existing tests**
        <text size="xs" color="secondary">Plain Java unit tests</text>
      </box>
    </grid-item>
    <grid-item>
      <box background="surface-secondary" radius="md" padding={2} gap={1}>
        **5 new tests**
        <text size="xs" color="secondary">Spring context and HTTP integration</text>
      </box>
    </grid-item>
  </grid>
  <caption>Expected if you retained the previous 46 tests and added these five. The count has not been verified against your local files.</caption>
</box>

A useful advantage of our architecture is that most tests still run without starting Spring. Only the integration tests need the Application Context.

## 14. Build and run the executable Spring Boot JAR

Run:

```powershell
mvn clean package

if ($LASTEXITCODE -ne 0) {
    throw "Spring Boot packaging failed."
}
```

The expected application archive is:

```text
target\learnhub-backend-0.1.0-SNAPSHOT.jar
```

Launch it:

```powershell
java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar
```

Spring Boot's Maven plugin repackages the archive with the dependencies and launcher needed to run the application.

This differs from the plain JAR we created in Lesson 1.

### Inspect the archive

In a second terminal:

```powershell
cd $HOME\angular-learning\learnhub-backend

jar tf target\learnhub-backend-0.1.0-SNAPSHOT.jar |
    Select-Object -First 25
```

You'll see Spring Boot archive entries such as:

```text
BOOT-INF/classes/
BOOT-INF/lib/
META-INF/
```

The application classes and runtime dependencies are packaged together.

## 15. Common Spring Boot migration problems

<box gap={3}>
  <box gap={1}>
    **Problem: Port 8080 is already in use**

    Find the process using the port:

    ```powershell
    Get-NetTCPConnection -LocalPort 8080 `
      -State Listen -ErrorAction SilentlyContinue |
      Select-Object LocalPort, OwningProcess
    ```

    Inspect the owning process:

    ```powershell
    Get-Process -Id <PID>
    ```

    Replace `<PID>` with the actual process ID. Stop it only if it's safe, or run LearnHub on another port.
  </box>
  <divider color="subtle"/>
  <box gap={1}>
    **Problem: Multiple CourseRepository beans**

    Check that `InMemoryCourseRepository` does not have `@Repository` when you're also registering it through `CourseRepositoryConfig`.

    ```powershell
    Get-ChildItem src\main\java -Recurse -Filter *.java |
      Select-String '@Repository|@Bean'
    ```

    We want exactly one repository bean in this lesson.
  </box>
  <divider color="subtle"/>
  <box gap={1}>
    **Problem: No qualifying bean of type CourseRepository**

    Confirm that `CourseRepositoryConfig` is under `com.learnhub` and has both `@Configuration` and the `@Bean` method.
  </box>
  <divider color="subtle"/>
  <box gap={1}>
    **Problem: Spring cannot find the controller**

    Confirm that `LearnHubStatusController` is under `com.learnhub.web` and has `@RestController`.

    Also check that the application class is in `com.learnhub`, not a deeper package that excludes the controller from component scanning.
  </box>
  <divider color="subtle"/>
  <box gap={1}>
    **Problem: Maven tests fail after migration**

    Run:

    ```powershell
    mvn clean test
    ```

    Inspect the reports:

    ```powershell
    Get-ChildItem target\surefire-reports -Filter *.txt |
      ForEach-Object {
        Get-Content $_.FullName |
          Select-String 'Tests run:|Failures:|Errors:'
      }
    ```

    Avoid deleting or disabling tests just to make the build pass.
  </box>
  <divider color="subtle"/>
  <box gap={1}>
    **Problem: Java version mismatch**

    Check:

    ```powershell
    java -version
    javac -version
    mvn -version
    ```

    Confirm all tools are using Java 21.
  </box>
</box>

## 16. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}><icon name="settings" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise A — Externalized configuration**

      Add a custom `learnhub.catalog.default-page-size` property to `application.yml`. Read it using `@ConfigurationProperties` or `@Value`, and add a test verifying the configured value.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}><icon name="layers" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise B — Spring bean discovery**

      Create a `CommandLineRunner` bean that prints the number of published courses at startup. Inject `CourseCatalogService` through the bean factory method instead of constructing it manually.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}><icon name="test-tube" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise C — Additional HTTP tests**

      Add a test confirming that `/api/v1/status` returns JSON and that a nonexistent endpoint returns HTTP 404. Keep the test isolated from any externally running server.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}><icon name="git-branch" size="lg"/></box>
    <box flex="1" gap={1}>
      **Exercise D — Spring profiles**

      Create `application-dev.yml` with a different log level. Start the application with `--spring.profiles.active=dev` and verify that the profile is active. Do not place passwords or secrets in profile files committed to Git.
    </box>
  </row>
</box>

## 17. Lesson 5 knowledge quiz

{@body const questions=[{q:"What does @SpringBootApplication provide?",options:["Only an HTTP controller","Spring Boot configuration, auto-configuration, and component scanning","Automatic PostgreSQL schema generation in every project","A replacement for Maven"],correct:1,why:"@SpringBootApplication combines configuration, auto-configuration, and component scanning."},{q:"Why did we register InMemoryCourseRepository with an @Bean method?",options:["Interfaces cannot be used with Spring","We want to construct a seeded instance explicitly without changing the plain Java repository","Spring requires every repository to use @Bean","It makes all repository operations transactional"],correct:1,why:"The factory method calls the existing seeded constructor and registers the result as one repository bean."},{q:"Which dependency injection style is preferred for required service dependencies?",options:["Public mutable fields","Static global variables","Constructor injection","Manual lookups from ApplicationContext inside every method"],correct:2,why:"Constructor injection makes dependencies explicit, supports final fields, and simplifies unit testing."},{q:"What is the default scope of a Spring bean?",options:["Prototype","Request","Session","Singleton"],correct:3,why:"Spring uses singleton scope by default, meaning one bean instance per Application Context."},{q:"Which command starts the Spring Boot app using the Maven plugin?",options:["mvn spring-boot:run","mvn test-only","java Main.java --spring","mvn generate-controller"],correct:0,why:"The Spring Boot Maven plugin provides the spring-boot:run goal."},{q:"What does a successful /actuator/health response guarantee?",options:["Every LearnHub feature and payment workflow works","The application's configured health indicators report UP","All database records are valid","The frontend is connected"],correct:1,why:"Actuator health reports the status of configured health contributors; it is not a comprehensive end-to-end feature test."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
{@body const score=questions.filter((q,i)=>answers[i]===q.correct).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Spring Boot fundamentals quiz**
    <text color="secondary" size="xs">{Object.keys(answers).length} of {questions.length} answered</text>
  </row>
  {#each questions as q,i}
    <box gap={2} key={i}>
      <text weight="medium">{i+1}. {q.q}</text>
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setGraded(false)}}>
        {#each q.options as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if graded}
        <row align="center" gap={2}>
          <icon name={answers[i]===q.correct?"check-circle":"alert-circle"} color={answers[i]===q.correct?"success":"danger"}/>
          <text size="sm" weight="medium">{answers[i]===q.correct?"Correct":"Correct answer: "+q.options[q.correct]}</text>
        </row>
        <text color="secondary" size="sm">{q.why}</text>
      {/if}
    </box>
    {#if i<questions.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if graded}
    <box background="surface-secondary" radius="md" padding={3} gap={1}>
      <title size="xl" tabularNums>{score}/{questions.length}</title>
      <text size="sm">{score===questions.length?"All answers correct.":"Review the explanations and try again."}</text>
    </box>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==questions.length} onClick={()=>setGraded(true)}>Check answers</button>
    <button color="secondary" variant="outline" onClick={()=>{setAnswers({});setGraded(false)}}>Reset</button>
  </row>
</box>

## 18. Lesson completion checklist

{@body const groups=[{name:"Spring Boot setup",items:["Verified JDK 21 and Maven","Updated pom.xml to Spring Boot","Created LearnHubApplication.java","Created application.yml","Understood Spring Initializr versus manual migration"]},{name:"Dependency injection",items:["Registered the seeded repository with @Bean","Added @Service to both application services","Avoided duplicate repository bean registration","Understood constructor injection and bean scopes"]},{name:"HTTP server",items:["Created LearnHubStatusResponse","Created LearnHubStatusController","Started Spring Boot on port 8080","Verified /api/v1/status","Verified /actuator/health"]},{name:"Testing and release",items:["Added Application Context tests","Added HTTP integration tests","Verified expected 51-test suite","Built the executable Spring Boot JAR","Ran the packaged JAR","Completed at least two exercises","Committed Lesson 5 changes"]}]}
{@body const [completed,setCompleted]=DIL.useState({})}
{@body const total=groups.reduce((sum,g)=>sum+g.items.length,0)}
{@body const count=Object.values(completed).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 5 progress**
      <text size="xs" color="secondary">Mark milestones only after verifying them locally.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" radius="full" height="8px" clip>
    <box background="rgba(109,40,217,0.85)" radius="full" height="100%" width={`${count/total*100}%`}/>
  </box>
  {#each groups as group,i}
    <box gap={2} key={i}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!completed[`${i}-${j}`]} onChange={v=>setCompleted(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if count===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 5 checklist completed.**
    </row>
  {/if}
</box>

## 19. Final verification and Git checkpoint

Run these commands from PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

# Verify compilation and automated tests.
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Lesson 5 Maven verification failed."
}

# Inspect the generated executable JAR.
Get-ChildItem target\*.jar

# Review project changes.
git status

# Commit the Spring Boot migration.
git add .
git commit -m "Migrate LearnHub backend to Spring Boot"
```

If the application is still running in another terminal, stop it with `Ctrl+C` when you're finished testing.

## 20. Next lesson — Build the Course REST API

<row align="start" gap={3}>
  <box background="surface-secondary" radius="lg" padding={3}>
    <icon name="network" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 6 — REST APIs with Spring MVC**

    We'll build the first real LearnHub course endpoints using `@RestController`, `@GetMapping`, `@PostMapping`, `@PathVariable`, `@RequestParam`, DTOs, JSON serialization, validation, and HTTP status codes.

    Our target will be a course API that can eventually replace the Angular frontend's `json-server` mock.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 6: REST APIs with Spring MVC, Course CRUD endpoints, @RestController, @GetMapping, @PostMapping, @PutMapping, @DeleteMapping, DTOs, validation, HTTP status codes, and PowerShell API testing. Continue from Lesson 5 with experienced-Java pacing, complete file paths, documented code, tests, exercises, quiz, and checklist. Align the course API with the existing Angular LearnHub frontend.")}>Start Lesson 6 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 5 deliverable:** Your existing LearnHub Java project is now a Spring Boot application with dependency injection, externalized configuration, a running embedded HTTP server, Actuator health monitoring, and integration-tested JSON responses.

The foundation is ready for REST endpoints, PostgreSQL, authentication, enrollments, and the other backend capabilities required by your Angular learning platform.