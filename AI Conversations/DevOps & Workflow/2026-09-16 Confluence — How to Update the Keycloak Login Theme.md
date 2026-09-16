---
date: 2026-09-16
source: Claude Code
project: infra (Keycloak)
type: confluence-page
tags: [confluence, devops, docker, ecr, ecs, keycloak, theme, runbook]
---

# How to Update the Keycloak Login Theme

**Audience:** developers who need to change the appearance or wording of the Keycloak login page
**Prerequisites:** Docker Desktop installed and running, AWS CLI configured for `ap-southeast-2`, ECR push permission on `minnovare/keycloak`
**Last verified:** 16 September 2026 — full loop proven end to end against the non-production cluster

---

## Read this first

Keycloak runs as a **container on ECS**. There is no server to log into and no file on a disk somewhere that you can edit. Every visual change is a **new container image**.

That has three consequences that shape everything below:

1. **Changes are made locally, then baked into an image, then deployed.** Editing anything inside a running container is lost the moment ECS replaces it.
2. **Pushing an image changes nothing by itself.** Running tasks keep the image they already pulled. A **forced deployment** is what makes ECS pull the new one.
3. **A new theme is invisible until a realm selects it.** Deploying an image that merely *contains* a new theme is a no-op visually, which is what makes this workflow safe.

⚠️ **The production image carries real database credentials.** Started with its default entrypoint it connects to a real Keycloak database in production mode. Never run it unmodified for experimentation — use the vanilla image (Step 2).

---

## What exists today

| Property | Value |
|---|---|
| ECR repository | `220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak` |
| Keycloak version | 20.0.1 (JVM 11) |
| Entrypoint | `/opt/keycloak/bin/kc.sh start --optimized` — production mode, fixed |
| Themes in the image | `Hexagon-New` (current), `Hexagon` (previous), `Minnovare` (legacy) |

**One repository, two tags, two clusters:**

| Tag | Used by |
|---|---|
| `latest` | All **non-production** ECS clusters |
| `production2` | The **production** ECS cluster |

Pushing `:latest` therefore **cannot** reach production. Production moves only when `production2` is pushed deliberately (Step 6).

**Set your working folder once.** Every command below assumes the theme lives here — substitute your own path consistently:

```
C:\Users\<you>\Desktop\keycloak-theme
```

---

## Step 1 — Get the theme onto your machine

Only needed the first time, or after someone else has changed the theme.

```
docker pull 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
docker rm kc-build
docker create --name kc-build 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
docker cp kc-build:/opt/keycloak/themes/Hexagon-New "C:\Users\<you>\Desktop\keycloak-theme"
```

`docker create` makes a container without starting it, so nothing connects to any database. `docker cp` works fine on a container that has never run.

> `docker rm kc-build` will error harmlessly the very first time, when no such container exists.

### Where things live

All paths are under `Hexagon-New/login/`.

| What you want to change | File |
|---|---|
| Colours, spacing, layout | `resources/css/login.css` |
| Logo and images | `resources/img/` |
| Page structure, which fields render | `login.ftl`, `template.ftl` |
| Any wording on the page | `messages/messages_en.properties` |

Useful selectors already in `login.css`:

- `.m-primary` / `.m-primary:hover` — the **Sign In** button
- `.m-primary-outline` — secondary button style used elsewhere
- `.forgot-pw` / `.forgot-pw:hover` — the **Forgot your password?** link
- `.card-pf` — the login card; its `border-top` is the coloured bar above the form

Two message keys worth knowing: the SSO rejection message is **`federatedIdentityUnavailableMessage`**, and the related broker failure is `identityProviderUnexpectedErrorMessage`. Both already have overrides in the theme, so customising them is a one-line edit.

---

## Step 2 — Test locally

Use a **vanilla** Keycloak of the same version. Our production image cannot run against a local database (see Troubleshooting).

```
docker run --name kc-theme -p 8081:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin -v "C:/Users/<you>/Desktop/keycloak-theme/Hexagon-New:/opt/keycloak/themes/Hexagon-New" quay.io/keycloak/keycloak:20.0.1 start-dev --spi-theme-cache-themes=false --spi-theme-cache-templates=false --spi-theme-static-max-age=-1
```

The three `--spi-theme-*` flags disable theme caching. Without them every edit needs a container restart; with them it is edit → save → refresh.

Then:

1. Open `http://localhost:8081` and sign in as `admin` / `admin`
2. **Realm settings → Themes → Login theme → `Hexagon-New` → Save**
3. Open the login page in an **incognito window** — your normal window is signed in as admin and skips it:

```
http://localhost:8081/realms/master/protocol/openid-connect/auth?client_id=account&redirect_uri=http://localhost:8081/realms/master/account&response_type=code&scope=openid&login_hint=test@example.com
```

Drop `&login_hint=` to see the page as a user arriving without a pre-supplied email address — the two render differently, so check both.

**The CORE application is not needed for theme work.** Do not repoint its `appsettings` at a local Keycloak.

---

## Step 3 — Label the current image for rollback

Do this **before** the tag moves, while `:latest` still points at the version you are replacing.

```
docker pull 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest

docker tag 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-<describe-this-change>
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-<describe-this-change>
```

`pre-<describe-this-change>` is the **only value you edit each time**. Make it descriptive — `pre-blue-buttons`, `pre-sso-message` — so the rollback target is still identifiable months later.

**Why this step is not optional:** a tag is a movable label, not the image. Committing over `:latest` does not modify the old image, it creates a new one and moves the name — leaving the old image untagged (`<none>:<none>`). Untagged images are deleted by `docker image prune` locally and by ECR lifecycle policies remotely, often within days. Labelling first is what keeps the rollback target alive.

---

## Step 4 — Build and push the image

```
docker rm kc-build
docker create --name kc-build 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest

docker cp "C:\Users\<you>\Desktop\keycloak-theme\Hexagon-New" kc-build:/opt/keycloak/themes/

docker commit kc-build 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
```

⚠️ **Never reuse an existing `kc-build` container.** It stays pinned to the image it was created from, so committing from a stale one **silently reverts anything else that has landed in `:latest` since**. This is why `docker pull` and `docker rm` / `docker create` appear every time rather than once. Confirmed 16 Sep 2026: a `kc-build` left over from an earlier session was still based on the pre-change image after `:latest` had already moved on.

### Verify before pushing — both checks matter

```
docker run --rm --entrypoint ls 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest /opt/keycloak/themes
```
Expect: `Hexagon  Hexagon-New  Minnovare  README.md`

```
docker inspect 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest --format "{{.Config.Entrypoint}} {{.Config.Cmd}}"
```
**Must** read: `[/opt/keycloak/bin/kc.sh start --optimized] []`

If `Cmd` is not empty, a development-mode command has been baked into the image — do not push. Rebuild from `docker create`.

```
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
```

---

## Step 5 — Deploy to non-production

**Pushing on its own changes nothing.** Running tasks keep the image they already pulled.

**ECS → Clusters →** non-production cluster **→ Services →** the Keycloak service **→ Update → tick ☑ Force new deployment → Update.**

**Leave the task definition revision unchanged.** The revision already points at `:latest`; the forced deployment is what makes ECS re-pull that tag. Creating a new revision is unnecessary and adds noise.

> CLI equivalent (fill in your cluster and service names):
> ```
> aws ecs update-service --cluster <non-prod-cluster> --service <keycloak-service> --force-new-deployment --region ap-southeast-2
> ```

Watch the service's **Events** tab until the new tasks reach steady state. If they fail health checks, ECS keeps the old tasks running — a safe failure, but one to notice rather than assume success.

### Then check the result

**Creating the theme for the first time:** it now appears in **Realm settings → Themes → Login theme**. That dropdown reads the *running container's* filesystem, so its presence is itself proof the new image is live. Switch **one realm first**, check the login page, then do the rest.

**Editing an existing theme:** any realm already set to `Hexagon-New` shows your change as soon as tasks cycle. This is **not** invisible — which is exactly why Step 2 matters more on repeat changes than it did the first time.

---

## Step 6 — Promote to production

Only after non-production has been checked by a human.

Same repository, so promotion is a re-tag: the **exact bytes** that were tested become production, with no rebuild.

```
# label the outgoing production image first
docker pull 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:production2
docker tag 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:production2 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-<describe-this-change>-prod
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-<describe-this-change>-prod

# promote the tested image
docker tag 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:production2
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:production2
```

Then force a new deployment on the **production** cluster, and switch its realms over individually.

---

## Rollback

**For a bad theme, the rollback is the realm dropdown.** Set **Realm settings → Themes → Login theme** back to the previous theme. Seconds, no deployment, no AWS access needed.

**For a bad build** — a stale commit, a broken entrypoint, a corrupted copy — move the tag back:

```
docker pull 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-<describe-this-change>
docker tag 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:pre-<describe-this-change> 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
docker push 220546320512.dkr.ecr.ap-southeast-2.amazonaws.com/minnovare/keycloak:latest
```

Then force a new deployment. Without the label from Step 3 you would have to find the old image by `sha256:` digest — possible, fiddlier, and impossible if it has already been pruned.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `kc.sh start --optimized start-dev` → parse error, exit 2 | The image's ENTRYPOINT **appends**, it does not replace. Override with `--entrypoint /opt/keycloak/bin/kc.sh`, or just use the vanilla image locally |
| *"configured datasource `<default>` not found"* | An **empty** env var is not an unset one. `-e KC_DB_URL=` makes Keycloak treat `""` as a real datasource. Supply a valid value or omit the variable entirely |
| Liquibase error: *Syntax error in SQL statement … `REPLACE(VALUE, …)`* | The production image is built for PostgreSQL and cannot migrate its schema onto local H2 (`VALUE` is a reserved word in H2 2.x). Use `quay.io/keycloak/keycloak:20.0.1` locally |
| `docker cp` succeeded but the image lacks the files | The commit silently did not run. Check `docker diff kc-build` (should list `A /opt/keycloak/themes/Hexagon-New`) and `docker images` — a successful commit produces an image created *seconds* ago. If `:latest` still shows months old, re-run the commit |
| Theme not in the admin console dropdown after pushing | Tasks have not cycled. The dropdown lists the **running** container's filesystem. Force a new deployment |
| `/opt/keycloak/...` becomes `C:/Users/.../opt/keycloak/...` | Git Bash rewrites Unix-style paths on Windows. Use **PowerShell** for Docker commands, or prefix with `MSYS_NO_PATHCONV=1` |
| `docker cp` rejects the argument | No spaces around the colon: `container:/path`, not `container : /path` |
| Container cannot read mounted theme files | A OneDrive-synced folder used as a mount must be set to **"Always keep on this device"** — the container cannot read cloud-only placeholders |

---

## Known limitation of this workflow

`docker commit` works, but it produces an **opaque image**: no record of what changed, not reproducible, not reviewable.

The theme files and a two-line Dockerfile belong in a repository:

```dockerfile
FROM quay.io/keycloak/keycloak:20.0.1
COPY themes/Hexagon-New /opt/keycloak/themes/Hexagon-New
```

That would make every theme change a pull request and put the theme under version control. Worth raising with whoever owns the Keycloak image before this manual workflow becomes permanent.
