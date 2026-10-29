# Lesson 18 — Code Reference

> Reconstructed guidance, not a complete verified replacement source file.

Important paths: `V9__create_curriculum_and_progress.sql; LessonProgressService`.

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'LessonProgress' | Select-Object -First 20
mvn clean verify
```

**Integration contract:** enrolled-only lesson content and idempotent completion. Include one success case, one unauthorized case, and one duplicate/concurrent case in the test suite.
