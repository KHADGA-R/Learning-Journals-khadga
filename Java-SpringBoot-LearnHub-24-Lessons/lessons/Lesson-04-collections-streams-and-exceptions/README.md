# Lesson 04 — Collections Streams and Exceptions

> Reconstructed from the LearnHub course history. Not a verbatim transcript. Adapt code to the actual project and verify before committing.

**Prerequisite:** Lesson 03. **Stack:** Java 21, Spring Boot 3.5.x, PostgreSQL, Angular as applicable.

## Objectives

Understand and implement streams, filtering, sorting, pagination, checked versus unchecked errors. Explain the domain invariants, test failure cases, and record your results.

## Lesson context

This lesson contributes to LearnHub's progressive backend architecture. Work in `$HOME\angular-learning\learnhub-backend` unless the referenced path belongs to Angular or the deployment repository. Newer lessons supersede older code signatures; merge rather than overwrite.

## Important files

- `catalog/CourseCatalog.java`
- `catalog/CourseFilter.java`
- `common/Result.java`

## Step-by-step workflow

1. Review [concepts](./CONCEPTS.md) and the existing application before editing.
2. Identify existing files and interfaces listed above; note differences from prior lessons.
3. Apply the [code and configuration examples](./CODE-EXAMPLES.md) to the correct layer.
4. Run the commands below and inspect failures instead of bypassing tests.
5. Complete [exercises](./EXERCISES.md), [quiz](./QUIZ.md), and [checklist](./CHECKLIST.md).

### Windows PowerShell

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn test
```

## Knowledge focus

- **streams** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **filtering** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **sorting** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **pagination** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **checked versus unchecked errors** — define the concept, identify the relevant code path, explain trade-offs and failure modes.

## Practical assignment

Build published-only filtered paginated catalog and test sort stability and boundary pages.

## Review questions

1. When should you use streams?
2. What is a stable sort?
3. When should an exception be domain-specific?

## Verification

```powershell
mvn test
git diff --check
git status --short
```

**Completion evidence:** Capture command results, test reports, and a brief explanation of what changed. Do not mark a test as passed unless you actually ran it.

**Next:** [Lesson 05](../Lesson-05-spring-boot-setup/README.md).
