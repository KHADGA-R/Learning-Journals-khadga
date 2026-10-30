# LearnHub Java & Spring Boot — Lesson 16

<text color="secondary" size="sm">Full-stack Angular integration · Spring Security sessions · Typed REST clients · Instructor CRUD · Contract testing</text>

**Stack:** Java 21 · Spring Boot 3.5.x · PostgreSQL · Spring Security · Angular standalone · TypeScript · RxJS · JUnit 5 · MockMvc · Testcontainers

**Estimated time:** 7–10 hours, including integration, debugging, and exercises.

## 1. Today's goal

In Lesson 15, we established that an instructor can manage only their own courses. Lesson 16 connects those protected backend operations to the existing LearnHub Angular frontend without weakening authentication, CSRF, or ownership enforcement.

By the end, LearnHub should support this workflow:

<box border radius="xl" padding={3} gap={2}>
  <box background="surface-secondary" padding={3} radius="lg" align="center" gap={1}>
    **Angular standalone application**
    <text color="secondary" size="xs">Catalog, login, instructor dashboard</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Typed HttpClient services**
    <text color="secondary" size="xs">Relative /api/v1 URLs · loading/error states</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Angular development proxy**
    <text color="secondary" size="xs">localhost:4200 → localhost:8080</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="lg" padding={3} align="center" gap={1}>
    **Spring Security**
    <text color="secondary" size="xs">JSESSIONID · CSRF · roles · course ownership</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box background="surface-secondary" padding={3} radius="lg" align="center" gap={1}>
    **Spring Data JPA + PostgreSQL**
    <text color="secondary" size="xs">Published catalog and instructor-owned drafts</text>
  </box>
</box>

### Acceptance criteria

| User | Expected experience |
|---|---|
| Anonymous visitor | Browse and search published courses |
| Student | Log in, browse courses, but cannot manage them |
| Instructor | View owned courses, create drafts, edit and publish owned drafts |
| Other instructor | Cannot modify another instructor's courses |
| Admin | Can use privileged backend operations |
| Returning visitor | Session is restored after a browser refresh |
| Logged-out visitor | Protected API requests are rejected |

**Architectural rule:** Angular controls presentation and navigation. Spring Boot controls authentication, authorization, validation, and database changes.

---

## 2. Baseline and workspace verification

We will work in two projects:

```text
$HOME\angular-learning\
├── learnhub\
│   └── src\app\
└── learnhub-backend\
    └── src\main\java\com\learnhub\
```

Open PowerShell:

```powershell
cd $HOME\angular-learning\learnhub-backend

java -version
mvn -version
docker info
git status
mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend baseline failed."
}
```

In a second terminal:

```powershell
cd $HOME\angular-learning\learnhub

node --version
npm --version
ng.cmd version
git status
ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular baseline failed."
}
```

Do not continue with a known failing baseline. Lesson 15 may require reconciling the exact JPA entity factory, DTO mapper, and exception package names before these commands pass.

### Inspect the existing Angular architecture

```powershell
cd $HOME\angular-learning\learnhub

Get-ChildItem src\app -Recurse -File |
    Where-Object {
        $_.Name -match 'course|auth|route|config|guard'
    } |
    Select-Object FullName
```

We will add an integration layer rather than replace the existing LearnHub UI.

---

## 3. Freeze the backend API contract

Before writing TypeScript, define exactly what the frontend consumes.

For this lesson, use the following endpoints. Preserve your actual Lesson 11–15 route names where they differ.

| Method | Endpoint | Access |
|---|---|---|
| `GET` | `/api/v1/courses` | Public |
| `GET` | `/api/v1/courses/{id}` | Public, published only |
| `GET` | `/api/v1/courses/categories` | Public |
| `GET` | `/api/v1/security/csrf` | Public |
| `POST` | `/api/v1/auth/register` | Public + CSRF |
| `POST` | `/api/v1/auth/login` | Public + CSRF |
| `GET` | `/api/v1/auth/me` | Authenticated |
| `POST` | `/api/v1/auth/logout` | Authenticated + CSRF |
| `GET` | `/api/v1/instructor/courses` | Instructor/Admin |
| `POST` | `/api/v1/courses` | Instructor/Admin + CSRF |
| `PUT` | `/api/v1/courses/{id}` | Owner/Admin + CSRF |
| `PATCH` | `/api/v1/courses/{id}/publish` | Owner/Admin + CSRF |
| `PATCH` | `/api/v1/courses/{id}/archive` | Owner/Admin + CSRF |
| `DELETE` | `/api/v1/courses/{id}` | Owner/Admin + CSRF |

### Public catalog page response

We'll use the Lesson 7/11 envelope:

```json
{
  "items": [
    {
      "id": 1,
      "title": "Angular Complete Guide",
      "description": "Build modern Angular applications.",
      "instructor": "Alex Morgan",
      "category": "Development",
      "price": 49.99,
      "rating": 4.8,
      "isFree": false,
      "level": "BEGINNER",
      "status": "PUBLISHED",
      "thumbnail": null,
      "durationHours": 12.5,
      "lastUpdated": "2026-10-01"
    }
  ],
  "totalItems": 1,
  "pageNumber": 0,
  "pageSize": 12,
  "totalPages": 1
}
```

This is an **illustrative contract example**, not a claim about your current database contents.

### Two backend alignment tasks

Before integration, ensure:

1. `GET /api/v1/instructor/courses` actually exists and returns DTOs, not JPA entities. Lesson 15 specified the service but left the final controller mapping to the existing project.
2. `GET /api/v1/courses` accepts the agreed search, filter, sort, and pagination parameters. Earlier lessons evolved these names; we'll use `search`, `category`, `freeOnly`, `level`, `sort`, `page`, and `size`.

If the backend differs, adapt the TypeScript adapter in one place rather than spreading assumptions throughout components.

---

## 4. Configure the same-origin development proxy

**Create or verify:**

`$HOME\angular-learning\learnhub\proxy.conf.json`

<CodeBlock language="json" editable>
{
  "/api": {
    "target": "http://localhost:8080",
    "secure": false,
    "changeOrigin": false
  }
}
</CodeBlock>

Start Angular with:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd serve --proxy-config proxy.conf.json
```

A browser request to:

```text
http://localhost:4200/api/v1/courses
```

is forwarded to Spring Boot at:

```text
http://localhost:8080/api/v1/courses
```

### Why this matters for sessions

The browser communicates with the Angular origin. It receives the session and XSRF cookies through that origin, and subsequent relative `/api/...` calls carry them naturally.

For production, configure the reverse proxy or hosting platform to serve the SPA and API from a compatible origin, with HTTPS and secure cookie settings.

Do not use a development-only permissive CORS configuration as a substitute for a deliberate production origin policy.

---

## 5. Typed API models

Create a dedicated API contract folder.

```powershell
cd $HOME\angular-learning\learnhub

New-Item -ItemType Directory `
  -Force src\app\core\api | Out-Null
```

**Create:** `src/app/core/api/course-api.models.ts`

<CodeBlock language="typescript" editable>
export type CourseLevel =
  | 'BEGINNER'
  | 'INTERMEDIATE'
  | 'ADVANCED';

export type CourseStatus =
  | 'DRAFT'
  | 'PUBLISHED'
  | 'ARCHIVED';

export type CourseSort =
  | 'DEFAULT'
  | 'PRICE_LOW_TO_HIGH'
  | 'PRICE_HIGH_TO_LOW'
  | 'RATING_HIGH_TO_LOW';

/**
 * Exact wire shape returned by Spring Boot.
 *
 * Do not mix API DTOs with the existing UI Course model.
 */
export interface CourseDto {
  id: number;
  title: string;
  description: string;
  instructor: string;
  category: string;
  price: number;
  rating: number;
  isFree: boolean;
  level: CourseLevel;
  status: CourseStatus;
  thumbnail: string | null;
  durationHours: number | null;
  lastUpdated: string | null;
}

export interface PageDto<T> {
  items: T[];
  totalItems: number;
  pageNumber: number;
  pageSize: number;
  totalPages: number;
}

/**
 * Fields accepted by the existing backend write DTO.
 *
 * instructor is retained for compatibility with older
 * Lesson 8 validation. The backend must ignore it for
 * ownership and server-control instructor display data.
 */
export interface CourseWriteDto {
  title: string;
  description: string;
  instructor: string;
  category: string;
  price: number;
  isFree: boolean;
  level: CourseLevel;
  rating: number;
  thumbnail: string | null;
  durationHours: number | null;
}

export interface CatalogQuery {
  search?: string;
  category?: string;
  freeOnly?: boolean;
  level?: CourseLevel;
  sort?: CourseSort;
  page?: number;
  size?: number;
}

export interface ApiProblem {
  type?: string;
  title?: string;
  status?: number;
  detail?: string;
  instance?: string;
  fieldErrors?: Record<string, string>;
}
</CodeBlock>

### Why keep API and UI models separate?

Your Angular LearnHub frontend predates the Spring backend and may contain fields such as image fallbacks, review counts, badges, or display durations that are not persisted in PostgreSQL.

Keep a mapping boundary:

```text
CourseDto (backend contract)
          ↓
Course (existing Angular presentation model)
```

Do not invent persistent values in the API DTO to satisfy UI expectations.

---

## 6. Build the typed CourseApiService

**Create:** `src/app/core/api/course-api.service.ts`

<CodeBlock language="typescript" editable>
import { Injectable, inject } from '@angular/core';
import {
  HttpClient,
  HttpParams,
} from '@angular/common/http';
import { Observable } from 'rxjs';

import {
  CatalogQuery,
  CourseDto,
  CourseWriteDto,
  PageDto,
} from './course-api.models';

@Injectable({ providedIn: 'root' })
export class CourseApiService {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/v1';

  /**
   * Public catalog. The server filters out drafts
   * and archived courses.
   */
  catalog(query: CatalogQuery = {}):
      Observable<PageDto<CourseDto>> {
    let params = new HttpParams();

    if (query.search?.trim()) {
      params = params.set('search', query.search.trim());
    }

    if (query.category?.trim()) {
      params = params.set(
        'category',
        query.category.trim()
      );
    }

    if (query.freeOnly !== undefined) {
      params = params.set(
        'freeOnly',
        String(query.freeOnly)
      );
    }

    if (query.level) {
      params = params.set('level', query.level);
    }

    if (query.sort) {
      params = params.set('sort', query.sort);
    }

    params = params
      .set('page', String(query.page ?? 0))
      .set('size', String(query.size ?? 12));

    return this.http.get<PageDto<CourseDto>>(
      `${this.base}/courses`,
      { params }
    );
  }

  publicCourse(id: number): Observable<CourseDto> {
    return this.http.get<CourseDto>(
      `${this.base}/courses/${id}`
    );
  }

  categories(): Observable<string[]> {
    return this.http.get<string[]>(
      `${this.base}/courses/categories`
    );
  }

  /**
   * Server derives instructor ID from the session.
   * There is deliberately no instructorId argument.
   */
  myCourses(
    page = 0,
    size = 10
  ): Observable<PageDto<CourseDto>> {
    const params = new HttpParams()
      .set('page', String(page))
      .set('size', String(size));

    return this.http.get<PageDto<CourseDto>>(
      `${this.base}/instructor/courses`,
      { params }
    );
  }

  createDraft(
    body: CourseWriteDto
  ): Observable<CourseDto> {
    return this.http.post<CourseDto>(
      `${this.base}/courses`,
      body
    );
  }

  updateDraft(
    id: number,
    body: CourseWriteDto
  ): Observable<CourseDto> {
    return this.http.put<CourseDto>(
      `${this.base}/courses/${id}`,
      body
    );
  }

  /**
   * Lesson 15 reference contract: 204 No Content.
   * If your backend returns CourseResponse instead,
   * change these two return types to CourseDto.
   */
  publish(id: number): Observable<void> {
    return this.http.patch<void>(
      `${this.base}/courses/${id}/publish`,
      {}
    );
  }

  archive(id: number): Observable<void> {
    return this.http.patch<void>(
      `${this.base}/courses/${id}/archive`,
      {}
    );
  }

  delete(id: number): Observable<void> {
    return this.http.delete<void>(
      `${this.base}/courses/${id}`
    );
  }
}
</CodeBlock>

### Important backend contract detail

The Lesson 15 `CourseWriteRequest` still required an `instructor` string for compatibility with older validation. That field is sent here, but **the backend must not trust it for ownership or impersonation**.

A later cleanup should remove the instructor field from the public write request entirely and derive it from the authenticated owner.

Also verify that the Lesson 12 sort enum uses the same names as `CourseSort`. If it uses different values, create an explicit mapping rather than relying on accidental string compatibility.

---

## 7. Map Spring DTOs to the existing Angular course model

Do not replace the existing UI model if your LearnHub cards, wishlist, and detail pages already depend on it.

Instead, create a mapper.

**Create:** `src/app/core/api/course-view.model.ts`

<CodeBlock language="typescript" editable>
import { CourseDto } from './course-api.models';

/**
 * A minimal presentation model for new integration
 * components. Existing LearnHub screens can adapt
 * this shape to their established Course interface.
 */
export interface CourseView {
  id: number;
  title: string;
  description: string;
  instructorName: string;
  category: string;
  price: number;
  priceLabel: string;
  rating: number;
  levelLabel: string;
  imageUrl: string;
  durationLabel: string;
  isFree: boolean;
}

const FALLBACK_IMAGE =
  '/assets/images/course-placeholder.svg';

export function toCourseView(
  dto: CourseDto
): CourseView {
  return {
    id: dto.id,
    title: dto.title,
    description: dto.description,
    instructorName: dto.instructor,
    category: dto.category,
    price: dto.price,
    priceLabel: dto.isFree
      ? 'Free'
      : new Intl.NumberFormat('en-US', {
          style: 'currency',
          currency: 'USD',
        }).format(dto.price),
    rating: dto.rating,
    levelLabel: dto.level
      .toLowerCase()
      .replace(/^./, character =>
        character.toUpperCase()
      ),
    imageUrl:
      dto.thumbnail?.trim() ||
      FALLBACK_IMAGE,
    durationLabel:
      dto.durationHours === null
        ? 'Duration not listed'
        : `${dto.durationHours} hours`,
    isFree: dto.isFree,
  };
}
</CodeBlock>

The placeholder path is an example. Point it to an image that actually exists in your Angular project, or create a simple placeholder asset.

**Do not use an arbitrary remote thumbnail URL as trusted HTML.** Bind it to Angular's normal image source binding, keep the backend's thumbnail policy restrictive, and do not bypass Angular sanitization.

### Why not copy mock ratings and enrollment counts?

Mock metadata can mislead users into believing it came from the database.

If a metric is not persisted yet, either omit it, label it as demo data, or use a clearly defined fallback.

---

## 8. Restore authentication state correctly

Lesson 14 introduced `AuthService.currentUser`, `login()`, `logout()`, and `loadCurrentUser()`.

For Lesson 16, we need to distinguish three states:

```text
loading
anonymous
authenticated
```

A `null` user alone cannot distinguish “not logged in” from “we haven't checked the server yet.”

### Replace the AuthService with an initialized state model

**Modify:** `src/app/core/auth/auth.service.ts`

<CodeBlock language="typescript" editable>
import {
  Injectable,
  computed,
  inject,
  signal,
} from '@angular/core';
import { HttpClient } from '@angular/common/http';

import {
  Observable,
  catchError,
  finalize,
  map,
  of,
  shareReplay,
  switchMap,
  tap,
  throwError,
} from 'rxjs';

import {
  AuthUser,
  CsrfResponse,
  LoginRequest,
  RegisterRequest,
} from './auth.models';

export type AuthState =
  | { status: 'loading'; user: null }
  | { status: 'anonymous'; user: null }
  | { status: 'authenticated'; user: AuthUser };

@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/v1';

  private readonly stateSignal =
    signal<AuthState>({
      status: 'loading',
      user: null,
    });

  readonly state = this.stateSignal.asReadonly();

  readonly currentUser = computed(
    () => this.stateSignal().user
  );

  readonly isAuthenticated = computed(
    () => this.stateSignal().status ===
      'authenticated'
  );

  private initialization$:
    Observable<AuthUser | null> | null = null;

  /**
   * The endpoint creates or retrieves the CSRF token.
   * The token itself is not stored in localStorage.
   */
  private csrf(): Observable<void> {
    return this.http.get<CsrfResponse>(
      `${this.base}/security/csrf`
    ).pipe(map(() => void 0));
  }

  /**
   * Use for application startup and protected routes.
   * Concurrent callers share one initialization request.
   */
  initialize(): Observable<AuthUser | null> {
    if (this.stateSignal().status !== 'loading') {
      return of(this.stateSignal().user);
    }

    if (!this.initialization$) {
      this.initialization$ =
        this.loadCurrentUser().pipe(
          finalize(() => {
            this.initialization$ = null;
          }),
          shareReplay({
            bufferSize: 1,
            refCount: false,
          })
        );
    }

    return this.initialization$;
  }

  loadCurrentUser():
      Observable<AuthUser | null> {
    return this.http.get<AuthUser>(
      `${this.base}/auth/me`
    ).pipe(
      tap(user => {
        this.stateSignal.set({
          status: 'authenticated',
          user,
        });
      }),
      catchError(error => {
        if (error.status === 401) {
          this.stateSignal.set({
            status: 'anonymous',
            user: null,
          });

          return of(null);
        }

        // A network outage is not evidence of logout.
        // Preserve the prior authenticated state, if any.
        // Startup remains loading until a retry succeeds.
        return throwError(() => error);
      })
    );
  }

  register(
    request: RegisterRequest
  ): Observable<AuthUser> {
    return this.csrf().pipe(
      switchMap(() =>
        this.http.post<AuthUser>(
          `${this.base}/auth/register`,
          request
        )
      )
    );
  }

  login(
    request: LoginRequest
  ): Observable<AuthUser> {
    return this.csrf().pipe(
      switchMap(() =>
        this.http.post<AuthUser>(
          `${this.base}/auth/login`,
          request
        )
      ),
      switchMap(user =>
        // Authentication invalidates the old CSRF token.
        this.csrf().pipe(map(() => user))
      ),
      tap(user => {
        this.stateSignal.set({
          status: 'authenticated',
          user,
        });
      })
    );
  }

  logout(): Observable<void> {
    return this.csrf().pipe(
      switchMap(() =>
        this.http.post<void>(
          `${this.base}/auth/logout`,
          {}
        )
      ),
      tap(() => {
        // The server has invalidated the session.
        this.stateSignal.set({
          status: 'anonymous',
          user: null,
        });
      }),
      switchMap(() => this.csrf())
    );
  }
}
</CodeBlock>

### A subtle logout edge case

The server may successfully invalidate the session while the subsequent CSRF refresh fails due to a network error.

Because we update the local state immediately after the successful logout response, the UI still recognizes the logout. The observable can report the later CSRF refresh failure separately.

For production, you may separate logout and token rebootstrap to make this distinction clearer to callers.

### Session storage rule

Do not put session IDs, passwords, or authentication tokens in `localStorage`.

For this architecture, the browser-managed HttpOnly session cookie is the authentication credential.

The CSRF token is separate and is not an authentication token.

---

## 9. Bootstrap authentication on application startup

For a clean initial navigation, use Angular's application initializer.

**Modify:** `src/app/app.config.ts`

Merge the following with your existing providers:

<CodeBlock language="typescript" editable>
import {
  provideAppInitializer,
  inject,
} from '@angular/core';

import {
  provideHttpClient,
  withXsrfConfiguration,
} from '@angular/common/http';

import { firstValueFrom } from 'rxjs';

import { AuthService }
  from './core/auth/auth.service';

// Inside your existing providers array:
provideHttpClient(
  withXsrfConfiguration({
    cookieName: 'XSRF-TOKEN',
    headerName: 'X-XSRF-TOKEN',
  })
),

provideAppInitializer(() => {
  const auth = inject(AuthService);

  return firstValueFrom(
    auth.initialize()
  ).then(
    () => undefined,
    () => undefined
  );
}),
</CodeBlock>

**Merge rather than duplicate** an existing `provideHttpClient()` registration.

The initializer attempts to restore the session before application startup finishes. In this reference, a temporary network error allows the app to render while auth state remains `loading`; show a retry or unavailable state rather than assuming the visitor is anonymous.

### Why use a same-origin proxy here?

Angular's built-in XSRF support is designed around eligible same-origin requests. Relative URLs are essential to this implementation.

For browser requests to a different origin, the cookie and XSRF behavior requires an explicitly designed CORS, credentials, and CSRF strategy.

---

## 10. Build a reusable API error interceptor

The frontend should handle HTTP errors consistently, but it must not hide the original status code.

**Create:** `src/app/core/api/api-error.ts`

<CodeBlock language="typescript" editable>
import { HttpErrorResponse } from '@angular/common/http';
import { ApiProblem } from './course-api.models';

export interface ApiFailure {
  status: number;
  message: string;
  fieldErrors: Record<string, string>;
  original: HttpErrorResponse;
}

export function toApiFailure(
  error: HttpErrorResponse
): ApiFailure {
  const body =
    error.error &&
    typeof error.error === 'object'
      ? error.error as ApiProblem
      : null;

  const fallback = (() => {
    switch (error.status) {
      case 0:
        return 'Cannot reach LearnHub. Check your connection.';
      case 400:
        return 'Please check the submitted information.';
      case 401:
        return 'Please sign in to continue.';
      case 403:
        return 'You do not have permission to do that.';
      case 404:
        return 'The requested resource was not found.';
      case 409:
        return 'This action conflicts with the current state.';
      default:
        return 'Something went wrong. Please try again.';
    }
  })();

  return {
    status: error.status,
    message: body?.detail || fallback,
    fieldErrors: body?.fieldErrors ?? {},
    original: error,
  };
}
</CodeBlock>

**Create:** `src/app/core/api/api-error.interceptor.ts`

<CodeBlock language="typescript" editable>
import {
  HttpErrorResponse,
  HttpInterceptorFn,
} from '@angular/common/http';

import {
  catchError,
  throwError,
} from 'rxjs';

import { toApiFailure } from './api-error';

/**
 * Normalizes API errors without changing their status.
 *
 * Does not redirect on every 401:
 * /auth/me intentionally returns 401 for visitors.
 */
export const apiErrorInterceptor:
    HttpInterceptorFn = (request, next) => {
  return next(request).pipe(
    catchError(error => {
      if (error instanceof HttpErrorResponse) {
        return throwError(
          () => toApiFailure(error)
        );
      }

      return throwError(() => error);
    })
  );
};
</CodeBlock>

Update `app.config.ts` so its single `provideHttpClient()` contains both XSRF configuration and the interceptor:

```typescript
import {
  provideHttpClient,
  withInterceptors,
  withXsrfConfiguration,
} from '@angular/common/http';

import { apiErrorInterceptor }
  from './core/api/api-error.interceptor';

// Existing providers:
provideHttpClient(
  withXsrfConfiguration({
    cookieName: 'XSRF-TOKEN',
    headerName: 'X-XSRF-TOKEN',
  }),
  withInterceptors([apiErrorInterceptor])
),
```

### One necessary compatibility fix

The interceptor transforms `HttpErrorResponse` into `ApiFailure`. The Lesson 14 auth service and the Lesson 16 `loadCurrentUser()` method currently check `error.status`, which still works because `ApiFailure` preserves it.

However, a component that checks `error instanceof HttpErrorResponse` will need updating.

An alternative production design is to leave the original error untouched and provide a standalone `toApiFailure()` helper in components. Choose one strategy consistently; don't mix incompatible assumptions.

### Don't automatically retry writes

A network failure after `POST /courses` does not prove the server failed to create the course.

Blindly retrying a non-idempotent write can create duplicates. Use explicit user retry flows and idempotency mechanisms where needed.

---

## 11. Implement a public catalog page

We will build a small standalone page that fetches real backend data, supports search and pagination, and shows loading, empty, and error states.

**Create:** `src/app/features/catalog/backend-catalog-page.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  inject,
  signal,
} from '@angular/core';

import {
  FormControl,
  ReactiveFormsModule,
} from '@angular/forms';
import { RouterLink } from '@angular/router';

import {
  catchError,
  debounceTime,
  distinctUntilChanged,
  of,
  startWith,
  switchMap,
  tap,
} from 'rxjs';

import { takeUntilDestroyed }
  from '@angular/core/rxjs-interop';

import { CourseApiService }
  from '../../core/api/course-api.service';

import { CourseDto }
  from '../../core/api/course-api.models';

import { toApiFailure }
  from '../../core/api/api-error';

@Component({
  selector: 'app-backend-catalog-page',
  standalone: true,
  imports: [ReactiveFormsModule, RouterLink],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main>
      <h1>Explore LearnHub courses</h1>

      <label for="course-search">Search courses</label>
      <input
        id="course-search"
        [formControl]="search"
        placeholder="Angular, Python, Spring Boot..."
      />

      @if (loading()) {
        <p role="status">Loading courses...</p>
      } @else if (error()) {
        <p role="alert">{{ error() }}</p>
        <button type="button" (click)="reload()">
          Try again
        </button>
      } @else if (courses().length === 0) {
        <p>No courses match your search.</p>
      } @else {
        <ul>
          @for (course of courses(); track course.id) {
            <li>
              <h2>
                <a [routerLink]="['/courses', course.id]">
                  {{ course.title }}
                </a>
              </h2>
              <p>{{ course.instructor }}</p>
              <p>{{ course.category }}</p>
              <p>
                {{ course.isFree ? 'Free' : '$' + course.price }}
              </p>
            </li>
          }
        </ul>
      }

      <nav aria-label="Catalog pagination">
        <button
          type="button"
          (click)="previous()"
          [disabled]="page() === 0 || loading()"
        >
          Previous
        </button>

        <span>
          Page {{ page() + 1 }} of
          {{ totalPages() || 1 }}
        </span>

        <button
          type="button"
          (click)="next()"
          [disabled]="
            loading() ||
            page() + 1 >= totalPages()
          "
        >
          Next
        </button>
      </nav>
    </main>
  `,
})
export class BackendCatalogPageComponent {
  private readonly api = inject(CourseApiService);
  private readonly destroyRef = inject(DestroyRef);

  readonly search = new FormControl('', {
    nonNullable: true,
  });

  readonly courses = signal<CourseDto[]>([]);
  readonly page = signal(0);
  readonly totalPages = signal(0);
  readonly loading = signal(false);
  readonly error = signal<string | null>(null);

  private readonly reloadTrigger =
    signal(0);

  constructor() {
    this.search.valueChanges.pipe(
      startWith(this.search.value),
      debounceTime(250),
      distinctUntilChanged(),
      tap(() => this.page.set(0)),
      switchMap(() => this.loadPage(0)),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe();

    // The constructor handles search changes.
    // Pagination is loaded by explicit button actions.
  }

  private loadPage(page: number) {
    this.loading.set(true);
    this.error.set(null);

    return this.api.catalog({
      search: this.search.value,
      page,
      size: 12,
    }).pipe(
      tap(result => {
        this.courses.set(result.items);
        this.totalPages.set(result.totalPages);
        this.page.set(result.pageNumber);
        this.loading.set(false);
      }),
      catchError(error => {
        this.error.set(
          (error as { message?: string }).message ||
          'Unable to load courses.'
        );
        this.loading.set(false);
        return of(null);
      })
    );
  }

  reload(): void {
    this.loadPage(this.page()).subscribe();
  }

  previous(): void {
    if (this.page() > 0) {
      this.loadPage(this.page() - 1).subscribe();
    }
  }

  next(): void {
    if (this.page() + 1 < this.totalPages()) {
      this.loadPage(this.page() + 1).subscribe();
    }
  }
}
</CodeBlock>

### Review: avoiding stale search results

The search stream uses `switchMap()`, which cancels its previous HTTP subscription when the query changes.

However, the explicit pagination subscriptions above are separate. They could race with a new search and overwrite its results.

For production, unify search and pagination into one reactive query stream. We'll do that in the next refinement.

### Production-ready refinement: one query state

Replace the component's constructor and pagination methods with a single observable pipeline driven by a query subject.

A clean approach is to use:

```typescript
private readonly query$ =
  new BehaviorSubject<{ search: string; page: number }>({
    search: '',
    page: 0,
  });
```

Then use `switchMap()` once for every query change.

This ensures that a late response for an old page cannot overwrite the current search results.

We'll use that approach in the exercises.

---

## 12. Complete the instructor-owned course endpoint

Lesson 15 introduced `InstructorCourseQueryService`, but the HTTP controller and DTO conversion were left for integration. We must finish that before Angular can call it.

For this section, the backend needs a mapper from `CourseEntity` to `CourseResponse`. The example below uses the existing Lesson 7 metadata service and response factory instead of inventing a new mapping API.

**Create:** `src/main/java/com/learnhub/web/InstructorCourseController.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.learnhub.catalog.PageResult;
import com.learnhub.model.Course;
import com.learnhub.persistence.CourseEntity;
import com.learnhub.persistence.CourseMetadataService;
import com.learnhub.service.InstructorCourseQueryService;
import com.learnhub.web.dto.CoursePageResponse;
import com.learnhub.web.dto.CourseResponse;

import org.springframework.data.domain.Page;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

/**
 * Instructor dashboard API.
 *
 * Never accepts an instructor ID from the browser.
 */
@RestController
@RequestMapping("/api/v1/instructor/courses")
public class InstructorCourseController {

    private final InstructorCourseQueryService service;
    private final CourseMetadataService metadata;

    public InstructorCourseController(
            InstructorCourseQueryService service,
            CourseMetadataService metadata
    ) {
        this.service = service;
        this.metadata = metadata;
    }

    @GetMapping
    public CoursePageResponse myCourses(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size
    ) {
        Page<CourseEntity> result =
                service.myCourses(page, size);

        // Adapt entities to the existing immutable domain
        // model using the same conversion rules as the
        // Lesson 7 PostgresCourseRepository.
        List<Course> domain = result.getContent()
                .stream()
                .map(this::toDomain)
                .toList();

        PageResult<Course> pageResult =
                new PageResult<>(
                    domain,
                    result.getNumber(),
                    result.getSize(),
                    result.getTotalElements(),
                    result.getTotalPages()
                );

        var ids = domain.stream()
                .map(Course::id)
                .toList();

        return CoursePageResponse.from(
                pageResult,
                metadata.findByCourseIds(ids)
        );
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

### Align these three method names

Your previous lessons may use slightly different names:

- `CourseEntity.isFree()` versus `getFree()`.
- `CourseMetadataService.findByCourseIds(...)` versus the existing bulk lookup method.
- `CoursePageResponse.from(PageResult, Map)` versus a Lesson 11 mapper.

Use the actual methods already present in your repository. The controller's essential behavior is fixed: **derive the actor from the security context, query only owned courses, and serialize explicit DTOs**.

### Performance consideration

The metadata service should bulk-load metadata for all page IDs rather than issuing one query per course.

Also, keep entity-to-domain mapping within the service's transaction if it ever starts reading lazy associations. The example maps scalar columns only.

### Security configuration

Ensure the existing `SecurityFilterChain` includes:

```java
.requestMatchers(
    HttpMethod.GET,
    "/api/v1/instructor/courses"
).hasAnyRole("INSTRUCTOR", "ADMIN")
```

The `InstructorCourseQueryService.myCourses()` method also has `@PreAuthorize`, so both the HTTP boundary and service boundary are protected.

---

## 13. Instructor dashboard: load owned courses

**Create:** `src/app/features/instructor/instructor-dashboard.component.ts`

If Lesson 15 already created the placeholder component, replace it with the implementation below.

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  inject,
  signal,
} from '@angular/core';

import { RouterLink } from '@angular/router';

import { CourseApiService }
  from '../../core/api/course-api.service';

import { CourseDto }
  from '../../core/api/course-api.models';

@Component({
  selector: 'app-instructor-dashboard',
  standalone: true,
  imports: [RouterLink],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main>
      <header>
        <h1>My courses</h1>

        <a routerLink="/instructor/courses/new">
          Create course
        </a>
      </header>

      @if (loading()) {
        <p role="status">Loading your courses...</p>
      } @else if (error()) {
        <p role="alert">{{ error() }}</p>
        <button type="button" (click)="load()">
          Retry
        </button>
      } @else if (courses().length === 0) {
        <p>You have not created any courses yet.</p>
      } @else {
        <ul>
          @for (course of courses(); track course.id) {
            <li>
              <h2>{{ course.title }}</h2>
              <p>Status: {{ course.status }}</p>

              <a
                [routerLink]="[
                  '/instructor/courses',
                  course.id,
                  'edit'
                ]"
              >
                Edit
              </a>

              @if (course.status === 'DRAFT') {
                <button
                  type="button"
                  (click)="publish(course)"
                  [disabled]="busyId() === course.id"
                >
                  Publish
                </button>
              }
            </li>
          }
        </ul>
      }
    </main>
  `,
})
export class InstructorDashboardComponent {
  private readonly api = inject(CourseApiService);

  readonly courses = signal<CourseDto[]>([]);
  readonly loading = signal(true);
  readonly error = signal<string | null>(null);
  readonly busyId = signal<number | null>(null);

  constructor() {
    this.load();
  }

  load(): void {
    this.loading.set(true);
    this.error.set(null);

    this.api.myCourses().subscribe({
      next: result => {
        this.courses.set(result.items);
        this.loading.set(false);
      },
      error: error => {
        this.error.set(
          error.message ||
          'Unable to load your courses.'
        );
        this.loading.set(false);
      },
    });
  }

  publish(course: CourseDto): void {
    if (this.busyId() !== null) {
      return;
    }

    this.busyId.set(course.id);
    this.error.set(null);

    this.api.publish(course.id).subscribe({
      next: () => {
        this.busyId.set(null);
        this.load();
      },
      error: error => {
        this.busyId.set(null);
        this.error.set(
          error.message || 'Unable to publish course.'
        );
      },
    });
  }
}
</CodeBlock>

### Why reload after publishing?

The backend owns the course lifecycle. After a successful publish request, reloading avoids guessing how the server transformed the record.

The same principle applies after archive, delete, or update.

### Instructor pagination

The first implementation displays the first ten courses. Before shipping a real instructor dashboard, add next/previous controls using the `totalPages` response and pass the selected page to `myCourses()`.

---

## 14. Course form: create and edit drafts

We'll use a shared standalone form component for both operations.

The backend still validates every field. Angular validation is for immediate user feedback.

**Create:** `src/app/features/instructor/course-editor.component.ts`

<CodeBlock language="typescript" editable>
import {
  ChangeDetectionStrategy,
  Component,
  inject,
  signal,
} from '@angular/core';

import {
  FormBuilder,
  ReactiveFormsModule,
  Validators,
} from '@angular/forms';

import {
  ActivatedRoute,
  Router,
} from '@angular/router';

import { finalize } from 'rxjs';

import { CourseApiService }
  from '../../core/api/course-api.service';

import {
  CourseLevel,
  CourseWriteDto,
} from '../../core/api/course-api.models';

import { AuthService }
  from '../../core/auth/auth.service';

@Component({
  selector: 'app-course-editor',
  standalone: true,
  imports: [ReactiveFormsModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <main>
      <h1>
        {{ courseId === null
          ? 'Create course'
          : 'Edit draft' }}
      </h1>

      @if (loading()) {
        <p role="status">Loading course...</p>
      } @else {
        <form
          [formGroup]="form"
          (ngSubmit)="save()"
        >
          <label for="title">Title</label>
          <input id="title" formControlName="title" />

          <label for="description">Description</label>
          <textarea
            id="description"
            formControlName="description"
          ></textarea>

          <label for="category">Category</label>
          <input
            id="category"
            formControlName="category"
          />

          <label for="price">Price (USD)</label>
          <input
            id="price"
            type="number"
            step="0.01"
            min="0"
            formControlName="price"
          />

          <label>
            <input
              type="checkbox"
              formControlName="isFree"
            />
            Free course
          </label>

          <label for="level">Level</label>
          <select id="level" formControlName="level">
            <option value="BEGINNER">Beginner</option>
            <option value="INTERMEDIATE">
              Intermediate
            </option>
            <option value="ADVANCED">Advanced</option>
          </select>

          <label for="duration">Duration (hours)</label>
          <input
            id="duration"
            type="number"
            min="0"
            step="0.01"
            formControlName="durationHours"
          />

          @if (error()) {
            <p role="alert">{{ error() }}</p>
          }

          <button
            type="submit"
            [disabled]="
              form.invalid ||
              saving() ||
              loading()
            "
          >
            {{ saving() ? 'Saving...' : 'Save draft' }}
          </button>
        </form>
      }
    </main>
  `,
})
export class CourseEditorComponent {
  private readonly fb = inject(FormBuilder);
  private readonly api = inject(CourseApiService);
  private readonly auth = inject(AuthService);
  private readonly router = inject(Router);
  private readonly route = inject(ActivatedRoute);

  readonly saving = signal(false);
  readonly loading = signal(false);
  readonly error = signal<string | null>(null);

  readonly courseId: number | null = (() => {
    const raw = this.route.snapshot.paramMap.get('id');

    if (raw === null) {
      return null;
    }

    const id = Number(raw);

    return Number.isSafeInteger(id) && id > 0
      ? id
      : NaN;
  })();

  readonly form = this.fb.nonNullable.group({
    title: ['', [
      Validators.required,
      Validators.maxLength(150),
    ]],
    description: ['', [
      Validators.required,
      Validators.maxLength(3000),
    ]],
    category: ['', [
      Validators.required,
      Validators.maxLength(100),
    ]],
    price: [0, [
      Validators.required,
      Validators.min(0),
    ]],
    isFree: [true],
    level: ['BEGINNER' as CourseLevel],
    durationHours: [0, [Validators.min(0)]],
  });

  constructor() {
    if (this.courseId !== null) {
      // Public GET hides drafts. Editing requires a
      // separate owner-authorized backend read route.
      this.error.set(
        'Draft editing requires the private course detail endpoint.'
      );
    }
  }

  save(): void {
    if (
      this.form.invalid ||
      this.saving() ||
      this.courseId !== null
    ) {
      return;
    }

    const value = this.form.getRawValue();

    if (value.isFree !== (value.price === 0)) {
      this.error.set(
        'Free courses must have a zero price.'
      );
      return;
    }

    const owner = this.auth.currentUser();

    if (!owner) {
      this.error.set('Please sign in.');
      return;
    }

    const body: CourseWriteDto = {
      title: value.title.trim(),
      description: value.description.trim(),
      instructor: owner.displayName,
      category: value.category.trim(),
      price: value.price,
      isFree: value.isFree,
      level: value.level,
      rating: 0,
      thumbnail: null,
      durationHours: value.durationHours,
    };

    this.saving.set(true);
    this.error.set(null);

    this.api.createDraft(body).pipe(
      finalize(() => this.saving.set(false))
    ).subscribe({
      next: () => {
        void this.router.navigateByUrl('/instructor');
      },
      error: error => {
        this.error.set(
          error.message || 'Unable to save course.'
        );
      },
    });
  }
}
</CodeBlock>

### Important limitation to resolve now

The existing public `GET /api/v1/courses/{id}` deliberately hides drafts.

Therefore, **the edit form cannot fetch an unpublished draft through that endpoint**.

We need a private owner-authorized read route. Add it next rather than weakening the public endpoint.

---

## 15. Add a private draft-detail API

The correct separation is:

```text
GET /api/v1/courses/{id}
    → Published courses only

GET /api/v1/instructor/courses/{id}
    → Owner or admin, including drafts
```

### 15.1 Add the service method

**Modify:** `src/main/java/com/learnhub/service/InstructorCourseQueryService.java`

Inject `CourseOwnershipPolicy` and the existing `JpaCourseRepository` if not already present, then add:

<CodeBlock language="java" editable>
@PreAuthorize("hasAnyRole('INSTRUCTOR', 'ADMIN')")
@Transactional(readOnly = true)
public CourseEntity findManageableCourse(long courseId) {
    var principal = actor.require();

    CourseEntity course = courses.findById(courseId)
            .orElseThrow(() ->
                new CourseNotFoundException(courseId)
            );

    ownership.requireCanManage(principal, course);

    return course;
}
</CodeBlock>

Here `actor` is the existing `CurrentActor` field. Add `CourseOwnershipPolicy ownership` as a constructor-injected dependency. Use the existing `CourseNotFoundException` package and constructor.

### 15.2 Return an explicit DTO

**Modify:** `src/main/java/com/learnhub/web/InstructorCourseController.java`

Add a method:

<CodeBlock language="java" editable>
@GetMapping("/{id}")
public CourseResponse manageableCourse(
        @PathVariable long id
) {
    CourseEntity entity =
            service.findManageableCourse(id);

    Course domain = toDomain(entity);

    return CourseResponse.from(
            domain,
            metadata.findByCourseId(id)
    );
}
</CodeBlock>

Use the existing metadata lookup method name and DTO factory from your backend. If those methods differ, align them with your actual Lesson 7–11 implementation.

### 15.3 Security rule

In the existing `SecurityFilterChain`, ensure this route is protected before the generic public course matcher:

```java
.requestMatchers(
    HttpMethod.GET,
    "/api/v1/instructor/courses/**"
).hasAnyRole("INSTRUCTOR", "ADMIN")
```

### 15.4 Add the typed frontend method

**Modify:** `src/app/core/api/course-api.service.ts`

```typescript
manageableCourse(id: number): Observable<CourseDto> {
  return this.http.get<CourseDto>(
    `${this.base}/instructor/courses/${id}`
  );
}
```

### 15.5 Complete the edit form

Replace the `CourseEditorComponent` constructor and `save()` logic from Section 14 with the following implementation details.

First, replace the constructor:

<CodeBlock language="typescript" editable>
constructor() {
  if (this.courseId !== null) {
    if (!Number.isSafeInteger(this.courseId)) {
      this.error.set('Invalid course ID.');
      return;
    }

    this.loading.set(true);

    this.api.manageableCourse(this.courseId).pipe(
      finalize(() => this.loading.set(false))
    ).subscribe({
      next: course => {
        if (course.status !== 'DRAFT') {
          this.error.set(
            'Only draft courses can be edited.'
          );
          return;
        }

        this.form.patchValue({
          title: course.title,
          description: course.description,
          category: course.category,
          price: course.price,
          isFree: course.isFree,
          level: course.level,
          durationHours: course.durationHours ?? 0,
        });
      },
      error: error => {
        this.error.set(
          error.message || 'Unable to load draft.'
        );
      },
    });
  }
}
</CodeBlock>

Then, in `save()`, replace the initial guard:

```typescript
if (this.form.invalid || this.saving() || this.loading()) {
  return;
}
```

Finally, replace the creation-only API call:

```typescript
const request$ =
  this.courseId === null
    ? this.api.createDraft(body)
    : this.api.updateDraft(this.courseId, body);

request$.pipe(
  finalize(() => this.saving.set(false))
).subscribe({
  next: () => {
    void this.router.navigateByUrl('/instructor');
  },
  error: error => {
    this.error.set(
      error.message || 'Unable to save course.'
    );
  },
});
```

Keep the existing form validation, body construction, `saving.set(true)`, and `error.set(null)` before this call. **Do not append a second subscription**; replace the original creation subscription.

### Two editing safeguards

The editor should remain disabled until a draft has been loaded successfully. Add a `draftLoaded` signal or equivalent state so a failed private GET cannot leave the form editable with default values.

Also, the editor currently sends `rating: 0` and `thumbnail: null` on update. That could unintentionally erase existing metadata. Preserve those fields from the loaded DTO or use a dedicated backend patch contract.

A safe local addition is:

```typescript
private readonly original =
  signal<CourseDto | null>(null);
```

Set it when the draft loads:

```typescript
this.original.set(course);
```

Then construct:

```typescript
const existing = this.original();

const body: CourseWriteDto = {
  title: value.title.trim(),
  description: value.description.trim(),
  instructor: owner.displayName,
  category: value.category.trim(),
  price: value.price,
  isFree: value.isFree,
  level: value.level,
  rating: existing?.rating ?? 0,
  thumbnail: existing?.thumbnail ?? null,
  durationHours: value.durationHours,
};
```

The backend must still preserve the authoritative owner and instructor display identity.

---

## 16. Configure Angular routes and guards

**Modify:** `src/app/app.routes.ts`

Merge the routes below into your existing route array. Do not duplicate existing catalog or instructor routes.

<CodeBlock language="typescript" editable>
import { Routes } from '@angular/router';

import { instructorGuard }
  from './core/auth/instructor.guard';

export const routes: Routes = [
  {
    path: 'courses',
    loadComponent: () =>
      import(
        './features/catalog/backend-catalog-page.component'
      ).then(m => m.BackendCatalogPageComponent),
  },
  {
    path: 'instructor',
    canActivate: [instructorGuard],
    children: [
      {
        path: '',
        loadComponent: () =>
          import(
            './features/instructor/instructor-dashboard.component'
          ).then(m => m.InstructorDashboardComponent),
      },
      {
        path: 'courses/new',
        loadComponent: () =>
          import(
            './features/instructor/course-editor.component'
          ).then(m => m.CourseEditorComponent),
      },
      {
        path: 'courses/:id/edit',
        loadComponent: () =>
          import(
            './features/instructor/course-editor.component'
          ).then(m => m.CourseEditorComponent),
      },
    ],
  },
];
</CodeBlock>

This is the relevant route fragment, not a replacement for your entire existing `app.routes.ts`. Preserve login, registration, course details, wishlist, and other LearnHub routes.

### Update the Lesson 15 guard

The new `AuthService` has an `initialize()` method. Update `instructor.guard.ts` to use it:

<CodeBlock language="typescript" editable>
import { inject } from '@angular/core';

import {
  CanActivateFn,
  Router,
} from '@angular/router';

import {
  catchError,  map,
  of,
} from 'rxjs';

import { AuthService } from './auth.service';

export const instructorGuard:
    CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  return auth.initialize().pipe(
    map(user => {
      if (!user) {
        return router.createUrlTree(['/login']);
      }

      return user.role === 'INSTRUCTOR' ||
        user.role === 'ADMIN'
        ? true
        : router.createUrlTree(['/']);
    }),
    catchError(() =>
      of(router.createUrlTree(['/login']))
    )
  );
};
</CodeBlock>

For a network outage, redirecting to login is a simple fallback but can mislead an already-authenticated user. A production guard should distinguish unavailable authentication service from confirmed `401`, and display a retry page for the former.

---

## 17. Backend HTTP contract tests with PostgreSQL

Our integration tests must verify more than JSON field names. They must prove the security rules still work through HTTP.

The following test uses your existing `PostgresIntegrationTest` Testcontainers base and `MockMvc`.

**Create:** `src/test/java/com/learnhub/web/FullStackContractTest.java`

<CodeBlock language="java" editable>
package com.learnhub.web;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.learnhub.LearnHubApplication;
import com.learnhub.PostgresIntegrationTest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.test.web.servlet.MockMvc;

import static org.junit.jupiter.api.Assertions.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest(classes = LearnHubApplication.class)
@AutoConfigureMockMvc
@Transactional
class FullStackContractTest
        extends PostgresIntegrationTest {

    @Autowired MockMvc mvc;
    @Autowired JdbcTemplate jdbc;
    @Autowired ObjectMapper mapper;

    @Test
    void publicCatalogUsesExpectedEnvelope()
            throws Exception {
        String json = mvc.perform(
                get("/api/v1/courses")
                    .param("page", "0")
                    .param("size", "5")
            )
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.items").isArray())
            .andExpect(jsonPath("$.totalItems").isNumber())
            .andExpect(jsonPath("$.pageNumber").value(0))
            .andExpect(jsonPath("$.pageSize").value(5))
            .andReturn()
            .getResponse()
            .getContentAsString();

        JsonNode root = mapper.readTree(json);

        for (JsonNode course : root.path("items")) {
            assertEquals(
                    "PUBLISHED",
                    course.path("status").asText()
            );

            assertTrue(course.has("id"));
            assertTrue(course.has("title"));
            assertTrue(course.has("isFree"));
            assertTrue(course.has("price"));
        }
    }

    @Test
    void anonymousInstructorListIsProtected()
            throws Exception {
        mvc.perform(
                get("/api/v1/instructor/courses")
            )
            .andExpect(status().isUnauthorized());
    }

    @Test
    void publicCourseDoesNotExposeSeededDraft()
            throws Exception {
        // The Lesson 7 seed uses course ID 5 as DRAFT.
        mvc.perform(get("/api/v1/courses/5"))
            .andExpect(status().isNotFound());
    }

    @Test
    void searchResultsRespectPublishedOnlyRule()
            throws Exception {
        mvc.perform(
                get("/api/v1/courses")
                    .param("search", "Angular")
                    .param("page", "0")
                    .param("size", "10")
            )
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.items").isArray());
    }

    @Test
    void invalidPageSizeIsRejected()
            throws Exception {
        mvc.perform(
                get("/api/v1/courses")
                    .param("page", "0")
                    .param("size", "10000")
            )
            .andExpect(status().isBadRequest());
    }
}
</CodeBlock>

### Contract-test assumptions

This test assumes your Lesson 12 controller enforces a maximum page size and that your Lesson 7 seed still includes draft course ID 5. If those assumptions have changed, use a dedicated test fixture instead of weakening the security assertion.

### Ownership tests to retain

The Lesson 15 tests remain essential:

- Instructor A can publish their own draft.
- Instructor B receives `403` for Instructor A's draft.
- A student receives `403` for instructor operations.
- An anonymous caller receives `401` for protected operations.
- Missing CSRF yields `403`.
- A denied mutation leaves PostgreSQL unchanged.

Do not replace those tests with frontend-only checks.

---

## 18. Angular HttpClient contract tests

Angular should verify request paths, query parameters, JSON bodies, and expected response mapping.

**Create:** `src/app/core/api/course-api.service.spec.ts`

<CodeBlock language="typescript" editable>
import {
  TestBed,
} from '@angular/core/testing';

import {
  provideHttpClient,
} from '@angular/common/http';

import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';

import { CourseApiService }
  from './course-api.service';

import {
  CourseDto,
  CourseWriteDto,
} from './course-api.models';

describe('CourseApiService', () => {
  let service: CourseApiService;
  let http: HttpTestingController;

  const course: CourseDto = {
    id: 101,
    title: 'Spring Boot Fundamentals',
    description: 'Learn Spring Boot.',
    instructor: 'Alex Morgan',
    category: 'Development',
    price: 0,
    rating: 0,
    isFree: true,
    level: 'BEGINNER',
    status: 'DRAFT',
    thumbnail: null,
    durationHours: 8,
    lastUpdated: '2026-10-01',
  };

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });

    service = TestBed.inject(CourseApiService);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    http.verify();
  });

  it('sends typed catalog query parameters', () => {
    service.catalog({
      search: 'spring',
      freeOnly: true,
      page: 1,
      size: 6,
    }).subscribe(response => {
      expect(response.items).toEqual([course]);
      expect(response.pageNumber).toBe(1);
    });

    const request = http.expectOne(req =>
      req.url === '/api/v1/courses'
    );

    expect(request.request.method).toBe('GET');
    expect(request.request.params.get('search'))
      .toBe('spring');
    expect(request.request.params.get('freeOnly'))
      .toBe('true');
    expect(request.request.params.get('page'))
      .toBe('1');

    request.flush({
      items: [course],
      totalItems: 1,
      pageNumber: 1,
      pageSize: 6,
      totalPages: 2,
    });
  });

  it('never accepts an instructor ID for myCourses', () => {
    service.myCourses(0, 10).subscribe();

    const request = http.expectOne(req =>
      req.url === '/api/v1/instructor/courses'
    );

    expect(request.request.method).toBe('GET');
    expect(request.request.params.has('instructorId'))
      .toBeFalse();

    request.flush({
      items: [],
      totalItems: 0,
      pageNumber: 0,
      pageSize: 10,
      totalPages: 0,
    });
  });

  it('posts a draft to the protected API', () => {
    const body: CourseWriteDto = {
      title: course.title,
      description: course.description,
      instructor: course.instructor,
      category: course.category,
      price: 0,
      isFree: true,
      level: 'BEGINNER',
      rating: 0,
      thumbnail: null,
      durationHours: 8,
    };

    service.createDraft(body).subscribe(result => {
      expect(result.id).toBe(101);
    });

    const request = http.expectOne(
      '/api/v1/courses'
    );

    expect(request.request.method).toBe('POST');
    expect(request.request.body).toEqual(body);

    request.flush(course);
  });

  it('loads a draft through the private detail API', () => {
    service.manageableCourse(101).subscribe(result => {
      expect(result.status).toBe('DRAFT');
    });

    const request = http.expectOne(
      '/api/v1/instructor/courses/101'
    );

    expect(request.request.method).toBe('GET');
    request.flush(course);
  });

  it('uses PATCH for publishing', () => {
    service.publish(101).subscribe();

    const request = http.expectOne(
      '/api/v1/courses/101/publish'
    );

    expect(request.request.method).toBe('PATCH');

    request.flush(null, {
      status: 204,
      statusText: 'No Content',
    });
  });
});
</CodeBlock>

### What these tests do not prove

`HttpTestingController` does not run the Spring backend, a real browser, or the Angular development proxy.

It proves that the TypeScript client constructs the intended HTTP requests.

The backend MockMvc tests prove the server contract. A browser smoke test completes the integration picture.

---

## 19. Verify XSRF behavior separately

The critical browser sequence is:

<list marker="number" gap={2}>
  <list-item>Request `GET /api/v1/security/csrf` to establish the XSRF cookie.</list-item>
  <list-item>Send `POST /api/v1/auth/login` with `X-XSRF-TOKEN`.</list-item>
  <list-item>After successful login, fetch a new CSRF token because the authentication transition clears the old one.</list-item>
  <list-item>Send `POST`, `PUT`, `PATCH`, or `DELETE` requests with the current token.</list-item>
  <list-item>After logout, confirm the old session cannot access instructor APIs.</list-item>
</list>

### Inspect browser DevTools

Open Chrome or Edge DevTools → Network, then create a draft.

Check the request:

```text
POST /api/v1/courses
Cookie: JSESSIONID=...
X-XSRF-TOKEN: ...
Content-Type: application/json
```

The actual session cookie value must be HttpOnly and should not be readable through JavaScript.

The `XSRF-TOKEN` cookie, by contrast, must be readable by Angular's XSRF mechanism, so it is normally not HttpOnly.

If a write returns `403`, inspect whether it was denied by CSRF, role authorization, or ownership. All three can produce `403`, but the causes are different.

### CSRF bootstrap edge case

The Spring Security CSRF endpoint must cause the token to be materialized and the cookie written. Merely exposing a lazy token object without accessing its value may not issue the cookie.

Your Lesson 14 `/security/csrf` implementation should access the token value and return it or otherwise ensure the cookie is generated.

---

## 20. End-to-end browser acceptance test

Run Spring Boot:

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn spring-boot:run `
  "-Dspring-boot.run.arguments=--spring.profiles.active=dev"
```

In another terminal, run Angular:

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd serve --proxy-config proxy.conf.json
```

Open `http://localhost:4200`.

### Acceptance scenario A — Anonymous catalog

1. Open `/courses` without logging in.
2. Confirm published courses appear.
3. Search for `Angular`.
4. Confirm the browser requests `/api/v1/courses?search=Angular...`.
5. Confirm drafts do not appear.
6. Open a published course detail page.
7. Confirm an unpublished course cannot be fetched through the public detail route.

### Acceptance scenario B — Student

1. Register a student using the Lesson 14 registration page.
2. Log in.
3. Refresh the browser.
4. Confirm `/api/v1/auth/me` restores the session.
5. Attempt to navigate to `/instructor`.
6. Confirm the guard blocks the page.
7. Attempt a protected course write directly.
8. Confirm the backend returns `403`.

### Acceptance scenario C — Instructor

Use a trusted admin-promoted instructor account.

1. Log in and refresh the page.
2. Open `/instructor`.
3. Confirm the dashboard shows only that instructor's courses.
4. Create a draft.
5. Edit its title and description.
6. Publish it.
7. Confirm it appears in the public catalog.
8. Confirm it cannot be edited as a draft after publishing.
9. Log out.
10. Confirm the old session cannot access the instructor API.

### Acceptance scenario D — Ownership

Use two distinct instructor accounts.

1. Instructor A creates a draft.
2. Instructor B attempts to edit or publish A's draft.
3. Confirm `403 Forbidden`.
4. Query PostgreSQL to confirm the draft was not modified.
5. Instructor A performs the same operation successfully.

This is the most important full-stack security scenario in Lesson 16.

---

## 21. Common integration failures

| Symptom | Likely cause | Check |
|---|---|---|
| Angular request gets `404` | Wrong route or missing proxy | Browser Network URL and Spring mappings |
| Login succeeds, `/me` returns `401` | SecurityContext not persisted or cookie not sent | Session cookie and Lesson 14 login service |
| POST returns `403` | Missing/old CSRF token | `XSRF-TOKEN` cookie and request header |
| Instructor sees `403` on own course | Missing owner FK or stale role | `instructor_user_id`, principal ID |
| Draft edit returns `404` | Using public detail API | Use private instructor detail route |
| Catalog shows drafts | Public repository query lacks published filter | Backend service query |
| Browser refresh appears logged out | Auth state not initialized | `/auth/me` startup request |
| Angular compile fails | Old UI model differs from API DTO | Adapter and component imports |
| Backend starts but JPA fails | Migration/entity mismatch | Flyway history and Hibernate validation |
| One instructor sees another's courses | Query trusts client instructor ID | Query must use authenticated principal |

### Useful PowerShell diagnostics

Inspect API routes in source:

```powershell
cd $HOME\angular-learning\learnhub-backend

Get-ChildItem src\main\java -Recurse -Filter *.java |
  Select-String 'RequestMapping|GetMapping|PostMapping|PutMapping|PatchMapping|DeleteMapping'
```

Inspect Angular hardcoded backend URLs:

```powershell
cd $HOME\angular-learning\learnhub

Get-ChildItem src\app -Recurse -Filter *.ts |
  Select-String 'http://localhost:8080|https://localhost:8080'
```

Inspect PostgreSQL ownership:

```powershell
$psql = "C:\Program Files\PostgreSQL\17\bin\psql.exe"

& $psql -U learnhub_app -h localhost -d learnhub `
  -c "SELECT id, title, status, instructor_user_id FROM courses ORDER BY id;"
```

Use the installed PostgreSQL version's `psql.exe` path if it differs.

---

## 22. Exercises

<box gap={3}>
  <row align="start" gap={3}>
    <icon name="search" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise A — Reactive catalog query state**

      Refactor the catalog page so search, filters, sorting, and pagination use one observable pipeline. Use `switchMap()` to prevent stale responses from overwriting newer results. Add a test for rapid search changes.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="shield-check" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise B — Full session contract test**

      Use MockMvc and Testcontainers to register, log in, refresh CSRF, create a draft, and fetch it through the private instructor route. Use a trusted instructor fixture and a real session.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="file-pen-line" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise C — Form validation**

      Add a cross-field Angular validator for `isFree` and `price`. Display backend `fieldErrors` next to the relevant controls and preserve server validation as authoritative.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="refresh-cw" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise D — Error and retry behavior**

      Simulate `401`, `403`, `409`, and network failure responses. Ensure the UI shows useful messages, preserves form input, and never automatically retries non-idempotent writes.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <icon name="monitor-check" color="secondary" size="lg"/>
    <box flex="1" gap={1}>
      **Exercise E — Browser automation**

      Add a Playwright test that signs in as an instructor, creates a draft, publishes it, and verifies it appears in the public catalog. Use isolated fixtures and deterministic cleanup.
    </box>
  </row>
</box>

---

## 23. Knowledge quiz

{@body const quiz=[{q:"Why should Angular use relative /api/v1 URLs with the development proxy?",a:["It eliminates server authorization checks","It provides a same-origin browser integration for cookies and XSRF","It stores the session in localStorage"],c:1,why:"The proxy forwards API requests while the browser sees the Angular origin, simplifying cookie and XSRF behavior."},{q:"Which endpoint should an instructor use to load their unpublished draft for editing?",a:["The public published-only course detail endpoint","The instructor-authorized private course detail endpoint","The public catalog search endpoint"],c:1,why:"Public detail routes must not expose drafts. A private owner-authorized endpoint is required."},{q:"What prevents Instructor B from modifying Instructor A's course?",a:["An Angular route guard alone","A hidden edit button","Backend role and database-backed ownership checks"],c:2,why:"Only the backend can enforce resource authorization against direct API calls."},{q:"Why should the frontend refresh CSRF after login?",a:["Authentication may invalidate the pre-login CSRF token","PostgreSQL changes the CSRF header","Angular resets all browser cookies on navigation"],c:0,why:"The authentication transition can clear the previous token; the next unsafe request needs the current token."},{q:"What is the main purpose of switchMap in a reactive search pipeline?",a:["It guarantees SQL uses an index","It unsubscribes from a previous request when a new query arrives","It automatically retries failed POST requests"],c:1,why:"SwitchMap avoids stale in-flight searches overwriting results for newer queries."}]}
{@body const [answers,setAnswers]=DIL.useState({})}
{@body const [showAnswers,setShowAnswers]=DIL.useState(false)}
<box border radius="xl" padding={3} gap={3}>
  <row justify="between" align="center">
    **Lesson 16 — Full-stack integration**
    <text color="secondary" size="xs">5 questions</text>
  </row>
  {#each quiz as q,i}
    <box key={i} gap={2}>
      **{i+1}. {q.q}**
      <radio-group direction="col" value={answers[i]===undefined?"":String(answers[i])} onChange={v=>{setAnswers(old=>({...old,[i]:Number(v)}));setShowAnswers(false);}}>
        {#each q.a as option,j}
          <radio value={String(j)}>{option}</radio>
        {/each}
      </radio-group>
      {#if showAnswers}
        <row align="start" gap={2}>
          <icon name={answers[i]===q.c?"check-circle":"info"} color={answers[i]===q.c?"success":"secondary"}/>
          <box flex="1" gap={1}>
            <text size="sm" weight="medium">{answers[i]===q.c?"Correct":"Correct answer: "+q.a[q.c]}</text>
            <text color="secondary" size="sm">{q.why}</text>
          </box>
        </row>
      {/if}
    </box>
    {#if i<quiz.length-1}<divider color="subtle"/>{/if}
  {/each}
  {#if showAnswers}
    <title size="lg">{quiz.filter((q,i)=>answers[i]===q.c).length} / 5 correct</title>
  {/if}
  <row gap={2}>
    <button block disabled={Object.keys(answers).length!==quiz.length} onClick={()=>setShowAnswers(true)}>Check answers</button>
    <button variant="outline" color="secondary" onClick={()=>{setAnswers({});setShowAnswers(false);}}>Reset</button>
  </row>
</box>

## 24. Completion checklist

{@body const groups=[{name:"Backend contract",items:["Verified Lesson 15 tests pass","Confirmed public catalog response DTO","Confirmed published-only public detail route","Completed instructor-owned list controller","Added private owner-authorized draft detail route","Verified all write endpoints use ownership-aware service","Verified CSRF and session security remain enabled"]},{name:"Angular integration",items:["Configured same-origin development proxy","Created typed course API models","Implemented CourseApiService","Mapped backend DTOs to presentation data","Implemented three-state authentication model","Configured startup auth restoration","Configured Angular XSRF support","Implemented consistent API error handling","Connected public catalog search and pagination","Connected instructor dashboard","Connected draft creation","Connected private draft editing","Preserved Angular route guards"]},{name:"Testing",items:["Added backend contract tests","Added Angular HttpClient tests","Verified anonymous catalog","Verified student authorization denial","Verified instructor-owned CRUD","Verified cross-instructor denial","Verified login and logout cookies","Verified CSRF header on writes","Ran backend Maven verification","Ran Angular tests and build"]},{name:"Wrap-up",items:["Completed two hands-on exercises","Completed knowledge quiz","Reviewed Git diffs","Committed verified backend and frontend changes"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const total=groups.reduce((sum,g)=>sum+g.items.length,0)}
{@body const completed=Object.values(done).filter(Boolean).length}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 16 progress**
      <text size="xs" color="secondary">Mark items only after local verification.</text>
    </box>
    <title size="lg" tabularNums>{completed}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" height="100%" width={`${completed/total*100}%`} radius="full"/>
  </box>
  {#each groups as group,i}
    <box key={i} gap={2}>
      **{group.name}**
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!done[`${i}-${j}`]} onChange={value=>setDone(old=>({...old,[`${i}-${j}`]:value}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if completed===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 16 checklist completed.**
    </row>
  {/if}
</box>

## 25. Final verification commands

### Backend

```powershell
cd $HOME\angular-learning\learnhub-backend

mvn "-Dtest=FullStackContractTest,CourseOwnershipIntegrationTest" test

if ($LASTEXITCODE -ne 0) {
    throw "Focused backend integration tests failed."
}

mvn clean verify

if ($LASTEXITCODE -ne 0) {
    throw "Backend verification failed."
}

git diff --check
git status
git diff --stat
```

### Angular

```powershell
cd $HOME\angular-learning\learnhub

ng.cmd test --watch=false

if ($LASTEXITCODE -ne 0) {
    throw "Angular unit tests failed."
}

ng.cmd build

if ($LASTEXITCODE -ne 0) {
    throw "Angular build failed."
}

git diff --check
git status
git diff --stat
```

### Pre-commit engineering corrections

The snippets in this lesson form an integration reference, but several details must be reconciled before treating the application as complete:

1. **Private DTO mapping:** Match `CourseEntity` getter names, metadata lookup methods, and `CourseResponse` factories to your existing code.
2. **Authentication error handling:** Ensure the error interceptor preserves `status` for the auth service and doesn't mistake network failures for logout.
3. **Catalog request races:** Replace separate pagination subscriptions with a single query stream before production use.
4. **Draft editing:** Preserve existing rating/thumbnail values, and disable saving if the private draft load fails.
5. **Backend status codes:** Keep the Angular `publish()` and `archive()` return types aligned with actual controller responses.
6. **Testing:** The supplied backend contract test checks basic HTTP shape. The existing ownership tests and a real-session lifecycle test are still necessary to prove security.

### Git checkpoints

After verifying both projects:

```powershell
cd $HOME\angular-learning\learnhub-backend

git add src\main src\test
git diff --cached --stat
git commit -m "Expose secure full-stack course integration APIs"
```

```powershell
cd $HOME\angular-learning\learnhub

git add src\app proxy.conf.json
git diff --cached --stat
git commit -m "Connect Angular LearnHub to Spring Boot backend"
```

Review staged changes before committing. Never commit database passwords, real credentials, session cookies, or environment secrets.

## 26. Lesson summary

| Layer | What we implemented |
|---|---|
| Angular HTTP | Typed `CourseApiService` |
| Authentication | Restored session state after refresh |
| CSRF | Same-origin cookie/header integration |
| Catalog | Backend-powered search and pagination |
| Instructor UI | Owned-course dashboard |
| Course management | Create, edit, publish via secured API |
| Authorization | Role and owner enforcement remains server-side |
| API contract | Public versus private course read separation |
| Error handling | Normalized Problem Details |
| Tests | MockMvc/Testcontainers and Angular HttpClient contracts |

The most important design decision is that **Angular never decides which instructor owns a course**. The backend derives the user ID from the authenticated session and checks ownership against PostgreSQL.

The code has not been executed against your local LearnHub repositories here, so passing builds and tests remain verification tasks rather than claimed results.

## 27. Next lesson

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="bookmark-check" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 17 — Wishlist and Enrollments**

    We'll replace the frontend's mock wishlist/enrollment behavior with persisted PostgreSQL relationships, authenticated student APIs, duplicate prevention, enrollment eligibility rules, transactional operations, and Angular integration.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 17: PostgreSQL-backed wishlist and enrollments, Flyway migrations, student-to-course relationships, authenticated session APIs, duplicate prevention, enrollment eligibility, ownership and role rules, transactional services, Spring Data JPA repositories, DTO mapping, Angular standalone wishlist and enrollment integration, MockMvc and Testcontainers tests, Windows PowerShell commands, complete documented code, exercises, quiz, and checklist. Continue from Lesson 16 with experienced-Java pacing.")}>Start Lesson 17 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 16 deliverable:** An integrated LearnHub frontend and backend architecture with typed HTTP contracts, server-backed catalog data, authenticated instructor workflows, session/CSRF handling, and cross-layer tests.