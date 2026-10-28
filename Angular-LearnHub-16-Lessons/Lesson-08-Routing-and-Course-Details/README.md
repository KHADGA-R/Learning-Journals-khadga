# Lesson 8: Angular Routing and Course Detail Pages

> Reconstructed and updated LearnHub curriculum. Windows PowerShell commands assume `$HOME\angular-learning\learnhub`. This lesson is a teaching chapter; later lessons evolve the same code, so do not apply every historical snippet simultaneously.

## Learning outcome
Navigate from course cards to URL-addressable detail pages.

## Prerequisites and workflow
Start in PowerShell with `cd $HOME\angular-learning\learnhub` for Lessons 2–16. Run `ng.cmd serve --open` during development and `ng.cmd build` to check compilation. Use the exact paths below, and merge imports into existing files instead of duplicating declarations.

## Step 1: Generate pages

```powershell
ng.cmd generate component pages/courses
ng.cmd generate component pages/course-detail
ng.cmd generate component pages/not-found
```

**Explanation:** The CLI creates standalone components with `.ts`, `.html`, and `.css` files.

## Step 2: Configure routes

```typescript
import { Routes } from '@angular/router';
import { Home } from './pages/home/home';
import { Courses } from './pages/courses/courses';
import { CourseDetail } from './pages/course-detail/course-detail';
import { NotFound } from './pages/not-found/not-found';
export const routes: Routes = [
  { path: '', component: Home },
  { path: 'courses', component: Courses },
  { path: 'courses/:id', component: CourseDetail },
  { path: '**', component: NotFound }
];
```

**Explanation:** Wildcard routes must be last.

## Step 3: Read a route parameter

```typescript
import { ActivatedRoute } from '@angular/router';
import { computed, inject } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs';
const route = inject(ActivatedRoute);
const rawId = toSignal(route.paramMap.pipe(map(p => p.get('id'))), { initialValue: route.snapshot.paramMap.get('id') });
const courseId = computed(() => { const value = rawId(); return value && /^[1-9]\d*$/.test(value) && Number.isSafeInteger(Number(value)) ? Number(value) : null; });
```

**Explanation:** Place these declarations inside the CourseDetail class. Use `CourseService.getCourseById()` and handle missing courses.

## Step 4: Link cards

```html
<a [routerLink]="['/courses', course().id]">{{ course().title }}</a>
```

**Explanation:** Import RouterLink in CourseCard.

## Hands-on exercise
Navigate between `/courses/1` and `/courses/2`; test invalid `/courses/abc` and missing IDs.

## Knowledge check
1. **What does `:id` mean?**
   - Answer: A route parameter.
2. **Why use `paramMap` reactively?**
   - Answer: The route parameter can change without destroying the component.
3. **Where must the wildcard route go?**
   - Answer: Last.

## Verification
Run `ng.cmd build`. Check the browser console and resolve template/import errors before continuing. Keep this lesson's changes in source control.

## Next lesson
Continue to Lesson 9 in the parent curriculum index.
