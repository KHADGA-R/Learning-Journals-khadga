# Lesson 05: Code and Configuration Reference

> **Reconstructed reference, not a complete verified source file.**

## Key files and patterns

`spring-boot-starter-web, spring-boot-starter-actuator`

### Windows PowerShell inspection

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'SpringBootApplication' | Select-Object -First 20
mvn clean verify
```

### Verification invariant

GET /actuator/health. Validate it with a focused test and a PostgreSQL integration test when persistence is involved.

### Merge guidance

Do not replace complete production classes with fragments. Verify package names, DTO constructors, enum types, and request mappings against the final application.
