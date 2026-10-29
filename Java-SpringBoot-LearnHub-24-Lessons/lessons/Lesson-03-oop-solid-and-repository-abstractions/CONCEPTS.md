# Lesson 03 — Concepts and Design Notes

## Core concepts

### 1. interfaces

Study **interfaces** in the context of LearnHub. Identify the public contract, the application-service responsibility, persistence implications, and the failure path. Document which existing class owns the concern and how it is tested.

### 2. dependency inversion

Study **dependency inversion** in the context of LearnHub. Identify the public contract, the application-service responsibility, persistence implications, and the failure path. Document which existing class owns the concern and how it is tested.

### 3. in-memory repositories

Study **in-memory repositories** in the context of LearnHub. Identify the public contract, the application-service responsibility, persistence implications, and the failure path. Document which existing class owns the concern and how it is tested.

### 4. services

Study **services** in the context of LearnHub. Identify the public contract, the application-service responsibility, persistence implications, and the failure path. Document which existing class owns the concern and how it is tested.


## Architecture questions

- Which layer owns validation and authorization?
- What invariant must hold after a failed or concurrent request?
- Which database constraint, if any, is the final safeguard?
- Which HTTP status should a client observe, and why?
- Which information must never be logged or returned?

## Course continuity

Earlier lessons are teaching snapshots. Prefer the final contracts from Lessons 19–24 when integrating a running application. Check [integration notes](../../../INTEGRATION-NOTES.md).
