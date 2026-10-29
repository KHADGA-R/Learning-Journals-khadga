# Lesson 19 — Code Reference

> Reconstructed guidance, not a complete verified replacement source file.

Important paths: `V10__course_version_and_publication_metadata.sql; CoursePublishingService`.

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'Publishing' | Select-Object -First 20
mvn clean verify
```

**Integration contract:** owner authorization and optimistic version. Include one success case, one unauthorized case, and one duplicate/concurrent case in the test suite.
