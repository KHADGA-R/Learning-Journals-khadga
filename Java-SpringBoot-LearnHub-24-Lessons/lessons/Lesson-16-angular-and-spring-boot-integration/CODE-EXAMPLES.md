# Lesson 16 — Code and Configuration Reference

> Reconstructed integration guidance; reconcile with actual repository source before copying.

Important files: `proxy.conf.json; course-api.service.ts; auth.service.ts`.

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn clean verify
cd $HOME\angular-learning\learnhub
npm run build
```

**Rule:** typed contracts and session-aware requests. Validate using Spring Security Test, MockMvc, and Angular HttpTestingController. Do not put secrets or trusted user IDs in frontend code.
