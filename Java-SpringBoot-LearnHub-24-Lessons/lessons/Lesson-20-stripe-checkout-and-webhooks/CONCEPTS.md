# Lesson 20 — Payments and refunds

## Domain model

Study `V11__create_payments.sql; StripeWebhookProcessor; CheckoutService`. The critical invariant is **verified paid events and refund-safe entitlement**. Explain state transitions, ownership, transaction boundaries, PostgreSQL uniqueness, and concurrency failure modes.

## Design review

A successful HTTP response is insufficient evidence: assert durable database state. Payment provider redirects are not proof of payment; lesson access must be enforced server-side. For state changes, handle retries, rollback and out-of-order events.
