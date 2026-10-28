# Lesson 1: Angular Fundamentals and Project Setup

> Reconstructed and updated LearnHub curriculum. Windows PowerShell commands assume `$HOME\angular-learning\learnhub`. This lesson is a teaching chapter; later lessons evolve the same code, so do not apply every historical snippet simultaneously.

## Learning outcome
Create the standalone LearnHub application and understand Angular's application shell.

## Prerequisites and workflow
Start in PowerShell with `cd $HOME\angular-learning\learnhub` for Lessons 2–16. Run `ng.cmd serve --open` during development and `ng.cmd build` to check compilation. Use the exact paths below, and merge imports into existing files instead of duplicating declarations.

## Step 1: Check your tools

```powershell
node --version
npm.cmd --version
ng.cmd version
```

**Explanation:** Use a supported Node.js version for your installed Angular CLI. If Angular CLI is missing: `npm.cmd install -g @angular/cli`.

## Step 2: Create LearnHub

```powershell
cd $HOME
New-Item -ItemType Directory -Force angular-learning
cd angular-learning
ng.cmd new learnhub --routing --style=css
cd learnhub
ng.cmd serve --open
```

**Explanation:** Choose standalone components when prompted. Open `src/app/app.ts`, `app.html`, `app.config.ts` and `app.routes.ts` to identify the root component, template, providers and router.

## Step 3: Build the first page

```html
<!-- src/app/app.html -->
<main class="container">
  <h1>Welcome to LearnHub</h1>
  <p>Learn practical skills with expert-led courses.</p>
</main>
```

**Explanation:** Keep the initial page simple; subsequent lessons will replace the root template with header, router outlet and footer.

## Step 4: Style the shell

```css
/* src/styles.css */
* { box-sizing: border-box; }
body { margin: 0; font-family: Arial, sans-serif; color: #1f2937; }
.container { max-width: 1100px; margin: auto; padding: 2rem; }
```

**Explanation:** Global styles apply throughout the application. Component styles belong in their respective CSS files.

## Hands-on exercise
Change the headline and add a second paragraph. Inspect the live reload and run `ng.cmd build`.

## Knowledge check
1. **What does `ng.cmd serve` do?**
   - Answer: Runs a development server with rebuilds.
2. **Where are application-wide providers configured?**
   - Answer: In `src/app/app.config.ts`.
3. **What does standalone mean?**
   - Answer: A component declares its own imports without requiring an NgModule.

## Verification
Run `ng.cmd build`. Check the browser console and resolve template/import errors before continuing. Keep this lesson's changes in source control.

## Next lesson
Continue to Lesson 2 in the parent curriculum index.
