# Windows PowerShell Setup

```powershell
$workspace = Join-Path $HOME 'angular-learning'
$backend = Join-Path $workspace 'learnhub-backend'
$frontend = Join-Path $workspace 'learnhub'
$deploy = Join-Path $workspace 'learnhub-deploy'

java -version
mvn -version
node --version
npm --version
docker compose version

Set-Location $backend
mvn clean verify

Set-Location $frontend
npm ci
npm run build

Set-Location $deploy
docker compose config --quiet
```

Use Java 21 and the Spring Boot version configured in your real pom.xml. PostgreSQL credentials and Stripe test secrets must be supplied through secure local environment configuration; never commit them.
