# Lesson 03 — OOP SOLID and Repository Abstractions

> Reconstructed from the LearnHub course history. Not a verbatim transcript. Adapt code to the actual project and verify before committing.

**Prerequisite:** Lesson 02. **Stack:** Java 21, Spring Boot 3.5.x, PostgreSQL, Angular as applicable.

## Objectives

Understand and implement interfaces, dependency inversion, in-memory repositories, services. Explain the domain invariants, test failure cases, and record your results.

## Lesson context

This lesson contributes to LearnHub's progressive backend architecture. Work in `$HOME\angular-learning\learnhub-backend` unless the referenced path belongs to Angular or the deployment repository. Newer lessons supersede older code signatures; merge rather than overwrite.

## Important files

- `repository/CourseRepository.java`
- `repository/InMemoryCourseRepository.java`
- `data/SampleCourses.java`

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

- **interfaces** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **dependency inversion** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **in-memory repositories** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **services** — define the concept, identify the relevant code path, explain trade-offs and failure modes.

## Practical assignment

Implement save/find/delete with deterministic IDs and tests; compare interface and implementation.

## Review questions

1. Why inject an interface?
2. Which layer owns business rules?
3. What can an in-memory repository not prove?

## Verification

```powershell
mvn test
git diff --check
git status --short
```

**Completion evidence:** Capture command results, test reports, and a brief explanation of what changed. Do not mark a test as passed unless you actually ran it.

**Next:** [Lesson 04](../Lesson-04-collections-streams-and-exceptions/README.md).
