# Lesson 6: Signals, Computed State, and Wishlist

> Reconstructed and updated LearnHub curriculum. Windows PowerShell commands assume `$HOME\angular-learning\learnhub`. This lesson is a teaching chapter; later lessons evolve the same code, so do not apply every historical snippet simultaneously.

## Learning outcome
Create reactive saved-course state and browser persistence.

## Prerequisites and workflow
Start in PowerShell with `cd $HOME\angular-learning\learnhub` for Lessons 2–16. Run `ng.cmd serve --open` during development and `ng.cmd build` to check compilation. Use the exact paths below, and merge imports into existing files instead of duplicating declarations.

## Step 1: Create a wishlist service

```typescript
// src/app/services/wishlist.ts
import { Injectable, computed, signal } from '@angular/core';
@Injectable({ providedIn: 'root' })
export class Wishlist {
  private readonly ids = signal<number[]>([]);
  readonly savedIds = this.ids.asReadonly();
  readonly count = computed(() => this.ids().length);
  isSaved(id: number): boolean { return this.ids().includes(id); }
  toggle(id: number): void {
    this.ids.update(ids => ids.includes(id) ? ids.filter(x => x !== id) : [...ids, id]);
  }
}
```

**Explanation:** Signal updates should produce new arrays rather than mutating existing ones.

## Step 2: Wire a wishlist button

```html
<button type="button" [attr.aria-pressed]="wishlist.isSaved(course().id)" (click)="wishlist.toggle(course().id)">
  {{ wishlist.isSaved(course().id) ? 'Remove from wishlist' : 'Save to wishlist' }}
</button>
```

**Explanation:** Inject the service in CourseCard. The signal-backed service is shared across components.

## Step 3: Persist safely

Read `localStorage` only when `isPlatformBrowser(inject(PLATFORM_ID))` is true. Parse JSON in `try/catch`, validate positive integer IDs, and persist signal updates using `effect()`. Do not filter stored IDs against an API catalog before that catalog finishes loading.

**Explanation:** undefined

## Step 4: Build a wishlist page

Compute saved courses by filtering `CourseService.courses()` against saved IDs. Show a friendly empty state and navigation back to Courses.

**Explanation:** undefined

## Hands-on exercise
Persist and restore the wishlist on refresh; test with an empty or corrupted localStorage value.

## Knowledge check
1. **What does `signal()` hold?**
   - Answer: Writable reactive state.
2. **What does `asReadonly()` provide?**
   - Answer: A read-only signal view.
3. **Why guard localStorage access?**
   - Answer: It is unavailable during server rendering.

## Verification
Run `ng.cmd build`. Check the browser console and resolve template/import errors before continuing. Keep this lesson's changes in source control.

## Next lesson
Continue to Lesson 7 in the parent curriculum index.
