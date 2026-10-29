# Lesson 22 — Code and Commands

> Reconstructed reference. Actual files, profiles and signatures must be verified in your project.

Key paths: `OpenAPI; Actuator; Micrometer; request ID MDC`.

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn clean verify
cd $HOME\angular-learning\learnhub
npm run build
cd $HOME\angular-learning\learnhub-deploy
docker compose config --quiet
```

**Acceptance condition:** safe contract and readiness. Verify actual exit codes and test reports. Do not claim deployment success merely because Docker containers started.
