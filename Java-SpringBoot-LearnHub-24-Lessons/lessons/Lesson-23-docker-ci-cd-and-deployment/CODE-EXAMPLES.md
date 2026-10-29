# Lesson 23 — Code and Commands

> Reconstructed reference. Actual files, profiles and signatures must be verified in your project.

Key paths: `Dockerfile; nginx.conf; compose.yaml; GitHub Actions`.

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn clean verify
cd $HOME\angular-learning\learnhub
npm run build
cd $HOME\angular-learning\learnhub-deploy
docker compose config --quiet
```

**Acceptance condition:** immutable deploy and tested restore. Verify actual exit codes and test reports. Do not claim deployment success merely because Docker containers started.
