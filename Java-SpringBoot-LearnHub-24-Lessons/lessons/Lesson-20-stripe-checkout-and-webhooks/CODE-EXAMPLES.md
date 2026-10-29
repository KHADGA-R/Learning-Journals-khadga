# Lesson 20 — Code Reference

> Reconstructed guidance, not a complete verified replacement source file.

Important paths: `V11__create_payments.sql; StripeWebhookProcessor; CheckoutService`.

```powershell
cd $HOME\angular-learning\learnhub-backend
Get-ChildItem src -Recurse -File | Select-String -Pattern 'Webhook' | Select-Object -First 20
mvn clean verify
```

**Integration contract:** verified paid events and refund-safe entitlement. Include one success case, one unauthorized case, and one duplicate/concurrent case in the test suite.
