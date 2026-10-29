# Lesson 10 — Code Reference

> Reconstructed integration notes; not an exact or compiled source file.

Key source identifiers: `AppUserEntity, EnrollmentEntity, @EntityGraph`.

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'EntityGraph' | Select-Object -First 20
mvn clean verify
```

Expected invariant: **N+1 avoidance**. Verify against real PostgreSQL and inspect generated SQL for performance-sensitive reads.
