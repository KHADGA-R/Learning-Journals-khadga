# Angular LearnHub — 16-Lesson Learning Journal

A progressive Angular course-marketplace curriculum, reconstructed from the LearnHub teaching sequence. Each lesson has its own folder and README with learning goals, PowerShell workflow, Angular code patterns, exercises, quizzes and completion checks.

**Important:** These are reconstructed, edited study notes rather than verbatim transcripts of earlier lessons. Code in a chapter may be illustrative or a targeted fragment; it is not necessarily a drop-in replacement for an entire existing project file. Follow the lessons in order and run a build after each integration.

## Windows setup

```powershell
cd $HOME\angular-learning\learnhub
npm.cmd install
ng.cmd serve --open
```

For lessons using the local mock API, start a second terminal:

```powershell
cd $HOME\angular-learning\learnhub
npx.cmd json-server api/db.json --port 3000
```

## Lesson index

1. [Lesson 01 — Angular Fundamentals and Setup](./Lesson-01-Angular-Fundamentals-and-Setup/README.md)
2. [Lesson 02 — TypeScript Models and Course Cards](./Lesson-02-TypeScript-Models-and-Course-Cards/README.md)
3. [Lesson 03 — Components and Layout](./Lesson-03-Components-and-Layout/README.md)
4. [Lesson 04 — Data Binding Events and Filters](./Lesson-04-Data-Binding-Events-and-Filters/README.md)
5. [Lesson 05 — Directives Pipes and Control Flow](./Lesson-05-Directives-Pipes-and-Control-Flow/README.md)
6. [Lesson 06 — Signals and Wishlist](./Lesson-06-Signals-and-Wishlist/README.md)
7. [Lesson 07 — Services and Dependency Injection](./Lesson-07-Services-and-Dependency-Injection/README.md)
8. [Lesson 08 — Routing and Course Details](./Lesson-08-Routing-and-Course-Details/README.md)
9. [Lesson 09 — Reactive Forms and Validation](./Lesson-09-Reactive-Forms-and-Validation/README.md)
10. [Lesson 10 — HTTP and REST APIs](./Lesson-10-HTTP-and-REST-APIs/README.md)
11. [Lesson 11 — RxJS and Search](./Lesson-11-RxJS-and-Search/README.md)
12. [Lesson 12 — Enrollment and Progress](./Lesson-12-Enrollment-and-Progress/README.md)
13. [Lesson 13 — Authentication and Roles](./Lesson-13-Authentication-and-Roles/README.md)
14. [Lesson 14 — Performance and Architecture](./Lesson-14-Performance-and-Architecture/README.md)
15. [Lesson 15 — Testing and Deployment](./Lesson-15-Testing-and-Deployment/README.md)
16. [Lesson 16 — Final MVP and Release](./Lesson-16-Final-MVP-and-Release/README.md)

## Release verification

```powershell
ng.cmd test --watch=false
ng.cmd build --configuration production
```

## Security scope

The project uses demo browser-local identities, wishlist and enrollment state, and JSON Server for catalog data. It does not implement secure authentication, account-specific authorization, payment verification, or a production backend. Never store passwords or secrets in browser code. Do not use client route guards as a backend security boundary.

## Study method

Read each lesson, implement changes in your local Angular project, complete the exercise and quiz, and check your build. The journal stores educational documentation, not a verified copy of a complete runnable application.
