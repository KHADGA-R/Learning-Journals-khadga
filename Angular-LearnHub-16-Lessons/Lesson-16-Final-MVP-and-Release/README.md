# Lesson 16: Final MVP Audit, Production Readiness, and Next Steps

> Reconstructed or updated LearnHub curriculum chapter. This chapter is part of a progressive application: earlier code evolves in later lessons. Merge targeted changes into existing files rather than pasting historical fragments together.

## Learning objectives
Audit routes, IDs, API errors, enrollment, wishlist, guards and storage isolation. Verify student and instructor journeys, test API outage recovery, run tests and a production build. A public marketplace still needs backend auth, database persistence, verified payments, entitlements and monitoring. Publish the frontend only as a clearly labeled educational demo.

## Step 1 — Open the project

```powershell
cd $HOME\angular-learning\learnhub
code .
ng.cmd version
```

Review existing files in src/app, especially app.routes.ts, app.config.ts, and relevant services. Preserve existing imports and providers.

## Step 2 — Implement the lesson concept

The following is a focused, documented code pattern; add the necessary Angular imports and place members inside the appropriate component or service rather than using this fragment as a complete file.

```powershell
# Windows PowerShell
ng.cmd test --watch=false
if ($LASTEXITCODE -ne 0) { throw 'Tests failed' }
ng.cmd build --configuration production
if ($LASTEXITCODE -ne 0) { throw 'Build failed' }
Get-ChildItem .\dist\learnhub -Recurse -Filter index.html
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
Walk through guest login, free enrollment, 67% progress, instructor guard and API outage recovery.

## Quiz with answers
1. **Does build success prove security?** — No
2. **Who verifies paid entitlements?** — Trusted backend

## Completion checklist
- [ ] The lesson feature works with the existing LearnHub code.
- [ ] I tested success and failure behavior.
- [ ] I understand security or scalability limitations.
- [ ] The Angular build completes without errors.

## Next lesson
Review the full curriculum and release checklist.
