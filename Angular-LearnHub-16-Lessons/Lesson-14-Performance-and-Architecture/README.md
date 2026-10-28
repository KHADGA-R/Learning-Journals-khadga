# Lesson 14: Lazy Loading, OnPush, and Architecture

> Reconstructed or updated LearnHub curriculum chapter. This chapter is part of a progressive application: earlier code evolves in later lessons. Merge targeted changes into existing files rather than pasting historical fragments together.

## Learning objectives
Replace eager route component references with loadComponent dynamic imports. Use OnPush for CourseCard, track course.id in @for, and NgOptimizedImage with dimensions. Avoid lazy loading critical LCP images. Prevent duplicate catalog requests and measure production bundle sizes. Consider core/shared/features architecture.

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
{path:'courses',loadComponent:()=>import('./pages/courses/courses').then(m=>m.Courses)}
// CourseCard decorator: changeDetection: ChangeDetectionStrategy.OnPush
// Template: <img [ngSrc]="course().thumbnail" width="600" height="350" loading="lazy" [alt]="course().title">
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
Measure initial and lazy chunks before and after optimization.

## Quiz with answers
1. **What enables lazy routes?** — loadComponent
2. **Why track IDs?** — Stable DOM identity

## Completion checklist
- [ ] The lesson feature works with the existing LearnHub code.
- [ ] I tested success and failure behavior.
- [ ] I understand security or scalability limitations.
- [ ] The Angular build completes without errors.

## Next lesson
Proceed to Lesson 15.
