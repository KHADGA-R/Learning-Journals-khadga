# Lesson 02 — Java Fundamentals and Immutable Models

> Reconstructed from the LearnHub course history. Not a verbatim transcript. Adapt code to the actual project and verify before committing.

**Prerequisite:** Lesson 01. **Stack:** Java 21, Spring Boot 3.5.x, PostgreSQL, Angular as applicable.

## Objectives

Understand and implement records, enums, BigDecimal, input validation, immutable domain modeling. Explain the domain invariants, test failure cases, and record your results.

## Lesson context

This lesson contributes to LearnHub's progressive backend architecture. Work in `$HOME\angular-learning\learnhub-backend` unless the referenced path belongs to Angular or the deployment repository. Newer lessons supersede older code signatures; merge rather than overwrite.

## Important files

- `model/Course.java`
- `model/CourseLevel.java`
- `model/CourseStatus.java`

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

- **records** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **enums** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **BigDecimal** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **input validation** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **immutable domain modeling** — define the concept, identify the relevant code path, explain trade-offs and failure modes.

## Practical assignment

Implement Course record with validated price, free flag, and rating; reject inconsistent inputs.

## Review questions

1. Why BigDecimal for money?
2. What does a record guarantee?
3. Why validate in a canonical constructor?

## Verification

```powershell
mvn test
git diff --check
git status --short
```

**Completion evidence:** Capture command results, test reports, and a brief explanation of what changed. Do not mark a test as passed unless you actually ran it.

**Next:** [Lesson 03](../Lesson-03-oop-solid-and-repository-abstractions/README.md).
