# Lesson 21 — Code and Commands

> Reconstructed reference. Actual files, profiles and signatures must be verified in your project.

Key paths: `MoneyTest; MockMvc; PostgresIntegrationTest; Testcontainers`.

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn clean verify
cd $HOME\angular-learning\learnhub
npm run build
cd $HOME\angular-learning\learnhub-deploy
docker compose config --quiet
```

**Acceptance condition:** observable response and durable state. Verify actual exit codes and test reports. Do not claim deployment success merely because Docker containers started.
