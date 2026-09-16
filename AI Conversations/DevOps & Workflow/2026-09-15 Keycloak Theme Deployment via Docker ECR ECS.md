---
date: 2026-09-15
updated: 2026-09-16
source: Claude Code
project: infra (Keycloak)
tags: [devops, docker, ecr, ecs, keycloak, theme, deployment, runbook]
---

# Keycloak theme changes — local development and deployment to ECS

> **Takeaway:** Keycloak runs as a **container on ECS**, so its files cannot be edited on a server — every change is a new image. Login branding lives in a custom theme called **`Hexagon`** baked into our ECR image. **Do not modify an existing theme: add a new one alongside it.** Deploying the image then changes nothing visually, and go-live is a per-realm dropdown you can reverse in seconds. Whole flow proven end to end 2026-09-16.

## How the pieces relate

| | Role |
|---|---|
| **Docker** | Packages the app and its whole filesystem into an immutable **image**. A **container** is a running instance. Anything written inside a running container is lost when it is replaced |
| **ECR** | AWS registry where images are stored |
| **ECS** | Runs containers. A **task definition** says which image and tag to run; a **service** keeps N tasks alive and replaces failures |

There is no server to log into. RDP-ing to the instance behind the load balancer leads nowhere — the ALB target group registers targets **by IP rather than instance ID**, the signature of ECS rather than plain EC2.

## Our setup

| Property | Value |
|---|---|
| ECR repository | `220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak` |
| Keycloak version | **20.0.1** (JVM 11) |
| Entrypoint | `/opt/keycloak/bin/kc.sh start --optimized` — production mode, fixed |
| Baked-in env vars | `KC_DB_URL`, `KC_DB_USERNAME`, `KC_DB_PASSWORD`, `KC_HOSTNAME`, `KC_PROXY`, `KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD` |
| Themes in image | `Hexagon` (current), `Minnovare` (older), plus any added |

**Environment topology — one repository, separated by tag:**

| Tag | Used by |
|---|---|
| `production2` | Production ECS cluster |
| `latest` | All non-production ECS clusters |

Two clusters, one repository. **Pushing `:latest` therefore cannot affect production** — production only moves when `production2` is pushed deliberately.

⚠️ **The image carries real database credentials.** Run it with its default entrypoint and it starts in production mode against whatever `KC_DB_URL` points at — a real Keycloak database. Never run it unmodified for experimentation.

## The Hexagon theme

A **full fork** of the base login theme (`parent=base`) — every FreeMarker template copied, plus substantial message customisation.

| What to change | File (under `<theme>/login/`) |
|---|---|
| Colours, spacing | `resources/css/login.css` |
| Logo and images | `resources/img/` |
| Page structure | `template.ftl`, `login.ftl` |
| Any wording | `messages/messages_en.properties` |

Known selectors in `login.css`:
- `.m-primary` / `.m-primary:hover` — **Sign In button**
- `.m-primary-outline` — secondary button style used elsewhere
- `.forgot-pw` / `.forgot-pw:hover` — **Forgot your password?** link
- `.card-pf` — login card; its `border-top` is the coloured bar above the form

**Message key correction:** the SSO rejection message is **`federatedIdentityUnavailableMessage`**, not `federatedIdentityUnavailableUser`. The override already exists in the theme with Keycloak's default wording, so customising it edits one line. The related broker-failure message is `identityProviderUnexpectedErrorMessage`.

---

# The procedure

> **Already done once — for subsequent edits to an existing theme, skip to [Repeat changes](#repeat-changes-the-normal-loop).** Sections A–G describe the first-time setup.

## A. Get the theme onto your machine

`docker cp` works on stopped or never-started containers.

```
mkdir "C:\Users\CULI\OneDrive - Hexagon\Desktop\keycloak-theme"
docker create --name kc-build 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
docker cp kc-build:/opt/keycloak/themes/Hexagon "C:\Users\CULI\OneDrive - Hexagon\Desktop\keycloak-theme"
```

## B. Create a new theme, never edit the existing one

```
cd "C:\Users\CULI\OneDrive - Hexagon\Desktop\keycloak-theme"
cp -r Hexagon Hexagon-New
```

Edit **only** `Hexagon-New`. The folder name is what appears in the admin console dropdown, so name it something an admin should see. `theme.properties` keeps `parent=base`.

This is what makes the deployment safe: the image gains a folder, nothing existing changes, and no realm uses it until selected.

## C. Test locally

Use a **vanilla** Keycloak of the same version — our production image cannot run on a local database (see traps). Mount both themes to compare them side by side.

```
docker run --name kc-theme -p 8081:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin -v "C:/Users/CULI/OneDrive - Hexagon/Desktop/keycloak-theme/Hexagon:/opt/keycloak/themes/Hexagon" -v "C:/Users/CULI/OneDrive - Hexagon/Desktop/keycloak-theme/Hexagon-New:/opt/keycloak/themes/Hexagon-New" quay.io/keycloak/keycloak:20.0.1 start-dev --spi-theme-cache-themes=false --spi-theme-cache-templates=false --spi-theme-static-max-age=-1
```

The three `--spi-theme-*` flags disable theme caching — without them every edit needs a restart; with them it is edit, save, refresh.

Then `localhost:8081` → admin/admin → **Realm settings → Themes → Login theme** → pick either theme.

View the login page in an **incognito window** (the normal one is signed in as admin and skips it):

```
http://localhost:8081/realms/master/protocol/openid-connect/auth?client_id=account&redirect_uri=http://localhost:8081/realms/master/account&response_type=code&scope=openid&login_hint=test@example.com
```

Drop `&login_hint=` to see the page without a pre-supplied email. **The CORE app is not needed** for theme work and its `appsettings` should not be repointed.

## D. Build the image

```
# 1. Pull fresh — committing from a stale local image silently reverts anyone else's changes
docker pull 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest

# 2. Label the current image BEFORE the tag moves, so rollback stays easy
docker tag 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-hexagon-new
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-hexagon-new

# 3. Create (do not start) a container — no runtime state, entrypoint preserved
docker rm kc-build
docker create --name kc-build 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest

# 4. Copy the new theme in
docker cp "C:\Users\CULI\OneDrive - Hexagon\Desktop\keycloak-theme\Hexagon-New" kc-build:/opt/keycloak/themes/

# 5. Commit
docker commit kc-build 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
```

**Verify before pushing — both checks matter:**

```
docker run --rm --entrypoint ls 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest /opt/keycloak/themes
#   expect: Hexagon  Hexagon-New  Minnovare  README.md

docker inspect 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest --format "{{.Config.Entrypoint}} {{.Config.Cmd}}"
#   MUST read: [/opt/keycloak/bin/kc.sh start --optimized] []
```

```
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
```

## E. Deploy to non-production

**Pushing changes nothing on its own.** Running tasks keep the image they already pulled.

**ECS → Clusters →** non-production cluster **→ Services →** Keycloak service **→ Update → ☑ Force new deployment**. **Leave the task definition revision unchanged** — the URI still says `:latest`; the forced deployment is what makes ECS re-pull it.

CLI equivalent:
```
aws ecs update-service --cluster <non-prod-cluster> --service <keycloak-service> --force-new-deployment --region ap-southeast-2
```

Watch the **Events** tab. If new tasks fail health checks ECS leaves the old ones running — a safe failure, but one worth noticing rather than assuming success.

## F. Switch realms over

Once tasks have cycled, the new theme appears in **Realm settings → Themes → Login theme**. That dropdown reads the *running container's* filesystem, so its presence is proof the new image is live.

Switch **one realm first**, check the login page, then do the rest. **Rollback is changing the dropdown back** — seconds, no deployment.

## G. Promote to production

Same repository, so promotion is a re-tag — the exact bytes tested become production:

```
# label the outgoing production image first
docker pull 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:production2
docker tag 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:production2 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-hexagon-new-prod
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-hexagon-new-prod

# promote the tested image
docker tag 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:production2
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:production2
```

Then force a new deployment on the **production** cluster, and switch its realms individually.

---

## Repeat changes — the normal loop

Once a theme exists, every later edit follows this. It is section D condensed, with the two commands that matter most: **pull first, and recreate the build container.**

⚠️ **Never reuse an existing `kc-build`.** It stays pinned to the image it was created from, so committing from a stale one silently reverts anything else that has landed in `:latest` since. Confirmed 2026-09-16: `kc-build` was still based on `acd03bc4d7bc` (pre-change) even after `:latest` had moved to `d527f97bbed2`.

```
# 1. edit the theme
#    C:\Users\CULI\OneDrive - Hexagon\Desktop\keycloak-theme\Hexagon-New

# 2. test locally (see section C), then:

docker pull 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest

docker tag 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-<describe-this-change>
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-<describe-this-change>

docker rm kc-build
docker create --name kc-build 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest

docker cp "C:\Users\CULI\OneDrive - Hexagon\Desktop\keycloak-theme\Hexagon-New" kc-build:/opt/keycloak/themes/

docker commit kc-build 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest

docker run --rm --entrypoint ls 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest /opt/keycloak/themes
docker inspect 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest --format "{{.Config.Entrypoint}} {{.Config.Cmd}}"

docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest

# 3. force a new deployment on the non-production cluster (section E)
```

`pre-<describe-this-change>` is the only value to change each time — make it descriptive (`pre-blue-buttons`, `pre-sso-message`) so the rollback target is identifiable months later.

**Realms already using the theme pick the change up as soon as tasks cycle** — unlike the first deployment, this is not invisible. Anything already switched to `Hexagon-New` shows the edit immediately, so local verification matters more on repeat changes than it did the first time.

---

## Tags and rollback

A tag is a **movable label**, not the image. Images are identified by content hash and are immutable.

`docker commit … :latest` therefore does **not** alter the existing image — it creates a new one and moves the name. Verified live on 2026-09-16:

```
:latest        →  d527f97bbed2  (new)  →  Hexagon, Hexagon-New, Minnovare
<none>:<none>  →  acd03bc4d7bc  (old)  →  Hexagon, Minnovare          ← unchanged, now unnamed
```

That `<none>:<none>` is the **untagged** state. The image still exists and works, it simply has no name. Two consequences:

- Locally, `docker image prune` deletes untagged images — an unlabelled rollback target can vanish during routine cleanup.
- In ECR, untagged images persist until a **lifecycle policy** removes them (`aws ecr get-lifecycle-policy`). Many repositories expire them within days.

Hence step D.2: label the outgoing image *before* the tag moves. Rolling back is then:

```
docker tag 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-hexagon-new 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
```
plus a forced deployment. Without that label you must find the old image by `sha256:` digest — possible, fiddlier, and impossible if it has been pruned.

**In practice, image rollback is rarely the tool you want here.** Adding a theme changes nothing until a realm selects it, so the real rollback is the realm dropdown. The image tag protects against a bad *build* (e.g. committing from a stale pull), not a bad theme.

## Traps

1. **The ENTRYPOINT appends, it does not replace.** Passing `start-dev` to our image produces `kc.sh start --optimized start-dev …` — two commands, parse error, exit 2. Override with `--entrypoint /opt/keycloak/bin/kc.sh`, or use the vanilla image.
2. **Empty env var ≠ unset.** `-e KC_DB_URL=` makes Keycloak treat `""` as a real datasource and fail with *"configured datasource `<default>` not found"*. Supply valid values or omit the variable.
3. **The production image cannot run on local H2.** Built for PostgreSQL; forcing `KC_DB=dev-file` runs the schema migration against H2 and dies on a reserved word (`VALUE` in the credential migration). Use the vanilla image locally.
4. **Never `docker commit` a container created with an overridden entrypoint or command** — commit preserves them, so a container started with `start-dev` would bake development mode into the production image. Commit only from one whose `Cmd` is `null`; `docker create` guarantees this.
5. **A commit can silently not happen.** Symptom: `docker cp` worked but the image lacks the files. Diagnose with `docker diff kc-build` (shows `A /opt/keycloak/themes/Hexagon-New` if the copy landed) and `docker images` (a successful commit produces an image created *seconds* ago — if `:latest` still shows months old, the commit never ran).
6. **A pushed image shows nothing until tasks cycle.** The admin console's theme dropdown lists the *running* container's filesystem, so the new theme cannot appear before a forced deployment.
7. **Git Bash rewrites Unix paths on Windows**, turning `/opt/keycloak/...` into `C:/Users/.../opt/keycloak/...`. Use PowerShell for Docker commands, or prefix with `MSYS_NO_PATHCONV=1`.
8. **`docker cp` takes no spaces around the colon** — `container:/path`, not `container : /path`.
9. **OneDrive-synced folders** used as mounts must be set to *"Always keep on this device"*, or the container cannot read cloud-only placeholder files.

## Standing recommendation

The `docker commit` workflow works but produces an opaque image — no record of what changed, not reproducible, not reviewable. The theme files and a three-line Dockerfile belong in a repository:

```dockerfile
FROM quay.io/keycloak/keycloak:20.0.1
COPY themes/Hexagon-New /opt/keycloak/themes/Hexagon-New
```

That makes each change a pull request and puts the theme under version control. Worth raising with whoever owns the Keycloak image before settling into the manual workflow permanently.

## Related
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]] — the realm configuration this theme sits on
- [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] — ticket 4.5 is the theme work, 4.2 the reset-credentials flow
