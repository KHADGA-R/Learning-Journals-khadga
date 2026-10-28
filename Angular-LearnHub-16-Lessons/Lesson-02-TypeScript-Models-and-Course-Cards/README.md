# Lesson 2: TypeScript, Models, and Course Cards

> Reconstructed and updated LearnHub curriculum. Windows PowerShell commands assume `$HOME\angular-learning\learnhub`. This lesson is a teaching chapter; later lessons evolve the same code, so do not apply every historical snippet simultaneously.

## Learning outcome
Define a typed Course model and render sample marketplace cards.

## Prerequisites and workflow
Start in PowerShell with `cd $HOME\angular-learning\learnhub` for Lessons 2–16. Run `ng.cmd serve --open` during development and `ng.cmd build` to check compilation. Use the exact paths below, and merge imports into existing files instead of duplicating declarations.

## Step 1: Create a typed model

```typescript
// src/app/models/course.model.ts
export type CourseLevel = 'Beginner' | 'Intermediate' | 'Advanced';
export interface Course {
  id: number; title: string; description: string;
  instructor: string; category: string; price: number;
  rating: number; thumbnail: string; isFree: boolean;
  level: CourseLevel; durationHours: number; lastUpdated: string;
}
```

**Explanation:** The interface defines a compile-time contract; it does not validate external API data.

## Step 2: Create sample data

```typescript
// src/app/data/courses.ts
import { Course } from '../models/course.model';
export const COURSES: Course[] = [
  { id: 1, title: 'Angular Complete Guide', description: 'Build Angular apps', instructor: 'Alex Morgan', category: 'Web Development', price: 49.99, rating: 4.8, thumbnail: 'https://placehold.co/600x350', isFree: false, level: 'Beginner', durationHours: 42.5, lastUpdated: '2026-09-15' },
  { id: 2, title: 'Python for Beginners', description: 'Start programming', instructor: 'Sarah Lee', category: 'Programming', price: 0, rating: 4.7, thumbnail: 'https://placehold.co/600x350', isFree: true, level: 'Beginner', durationHours: 18, lastUpdated: '2026-08-10' }
];
```

**Explanation:** Maintain consistent numeric IDs within Angular.

## Step 3: Generate a card

```powershell
ng.cmd generate component components/course-card
```

**Explanation:** In `course-card.ts`, add `readonly course = input.required<Course>()` and import `input` and the model. In its template render `{{ course().title }}`, `{{ course().instructor }}`, and `[src]="course().thumbnail"`.

## Hands-on exercise
Add a third typed course and display price with `currency` pipe after importing `CurrencyPipe`.

## Knowledge check
1. **What is a TypeScript interface?**
   - Answer: A compile-time structural contract.
2. **How is a signal input read?**
   - Answer: By calling it, for example `course()`.
3. **Should external API JSON be trusted because it has a TypeScript type?**
   - Answer: No; runtime validation is still needed.

## Verification
Run `ng.cmd build`. Check the browser console and resolve template/import errors before continuing. Keep this lesson's changes in source control.

## Next lesson
Continue to Lesson 3 in the parent curriculum index.
