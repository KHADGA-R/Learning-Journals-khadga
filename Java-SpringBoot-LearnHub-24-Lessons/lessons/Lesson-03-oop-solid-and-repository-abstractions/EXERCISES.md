# Lesson 03 — Exercises

## A. Guided implementation

Implement save/find/delete with deterministic IDs and tests; compare interface and implementation.

**Acceptance criteria:** State the input, expected output, error behavior, database effects (where applicable), and automated test. Capture a passing command result.

## B. Negative test

Create an invalid input or unauthorized request for this lesson's feature. Verify that it fails safely and does not partially persist changes.

## C. Integration review

Identify an API, DTO, entity, or configuration change introduced here that could break an earlier lesson. Explain how to migrate consumers without silently changing contracts.

## D. Git checkpoint

```powershell
cd $HOME\angular-learning\learnhub-backend
git diff --check
git status --short
git diff --stat
```

Commit in the appropriate **implementation** repository only after verifying the relevant tests.
