---
date: 2026-09-15
updated: 2026-09-15
source: Claude Code
project: infra (Keycloak)
tags: [devops, docker, ecr, ecs, keycloak, theme, deployment, runbook]
---

# Keycloak theme changes — local development and deployment to ECS

> **Takeaway:** Our Keycloak runs as a **container on ECS**, so its files cannot be edited on a server — every change is a new image. The login page branding lives in a custom theme called **`Hexagon`** that is already baked into our ECR image (not a vanilla Keycloak). Local theme work is done against a **vanilla Keycloak 20.0.1 container with the theme folder mounted**, because our production image is built for PostgreSQL and fails to start on a local database. Deployment is `docker cp` → `docker commit` → `docker push` → **force new deployment in ECS** (pushing alone changes nothing).

## How the pieces relate

| | Role |
|---|---|
| **Docker** | Packages the app and its whole filesystem into an immutable **image**. A **container** is a running instance of one. Anything written inside a running container is lost when it is replaced |
| **ECR** | AWS's registry where images are stored |
| **ECS** | Runs containers. A **task definition** (versioned as **revisions**) says which image and tag to run; a **service** keeps N tasks alive and replaces failures |

Consequence: there is no server to log into and no file to edit in place. This is why RDP-ing to the instance behind the load balancer leads nowhere — the ALB target group registers targets **by IP address rather than instance ID**, which is the signature of ECS rather than plain EC2.

## Our Keycloak image — facts

| Property | Value |
|---|---|
| ECR image | `220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest` |
| Keycloak version | **20.0.1** (JVM 11) |
| Entrypoint | `/opt/keycloak/bin/kc.sh start --optimized` — production mode, fixed |
| Baked-in env vars | `KC_DB_URL`, `KC_DB_USERNAME`, `KC_DB_PASSWORD`, `KC_HOSTNAME`, `KC_PROXY`, `KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD` |
| Custom themes present | **`Hexagon`** (current, updated Sep 2025) and `Minnovare` (older, Dec 2022) |
| Task definitions | All revisions reference **`:latest`** |

⚠️ **The image carries real database credentials.** Running it locally with its default entrypoint starts it in production mode and it will connect to whatever `KC_DB_URL` points at — i.e. a real Keycloak database, not a local one. Never run this image unmodified for experimentation.

## The Hexagon theme

A **full fork** of the base login theme (`parent=base`) — every FreeMarker template copied, plus substantial message customisation. Files that matter:

| What to change | File (under `Hexagon/login/`) |
|---|---|
| Colours, spacing | `resources/css/login.css` |
| Logo and images | `resources/img/` |
| Page structure | `template.ftl`, `login.ftl` |
| Any wording | `messages/messages_en.properties` |

Known selectors in `login.css`:
- `.m-primary` / `.m-primary:hover` — the **Sign In button**
- `.m-primary-outline` — a secondary button style used elsewhere
- `.forgot-pw` / `.forgot-pw:hover` — the **Forgot your password?** link
- `.card-pf` — the login card; its `border-top` is the coloured bar above the form

**Message key correction:** the SSO rejection message is **`federatedIdentityUnavailableMessage`**, not `federatedIdentityUnavailableUser`. The override already exists in the Hexagon theme carrying Keycloak's default wording, so customising it is editing one line, not adding one. The related broker-failure message is `identityProviderUnexpectedErrorMessage`.

## Local theme development

Use a **vanilla** Keycloak of the same version with the theme folder mounted. Do not use the ECR image for this (see traps).

**1. Copy the theme out of a container made from the ECR image:**
```
docker cp <container>:/opt/keycloak/themes/Hexagon <local folder>
```

**2. Run vanilla Keycloak with that folder mounted:**
```
docker run --name kc-theme -p 8081:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin -v "<local folder>/Hexagon:/opt/keycloak/themes/Hexagon" quay.io/keycloak/keycloak:20.0.1 start-dev --spi-theme-cache-themes=false --spi-theme-cache-templates=false --spi-theme-static-max-age=-1
```
The three `--spi-theme-*` flags disable theme caching — without them every edit needs a container restart. With them it is edit, save, refresh.

**3. Point the realm at the theme:** `localhost:8081` → admin/admin → **Realm settings → Themes → Login theme → Hexagon**.

**4. View the login page** in an **incognito window** (the normal one is signed in as admin and skips it):
```
http://localhost:8081/realms/master/protocol/openid-connect/auth?client_id=account&redirect_uri=http://localhost:8081/realms/master/account&response_type=code&scope=openid&login_hint=test@example.com
```
Drop the `&login_hint=` parameter to see the page as it appears without a pre-supplied email.

The CORE app is **not** needed for theme work, and its `appsettings` should not be repointed.

## Deploying a theme change

```
# 1. Copy the finished theme into a container made from the PRODUCTION image
docker cp "<local folder>\Hexagon" <ecr-container>:/opt/keycloak/themes/

# 2. Commit to a VERSIONED tag first
docker commit <ecr-container> <ecr-repo>:theme-v1

# 3. Verify before pushing — theme present, entrypoint intact
docker run --rm --entrypoint grep <ecr-repo>:theme-v1 "<a value you changed>" /opt/keycloak/themes/Hexagon/login/resources/css/login.css
docker inspect <ecr-repo>:theme-v1 --format "{{.Config.Entrypoint}} {{.Config.Cmd}}"
#   MUST read: [/opt/keycloak/bin/kc.sh start --optimized] []

# 4. Push the version tag, then latest
docker push <ecr-repo>:theme-v1
docker tag <ecr-repo>:theme-v1 <ecr-repo>:latest
docker push <ecr-repo>:latest
```

**5. ECS will not notice on its own.** Pushing to ECR neither restarts nor updates anything — running tasks keep the image they already pulled. Deploy with **Service → Update → Force new deployment** (or `aws ecs update-service --force-new-deployment`). Because task definitions track `:latest`, **no new revision is needed** — a forced deployment is enough.

**6. Rollback.** With everything on `:latest` there is no previous revision to return to, which is why step 2 tags a version. To roll back: re-tag the previous version as `:latest`, push, and force another deployment. Without a version tag the old image is only addressable by SHA digest — and may already have been removed if the repository has a lifecycle policy pruning untagged images (`aws ecr get-lifecycle-policy`).

## Traps

1. **The ENTRYPOINT appends, it does not replace.** Passing `start-dev` to our image produces `kc.sh start --optimized start-dev …` — two commands, parse error, exit code 2. Override with `--entrypoint /opt/keycloak/bin/kc.sh` if you must, or use the vanilla image.
2. **Empty env var ≠ unset.** `-e KC_DB_URL=` makes Keycloak treat `""` as a real datasource URL and fail with *"configured datasource `<default>` not found"*. Supply valid values or omit the variable entirely.
3. **The production image cannot run on a local H2 database.** It is built for PostgreSQL; forcing `KC_DB=dev-file` runs the schema migration against H2 and dies on a reserved-word error (`VALUE` in the credential migration). This is why local theme work uses the vanilla image.
4. **Never `docker commit` a container that was created with an overridden entrypoint or command** — commit preserves them, so committing a container started with `start-dev` would bake development mode into the production image. Commit only from a container whose `Cmd` is `null`.
5. **Git Bash rewrites Unix paths on Windows**, turning `/opt/keycloak/...` into `C:/Users/.../opt/keycloak/...`. Use PowerShell for Docker commands, or prefix with `MSYS_NO_PATHCONV=1`.
6. **`docker cp` needs no spaces around the colon** — `container:/path`, not `container : /path`.
7. **OneDrive-synced folders** used as mounts must be set to *"Always keep on this device"*, or the container cannot read cloud-only placeholder files.
8. **`:latest` reaches production without a deployment.** ECS replaces failed tasks by pulling the tag again, so overwriting `:latest` can reach production on its own the next time a task restarts. Pushing to that tag is itself the risky act, not the deploy.

## Standing recommendation

The `docker commit` workflow works but produces an opaque image — no record of what changed, not reproducible, not reviewable. Since this theme will be changed repeatedly (branding, SSO messages, each new environment), the theme files and a three-line Dockerfile belong in a repository:

```dockerfile
FROM quay.io/keycloak/keycloak:20.0.1
COPY themes/Hexagon /opt/keycloak/themes/Hexagon
```

That turns each change into a pull request and puts the theme under version control. Worth raising with whoever owns the Keycloak image before settling into the manual workflow.

## Related
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]] — the Keycloak realm configuration this theme sits on
- [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] — ticket 4.5 is the theme work; 4.2 the reset-credentials flow
