# Lesson 7: Services and Dependency Injection

> Reconstructed and updated LearnHub curriculum. Windows PowerShell commands assume `$HOME\angular-learning\learnhub`. This lesson is a teaching chapter; later lessons evolve the same code, so do not apply every historical snippet simultaneously.

## Learning outcome
Centralize catalog state in a reusable singleton service.

## Prerequisites and workflow
Start in PowerShell with `cd $HOME\angular-learning\learnhub` for Lessons 2–16. Run `ng.cmd serve --open` during development and `ng.cmd build` to check compilation. Use the exact paths below, and merge imports into existing files instead of duplicating declarations.

## Step 1: Generate a service

```powershell
ng.cmd generate service services/course
```

**Explanation:** Use `providedIn: 'root'` for a shared application service.

## Step 2: Implement shared catalog state

```typescript
// src/app/services/course.ts (early in-memory version)
import { Injectable, computed, signal } from '@angular/core';
import { Course } from '../models/course.model';
import { COURSES } from '../data/courses';
@Injectable({ providedIn: 'root' })
export class CourseService {
  private readonly state = signal<Course[]>(COURSES);
  readonly courses = this.state.asReadonly();
  readonly totalCourses = computed(() => this.courses().length);
  readonly freeCourses = computed(() => this.courses().filter(c => c.isFree));
  getCourseById(id: number): Course | undefined { return this.courses().find(c => c.id === id); }
  addCourse(course: Course): void {
    if (this.getCourseById(course.id)) throw new Error('Duplicate course ID');
    this.state.update(items => [...items, course]);
  }
}
```

**Explanation:** Lesson 10 will replace the initial hardcoded catalog with HTTP loading.

## Step 3: Inject into a page

```typescript
import { inject } from '@angular/core';
import { CourseService } from '../../services/course';
readonly courseService = inject(CourseService);
```

**Explanation:** Dependency injection makes logic reusable and testable.

## Step 4: Render shared state

```html
<p>{{ courseService.totalCourses() }} courses</p>
@for (course of courseService.courses(); track course.id) { <app-course-card [course]="course" /> }
```

**Explanation:** Import CourseCard in the page's component decorator.

## Hands-on exercise
Add a `paidCourses` computed signal and show free/paid totals.

## Knowledge check
1. **What does `providedIn: 'root'` mean?**
   - Answer: A root-provided service is shared through the root injector.
2. **Why inject services?**
   - Answer: To separate responsibilities and allow testing.
3. **Where should catalog state live?**
   - Answer: In the shared CourseService.

## Verification
Run `ng.cmd build`. Check the browser console and resolve template/import errors before continuing. Keep this lesson's changes in source control.

## Next lesson
Continue to Lesson 8 in the parent curriculum index.
