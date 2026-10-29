# Lesson 12 — Search Filtering and Performance

> **Reconstructed lesson notes**, not an exact transcript. Code and tests are not verified against your local application.

## Objectives and concepts

- **V5 pg_trgm indexes**: explain the concept, implement it in LearnHub, identify failure modes and test it.
- **escaped LIKE**: explain the concept, implement it in LearnHub, identify failure modes and test it.
- **keyset pagination**: explain the concept, implement it in LearnHub, identify failure modes and test it.

## Context and implementation

This lesson extends the Java 21 + Spring Boot LearnHub marketplace. Review the earlier layers before changing controllers, DTOs, entities or migrations. Preserve server-side authorization and database constraints. Consult the course's original conversation for detailed code and examples where available.

## Windows PowerShell

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn test
git diff --check
git status --short
```

## Exercises

1. Implement a small end-to-end slice demonstrating v5 pg_trgm indexes.
2. Add a negative or unauthorized request and verify database state does not change.
3. Explain an API or schema compatibility risk from this lesson and its mitigation.
4. Run relevant tests and record the actual result; do not claim unexecuted tests passed.

## Quiz

1. Which domain invariant is introduced or strengthened here?
2. Which application layer should enforce it and why?
3. What database or concurrency failure could violate it?
4. Which test would catch a regression?

## Knowledge checklist

- [ ] I can explain each core concept without looking at notes.
- [ ] I located the corresponding implementation files.
- [ ] I understand the public API and persistence effects.
- [ ] I tested the happy path and at least one failure path.
- [ ] I checked authentication, ownership, and secret exposure where applicable.
- [ ] I ran the build and reviewed errors.
- [ ] I recorded exercises, quiz answers, and Git checkpoint.

## Important

Course lessons were incremental. Later implementations may supersede earlier signatures. Reconcile rather than blindly copying old files. This page is a structured study reconstruction; expand it with the exact original code when available.
