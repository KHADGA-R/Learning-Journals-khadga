# Lesson 11 — Code Reference

> Reconstructed integration notes; not an exact or compiled source file.

Key source identifiers: `CourseSpecifications, CourseApplicationService`.

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'Specification' | Select-Object -First 20
mvn clean verify
```

Expected invariant: **published-only catalog**. Verify against real PostgreSQL and inspect generated SQL for performance-sensitive reads.
