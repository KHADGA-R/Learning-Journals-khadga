# Lesson 15: Unit Testing, HTTP Testing, and Deployment

> Reconstructed or updated LearnHub curriculum chapter. This chapter is part of a progressive application: earlier code evolves in later lessons. Merge targeted changes into existing files rather than pasting historical fragments together.

## Learning objectives
Test EnrollmentService with TestBed and controlled providers, CourseCard with setInput, functional guards with runInInjectionContext, and CourseApiService with HttpTestingController. Run ng.cmd test --watch=false and production builds. Configure environment API URLs and SPA rewrites; do not place secrets in browser environment files.

## Step 1 — Open the project

```powershell
cd $HOME\angular-learning\learnhub
code .
ng.cmd version
```

Review existing files in src/app, especially app.routes.ts, app.config.ts, and relevant services. Preserve existing imports and providers.

## Step 2 — Implement the lesson concept

The following is a focused, documented code pattern; add the necessary Angular imports and place members inside the appropriate component or service rather than using this fragment as a complete file.

```typescript
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';
TestBed.configureTestingModule({providers:[CourseApiService,provideHttpClient(),provideHttpClientTesting()]});
const http=TestBed.inject(HttpTestingController);
service.getCourses().subscribe(rows=>expect(rows.length).toBe(0));
http.expectOne('http://localhost:3000/courses').flush([]);
http.verify();
```

## Step 3 — Integrate and validate

Use the Angular development server:

```powershell
ng.cmd serve --open
```

For HTTP-dependent lessons, run the local API in a separate PowerShell terminal:

```powershell
npx.cmd json-server api/db.json --port 3000
```

Check the browser console, Network tab, and application error states. Run `ng.cmd build` after changes.

## Step 4 — Hands-on exercise
Test paid enrollment rejection, guard redirects and an HTTP 500 error.

## Quiz with answers
1. **What mocks HTTP requests?** — HttpTestingController
2. **Why SPA rewrites?** — Direct route refresh

## Completion checklist
- [ ] The lesson feature works with the existing LearnHub code.
- [ ] I tested success and failure behavior.
- [ ] I understand security or scalability limitations.
- [ ] The Angular build completes without errors.

## Next lesson
Proceed to Lesson 16.
