# Lesson 12 — Code Reference

> Reconstructed integration notes; not an exact or compiled source file.

Key source identifiers: `pg_trgm, escaped LIKE, keyset pagination`.

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'pg_trgm' | Select-Object -First 20
mvn clean verify
```

Expected invariant: **stable pagination**. Verify against real PostgreSQL and inspect generated SQL for performance-sensitive reads.
