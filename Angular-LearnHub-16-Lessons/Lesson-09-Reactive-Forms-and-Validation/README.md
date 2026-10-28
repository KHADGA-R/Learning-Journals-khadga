# Lesson 9: Reactive Forms, Login, and Registration

> Reconstructed or updated LearnHub curriculum chapter. This chapter is part of a progressive application: earlier code evolves in later lessons. Merge targeted changes into existing files rather than pasting historical fragments together.

## Learning objectives
Generate login and register pages. Import ReactiveFormsModule. Use NonNullableFormBuilder with email, password, name, confirmPassword; Validators.required, Validators.email, Validators.minLength(8). Mark all touched on invalid submission; show accessible inline errors. A valid form is not authentication.

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
import { NonNullableFormBuilder, Validators } from '@angular/forms';
private readonly fb=inject(NonNullableFormBuilder);
readonly form=this.fb.group({email:['',[Validators.required,Validators.email]],password:['',[Validators.required,Validators.minLength(8)]]});
submit(){if(this.form.invalid){this.form.markAllAsTouched();return;} console.info('Demo validation only');}
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
Add cross-field password matching validation to Register.

## Quiz with answers
1. **What does ReactiveFormsModule enable?** — formGroup and formControlName
2. **Does form validation create a user?** — No

## Completion checklist
- [ ] The lesson feature works with the existing LearnHub code.
- [ ] I tested success and failure behavior.
- [ ] I understand security or scalability limitations.
- [ ] The Angular build completes without errors.

## Next lesson
Proceed to Lesson 10.
