# Lesson 10: HttpClient, REST APIs, and JSON Server

> Reconstructed or updated LearnHub curriculum chapter. This chapter is part of a progressive application: earlier code evolves in later lessons. Merge targeted changes into existing files rather than pasting historical fragments together.

## Learning objectives
Install JSON Server, create api/db.json with a courses array, and start the mock API on port 3000. Add provideHttpClient(withFetch()) to app.config.ts. Convert string API IDs to numeric Course IDs. Show loading, error and retry states. JSON Server is not production authentication.

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
import { HttpClient } from '@angular/common/http';
import { map } from 'rxjs';
private readonly http=inject(HttpClient);
getCourses(){return this.http.get<Array<Omit<Course,'id'>&{id:string}>>('http://localhost:3000/courses').pipe(map(rows=>rows.map(c=>({...c,id:Number(c.id)}))),map(rows=>{if(rows.some(c=>!Number.isSafeInteger(c.id)||c.id<=0))throw new Error('Invalid ID');return rows;}));}
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
Stop the API and verify the error and retry experience.

## Quiz with answers
1. **What does HttpClient return?** — Observable
2. **Where should IDs be normalized?** — API boundary

## Completion checklist
- [ ] The lesson feature works with the existing LearnHub code.
- [ ] I tested success and failure behavior.
- [ ] I understand security or scalability limitations.
- [ ] The Angular build completes without errors.

## Next lesson
Proceed to Lesson 11.
