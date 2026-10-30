# LearnHub Java & Spring Boot — Lesson 23
<text color="secondary" size="sm">Docker · Java 21 · Angular · PostgreSQL · Flyway · Nginx · GitHub Actions · CI/CD · Production Deployment</text>

**Level:** Experienced Java developer  
**Estimated time:** 12–16 hours, including deployment exercises  
**Starting point:** Lesson 22's tested and observable LearnHub application  
**Goal:** Package LearnHub into reproducible containers, run the full stack with Docker Compose, and establish a secure build-and-deploy pipeline.

This is the penultimate lesson in our 24-lesson backend roadmap. Lesson 24 will bring the entire LearnHub marketplace together as a capstone.

## 1. What we're deploying

LearnHub has three essential runtime components:

1. An Angular standalone frontend, served by Nginx.
2. A Spring Boot Java 21 backend, responsible for authentication, courses, enrollment, payments, and API access.
3. PostgreSQL, with its schema managed by Flyway.

We'll use a **single public origin**. Nginx serves Angular and proxies `/api/` to Spring Boot, preserving the session-cookie and CSRF design established in Lessons 14 and 16.

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    <icon name="globe" size="lg"/>
    **Browser**
    <text size="xs" color="secondary">https://learnhub.example.com</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} gap={2}>
    <row justify="between" align="center">
      **Nginx / Angular container**
      <badge color="success">Public</badge>
    </row>
    <grid columns={2} gap={2}>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2} align="center">
          **`/`**
          <text color="secondary" size="xs">Angular SPA</text>
        </box>
      </grid-item>
      <grid-item>
        <box background="surface-secondary" radius="md" padding={2} align="center">
          **`/api/`**
          <text color="secondary" size="xs">Reverse proxy</text>
        </box>
      </grid-item>
    </grid>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring Boot container**
    <text size="xs" color="secondary">Port 8080 · private Docker network</text>
    <text size="xs" color="secondary">Flyway · Security · JPA · Actuator</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" radius="lg" padding={3} align="center" gap={1}>
    <icon name="database" size="lg"/>
    **PostgreSQL**
    <text size="xs" color="secondary">Persistent volume · no public port</text>
  </box>
</box>

The payment provider communicates with the backend through Nginx at `/api/v1/webhooks/stripe`. The webhook uses signature verification, not a browser session.

**Important:** The initial Compose configuration is a **local production-like environment**. Publishing it directly on the internet requires TLS termination, secure secrets, backups, monitoring, and deployment controls.

---

## 2. Workspace and baseline verification

Your existing layout is:

```text
$HOME\angular-learning\
├── learnhub\                  # Angular frontend
└── learnhub-backend\          # Spring Boot backend
```

We'll create a deployment directory alongside them:

```text
$HOME\angular-learning\
├── learnhub\
├── learnhub-backend\
└── learnhub-deploy\
    ├── compose.yaml
    ├── .env.example
    ├── .gitignore
    └── scripts\
```

### PowerShell setup

```powershell
$workspace = Join-Path $HOME "angular-learning"
$frontend = Join-Path $workspace "learnhub"
$backend = Join-Path $workspace "learnhub-backend"
$deploy = Join-Path $workspace "learnhub-deploy"

New-Item -ItemType Directory -Force `
  $deploy, `
  (Join-Path $deploy "scripts") |
  Out-Null

docker --version
docker compose version
docker info

Set-Location $backend
java -version
mvn -version
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend baseline failed."
}

Set-Location $frontend
node --version
npm --version
npm run build

if ($LASTEXITCODE -ne 0) {
    throw "Angular baseline failed."
}
```

The commands assume Docker Desktop is running with Linux containers enabled.

---

## 3. Docker fundamentals for LearnHub

A Dockerfile is a recipe for building an image. A container is a running instance of that image.

For production-style images, we want:

- **Multi-stage builds:** build tools do not ship in runtime images.
- **Non-root runtime users:** reduce the impact of a compromised process.
- **Small runtime footprint:** fewer unnecessary packages.
- **Immutable artifacts:** deploy the same image that passed tests.
- **Externalized configuration:** no database passwords or payment secrets baked into images.
- **Health checks:** report readiness rather than assuming a started process is usable.

### Image versus container versus volume

| Concept | LearnHub example | Purpose |
|---|---|---|
| Image | `learnhub-backend:lesson23` | Packaged Java application |
| Container | `learnhub-backend-1` | Running backend instance |
| Volume | `learnhub_pgdata` | Persistent PostgreSQL files |
| Network | Compose internal network | Backend/database communication |
| Environment variable | `LEARNHUB_DB_URL` | Runtime configuration |
| Secret | Stripe API key | Sensitive runtime credential |

A Docker image should be rebuildable from source. A PostgreSQL volume is stateful and must be backed up separately.

---

## 4. Backend multi-stage Dockerfile

**Create:** `learnhub-backend/Dockerfile`

<CodeBlock language="dockerfile" editable>
# syntax=docker/dockerfile:1

# --------------------------------------------------
# Stage 1: Build the Spring Boot application
# --------------------------------------------------
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /workspace

# Cache Maven dependencies when application source changes.
COPY pom.xml ./

RUN mvn -B -ntp dependency:go-offline

COPY src ./src

# Tests run in CI, including Testcontainers.
# The image build packages the already-reviewed source.
RUN mvn -B -ntp -DskipTests package


# --------------------------------------------------
# Stage 2: Java 21 runtime
# --------------------------------------------------
FROM eclipse-temurin:21-jre

WORKDIR /app

# Create a dedicated unprivileged application user.
RUN groupadd --system learnhub \
    && useradd --system \
       --gid learnhub \
       --home-dir /app \
       --no-create-home learnhub

COPY --from=build \
    --chown=learnhub:learnhub \
    /workspace/target/*.jar \
    /app/app.jar

USER learnhub

EXPOSE 8080

ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0"

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
</CodeBlock>

### Why two stages?

The Maven stage contains compilers, source files, and build dependencies. The runtime stage contains the Java runtime and the built application.

This reduces image size and removes Maven from the running container.

### Important packaging caveat

The wildcard `/workspace/target/*.jar` assumes exactly one deployable JAR. If Maven produces an additional `*-plain.jar` or another artifact, make the copy path explicit.

Also, the final image is Linux-based. Building it requires Docker Desktop configured for Linux containers.

### `.dockerignore`

**Create:** `learnhub-backend/.dockerignore`

<CodeBlock language="text" editable>
.git
.github
.idea
.vscode
target
*.log
.env
.env.*
!.env.example
README.local.md
</CodeBlock>

Do not put database credentials, Stripe secrets, or production certificates in the image build context.

### Build locally

```powershell
cd $HOME\angular-learning\learnhub-backend

docker build `
  --tag learnhub-backend:lesson23 `
  .
```

---

## 5. Angular multi-stage Dockerfile

Angular builds into static assets. Nginx serves those assets and proxies API requests.

**Create:** `learnhub/Dockerfile`

<CodeBlock language="dockerfile" editable>
# syntax=docker/dockerfile:1

# --------------------------------------------------
# Stage 1: Build Angular
# --------------------------------------------------
FROM node:22-alpine AS build

WORKDIR /workspace

COPY package.json package-lock.json ./

RUN npm ci

COPY . .

RUN npm run build -- --configuration production


# --------------------------------------------------
# Stage 2: Serve static assets
# --------------------------------------------------
FROM nginx:1.27-alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf

# The Angular output path must match angular.json.
COPY --from=build \
    /workspace/dist/learnhub/browser/ \
    /usr/share/nginx/html/

EXPOSE 80
</CodeBlock>

The Angular output path varies by builder and `angular.json` configuration. Confirm it locally:

```powershell
cd $HOME\angular-learning\learnhub

Get-Content angular.json |
    Select-String -Pattern "outputPath" -Context 2,4

npm run build

Get-ChildItem dist -Recurse -Filter index.html |
    Select-Object FullName
```

If your output is `dist/learnhub/` rather than `dist/learnhub/browser/`, adjust the Dockerfile accordingly.

### `.dockerignore`

**Create:** `learnhub/.dockerignore`

<CodeBlock language="text" editable>
.git
.github
.angular
node_modules
dist
coverage
.env
.env.*
!.env.example
*.log
</CodeBlock>

---

## 6. Nginx reverse proxy and Angular routing

**Create:** `learnhub/nginx.conf`

<CodeBlock language="nginx" editable>
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    server_tokens off;
    client_max_body_size 1m;

    # Forward API requests to Spring Boot without
    # rewriting the /api prefix.
    location /api/ {
        proxy_pass http://backend:8080;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For
            $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 30s;
    }

    # Local readiness check for the frontend container.
    location = /nginx-health {
        access_log off;
        default_type text/plain;
        return 200 "ok";
    }

    # Do not expose backend operational endpoints
    # through the public frontend.
    location /actuator/ {
        return 404;
    }

    location /v3/api-docs {
        return 404;
    }

    location /swagger-ui {
        return 404;
    }

    # Cache versioned Angular assets.
    location ~* \.(?:js|css|woff2?|png|jpg|jpeg|svg|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # Angular client-side routes must load index.html.
    location / {
        try_files $uri $uri/ /index.html;
    }
}
</CodeBlock>

### Why the trailing slash matters

Here:

```nginx
proxy_pass http://backend:8080;
```

Nginx forwards `/api/v1/courses` unchanged.

If you instead write `proxy_pass http://backend:8080/;`, Nginx can replace the matched `/api/` prefix and forward `/v1/courses`, breaking the backend route.

### Security implications

The browser sees one origin, so the Angular session and XSRF design remains straightforward.

However, Nginx does not itself provide TLS in this example. For internet deployment, terminate HTTPS at a trusted load balancer or ingress, configure trusted forwarded headers, and ensure session cookies are marked `Secure`.

A public reverse proxy should also apply appropriate rate limits, request-size limits, and timeouts. Payment webhooks may need a different body-size or timeout policy from general API routes.

### Container image health check

The stock Nginx image may not contain `curl`. We'll use `wget` in the Compose health check, which is available in the Alpine-based image.

---

## 7. PostgreSQL and the full Docker Compose stack

We will run all three containers on a private Compose network. Only Nginx will publish a host port.

### 7.1 Environment template

**Create:** `learnhub-deploy/.env.example`

<CodeBlock language="dotenv" editable>
# Local development only.
# Never commit a populated .env file.

POSTGRES_DB=learnhub
POSTGRES_USER=learnhub_app
POSTGRES_PASSWORD=replace-with-a-long-random-password

LEARNHUB_PUBLIC_URL=http://localhost:8088

# Test-mode credentials only for local integration.
STRIPE_SECRET_KEY=replace-with-stripe-test-secret
STRIPE_WEBHOOK_SECRET=replace-with-test-webhook-secret

# Pin image tags for deployment.
BACKEND_IMAGE=learnhub-backend:lesson23
FRONTEND_IMAGE=learnhub-frontend:lesson23
</CodeBlock>

**Create:** `learnhub-deploy/.gitignore`

<CodeBlock language="gitignore" editable>
.env
.env.*
!.env.example
backups/
*.dump
*.sql.gz
secrets/
</CodeBlock>

Copy the template:

```powershell
cd $HOME\angular-learning\learnhub-deploy

Copy-Item .env.example .env

notepad .env
```

Use unique values. The placeholders will not work for real checkout requests.

### 7.2 Compose configuration

**Create:** `learnhub-deploy/compose.yaml`

<CodeBlock language="yaml" editable>
name: learnhub

services:
  db:
    image: postgres:17
    restart: unless-stopped

    environment:
      POSTGRES_DB: ${POSTGRES_DB:?Required}
      POSTGRES_USER: ${POSTGRES_USER:?Required}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Required}

    volumes:
      - pgdata:/var/lib/postgresql/data

    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"
      interval: 5s
      timeout: 3s
      retries: 12
      start_period: 10s

    networks:
      - internal

  backend:
    image: ${BACKEND_IMAGE:-learnhub-backend:lesson23}
    build:
      context: ../learnhub-backend

    restart: unless-stopped

    environment:
      SPRING_PROFILES_ACTIVE: production

      LEARNHUB_DB_URL: jdbc:postgresql://db:5432/${POSTGRES_DB}
      LEARNHUB_DB_USER: ${POSTGRES_USER}
      LEARNHUB_DB_PASSWORD: ${POSTGRES_PASSWORD}

      # Match these names to Lesson 20's payment properties.
      LEARNHUB_PAYMENTS_STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY:?Required}
      LEARNHUB_PAYMENTS_STRIPE_WEBHOOK_SECRET: ${STRIPE_WEBHOOK_SECRET:?Required}
      LEARNHUB_PAYMENTS_PUBLIC_BASE_URL: ${LEARNHUB_PUBLIC_URL:?Required}

      SERVER_FORWARD_HEADERS_STRATEGY: framework

    depends_on:
      db:
        condition: service_healthy

    healthcheck:
      test:
        - CMD-SHELL
        - >
          java -cp /app/app.jar
          org.springframework.boot.loader.launch.JarLauncher
          --help >/dev/null 2>&1
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 90s

    networks:
      - internal

  frontend:
    image: ${FRONTEND_IMAGE:-learnhub-frontend:lesson23}
    build:
      context: ../learnhub

    restart: unless-stopped

    ports:
      - "127.0.0.1:8088:80"

    depends_on:
      backend:
        condition: service_started

    healthcheck:
      test:
        - CMD-SHELL
        - wget -q -O /dev/null http://127.0.0.1/nginx-health
      interval: 15s
      timeout: 3s
      retries: 3

    networks:
      - internal

volumes:
  pgdata:

networks:
  internal:
    driver: bridge
</CodeBlock>

### Critical correction: backend health check

The Java command above **does not check HTTP readiness**. It only attempts to start the Spring Boot launcher and is unsuitable as a real health check. Do not use it as a deployment gate.

Instead, add a tiny Java readiness probe to the backend image.

**Create:** `learnhub-backend/docker/ReadinessProbe.java`

<CodeBlock language="java" editable>
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;

public final class ReadinessProbe {

    public static void main(String[] args)
            throws Exception {

        try (HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(2))
                .build()) {

            HttpRequest request = HttpRequest.newBuilder(
                    URI.create(
                        "http://127.0.0.1:8080/actuator/health/readiness"
                    )
            ).timeout(Duration.ofSeconds(3))
             .GET()
             .build();

            HttpResponse<Void> response = client.send(
                    request,
                    HttpResponse.BodyHandlers.discarding()
            );

            if (response.statusCode() != 200) {
                System.exit(1);
            }
        }
    }
}
</CodeBlock>

Compile this class in the Docker build stage and copy the resulting class into the runtime image.

Add to the **build stage** of the backend Dockerfile, after the application package step:

<CodeBlock language="dockerfile" editable>
COPY docker/ReadinessProbe.java /workspace/ReadinessProbe.java

RUN javac \
    -d /workspace/probe \
    /workspace/ReadinessProbe.java
</CodeBlock>

Add to the **runtime stage**, before `USER learnhub`:

<CodeBlock language="dockerfile" editable>
COPY --from=build \
    --chown=learnhub:learnhub \
    /workspace/probe/ReadinessProbe.class \
    /app/probe/ReadinessProbe.class
</CodeBlock>

Replace the backend Compose health check command with:

<CodeBlock language="yaml" editable>
healthcheck:
  test:
    - CMD
    - java
    - -cp
    - /app/probe
    - ReadinessProbe
  interval: 15s
  timeout: 5s
  retries: 6
  start_period: 90s
</CodeBlock>

Also change the frontend dependency to wait for actual backend readiness:

<CodeBlock language="yaml" editable>
depends_on:
  backend:
    condition: service_healthy
</CodeBlock>

**Why this approach?** We avoid installing `curl` or `wget` in the Java runtime image while checking the actual Spring Boot readiness endpoint.

The Java probe uses the JDK HTTP client. The runtime image must contain the required Java modules; Eclipse Temurin's standard JRE image normally does.

### 7.3 Compose secrets caveat

Environment variables are convenient for local development but can be exposed through container inspection, process diagnostics, or orchestration metadata.

For production, prefer a managed secret store or Docker/Kubernetes secret mounts, and configure Spring Boot to read credentials from mounted files or an approved secrets integration.

Do not mistake a local `.env` file for a production secret-management system.

---

## 8. Flyway migration safety

The backend should apply Flyway migrations before serving traffic.

In Lesson 9, the application was configured with:

```yaml
spring:
  flyway:
    enabled: true

  jpa:
    hibernate:
      ddl-auto: validate
```

Keep both settings.

Flyway creates or updates the schema. Hibernate validates that the entity mappings match the resulting database schema.

### Startup sequence

<box border radius="lg" padding={3} gap={2}>
  <row align="center" gap={2}>
    <icon name="database" color="secondary"/>
    **1. PostgreSQL becomes healthy**
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row align="center" gap={2}>
    <icon name="files" color="secondary"/>
    **2. Flyway validates and applies pending migrations**
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row align="center" gap={2}>
    <icon name="check-circle" color="secondary"/>
    **3. Hibernate validates the schema**
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row align="center" gap={2}>
    <icon name="heart-pulse" color="secondary"/>
    **4. Spring Boot reports readiness**
  </row>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <row align="center" gap={2}>
    <icon name="globe" color="secondary"/>
    **5. Nginx begins forwarding requests**
  </row>
</box>

### Production migration strategy

For a single local backend container, startup migrations are acceptable.

For multiple production replicas, prefer a separate migration job that runs before the new application version is rolled out. That gives you an explicit migration stage and reduces the chance of multiple instances racing during deployment.

Flyway uses schema-history locking, but that is not a substitute for a controlled release process.

### Expand-and-contract migrations

Suppose Lesson 24 changes `courses.description` to a new structured content model.

Do not deploy a migration that immediately drops the old column while old backend instances are still running.

Instead:

1. **Expand:** Add new columns or tables while retaining old ones.
2. **Deploy compatible code:** Read/write both representations as needed.
3. **Backfill:** Migrate existing records.
4. **Switch readers:** Use the new representation.
5. **Contract:** Remove obsolete schema only after old application versions are retired.

Database rollback is often more dangerous than application rollback. Design migrations to preserve compatibility across releases.

---

## 9. Build and run the complete stack

### 9.1 Validate Compose

```powershell
cd $HOME\angular-learning\learnhub-deploy

docker compose config --quiet

if ($LASTEXITCODE -ne 0) {
    throw "Compose configuration is invalid."
}
```

Avoid posting the unredacted output of `docker compose config` because it can contain resolved environment variables and secrets.

### 9.2 Build and start

```powershell
docker compose build

if ($LASTEXITCODE -ne 0) {
    throw "Docker build failed."
}

docker compose up -d

if ($LASTEXITCODE -ne 0) {
    throw "Docker Compose startup failed."
}
```

### 9.3 Check status

```powershell
docker compose ps

docker compose logs --tail=100 backend
docker compose logs --tail=100 frontend
docker compose logs --tail=100 db
```

Look for Flyway completion, Hibernate validation, and readiness status.

### 9.4 Test the frontend and API

```powershell
$origin = "http://localhost:8088"

Invoke-WebRequest "$origin/" -UseBasicParsing |
    Select-Object StatusCode

Invoke-RestMethod "$origin/api/v1/courses?page=0&size=5"
```

Open `http://localhost:8088` in your browser.

Because we bound Nginx to `127.0.0.1`, it is reachable from your own machine, not directly from other machines on the network.

### 9.5 Verify container health

```powershell
docker inspect `
  --format '{{json .State.Health}}' `
  learnhub-backend-1
```

The generated container name can vary. Use `docker compose ps -q backend` for a stable approach:

```powershell
$containerId = docker compose ps -q backend

docker inspect `
  --format '{{.State.Health.Status}}' `
  $containerId
```

Expected after startup:

```text
healthy
```

### 9.6 Stop without deleting data

```powershell
docker compose down
```

This stops and removes containers and networks, but retains the named PostgreSQL volume.

**Do not run `docker compose down -v` against a database containing data you want to keep.** The `-v` flag removes Compose-managed volumes.

---

## 10. Production configuration and secure cookies

The backend must not trust arbitrary forwarded headers from untrusted clients.

We configured:

```yaml
SERVER_FORWARD_HEADERS_STRATEGY: framework
```

This is appropriate only when the backend is reachable exclusively through a trusted proxy that controls forwarded headers.

For production HTTPS, make sure the external load balancer forwards the original scheme correctly and does not accept spoofed `X-Forwarded-*` values from clients.

### Secure session cookies

In a production profile, configure:

**Create:** `learnhub-backend/src/main/resources/application-production.yml`

<CodeBlock language="yaml" editable>
server:
  servlet:
    session:
      cookie:
        http-only: true
        secure: true
        same-site: lax

spring:
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate

  flyway:
    enabled: true

springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false

management:
  endpoint:
    health:
      show-details: never
      probes:
        enabled: true
      group:
        readiness:
          include: readinessState,db

  endpoints:
    web:
      exposure:
        include: health,info,prometheus
</CodeBlock>

**Important local-development caveat:** This profile sets `Secure` cookies, which browsers normally require to be used over HTTPS. Our local Compose example uses HTTP at `localhost:8088`.

To keep local development reliable, use a separate `compose` or `local-container` profile with `secure: false`, while retaining `secure: true` in production. Do not weaken production cookie security merely to make the HTTP Compose demo work.

For example, change the local Compose backend environment to:

```yaml
SPRING_PROFILES_ACTIVE: local-container
```

Then create:

**File:** `src/main/resources/application-local-container.yml`

<CodeBlock language="yaml" editable>
server:
  servlet:
    session:
      cookie:
        http-only: true
        secure: false
        same-site: lax

springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false

management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        readiness:
          include: readinessState,db
</CodeBlock>

The base `application.yml` should continue to supply the PostgreSQL, Flyway, and other common settings.

For actual deployment, use the `production` profile behind HTTPS.

---

## 11. GitHub Actions CI: build, test, and package

CI should run on every pull request before images are published.

We'll create separate workflows for backend and frontend, then a release workflow for container images.

### 11.1 Backend CI

**Create:** `learnhub-backend/.github/workflows/backend-ci.yml`

<CodeBlock language="yaml" editable>
name: Backend CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Check Docker availability
        run: docker info

      - name: Run Maven verification
        run: mvn -B -ntp clean verify
      - name: Upload test reports on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: backend-test-reports
          path: |
            target/surefire-reports/
            target/failsafe-reports/
          if-no-files-found: ignore

      - name: Build Docker image
        run: docker build -t learnhub-backend:ci .
</CodeBlock>

GitHub-hosted Ubuntu runners generally provide Docker. Testcontainers uses the runner's Docker daemon.

### Testcontainers CI requirements

Ensure tests use PostgreSQL Testcontainers, not a hardcoded local database URL.

Your Lesson 21 base class supplies database properties through `@DynamicPropertySource`.

The test suite must not depend on:

- A manually started PostgreSQL service.
- A developer's local `.env` file.
- A locally installed Stripe CLI.
- Real Stripe network calls.

Use a fake checkout provider and locally signed webhook payloads in tests.

### 11.2 Angular CI

**Create:** `learnhub/.github/workflows/frontend-ci.yml`

<CodeBlock language="yaml" editable>
name: Frontend CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test-and-build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run Angular tests
        run: npm test -- --watch=false

      - name: Build Angular
        run: npm run build -- --configuration production

      - name: Build frontend image
        run: docker build -t learnhub-frontend:ci .
</CodeBlock>

If your Angular test runner requires Chrome, install or configure a headless browser in CI. The exact command depends on whether your project uses Karma, Vitest, or another runner.

**Do not disable failing tests in CI to make deployment green.**

---

## 12. Container image publishing

Once CI passes, publish immutable images to a registry.

We'll use GitHub Container Registry (GHCR) as an example.

### 12.1 Backend image release

**Create:** `learnhub-backend/.github/workflows/publish-image.yml`

<CodeBlock language="yaml" editable>
name: Publish Backend Image

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: read
  packages: write

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Verify tests
        run: mvn -B -ntp clean verify

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and publish
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}:${{ github.sha }}
</CodeBlock>

This workflow publishes an image when a version tag is pushed. It does **not** deploy it.

### 12.2 Frontend image release

Create a corresponding workflow in the Angular repository, with the same test-before-publish rule.

Use the repository's own image name and immutable SHA tag.

### 12.3 Security improvements for a production pipeline

Pin GitHub Actions to reviewed commit SHAs instead of mutable major-version tags, generate an SBOM, scan dependencies and container images, and sign published images where your registry and deployment tooling support it.

Use minimal GitHub token permissions. Avoid long-lived registry passwords.

### 12.4 Avoid deploying `latest`

A mutable `latest` tag is convenient for development but unreliable for rollback.

Prefer a deployment manifest containing exact image digests:

```text
ghcr.io/organization/learnhub-backend@sha256:...
ghcr.io/organization/learnhub-frontend@sha256:...
```

The digest uniquely identifies the artifact that was tested and approved.

---

## 13. Deployment pipeline and release gates

Publishing an image is not the same as deploying it.

A safe LearnHub release has explicit gates:

<box border radius="xl" padding={3} gap={2}>
  {#each [{t:"Pull request",d:"Review code, configuration, and migrations"},{t:"CI verification",d:"JUnit, MockMvc, Testcontainers, Angular tests"},{t:"Build and publish",d:"Immutable backend and frontend images"},{t:"Staging deployment",d:"Run Flyway, readiness checks, smoke tests"},{t:"Approval",d:"Review schema changes and payment behavior"},{t:"Production rollout",d:"Deploy images, monitor errors and latency"},{t:"Post-deploy verification",d:"Check login, catalog, enrollment, checkout, webhooks"}] as step,i}
    <row gap={3} align="start" key={i}>
      <box background="surface-secondary" radius="md" padding={2}>
        <icon name={i===0?"git-pull-request":i===1?"test-tube":i===2?"package":i===3?"server":i===4?"shield-check":i===5?"rocket":"check-circle"} color="secondary"/>
      </box>
      <box flex="1" gap={1}>
        **{step.t}**
        <text color="secondary" size="sm">{step.d}</text>
      </box>
    </row>
    {#if i<6}<divider color="subtle"/>{/if}
  {/each}
</box>

### Deployment options

| Strategy | Advantages | Trade-offs |
|---|---|---|
| Recreate | Simplest, suitable for local Compose | Brief downtime |
| Rolling | Incremental replacement | Old/new versions must coexist |
| Blue-green | Fast traffic switch and rollback | Extra capacity, schema compatibility |
| Canary | Limits exposure to a small share of traffic | More routing and monitoring complexity |

For the first LearnHub deployment, a controlled single-instance release is reasonable. For a higher-availability deployment, use a platform with health-gated rolling or blue-green deployments.

### Payment-specific deployment requirement

Never deploy a new webhook processor that silently ignores events from the previous version. Payment providers retry deliveries, and events can arrive late or out of order.

Keep webhook handlers backward-compatible, persist event IDs, and maintain idempotent fulfillment.

---

## 14. PostgreSQL backup and restore

A Docker volume protects data from ordinary container replacement. It is **not a backup**.

Backups must be stored outside the database container and, ideally, outside the deployment host.

### 14.1 Create a backup

**Create:** `learnhub-deploy/scripts/backup.ps1`

<CodeBlock language="powershell" editable>
param(
    [string]$OutputDirectory = (
        Join-Path $PSScriptRoot "..\backups"
    )
)

$ErrorActionPreference = "Stop"

New-Item -ItemType Directory `
    -Force `
    -Path $OutputDirectory |
    Out-Null

$timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
$backupPath = Join-Path `
    $OutputDirectory `
    "learnhub-$timestamp.dump"

Push-Location (Join-Path $PSScriptRoot "..")

try {
    $containerId = docker compose ps -q db

    if (-not $containerId) {
        throw "Database container is not running."
    }

    $database = docker compose exec -T db `
        printenv POSTGRES_DB

    $username = docker compose exec -T db `
        printenv POSTGRES_USER

    $database = $database.Trim()
    $username = $username.Trim()

    # Stream pg_dump's binary custom-format output
    # directly to a host file without text encoding.
    $command = @(
        "docker", "compose", "exec", "-T",
        "db", "pg_dump",
        "-U", $username,
        "-d", $database,
        "-Fc",
        "--no-owner",
        "--no-acl"
    )

    $process = Start-Process `
        -FilePath $command[0] `
        -ArgumentList $command[1..($command.Length - 1)] `
        -NoNewWindow `
        -Wait `
        -PassThru `
        -RedirectStandardOutput $backupPath

    if ($process.ExitCode -ne 0) {
        Remove-Item $backupPath -ErrorAction SilentlyContinue
        throw "pg_dump failed with exit code $($process.ExitCode)."
    }

    Write-Host "Backup created: $backupPath"
}
finally {
    Pop-Location
}
</CodeBlock>

This example uses `Start-Process` to avoid PowerShell text-pipeline encoding of binary `pg_dump -Fc` output. Validate the resulting archive with `pg_restore --list` before relying on it.

**Windows caveat:** PowerShell's native-process argument handling differs across versions. Test this script in your installed PowerShell version. A production backup job is better implemented in a controlled Linux environment or managed database backup service.

### 14.2 Simpler alternative: write inside the container

For a local exercise, this method avoids streaming binary data through PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-deploy

docker compose exec -T db sh -c `
  'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc -f /tmp/learnhub.dump'

$databaseContainer = docker compose ps -q db

New-Item -ItemType Directory -Force .\backups |
    Out-Null

docker cp `
  "${databaseContainer}:/tmp/learnhub.dump" `
  ".\backups\learnhub.dump"

docker compose exec -T db `
  pg_restore --list /tmp/learnhub.dump
```

This temporarily writes the backup inside the database container. Copy it off the host to secure storage and remove the temporary file afterward.

### 14.3 Restore procedure

**Never restore over a running production database without a tested recovery plan.**

For a controlled local restoration exercise:

1. Stop the frontend and backend.
2. Create a fresh PostgreSQL database or volume.
3. Copy the backup archive into the database container.
4. Restore with `pg_restore`.
5. Validate Flyway history, user counts, enrollments, payments, and referential integrity.
6. Restart the backend and run smoke tests.

Example commands for a **fresh, empty test database**:

```powershell
cd $HOME\angular-learning\learnhub-deploy

$databaseContainer = docker compose ps -q db

docker cp `
  ".\backups\learnhub.dump" `
  "${databaseContainer}:/tmp/learnhub.dump"

docker compose exec -T db sh -c `
  'pg_restore -U "$POSTGRES_USER" -d "$POSTGRES_DB" --no-owner --no-acl /tmp/learnhub.dump'
```

Do not run that restore against an already populated LearnHub database without planning for existing objects and data.

### Backup requirements

A production backup policy should specify recovery point objective (RPO), recovery time objective (RTO), encryption, offsite retention, restore testing, and point-in-time recovery.

A successful `pg_dump` exit code does not prove that you can recover the application.

---

## 15. Rollback strategy

A deployment rollback and a database rollback are different operations.

### Application rollback

If the new release fails but the schema remains backward-compatible:

1. Stop routing traffic to the failed release.
2. Restore the previous approved image digest.
3. Wait for readiness.
4. Run login, catalog, enrollment, and payment-status smoke tests.
5. Monitor webhook retries and errors.

### Database rollback

If a migration has dropped data or changed semantics incompatibly, redeploying an old JAR may not work.

Prefer forward-fixing the schema or restoring from a verified backup according to the recovery plan.

### Example release record

<CodeBlock language="yaml" editable>
release:
  version: v1.0.0
  backendImage: ghcr.io/example/learnhub-backend@sha256:REPLACE
  frontendImage: ghcr.io/example/learnhub-frontend@sha256:REPLACE

database:
  migrationBaseline: V8
  backupVerified: true
  backwardCompatible: true

verification:
  readiness: passed
  publicCatalog: passed
  authentication: passed
  enrollment: passed
  checkoutTestMode: passed
  webhookSignature: passed
</CodeBlock>

The values above are illustrative placeholders, not actual image digests or verified results.

---

## 16. Deployment and configuration tests

Lesson 21 established a testing pyramid. Lesson 23 adds deployment-specific checks.

### 16.1 Verify production security configuration

**Create:** `learnhub-backend/src/test/java/com/learnhub/deployment/ProductionSecurityIntegrationTest.java`

<CodeBlock language="java" editable>
package com.learnhub.deployment;

import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles({"test", "production"})
class ProductionSecurityIntegrationTest
        extends PostgresIntegrationTest {

    @Autowired
    MockMvc mvc;

    @Test
    void swaggerIsUnavailableInProduction()
            throws Exception {

        mvc.perform(get("/swagger-ui/index.html"))
                .andExpect(status().isNotFound());
    }

    @Test
    void openApiDocumentIsUnavailableInProduction()
            throws Exception {

        mvc.perform(get("/v3/api-docs"))
                .andExpect(status().isNotFound());
    }

    @Test
    void readinessIsAvailable()
            throws Exception {

        mvc.perform(get("/actuator/health/readiness"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("UP"));
    }
}
</CodeBlock>

**Test configuration correction:** The `test` profile from Lesson 22 enables OpenAPI. Combining `test` and `production` can result in profile-order-dependent behavior. For a reliable production test, use only `production` and keep the Testcontainers database properties supplied by the base class. Remove `@ActiveProfiles({"test", "production"})` and replace it with:

```java
@ActiveProfiles("production")
```

The base class's own `@ActiveProfiles("test")` may still be inherited. To avoid that ambiguity completely, extract the Testcontainers database setup into a profile-neutral base class and have this test extend it.

This is a useful exercise in itself: **production configuration tests must actually load production configuration**.

### 16.2 Docker smoke test

**Create:** `learnhub-deploy/scripts/smoke-test.ps1`

<CodeBlock language="powershell" editable>
param(
    [string]$Origin = "http://localhost:8088"
)

$ErrorActionPreference = "Stop"

Write-Host "Checking frontend..."

$frontend = Invoke-WebRequest `
    -Uri "$Origin/" `
    -UseBasicParsing

if ($frontend.StatusCode -ne 200) {
    throw "Frontend failed."
}

Write-Host "Checking public catalog..."

$catalog = Invoke-RestMethod `
    -Uri "$Origin/api/v1/courses?page=0&size=5"

if ($null -eq $catalog.items) {
    throw "Catalog response has no items field."
}

Write-Host "Checking unauthenticated API security..."

try {
    Invoke-WebRequest `
        -Uri "$Origin/api/v1/auth/me" `
        -UseBasicParsing `
        -ErrorAction Stop

    throw "Anonymous auth/me unexpectedly succeeded."
}
catch {
    if ($_.Exception.Response.StatusCode.value__ -ne 401) {
        throw
    }
}

Write-Host "Smoke tests passed."
</CodeBlock>

This tests frontend availability, public API routing, and an important security boundary. Extend it with authenticated session and test-mode payment flows in staging.

### 16.3 CI/CD verification matrix

| Test | Local | CI | Staging | Production |
|---|---|---|---|---|
| Java unit tests | Yes | Yes | — | — |
| Testcontainers integration | Yes | Yes | — | — |
| Angular tests | Yes | Yes | — | — |
| Docker build | Yes | Yes | — | — |
| Compose smoke test | Yes | Optional | Yes | — |
| Flyway validation | Yes | Yes | Yes | Yes |
| Readiness probe | Yes | Optional | Yes | Yes |
| Session/CSRF journey | Yes | Yes | Yes | Smoke only |
| Stripe test-mode checkout | Yes | Mocked | Yes | No live charge |
| Backup restoration | Exercise | Scheduled | Yes | Periodically tested |

Do not run destructive tests or real payment charges against production.

---

## 17. Hands-on exercises

<box gap={3}>
  <row gap={3} align="start">
    <icon name="container" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise A — Build both images**

      Build the Java and Angular images, inspect their sizes, confirm that the backend runs as a non-root user, and verify that neither image contains `.env` files or secret values.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="network" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise B — Prove network isolation**

      Confirm PostgreSQL and the backend have no published host ports. Access the public catalog through Nginx and verify that the backend is not reachable directly from another machine.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="database-backup" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise C — Backup and restore**

      Create a PostgreSQL backup, restore it into a separate empty test database, and compare course, enrollment, payment, and Flyway history counts.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="git-branch" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise D — CI failure gate**

      Introduce a deliberate failing JUnit test in a temporary branch. Verify GitHub Actions blocks the release, then fix the test and confirm CI passes.
    </box>
  </row>
  <divider color="subtle"/>
  <row gap={3} align="start">
    <icon name="rotate-ccw" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise E — Practice rollback**

      Deploy two compatible image versions to a staging environment. Revert to the previous digest and confirm readiness, login, and enrollment behavior without modifying the database.
    </box>
  </row>
</box>

## 18. Interactive knowledge quiz

{@body const questions=[{q:"Why use a multi-stage Dockerfile?",options:["To run multiple databases in one container","To separate build tools from the smaller runtime image","To store application secrets in the image"],correct:1,explanation:"The build stage contains Maven or Node; the runtime stage ships only what is needed to run the application."},{q:"What does Docker Compose depends_on with service_healthy guarantee?",options:["The dependency passed its configured health check before dependent startup","The database will never fail afterward","All application transactions are safe"],correct:0,explanation:"Startup ordering is useful, but applications still need retries and resilience when dependencies fail later."},{q:"Why should Flyway migrations be backward-compatible during rolling deployments?",options:["Old and new application versions may run simultaneously","PostgreSQL does not support ALTER TABLE","Docker cannot run SQL migrations"],correct:0,explanation:"Expand-and-contract changes allow old and new application versions to coexist while traffic is shifted."},{q:"What is the safest artifact identifier for production rollback?",options:["The latest tag","A developer's local image name","An immutable image digest"],correct:2,explanation:"A digest identifies exactly which image was tested and deployed."},{q:"Why is a PostgreSQL Docker volume not sufficient as a backup?",options:["Volumes disappear after every container restart","A volume can be corrupted, deleted, or lost with its host","Volumes cannot contain SQL data"],correct:1,explanation:"Recovery requires independently stored, validated backups and a tested restore procedure."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [graded,setGraded]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    **Lesson 23 — Deployment mastery**
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
            <text size="sm" color="secondary">{q.explanation}</text>
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

{@body const groups=[{name:"Containerization",items:["Verified backend and Angular baseline","Created backend multi-stage Dockerfile","Created Angular multi-stage Dockerfile","Created both .dockerignore files","Configured Nginx SPA and API routing","Verified Angular output directory","Confirmed backend runs as non-root"]},{name:"Compose and database",items:["Created deployment .env template","Protected populated .env from Git","Created PostgreSQL Compose service","Configured persistent database volume","Configured backend environment variables","Added real backend readiness probe","Configured frontend health check","Verified internal network isolation","Confirmed Flyway migrations and JPA validation","Tested local session cookies"]},{name:"CI/CD and operations",items:["Added backend GitHub Actions CI","Added frontend GitHub Actions CI","Verified Testcontainers in CI","Configured immutable image publishing","Reviewed secrets and permissions","Implemented backup procedure","Tested restoration into a fresh database","Documented deployment rollback","Ran Docker smoke tests","Reviewed production security settings"]},{name:"Learning and Git",items:["Completed at least two exercises","Completed the quiz","Ran final verification commands","Reviewed Git changes","Committed Lesson 23 checkpoint"]}]}
{@body const [checked,setChecked]=DIL.useState({})}
{@body const total=groups.reduce((n,g)=>n+g.items.length,0)}
{@body const completed=Object.values(checked).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    <box gap={1}>
      **Lesson 23 progress**
      <text color="secondary" size="xs">Mark only work verified in your environment.</text>
    </box>
    <title size="lg" tabularNums>{completed}/{total}</title>
  </row>
  <box background="surface-tertiary" radius="full" height="8px" clip>
    <box background="rgba(109,40,217,0.85)" radius="full" height="100%" width={`${100*completed/total}%`}/>
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
      **Lesson 23 checklist complete.**
    </row>
  {/if}
</box>

## 20. Final verification and Git checkpoint

### Backend

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend verification failed."
}

docker build -t learnhub-backend:lesson23 .

if ($LASTEXITCODE -ne 0) {
    throw "Backend image build failed."
}

git diff --check
git status --short

git add Dockerfile .dockerignore docker src .github
git diff --cached --stat

git commit -m "Containerize Spring Boot and add deployment CI"
```

### Angular

```powershell
cd $HOME\angular-learning\learnhub

npm ci
npm run build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}

docker build -t learnhub-frontend:lesson23 .

if ($LASTEXITCODE -ne 0) {
    throw "Frontend image build failed."
}

git diff --check
git status --short

git add Dockerfile .dockerignore nginx.conf .github
git diff --cached --stat

git commit -m "Containerize Angular with Nginx reverse proxy"
```

### Deployment directory

```powershell
cd $HOME\angular-learning\learnhub-deploy

docker compose config --quiet

if ($LASTEXITCODE -ne 0) {
    throw "Compose configuration failed."
}

docker compose up -d --build
docker compose ps

.\scripts\smoke-test.ps1
```

If `learnhub-deploy` is a separate Git repository, initialize and commit it independently. Never stage the populated `.env` or backup files.

## 21. Important pre-production corrections

Before considering the Compose example deployable to a real production environment, resolve these issues:

- **HTTP versus HTTPS:** Local Compose uses HTTP and must use the local-container cookie profile. Production must use HTTPS and secure cookies.
- **Secrets:** Replace `.env`-based production credentials with managed secrets or securely mounted secret files.
- **Image provenance:** Pin base images and deployment images to reviewed digests.
- **Database recovery:** Establish tested backups and point-in-time recovery before handling real transactions.
- **Deployment readiness:** Confirm the backend readiness probe succeeds only after Flyway and application initialization.
- **Payment reconciliation:** Ensure Stripe webhooks and retries remain correct during deployment, restarts, and rollback.
- **CI completeness:** Ensure Maven integration tests are actually discovered and Angular browser tests run in the CI environment.
- **Operational access:** Keep Actuator metrics, Swagger, database ports, and administrative interfaces off the public surface.

The code is a teaching implementation. I have not run it against your local repository, so treat compilation, health checks, and smoke tests as required acceptance gates.

---

## 22. Next lesson — the capstone

<row gap={3} align="start">
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="graduation-cap" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 24 — LearnHub Full-Stack Capstone**

    We'll audit and integrate the entire system: Angular storefront, authentication, instructor workflows, course curriculum, student enrollment and progress, payments, PostgreSQL, security, observability, testing, and deployment.

    The capstone will include end-to-end acceptance criteria, architectural review, failure scenarios, performance considerations, a final release checklist, and portfolio documentation.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 24: full-stack LearnHub capstone, end-to-end Angular and Spring Boot integration, PostgreSQL and Flyway schema audit, student and instructor journeys, secure authentication and CSRF, course publishing, curriculum and progress, paid checkout and webhook fulfillment, refund and entitlement consistency, architecture review, performance and security hardening, comprehensive automated testing, Docker Compose and CI/CD release validation, Windows PowerShell commands, complete documented code, exercises, final project rubric, portfolio README, and graduation checklist. Continue from Lesson 23 with experienced-Java pacing.")}>Start Lesson 24 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 23 deliverable:** A containerized LearnHub stack with Java 21 Spring Boot, Angular/Nginx, PostgreSQL, Flyway, private service networking, health probes, GitHub Actions CI, image publishing, backup procedures, and a deployment/rollback plan.

**Core principle:** A reliable deployment is not merely an image that starts. It is a tested, secure, observable, recoverable release whose application code and database state remain consistent throughout rollout and rollback.