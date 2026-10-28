# Lesson 12: Enrollment, Lesson Completion, and Progress

> Reconstructed or updated LearnHub curriculum chapter. This chapter is part of a progressive application: earlier code evolves in later lessons. Merge targeted changes into existing files rather than pasting historical fragments together.

## Learning objectives
Model Enrollment with courseId, enrolledAt and completedLessonIds. Build EnrollmentService with signals. Enroll only in existing free courses; reject duplicates. Validate lesson membership before toggling completion. Calculate rounded progress as completed valid lessons divided by total lessons. Persist safely in browser storage, acknowledging that it is not private or account-specific.

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
export interface Enrollment {courseId:number;enrolledAt:string;completedLessonIds:number[];}
progressFor(courseId:number):number {const e=this.enrollments().find(x=>x.courseId===courseId);const lessons=this.courseService.getLessonsByCourseId(courseId);if(!e||!lessons.length)return 0;return Math.round(100*lessons.filter(l=>e.completedLessonIds.includes(l.id)).length/lessons.length);}
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
Complete two of three lessons and confirm 67%. Reject unknown lesson IDs.

## Quiz with answers
1. **How is progress calculated?** — Completed divided by total
2. **Can the demo enroll paid courses?** — No

## Completion checklist
- [ ] The lesson feature works with the existing LearnHub code.
- [ ] I tested success and failure behavior.
- [ ] I understand security or scalability limitations.
- [ ] The Angular build completes without errors.

## Next lesson
Proceed to Lesson 13.
