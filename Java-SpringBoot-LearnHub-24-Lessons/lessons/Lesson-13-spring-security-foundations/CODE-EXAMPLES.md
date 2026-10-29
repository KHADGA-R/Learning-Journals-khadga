# Lesson 13 — Code and Configuration Reference

> Reconstructed integration guidance; reconcile with actual repository source before copying.

Important files: `SecurityFilterChain; BCrypt; DatabaseUserDetailsService`.

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn clean verify
cd $HOME\angular-learning\learnhub
npm run build
```

**Rule:** default deny and least privilege. Validate using Spring Security Test, MockMvc, and Angular HttpTestingController. Do not put secrets or trusted user IDs in frontend code.
