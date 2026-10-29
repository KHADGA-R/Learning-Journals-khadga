# Lesson 14 — Sessions and CSRF

## Deep dive

Study `AuthController; LearnHubPrincipal; CookieCsrfTokenRepository` and how they enforce **server principal and CSRF on unsafe methods**. Differentiate authentication, authorization, transport contracts, and application-service policy. Model direct HTTP attackers rather than trusting Angular route guards.

## Failure scenarios

Consider anonymous requests, CSRF omissions, cross-user IDs, stale sessions, and data leaks. Document expected status codes and persistence behavior.
