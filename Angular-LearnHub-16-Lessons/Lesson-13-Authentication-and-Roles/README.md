# Lesson 13: Authentication, Authorization, Guards, and Roles

> Reconstructed or updated LearnHub curriculum chapter. This chapter is part of a progressive application: earlier code evolves in later lessons. Merge targeted changes into existing files rather than pasting historical fragments together.

## Learning objectives
Create a demo AuthService with a signal-backed AuthUser and roles student/instructor. Provide demo sign-in and sign-out, guest authGuard and instructorGuard, role-aware header links, and an instructor dashboard. Do not store passwords. Browser role state and route guards are UX, not trusted authentication or authorization.

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
export type UserRole='student'|'instructor';
export interface AuthUser {id:string;name:string;role:UserRole;}
private readonly state=signal<AuthUser|null>(null);
readonly user=this.state.asReadonly();
readonly isAuthenticated=computed(()=>this.user()!==null);
readonly isInstructor=computed(()=>this.user()?.role==='instructor');
signInDemo(role:UserRole){this.state.set({id:'demo-'+role,name:'Demo '+role,role});}
signOut(){this.state.set(null);}
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
Test guest, student and instructor access to /my-learning and /instructor.

## Quiz with answers
1. **Authentication versus authorization?** — Identity versus permissions
2. **Are route guards security?** — No

## Completion checklist
- [ ] The lesson feature works with the existing LearnHub code.
- [ ] I tested success and failure behavior.
- [ ] I understand security or scalability limitations.
- [ ] The Angular build completes without errors.

## Next lesson
Proceed to Lesson 14.
