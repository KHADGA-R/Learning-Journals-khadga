# Lesson 4: Data Binding, Events, Search, and Filters

> Reconstructed and updated LearnHub curriculum. Windows PowerShell commands assume `$HOME\angular-learning\learnhub`. This lesson is a teaching chapter; later lessons evolve the same code, so do not apply every historical snippet simultaneously.

## Learning outcome
Connect user input to a reactive catalog.

## Prerequisites and workflow
Start in PowerShell with `cd $HOME\angular-learning\learnhub` for Lessons 2–16. Run `ng.cmd serve --open` during development and `ng.cmd build` to check compilation. Use the exact paths below, and merge imports into existing files instead of duplicating declarations.

## Step 1: Understand bindings

```html
<h2>{{ course.title }}</h2>
<img [src]="course.thumbnail" [alt]="course.title" />
<button (click)="selectCourse(course.id)">Select</button>
```

**Explanation:** Interpolation renders text, property binding sets DOM properties, and event binding handles interactions.

## Step 2: Build a signal search

```typescript
import { Component, computed, signal } from '@angular/core';
import { COURSES } from '../../data/courses';
@Component({ selector: 'app-courses', standalone: true, templateUrl: './courses.html' })
export class Courses {
  readonly search = signal('');
  readonly filtered = computed(() => COURSES.filter(c =>
    c.title.toLowerCase().includes(this.search().trim().toLowerCase())));
  updateSearch(value: string): void { this.search.set(value); }
}
```

**Explanation:** Computed signals update when their dependencies change.

## Step 3: Bind the search UI

```html
<input #searchBox type="search" aria-label="Search courses" (input)="updateSearch(searchBox.value)" />
@for (course of filtered(); track course.id) {
  <article><h2>{{ course.title }}</h2><p>{{ course.instructor }}</p></article>
} @empty { <p>No matching courses.</p> }
```

**Explanation:** Use a stable course ID for tracking.

## Step 4: Add category and price filters

Use signals for category (`'all'` by default) and price (`'all'|'free'|'paid'`). Extend the computed predicate with each filter, keeping it pure.

**Explanation:** undefined

## Hands-on exercise
Add a Reset Filters button and a live result count.

## Knowledge check
1. **What does `(input)` represent?**
   - Answer: An event binding.
2. **What does `computed()` do?**
   - Answer: Derives reactive state.
3. **Why use `track course.id`?**
   - Answer: To preserve stable list item identity.

## Verification
Run `ng.cmd build`. Check the browser console and resolve template/import errors before continuing. Keep this lesson's changes in source control.

## Next lesson
Continue to Lesson 5 in the parent curriculum index.
