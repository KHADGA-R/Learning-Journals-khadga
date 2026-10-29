# Lesson 16 — Angular integration

## Deep dive

Study `proxy.conf.json; course-api.service.ts; auth.service.ts` and how they enforce **typed contracts and session-aware requests**. Differentiate authentication, authorization, transport contracts, and application-service policy. Model direct HTTP attackers rather than trusting Angular route guards.

## Failure scenarios

Consider anonymous requests, CSRF omissions, cross-user IDs, stale sessions, and data leaks. Document expected status codes and persistence behavior.
