# Lesson 19 — Publishing and analytics

## Domain model

Study `V10__course_version_and_publication_metadata.sql; CoursePublishingService`. The critical invariant is **owner authorization and optimistic version**. Explain state transitions, ownership, transaction boundaries, PostgreSQL uniqueness, and concurrency failure modes.

## Design review

A successful HTTP response is insufficient evidence: assert durable database state. Payment provider redirects are not proof of payment; lesson access must be enforced server-side. For state changes, handle retries, rollback and out-of-order events.
