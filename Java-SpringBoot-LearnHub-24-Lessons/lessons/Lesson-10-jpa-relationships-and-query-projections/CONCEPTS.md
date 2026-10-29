# Lesson 10 — JPA relationships and projections

**Implementation:** AppUserEntity, EnrollmentEntity, @EntityGraph.

**Invariant:** N+1 avoidance.

## Deep dive

Explain the API contract, database representation, transaction boundaries, failure states, and indexing/locking choices. Demonstrate how Java services and PostgreSQL constraints complement each other. Avoid copying outdated signatures from earlier lessons.

## Questions to investigate

- Which class owns the invariant?
- What is the observable HTTP contract?
- What happens during two concurrent writes?
- Which integration test proves the database behavior?
