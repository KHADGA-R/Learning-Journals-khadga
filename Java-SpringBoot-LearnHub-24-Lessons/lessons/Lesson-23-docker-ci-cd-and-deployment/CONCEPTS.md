# Lesson 23 — Deployment and recovery

## Deep dive

Review `Dockerfile; nginx.conf; compose.yaml; GitHub Actions`. The key release concern is **immutable deploy and tested restore**. Define measurable acceptance criteria, identify trust boundaries, document rollback/retry behavior, and separate local simulation from production evidence.

## Operational review

- Can the system recover from a database or provider outage?
- Are secrets absent from logs and Git?
- Are health probes measuring real readiness?
- Is the tested artifact the deployed artifact?
- Are known limitations documented?
