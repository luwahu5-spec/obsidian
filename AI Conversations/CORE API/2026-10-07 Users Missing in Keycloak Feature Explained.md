---
date: 2026-10-07
updated: 2026-10-07
source: Claude Code
project: both
tags: [domain-knowledge, users, keycloak, user-management, local-testing]
---

# Users missing in Keycloak — what each user page does

> **Takeaway (changed 2026-10-07):** For a CORE user whose Keycloak account no longer exists,
> **details and edit return 404**, and **only the delete page opens**, so an admin can remove the
> broken user.
> - It works through an admin-only opt-in, `GET /Users/{id}?allowMissingKeycloak=true`, which only the delete page sends.
> - Saving an edit would fail anyway (502, `0x800501f6`), because `PutUserAsync` must fetch the Keycloak user first.
> - Locally this hits most users, because the database is a production copy while the API talks to the `corelocal` realm.
>
> History:
> - Before `0b71391`, everything was 404, including delete.
> - `0b71391` (2026-09-29) tolerated the 404 for every page, which confused people: pages opened, then failed on save.
> - 2026-10-07: tolerance now applies to delete only.

## The real-world situation being modelled

Every CORE user is two records linked by `KcUserMapping` (`KcId` ↔ `UserId`):
- the CORE record (`AspNetUsers`), which holds sites, claims and app data;
- the Keycloak account, which handles sign-in.

The Keycloak account can disappear while the CORE record stays. That happens when someone deletes it manually in the Keycloak console, or (locally) when the realm never had it.

## Stale mapping repair (added 2026-10-07)

"Missing in Keycloak" can be false: the account may exist under a **different id** than `KcUserMapping.KcId`.

`GetUserAsync` now resolves the account in three steps:
1. `GetKcUser(mappedKcId)`. If found, done.
2. On 404, `KcClient.FindKcUserByEmail(user.Email)`: `users?exact=true&email=…`, first result or null. Keycloak keeps emails unique per realm, so there's at most one.
3. If found, `KcUserMappingRepository.RelinkAsync(old, new)`:
   - `KcId` is the primary key, so the row is removed and re-added in one `SaveChanges`.
   - **Guard:** this is refused when the new id is already mapped to another CORE user, because prod has CORE accounts that share an email.
   - A warning log records the old and new ids.
4. Only if steps 1–3 find nothing is the user truly missing (the 404 rules below apply).

- Every user page loads through `GetUserAsync` first, so the repair happens before any save, role change or delete. That's why it lives only there.
- Locally this rewrites prod-copy mappings to `corelocal` ids, for users whose email exists in `corelocal`.

## Several mappings for one user (added 2026-10-08)

`KcUserMapping` is keyed by `KcId` only; nothing stops one `UserId` having several rows.
- **Defect (before):** every lookup by user id took `FirstOrDefault()`, an arbitrary row: `GetUserAsync`, `PutUserAsync`, `CheckUser` (role endpoints), `SendConfirmEmail`, and the duplicate check in user creation.
- **Defect (before):** `DeleteAsync` removed only that one row. That's how mappings outlive their user; the local prod copy has **47 mappings with no CORE user**.

Now `GetUserAsync` loads **all** of the user's mappings:
- **No mapping at all (2026-10-08):** go straight to email. Found → `ReconcileAsync` creates the mapping. Not found → treated as missing (404 / delete page).
  - `DeleteAsync` with **zero** mappings (option A, decided 2026-10-08) looks the account up by email:
    - Found and not mapped to anyone → delete that Keycloak account and the CORE user.
    - Found but mapped to another CORE user → keep that login, delete the CORE user only.
    - Not found → delete the CORE user only.
    - Keycloak unreachable → 502, CORE kept for retry.
    - Replaces the old "502 BadMapping, refuse" rule.
- **One mapping:** `GetKcUser(id)`. If it's gone, fall back to email.
- **Several mappings:** no id is trusted; go straight to `FindKcUserByEmail`. Then:
  1. **No account with that email** → none is right. Mappings untouched; details/edit 404, the delete page opens (admin), and delete cleans everything up.
  2. **Account found, no mapping matches** → all removed, one created for the found id.
  3. **Account found, one mapping matches** → that one kept, the rest removed.
- All three go through `KcUserMappingRepository.ReconcileAsync(userId, kcId)`: removes the user's other rows, adds the correct one if missing, one `SaveChanges`. It is refused if `kcId` already belongs to another CORE user.
- A non-admin is allowed if their token subject is **any** of their mappings (not only the first). The Entra oid capture uses the same check.
- `DeleteAsync` deletes **every** mapped Keycloak id (404 = already gone) and removes **every** mapping row.
- Tests: `SSOUserTest` `GetUserAsync_SeveralMappings_*`, `DeleteAsync_UserWithSeveralMappings_*`; `KcUserMappingRepositoryTests` (5 cases including the guard).
- Other `FirstOrDefault` readers (`PutUserAsync`, `CheckUser`, `SendConfirmEmail`, creation duplicate check) are unchanged. Every page loads through `GetUserAsync` first, so by then only one mapping is left.

## Where it takes effect

| Page / action | API path | On Keycloak 404 |
|---|---|---|
| Details | `GET /Users/{id}` → `GetUserAsync` | Warning logged, **404**. Page redirects to NotFound |
| Edit (open) | same `GetUserAsync` | **404** |
| Delete (open) | `GET /Users/{id}?allowMissingKeycloak=true` (core-web `DeleteUser.razor.cs:37-38`, `UserRepository.GetUser(..., allowMissingKeycloak)`) | Admin only: CORE record returned, page loads. Non-admins still get 404 (`UsersController.cs` `GetUserAsync` 404 branch) |
| Edit (save) | `PUT /Users/{id}` → `PutUserAsync` → `KcClient.UpdateUser` → `GetKcUser` | `KcRequestException(404)`, treated as non-409, so **502** (`UsersController.cs:356-368`, `KcClient.cs:448`, `KcClient.cs:226-230`) |
| Delete | `DELETE /Users/{id}` → `DeleteAsync` → `KcClient.DeleteUser` | 404 = "already deleted", so CORE user + mapping are removed (`KcClient.cs:477-483`, `UsersController.cs:950-955`) |

- `GetUserAsync` calls Keycloak only to read `EmailVerified` (`UsersController.cs:264-267`). With a 404 the field just keeps its CORE value.
- **Before `0b71391`** the 404 branch returned `NotFound()`, so details, edit and delete pages all failed to open. That was the old behaviour people remember.

## How it shows in the UI

- core-web shows `0x80050000 + HTTP status` (`EditUser.razor.cs:176`). 502 → `0x800501f6`, with the "Error processing update on Authentication Server…" text (`EditUser.razor.cs:166-170`).
- A real duplicate email is **409** → "Failed to update User. (0x80050199)". **`0x800501f6` is therefore never a duplicate email.**

## Design subtleties & edge cases

- **Local testing trap.**
  - The local DB (`MinnovareProd`) is a production copy, so its mappings hold **production** Keycloak IDs.
  - The local API uses the `corelocal` realm (`appsettings.Development.json` `KeyCloak:Realm`), where those IDs don't exist.
  - Copied users open fine but can't be saved. Test edits with users created locally (or the SSO test users).
- **Edit form keeps failed input.** After an error the page doesn't reload the user (`EditUser.razor.cs:163-177`), so a retry resends whatever is still in the form.
- **Reading the API's logs locally.** Under the IIS Express profile there's no console, and `ClearProviders` (`Program.cs:91`) removes Debug output. Use the `Minnovare.Core.WebApi` profile to get the console window.
  - That profile serves HTTPS on 5000 and HTTP on 5001, while core-web's `CoreAPI:BaseUrl` expects `http://localhost:5000` (the IIS Express setup). Point core-web to `http://localhost:5001` while debugging, and revert before committing.
- Delete is idempotent on the Keycloak side, by design. A retry after a partial failure completes cleanly.

## Questions asked
- 2026-10-07 — "User save fails every time after one failed duplicate-email attempt, but details/edit/delete pages work — why?" → The Keycloak account doesn't exist (404), and it isn't related to the earlier attempt. GET and DELETE tolerate 404 since `0b71391`; PUT can't. See *Where it takes effect* and *Local testing trap*.
- 2026-10-07 — "Tolerating it confuses people. Don't tolerate on details/edit, only on delete" → Done via the admin-only `allowMissingKeycloak` query flag, sent only by the delete page. Tests in `SSOUserTest.cs` (`GetUserAsync_KeycloakAccountMissing_*`).
- 2026-10-07 — "Delete page still says not found" → core-web was still running the old build (process started 09:56, before the edit). Restart fixes it.
- 2026-10-07 — "User exists in Keycloak but under a different id than the mapping. Fall back to email and fix the mapping" → Done. See *Stale mapping repair*. Tests `GetUserAsync_MappedKeycloakIdStale_*` and `GetUserAsync_AccountFoundByEmailAlreadyMappedToAnotherUser_*`.
- 2026-10-08 — "Multiple mappings for one userId: resolve by email (none found / none match / one matches), and delete all mappings with the user" → Done. See *Several mappings for one user*. Case 1 deliberately does not auto-delete on a GET; the admin deletes from the delete page.
- 2026-10-08 — "Is the first `FindAsync(m => m.KcId == id).FirstOrDefault()` redundant with the all-mappings query?" → No.
  - It searches by `KcId`, the primary key, so 0 or 1 row; `FirstOrDefault` can't pick a wrong one.
  - Its job is converting a Keycloak id (token `sub`, sent by profile/edit, ConfirmEmail, ProductSelector, ProductivityReport, UserTimeService) into the CORE user id.
  - The second query (by `UserId`, not unique) is the one that can return several rows, and it now loads them all.
  - Removing the first would 404 every self-lookup by `sub`.

## Related
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]] — dual-store user management
- [[2026-09-23 SSO User Creation - Deferred Keycloak Orphan Rollback]] — the opposite orphan (Keycloak user without CORE)
- [[2026-10-06 Sign-in Lookup Rate Limiting and Forwarded Headers Feature Explained]]
