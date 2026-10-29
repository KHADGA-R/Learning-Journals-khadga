# Lesson 01 — Java 21 Setup and Maven

> Reconstructed from the LearnHub course history. Not a verbatim transcript. Adapt code to the actual project and verify before committing.

**Prerequisite:** Lesson 00 (Java setup). **Stack:** Java 21, Spring Boot 3.5.x, PostgreSQL, Angular as applicable.

## Objectives

Understand and implement Java 21, Maven lifecycle, JUnit 5, project structure. Explain the domain invariants, test failure cases, and record your results.

## Lesson context

This lesson contributes to LearnHub's progressive backend architecture. Work in `$HOME\angular-learning\learnhub-backend` unless the referenced path belongs to Angular or the deployment repository. Newer lessons supersede older code signatures; merge rather than overwrite.

## Important files

- `src/main/java/com/learnhub/LearnHubApplication.java`
- `pom.xml`
- `src/test/java/...`

## Step-by-step workflow

1. Review [concepts](./CONCEPTS.md) and the existing application before editing.
2. Identify existing files and interfaces listed above; note differences from prior lessons.
3. Apply the [code and configuration examples](./CODE-EXAMPLES.md) to the correct layer.
4. Run the commands below and inspect failures instead of bypassing tests.
5. Complete [exercises](./EXERCISES.md), [quiz](./QUIZ.md), and [checklist](./CHECKLIST.md).

### Windows PowerShell

```powershell
cd $HOME\angular-learning\learnhub-backend
java -version
mvn -version
mvn clean test
```

## Knowledge focus

- **Java 21** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **Maven lifecycle** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **JUnit 5** — define the concept, identify the relevant code path, explain trade-offs and failure modes.
- **project structure** — define the concept, identify the relevant code path, explain trade-offs and failure modes.

## Practical assignment

Explain the distinction between JDK and JRE; inspect Maven dependency tree; create a failing and then passing JUnit test.

## Review questions

1. Why pin Java 21?
2. What does `mvn clean verify` do?
3. Why separate test and production sources?

## Verification

```powershell
mvn clean test
git diff --check
git status --short
```

**Completion evidence:** Capture command results, test reports, and a brief explanation of what changed. Do not mark a test as passed unless you actually ran it.

**Next:** [Lesson 02](../Lesson-02-java-fundamentals-and-immutable-models/README.md).
