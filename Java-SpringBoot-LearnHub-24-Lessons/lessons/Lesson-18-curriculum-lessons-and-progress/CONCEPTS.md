# Lesson 18 — Curriculum and progress

## Domain model

Study `V9__create_curriculum_and_progress.sql; LessonProgressService`. The critical invariant is **enrolled-only lesson content and idempotent completion**. Explain state transitions, ownership, transaction boundaries, PostgreSQL uniqueness, and concurrency failure modes.

## Design review

A successful HTTP response is insufficient evidence: assert durable database state. Payment provider redirects are not proof of payment; lesson access must be enforced server-side. For state changes, handle retries, rollback and out-of-order events.
