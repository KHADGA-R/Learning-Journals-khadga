# Lesson 07: Code and Configuration Reference

> **Reconstructed reference, not a complete verified source file.**

## Key files and patterns

`V1__create_courses.sql and V2__seed_courses.sql`

### Windows PowerShell inspection

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'Flyway' | Select-Object -First 20
mvn clean verify
```

### Verification invariant

ddl-auto: validate. Validate it with a focused test and a PostgreSQL integration test when persistence is involved.

### Merge guidance

Do not replace complete production classes with fragments. Verify package names, DTO constructors, enum types, and request mappings against the final application.
