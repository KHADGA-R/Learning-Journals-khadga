# Lesson 09 — Code Reference

> Reconstructed integration notes; not an exact or compiled source file.

Key source identifiers: `V3__create_users_and_enrollments.sql`.

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'enrollments' | Select-Object -First 20
mvn clean verify
```

Expected invariant: **unique normalized email and enrollment**. Verify against real PostgreSQL and inspect generated SQL for performance-sensitive reads.
