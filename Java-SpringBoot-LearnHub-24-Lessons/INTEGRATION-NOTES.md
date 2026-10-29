# Integration Notes and Known Course Drift

These 24 chapters are reconstructed study material, **not original transcripts or a verified compilable monorepo**. The actual Java and Angular implementation repositories remain the source of truth.

## Known mismatches to reconcile

1. Early `CourseRepository.deleteById` returned boolean; later versions changed to void. Update adapters and tests consistently.
2. `CourseWriteRequest` and `CourseCommandRequest` differ in metadata fields and constructors. Choose one canonical API.
3. Publish/archive were illustrated with both POST and PATCH. Align Spring mappings, Angular service and OpenAPI.
4. Search used both `free` and `freeOnly` query names. Align the wire contract.
5. Instructor draft detail must not use the public published-only endpoint.
6. Lesson 20 payment status lookup needs a trusted server-scoped payment attempt or session mapping; a redirect is not proof of purchase.
7. Refund revocation must preserve unrelated entitlements (promotions, administrative grants, separate valid purchases).
8. Full refunds, duplicate paid events, and out-of-order webhook events require a deterministic transactional state machine.
9. Curriculum changes and publishing readiness must share a concurrency/version protocol.
10. Existing students' access to archived courses must follow one documented rule.
11. Production session cookies require HTTPS; the local Compose example uses a distinct HTTP profile.
12. Backend Docker health must perform an actual HTTP readiness check, not merely launch a JVM.
13. Flyway migration filenames/versions must match applied schema history; never edit previously applied migrations.
14. Some tests and examples depend on fixture IDs or methods that may not exist in the final code; adapt them before running.

## Verification commands

```powershell
cd $HOME\angular-learning\learnhub-backend
mvn clean verify
cd $HOME\angular-learning\learnhub
npm test -- --watch=false
npm run build
cd $HOME\angular-learning\learnhub-deploy
docker compose config --quiet
```

Do not claim tests passed unless run in the user's environment.
