---
date: 2026-09-23
source: Claude Code
project: core-web-api
status: deferred
tags: [sso, keycloak, user-creation, rollback, deferred-optimisation, MC-1878]
---

# SSO user creation — deferred: Keycloak orphan rollback

> **Takeaway:** User creation spans two systems with no shared transaction, so rollback is compensation, and compensation is only as good as what the caller *knows*. When the Keycloak create call fails **after** Keycloak has already created the user, CORE is rolled back but the Keycloak user survives, and every retry then fails with *"User already exists in KeyCloak"* until someone deletes it by hand. A fix is written, tested and parked; this note is the reason to pick it up again.

## Takeaways

- **"The request failed" ≠ "nothing was created."** A timeout means *no response came back*, not *no work was done*. Treat a failed create as "outcome unknown" and find out before rolling back.
- **The pre-create lookup is what makes resolution safe.** `MigrateUser` already proves no Keycloak user with that username exists before creating one. So after a failed create, exactly one match can only be the user this request created, and it's safe to delete.
- **Never claim a clean rollback you can't prove.** If a Keycloak identity *may* exist but can't be found, CORE should still be rolled back, but the response must say *manual reconciliation required*, not "rolled back".
- **Only look when an orphan is possible.** Resolving on *every* failure would raise false alarms while Keycloak is simply down (the pre-create lookup fails, nothing was ever sent). Resolve only once the POST has actually been sent.

## When it happens

| Trigger | Keycloak state | Today's result |
|---|---|---|
| POST times out, but Keycloak processed it | user **exists** | CORE rolled back, Keycloak user orphaned. Response says "rolled back" |
| 201 Created with no `Location` header, and the fallback lookup fails | user **exists**, ID unknown | same: orphan, and "rolled back" is untrue |
| Any failure *before* the POST (login, pre-create lookup) | nothing created | correct: nothing to clean up |

Rare now. The earlier switch to reading the new ID from the `Location` header removed the common cause (create succeeded, then the follow-up search failed). The residual is Keycloak being slow enough for `HttpClient` to time out, which is plausible under load or during a Keycloak redeploy.

**Symptom to recognise:** a user who can never be created: *"User already exists in KeyCloak"* on every attempt, yet no such user exists in CORE. **Manual fix today:** delete the stray user in the Keycloak admin console, then retry.

## The parked design

1. **`KcClient.MigrateUser`**: wrap the create POST. On `KcPolicyException` (how `TryCall` surfaces a timeout), run an exact-username lookup:
   - 0 matches → nothing was created → rethrow the original failure unchanged
   - 1 match → throw `KcUserCreationUncertainException` carrying that user's ID
   - lookup fails / ambiguous → throw it with **no** ID
   The two "created but can't identify it" branches after a 201 also throw it with no ID.
2. **`KcUserCreationUncertainException`** (new, `: KcClientException`, so the controller's existing `IsKeyCloakFailure` filter still catches it). Carries `Guid? CreatedUserId`.
3. **`UsersController.CompensateAfterKeycloakFailureAsync`**: used by both creation catch blocks:
   - passes `CreatedUserId` to `CompensateFailedUserCreationAsync`, so the Keycloak identity is deleted too
   - reports rollback as **incomplete** when an identity may exist but has no ID

Tests written with it: two controller tests (rollback with ID → both identities deleted; without ID → "manual reconciliation"), and two `KcClient` tests (timeout + user created → uncertain with ID; timeout + nothing created → original failure). They simulate the timeout by letting a `null` entry in the test `SequenceHttpMessageHandler` throw `TaskCanceledException`.

## Before picking it up

- Re-check the `MigrateUser` flow hasn't changed shape. The design depends on the pre-create lookup still running first.
- The patch also changes the shared `SequenceHttpMessageHandler` test helper. Apply it before adding other tests that rely on it.

## Reference

- **Parked code:** git stash on `core-web-api`, message *"MC-1878: Keycloak orphan rollback (deferred) - see vault note 2026-09-23"*. Its base is the SSO creation fixes, so it holds only this change.
- **Durable copy:** `2026-09-23 SSO Keycloak Orphan Rollback.patch` (next to this note). Use it if the stash is ever lost; `git apply` it onto `MC-1878`.
- Related: [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]]
