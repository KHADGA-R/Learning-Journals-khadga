# Lesson 11: RxJS, Debounced Search, and Cancellation

> Reconstructed or updated LearnHub curriculum chapter. This chapter is part of a progressive application: earlier code evolves in later lessons. Merge targeted changes into existing files rather than pasting historical fragments together.

## Learning objectives
Use Subject, debounceTime(300), distinctUntilChanged, switchMap, catchError, and takeUntilDestroyed to search the catalog. switchMap cancels the previous subscription when a newer query arrives. Catch errors inside switchMap to preserve the outer stream. For large catalogs use server search and pagination.

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
this.searchInput$.pipe(debounceTime(300),distinctUntilChanged(),switchMap(term=>this.api.searchCourses(term).pipe(catchError(()=>of([])))),takeUntilDestroyed(this.destroyRef)).subscribe(results=>this.searchResults.set(results));
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
Type quickly and inspect the Network tab. Add retry and error messages.

## Quiz with answers
1. **Why debounce?** — Avoid excessive requests
2. **Why switchMap?** — Use the latest query

## Completion checklist
- [ ] The lesson feature works with the existing LearnHub code.
- [ ] I tested success and failure behavior.
- [ ] I understand security or scalability limitations.
- [ ] The Angular build completes without errors.

## Next lesson
Proceed to Lesson 12.
