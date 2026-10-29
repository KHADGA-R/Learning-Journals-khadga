# Lesson 24 — Full stack capstone

## Deep dive

Review `schema-audit.sql; release verification scripts; README`. The key release concern is **student/instructor/payment journeys**. Define measurable acceptance criteria, identify trust boundaries, document rollback/retry behavior, and separate local simulation from production evidence.

## Operational review

- Can the system recover from a database or provider outage?
- Are secrets absent from logs and Git?
- Are health probes measuring real readiness?
- Is the tested artifact the deployed artifact?
- Are known limitations documented?
