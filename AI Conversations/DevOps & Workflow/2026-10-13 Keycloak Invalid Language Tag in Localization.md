---
date: 2026-10-13
updated: 2026-10-13
source: Claude Code
project: infra (Keycloak)
tags: [keycloak, localization, realm-settings, debugging, root-cause, admin-console, kcadm]
---

# Keycloak "Invalid language tag" in Realm settings → Localization — root cause and fix

> **Takeaway:** The Localization tab failed in two realms with `RangeError: Invalid language tag:` (nothing after the colon). The empty tag was the clue: those realms had `"supportedLocales": [ "" ]` stored, one **empty string** in the list. The admin console loops over that list and asks the browser for a display name for each entry (`new Intl.DisplayNames("")`), which throws on the blank one. Fix: remove the blank entry with `kcadm.sh update realms/<realm> -s "supportedLocales=[]"`. Fixed in `coredev` 2026-10-13; `corestaging` command given, result not yet confirmed.

## What happened

- Working on the `Hexagon-New` login theme (language picker with five locales). Setting up Supported Locales in the realms.
- In **two realms** Realm settings → Localization would not work. The other realms and `master` worked.
- Browser console showed:

```
RangeError: Invalid language tag:
    at new DisplayNames (<anonymous>)
    at Ee (RealmSettingsSection.b5a2418b.js:1:64523)
    at Array.map (<anonymous>)
    at Ss (RealmSettingsSection.b5a2418b.js:1:67230)
```

## How the root cause was found

1. **First guess (wrong):** a malformed tag already stored, most likely an underscore (`pt_BR`) instead of a hyphen (`pt-BR`). That would have printed the tag in the message (`Invalid language tag: pt_BR`). It did not.
2. **`Ee` is not the bad tag.** It is the build tool's name for a function in the admin console bundle (minified); it carries no meaning. The tag is the text *after* the colon.
3. **The message ends with nothing after the colon** → the value passed in was an **empty string** (or only spaces).
4. **The stack shows `Array.map`** → the page loops over a list of locales and fails on one blank entry. The likeliest list is the realm's `supportedLocales`.
5. **Confirmed from the realm JSON** (browser DevTools → Network → the `GET /admin/realms/<realm>` response). Relevant fields for `coredev`:

```
"internationalizationEnabled": false,
"supportedLocales": [ "" ],
```

Other realms hold valid lists (or an empty list), so only the realms with `[""]` crash.

## The fix

Run from the local `kc-theme` container (it has `kcadm.sh`; the production image must **not** be used, it carries real DB credentials). Log in once with the master-realm admin, then:

```
docker exec -it kc-theme /opt/keycloak/bin/kcadm.sh config credentials --server <keycloak-url> --realm master --user <admin> --client admin-cli

docker exec kc-theme /opt/keycloak/bin/kcadm.sh get realms/<realm> --fields supportedLocales,internationalizationEnabled
docker exec kc-theme /opt/keycloak/bin/kcadm.sh update realms/<realm> -s "supportedLocales=[]"
docker exec kc-theme /opt/keycloak/bin/kcadm.sh get realms/<realm> --fields supportedLocales,internationalizationEnabled
```

- Run the update **only if the first `get` shows `[ "" ]`**. A valid list means the cause is different.
- Internationalization stays off, so the login page does not change.
- `config credentials` prints nothing on success. The "What's next: Try Docker Debug" lines Docker prints afterwards are a Docker hint, not an error. Check with `kcadm.sh get realms/<realm> --fields realm`.
- If PowerShell mangles the quotes, use single quotes: `-s 'supportedLocales=[]'`.

**Status:** `coredev` fixed and Localization tab loads. `corestaging`: same commands given; result to be confirmed.

## Related finding — why the extra locales did not show up in the console (Keycloak 20)

While setting up the five picker languages (`en-AU`, `en-US`, `fr-CA`, `pt-BR`, `es`):

- The console's **Supported Locales** box only offers locales that the realm's **login, account and email themes all declare** (`locales=` in each `theme.properties`). Only `en`, `es`, `pt-BR` were offered because the fork's account and email themes declared nothing.
- Fix applied locally: `locales=en,en-AU,en-US,fr-CA,pt-BR,es` added to `login`, `account` and `email` `theme.properties` in `Hexagon-New`, container restarted, login/account/email themes set to `Hexagon-New`.
- **`coredev` still uses the old `Hexagon` theme** (login, account, email, admin), which declares no locales. To use the five languages there: deploy the image containing `Hexagon-New`, switch the three themes, then set `internationalizationEnabled=true`, the five `supportedLocales` and `defaultLocale=en-AU`.
- Switching account and email themes also changes how the account page and Keycloak emails look. Decide that before doing it in staging or production.

## Prevention

- Before enabling locales in any realm, run `get realms/<realm> --fields supportedLocales,defaultLocale` and check there is no blank entry.
- Locale codes use **hyphens** (`pt-BR`), never underscores. Message *file names* use underscores (`messages_pt_BR.properties`); realm settings do not.
- Add the `supportedLocales` check to the Keycloak realm runbook (ticket 4.4).

## Questions asked

- 2026-10-13 — "Localization in realm settings won't work for 2 realms … Invalid language tag, what could happen?" → First answer ranked guesses (malformed stored tag); the stack trace then pointed at an empty value.
- 2026-10-13 — "Could it be `Ee`?" → No, minified function name; the empty tag after the colon is the clue.
- 2026-10-13 — pasted the `coredev` realm JSON → `"supportedLocales": [ "" ]` confirmed the cause.
- 2026-10-13 — "Where do I run kcadm?" / "password entered but it shows a Docker hint" → run via the local `kc-theme` container; the Docker banner is not an error.

## Related
- [[2026-09-15 Keycloak Theme Deployment via Docker ECR ECS]] — theme runbook (the `Hexagon-New` theme and its image deployment)
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]] — realm configuration this theme sits on
- [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] — ticket 4.4 realm runbook, 4.5 login theme
