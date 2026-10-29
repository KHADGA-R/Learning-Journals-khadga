# Lesson 17 — Code Reference

> Reconstructed guidance, not a complete verified replacement source file.

Important paths: `V8__create_wishlists.sql; EnrollmentService`.

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'Enrollment' | Select-Object -First 20
mvn clean verify
```

**Integration contract:** idempotent student/course uniqueness. Include one success case, one unauthorized case, and one duplicate/concurrent case in the test suite.
