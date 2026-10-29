# Lesson 17 — Wishlist and enrollment

## Domain model

Study `V8__create_wishlists.sql; EnrollmentService`. The critical invariant is **idempotent student/course uniqueness**. Explain state transitions, ownership, transaction boundaries, PostgreSQL uniqueness, and concurrency failure modes.

## Design review

A successful HTTP response is insufficient evidence: assert durable database state. Payment provider redirects are not proof of payment; lesson access must be enforced server-side. For state changes, handle retries, rollback and out-of-order events.
