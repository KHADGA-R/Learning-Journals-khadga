# Lesson 08: Exercises

1. Locate the implementation of **Validation and RFC 9457**.
2. Reproduce a successful operation demonstrating 400 / 404 / 409.
3. Add an invalid or conflicting operation and assert both HTTP response and database state.
4. Write a regression test and run `mvn clean verify`.
5. Record the exact files changed and a Git checkpoint.

## Challenge

Explain one realistic failure mode (concurrent requests, bad input, missing migration, or dependency outage) and how the design handles it.
