# Lesson 14 — Code and Configuration Reference

> Reconstructed integration guidance; reconcile with actual repository source before copying.

Important files: `AuthController; LearnHubPrincipal; CookieCsrfTokenRepository`.

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn clean verify
cd $HOME\angular-learning\learnhub
npm run build
```

**Rule:** server principal and CSRF on unsafe methods. Validate using Spring Security Test, MockMvc, and Angular HttpTestingController. Do not put secrets or trusted user IDs in frontend code.
