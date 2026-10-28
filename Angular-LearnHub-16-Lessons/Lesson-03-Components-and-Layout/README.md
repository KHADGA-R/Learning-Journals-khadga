# Lesson 3: Reusable Components, Header, Footer, and Layout

> Reconstructed and updated LearnHub curriculum. Windows PowerShell commands assume `$HOME\angular-learning\learnhub`. This lesson is a teaching chapter; later lessons evolve the same code, so do not apply every historical snippet simultaneously.

## Learning outcome
Compose a reusable app shell with routed page content.

## Prerequisites and workflow
Start in PowerShell with `cd $HOME\angular-learning\learnhub` for Lessons 2–16. Run `ng.cmd serve --open` during development and `ng.cmd build` to check compilation. Use the exact paths below, and merge imports into existing files instead of duplicating declarations.

## Step 1: Generate components

```powershell
ng.cmd generate component components/header
ng.cmd generate component components/footer
ng.cmd generate component pages/home
```

**Explanation:** Keep each component focused on one responsibility.

## Step 2: Create the app shell

```typescript
// src/app/app.ts (illustrative complete root)
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { Header } from './components/header/header';
import { Footer } from './components/footer/footer';
@Component({ selector: 'app-root', standalone: true, imports: [RouterOutlet, Header, Footer], templateUrl: './app.html' })
export class App {}
```
```html
<!-- src/app/app.html -->
<app-header />
<main><router-outlet /></main>
<app-footer />
```

**Explanation:** RouterOutlet displays the active page between the persistent header and footer.

## Step 3: Configure a home route

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';
import { Home } from './pages/home/home';
export const routes: Routes = [{ path: '', component: Home, title: 'LearnHub — Home' }];
```

**Explanation:** Make sure `app.config.ts` includes `provideRouter(routes)`.

## Step 4: Add accessible navigation

```html
<!-- src/app/components/header/header.html -->
<header><a routerLink="/">LearnHub</a><nav aria-label="Main navigation"><a routerLink="/">Home</a></nav></header>
```

**Explanation:** Import `RouterLink` in the Header component. Add responsive CSS for narrow screens.

## Hands-on exercise
Build a footer with a copyright notice and make the header responsive.

## Knowledge check
1. **What renders a routed page?**
   - Answer: RouterOutlet.
2. **Where should reusable navigation live?**
   - Answer: A shared Header component.
3. **What must a standalone component import to use routerLink?**
   - Answer: RouterLink.

## Verification
Run `ng.cmd build`. Check the browser console and resolve template/import errors before continuing. Keep this lesson's changes in source control.

## Next lesson
Continue to Lesson 4 in the parent curriculum index.
