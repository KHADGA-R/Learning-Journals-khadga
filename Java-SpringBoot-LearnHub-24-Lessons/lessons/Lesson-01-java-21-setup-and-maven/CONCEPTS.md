# Lesson 01 — Concepts and Design Notes

## Core concepts

### 1. Java 21

Study **Java 21** in the context of LearnHub. Identify the public contract, the application-service responsibility, persistence implications, and the failure path. Document which existing class owns the concern and how it is tested.

### 2. Maven lifecycle

Study **Maven lifecycle** in the context of LearnHub. Identify the public contract, the application-service responsibility, persistence implications, and the failure path. Document which existing class owns the concern and how it is tested.

### 3. JUnit 5

Study **JUnit 5** in the context of LearnHub. Identify the public contract, the application-service responsibility, persistence implications, and the failure path. Document which existing class owns the concern and how it is tested.

### 4. project structure

Study **project structure** in the context of LearnHub. Identify the public contract, the application-service responsibility, persistence implications, and the failure path. Document which existing class owns the concern and how it is tested.


## Architecture questions

- Which layer owns validation and authorization?
- What invariant must hold after a failed or concurrent request?
- Which database constraint, if any, is the final safeguard?
- Which HTTP status should a client observe, and why?
- Which information must never be logged or returned?

## Course continuity

Earlier lessons are teaching snapshots. Prefer the final contracts from Lessons 19–24 when integrating a running application. Check [integration notes](../../../INTEGRATION-NOTES.md).
