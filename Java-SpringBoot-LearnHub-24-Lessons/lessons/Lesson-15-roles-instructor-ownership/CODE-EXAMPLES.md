# Lesson 15 — Code and Configuration Reference

> Reconstructed integration guidance; reconcile with actual repository source before copying.

Important files: `CurrentActor; CourseOwnershipPolicy`.

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn clean verify
cd $HOME\angular-learning\learnhub
npm run build
```

**Rule:** object-level ownership. Validate using Spring Security Test, MockMvc, and Angular HttpTestingController. Do not put secrets or trusted user IDs in frontend code.
