# Lesson 5: Directives, Pipes, and Angular Control Flow

> Reconstructed and updated LearnHub curriculum. Windows PowerShell commands assume `$HOME\angular-learning\learnhub`. This lesson is a teaching chapter; later lessons evolve the same code, so do not apply every historical snippet simultaneously.

## Learning outcome
Format data and conditionally render marketplace content.

## Prerequisites and workflow
Start in PowerShell with `cd $HOME\angular-learning\learnhub` for Lessons 2–16. Run `ng.cmd serve --open` during development and `ng.cmd build` to check compilation. Use the exact paths below, and merge imports into existing files instead of duplicating declarations.

## Step 1: Use built-in pipes

```html
<p>{{ course.price | currency:'USD' }}</p>
<p>{{ course.lastUpdated | date:'mediumDate' }}</p>
<p>{{ course.rating | number:'1.1-1' }}</p>
```

**Explanation:** Import `CurrencyPipe`, `DatePipe`, and `DecimalPipe` in standalone components.

## Step 2: Use conditional control flow

```html
@if (course.isFree) { <strong>FREE</strong> }
@else { <span>{{ course.price | currency:'USD' }}</span> }
@for (course of courses; track course.id) { <app-course-card [course]="course" /> }
```

**Explanation:** `@if` and `@for` are built-in Angular template syntax.

## Step 3: Create a custom price pipe

```typescript
// src/app/pipes/course-price-pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
@Pipe({ name: 'coursePrice', standalone: true })
export class CoursePricePipe implements PipeTransform {
  transform(price: number): string {
    return price === 0 ? 'FREE' : `$${price.toFixed(2)}`;
  }
}
```

**Explanation:** Keep pure presentation transformations in pipes. Import CoursePricePipe where used.

## Step 4: Create a highlight directive

```typescript
// src/app/directives/highlight.ts
import { Directive, ElementRef, inject } from '@angular/core';
@Directive({ selector: '[appHighlight]', standalone: true })
export class Highlight {
  private readonly element = inject(ElementRef<HTMLElement>);
  constructor() { this.element.nativeElement.style.outlineOffset = '2px'; }
}
```

**Explanation:** Prefer host bindings and CSS classes for richer interactive behavior.

## Hands-on exercise
Display FREE for zero-price courses and add an empty catalog message.

## Knowledge check
1. **What is a pipe?**
   - Answer: A template value transformation.
2. **What does `@if` do?**
   - Answer: Conditionally renders template blocks.
3. **Why prefer pure pipes?**
   - Answer: They avoid unnecessary recalculation when inputs are unchanged.

## Verification
Run `ng.cmd build`. Check the browser console and resolve template/import errors before continuing. Keep this lesson's changes in source control.

## Next lesson
Continue to Lesson 6 in the parent curriculum index.
